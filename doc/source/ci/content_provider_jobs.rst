Content Provider Jobs
=====================

This section gives an overview and some details about the 'content provider'
zuul jobs. They are so called because they consist of a parent job that builds
containers which are then consumed by any number of child jobs. Thus the parent
jobs are the 'content provider' for the child jobs.

Why Do We Need Content Providers?
---------------------------------

The content provider jobs were added by the Tripleo CI squad during the
Victoria development cycle. Prior to this `check and gate tripleo-ci jobs`_
running on review.opendev.org code submissions were pulling the promoted
'current-tripleo' containers from docker.io.

Having all jobs pull directly from a remote registry obviously puts a strain on
resources; consider multiple jobs per code submission with tens of
container pulls for each. We have over time been affected by a number of issues
related to the container pulls (such as timeouts) that would cause jobs to fail
and block the gates. Furthermore, `docker has recently announced`_ that requests
will be rate limited to one or two hundred pull requests per six hours (without and
with authentication respectively) on the free plan effective 01 November 2020.

In anticipation of this the TripleO CI squad has moved all jobs to the new
content provider architecture.

The Content Provider
--------------------

The main task executed by the content provider job is to build the containers
needed to deploy TripleO. This is achieved with a collection of ansible plays
defined in the `multinode-standalone-pre.yml`_ tripleo-quickstart-extras
playbook.

Once built, the content provider then needs to make those containers available
for use by the child jobs. The `build-container role itself`_ as invoked in
`multinode-standalone-pre.yml`_ ensures containers are pushed to the
a local registry on the content provider node. However the child job will need
to know the IP address on which they can reach that registry.

To achieve this we use the `zuul_return module`_ that allows for a parent
job to return data for consumption within child jobs. We set the required
zuul_return data in the `run-provider.yml playbook`_::

    - name: Set registry IP address
      zuul_return:
        data:
          zuul:
            pause: true
          registry_ip_address: "{{ hostvars[groups.all[0]].ansible_host }}"
          provider_dlrn_hash: "{{ dlrn_hash|default('') }}"
          provider_dlrn_hash_tag: "{{ dlrn_hash_tag|default('') }}"
          provider_job_branch: "{{ provider_job_branch }}"
          registry_ip_address_branch: "{{ registry_ip_address_branch }}"
          provider_dlrn_hash_branch: "{{ provider_dlrn_hash_branch }}"
      tags:
        - skip_ansible_lint

Child jobs retrieve the IP address for the content provider container
registry via the `registry_ip_address_branch dictionary`_. This contains a
mapping between the release (master, victoria, ussuri etc) and the IP address
of the content provider container registry with images for that release.
For example::

    registry_ip_address_branch:
      master: 38.145.33.72

Most jobs will only ever have one release in this dictionary but upgrade jobs
will require two (more on that later). Note that besides setting the
zuul_return data the task above sets the `zuul pause: true`_. As the name
suggests, this allows the parent content provider job to be paused until all
children have executed.

Given all the above, it should be of little surprise ;) that the
`content provider zuul job definition`_ is as follows (at time of writing)::

    - job:
        name: tripleo-ci-centos-8-content-provider
        parent: tripleo-ci-base-standalone-centos-8
        branches: ^(?!stable/(newton|ocata|pike|queens|rocky|stein)).*$
        run:
          - playbooks/tripleo-ci/run-v3.yaml
          - playbooks/tripleo-ci/run-provider.yml
        vars:
          featureset: '052'
          provider_job: true
          build_container_images: true
          ib_create_web_repo: true
          playbooks:
            - quickstart.yml
            - multinode-standalone-pre.yml

It uses the `same featureset as the standalone job`_. Notice the
`multinode-standalone-pre.yml`_ passed to tripleo-quickstart for execution.
The `run-provider.yml playbook`_ is executed as the last of the zuul `run` plays.

Finally, one other important task performed by the content provider job is to
build any dependent changes (i.e. depends-on in the code submission). This is
done with `build-test-packages`_ invoked in the `multinode-standalone-pre.yml`_.
We ensure that the built repo is available to child jobs by setting the
`ib_create_web_repo variable`_ when built-test-packages is invoked by a
provider job. This `makes the repo available via a HTTP server`_ on the
content provider node that consumers then retrieve as described below.

The Content Consumers
---------------------

The child jobs or content consumers must use the container registry available
from the content provider. To do this `we set the docker_registry_host`_
variable using the `job.registry_ip_address_branch` zuul_data returned from
the parent content provider.

Any dependent changes built by `build-test-packages`_ are installed into
consumer jobs using the `install-built-repo`_ playbook. This has been added
into the `appropriate base job definitions`_ as a *pre-run:* play.

