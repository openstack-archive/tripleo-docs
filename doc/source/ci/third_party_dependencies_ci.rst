Gating github projects using TripleO CI jobs
============================================

In TripleO deployment, we consume OpenStack and non-openstack projects.
In order to catch issues early, every patchset of the OpenStack projects
is gated with TripleO CI jobs using Zuul.

With the help of an RDO software factory instance, we can also now gate
non-openstack projects hosted on Github.

ceph-ansible and podman are the two non-openstack projects which are heavily
used in TripleO deployments and are hosted on github and for which we have
enabled TripleO CI jobs via github pull requests as described below.

Jobs running against ceph-ansible
---------------------------------

ceph-ansible_ is used to deploy Ceph in standalone scenario 1 and 4 jobs.
These jobs are defined in rdo-jobs_ repo.

On any ceph-ansible pull request, A user can trigger these jobs by leaving a
comment with 'check-rdo' on a pull request. It is currently done manually by
the OpenStack developers.

Then, those jobs will appear in the RDO software factory Zuul_ status page
under `github-check` pipeline.

On merged patches, periodic jobs are also triggered in
`openstack-periodic-weekend` pipeline_.

.. _ceph-ansible: https://github.com/ceph/ceph-ansible
.. _rdo-jobs: https://github.com/rdo-infra/rdo-jobs/blob/master/zuul.d/ceph-ansible.yaml
.. _Zuul: https://review.rdoproject.org/zuul/status
.. _pipeline: https://review.rdoproject.org/zuul/builds?pipeline=openstack-periodic-weekend&project=ceph%2Fceph-ansible

Jobs running against podman
---------------------------

In TripleO, OpenStack services are running in containers.
The container lifecycle, healthcheck and execution is managed via systemd using
paunch. Paunch under the hood uses podman.

The `podman` utility comes from libpod_ project.

Currently on each libpod pull request, tripleo ci based jobs get triggered
automatically and get queued in `github-check` pipeline in RDO software factory
Zuul instance.

TripleO jobs related to podman are defined in rdo-jobs-repo_.

For gating libpod project, we run keystone based scenario000 minimal tripleo
deployment job which tests the functionality of podman with keystone services.
It takes 30 mins to finish the tripleo deployment.

Below is the example job definition for scenario000-job_::

    - job:
        name: tripleo-podman-integration-rhel-8-scenario000-standalone
        parent: tripleo-ci-base-standalone-periodic
        nodeset: single-rhel-8-node
        branches: ^master$
        run: playbooks/podman/install-podman-rpm.yaml
        required-projects:
          - name: github.com/containers/libpod
        vars:
          featureset: '052'
          release: master
          registry_login_enabled: false
          featureset_override:
            standalone_environment_files:
              - 'environments/low-memory-usage.yaml'
              - 'ci/environments/scenario000-standalone.yaml'
              - 'environments/podman.yaml'
            run_tempest: false
            use_os_tempest: false

For re-running the tripleo jobs on libpod pull request, we can add
`check-github` comment on the libpod pull requests itself.

On merged patches, periodic jobs also get triggerd in
`openstack-regular` rdo-job-pipeline_.

.. _libpod: https://github.com/containers/libpod
.. _rdo-jobs-repo: https://github.com/rdo-infra/rdo-jobs/blob/master/zuul.d/podman.yaml
.. _scenario000-job: https://github.com/rdo-infra/rdo-jobs/blob/0186d637063c7e410ab9e0afc91b266c19e92473/zuul.d/podman.yaml#L50-L67
.. _rdo-job-pipeline: https://review.rdoproject.org/zuul/builds?pipeline=openstack-regular&project=containers%2Flibpod


Report bugs when Jobs start failing
-----------------------------------

TripleO Jobs running against libpod and ceph-ansible projects might fail due to
issue in libpod/ceph-ansible or in TripleO itself.

Once the status of any job is *FAILED* or *POST_FAILURE* or *RETRY_LIMIT*.
Click on the job link and it will open the build result page. Then click on
`log_url`, click on the `job-output.txt`. It contains the results of
ansible playbook runs.
Look for *ERROR* or failed messages.
If looks something ovious.
Please go ahead and create the bug on launchpad_ against tripleo project with
all the information.

Once the bug is created, please add `depcheck` tag on the filed launchpad bug.
This tag is explicitly used for listing bugs related to TripleO CI job failue
against ceph-ansible and podman projects.

.. _launchpad: https://bugs.launchpad.net/tripleo/+filebug

`check-rdo` vs `check-github`
-----------------------------

`check-rdo` and `check-github` comments are used to trigger TripleO based zuul
jobs against github projects (ceph-ansible/podman) 's pull requests.

.. note::

  On commenting `check-rdo` or `check-github`, not all jobs will appears in the
  github-manual pipeline. It depends whether the jobs are configured in the
  particular pipeline to get triggered. If the jobs are not defined there
  then, nothing will happen.

check-rdo
*********

It is used against ceph-ansible pull requests especially. The jobs will be gets
triggered and land in `github-check` pipeline.

check-github
************

If a TripleO job fails against ceph-ansible or podman PRs, then it can be
relaunched using `check-github` comment. The job will appear in `github-manual`
pipeline.

Using `Depends-On` on ceph-ansible/podman pull requests
-------------------------------------------------------

One can also create/put OpenStack or RDO gerrit reviews against
ceph-ansible/podman pull requests by putting
`Depends-On: <openstack/rdo gerrit review link>` in the first message
of the github pull request_.

.. _request: https://github.com/ceph/ceph-ansible/pull/3576