Finally, in order to make a given zuul job a *consumer* job we must set the
content provider as dependency and pass the relevant variables. For example
in order to `run tripleo-ci-centos-8-scenario001-standalone as a consumer job`_::

        - tripleo-ci-centos-8-content-provider
        - tripleo-ci-centos-8-scenario001-standalone:
            files: *scen1_files
            vars: &consumer_vars
              consumer_job: true
              build_container_images: false
              tags:
                - standalone
            dependencies:
              - tripleo-ci-centos-8-content-provider


Upgrade Jobs
------------

Upgrade jobs are a special case because they require content from more than
one release. For instance tripleo-ci-centos-8-standalone-upgrade-ussuri will
deploy train containers and then upgrade to ussuri containers.

To achieve this we use two content provider jobs as dependencies for the upgrade
jobs that require them (not all do)::

        - tripleo-ci-centos-8-standalone-upgrade:
            vars: *consumer_vars
            dependencies:
              - tripleo-ci-centos-8-content-provider
              - tripleo-ci-centos-8-content-provider-ussuri

As shown earlier in this document the `registry_ip_address_branch dictionary`_
maps release to the appropriate registry. This is set by each of the two parent
jobs and once both have executed the dictionary will contain more than one
entry. For example::

    registry_ip_address_branch:
      master: 213.32.75.192
      ussuri: 158.69.75.154

The consumer upgrade jobs then use the appropriate registry for the deployment
or upgrade part of the test.

.. _`check and gate tripleo-ci jobs`: ci_primer.html
.. _`docker has recently announced`: https://www.docker.com/blog/scaling-docker-to-serve-millions-more-developers-network-egress/
.. _`content provider zuul job definition`: https://opendev.org/openstack/tripleo-ci/src/commit/fbaaa3324712b9a718ce17c82bb190d09cca95be/zuul.d/standalone-jobs.yaml#L1032
.. _`multinode-standalone-pre.yml`: https://opendev.org/openstack/tripleo-quickstart-extras/src/commit/e61200fec8acccb3d5fe20f68b64156a3daadb8a/playbooks/multinode-standalone-pre.yml
.. _`build-container role itself`: https://opendev.org/openstack/tripleo-ci/src/commit/fbaaa3324712b9a718ce17c82bb190d09cca95be/roles/build-containers/tasks/main.yaml#L265-L270
.. _`zuul_return module`: https://zuul-ci.org/docs/zuul/reference/jobs.html?highlight=zuul_return#return-values
.. _`run-provider.yml playbook`: https://opendev.org/openstack/tripleo-ci/src/commit/fbaaa3324712b9a718ce17c82bb190d09cca95be/playbooks/tripleo-ci/run-provider.yml#L56
.. _`zuul pause: true`: https://zuul-ci.org/docs/zuul/reference/jobs.html?highlight=pause#pausing-the-job
.. _`we set the docker_registry_host`: https://opendev.org/openstack/tripleo-quickstart-extras/src/commit/e61200fec8acccb3d5fe20f68b64156a3daadb8a/roles/extras-common/defaults/main.yml#L44
.. _`build-test-packages`: https://opendev.org/openstack/tripleo-quickstart-extras/src/branch/master/roles/build-test-packages/
.. _`ib_create_web_repo variable`: https://opendev.org/openstack/tripleo-quickstart-extras/src/commit/e61200fec8acccb3d5fe20f68b64156a3daadb8a/roles/install-built-repo/defaults/main.yml#L11
.. _`makes the repo available via a HTTP server`: https://opendev.org/openstack/tripleo-quickstart-extras/src/commit/e61200fec8acccb3d5fe20f68b64156a3daadb8a/roles/install-built-repo/templates/install-built-repo.sh.j2#L17-L23
.. _`install-built-repo`: https://opendev.org/openstack/tripleo-ci/src/commit/fbaaa3324712b9a718ce17c82bb190d09cca95be/playbooks/tripleo-ci/install-built-repo.yml#L16-L27
.. _`appropriate base job definitions`: https://opendev.org/openstack/tripleo-ci/src/commit/fbaaa3324712b9a718ce17c82bb190d09cca95be/zuul.d/base.yaml#L184
.. _`run tripleo-ci-centos-8-scenario001-standalone as a consumer job`: https://opendev.org/openstack/tripleo-ci/src/commit/fbaaa3324712b9a718ce17c82bb190d09cca95be/zuul.d/standalone-jobs.yaml#L483-L492
.. _`registry_ip_address_branch dictionary`: https://opendev.org/openstack/tripleo-ci/src/commit/fbaaa3324712b9a718ce17c82bb190d09cca95be/playbooks/tripleo-ci/run-provider.yml#L26
.. _`same featureset as the standalone job`: https://github.com/openstack/tripleo-quickstart/blob/671893a60467ad76359eaaf2199c55b64cc20702/config/general_config/featureset052.yml#L2
