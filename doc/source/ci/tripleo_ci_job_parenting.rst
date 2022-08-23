TripleO CI Zuul Jobs Parenting
==============================

When a developer submits a patch to TripleO repositories, their code is
tested against a series of different TripleO CI jobs.
Each job creates a different scenario for testing purposes.

The TripleO CI jobs are Zuul jobs, defined within TripleO projects under
one of several locations: `zuul.d`_ directory, .zuul.yaml or zuul.yaml.

A Zuul job can be inherited in various child jobs as `parent`_.


Zuul Job Parenting
++++++++++++++++++

In order to re-use a particular Zuul job, we create
a set of standard base jobs, which contain
ansible variables, required projects, pre-run, run,
post-run steps and Zuul related variables.

These base job definitions are used as `parent`_ in various tripleo-ci
jobs. The child job inherits attributes from the parent unless
these are overridden by the child.

A child job can override the variable which is also defined
in parent job.

TripleO CI Base jobs
++++++++++++++++++++

TripleO CI base jobs are defined in `zuul.d/base.yaml`_ file
in tripleo-ci repo.

Below is the list of base jobs and each is explained in a little more detail
in subsequent sections:

* tripleo-ci-base-common-required-projects
* tripleo-ci-base-standard
* tripleo-ci-base-multinode-standard
* tripleo-ci-base-singlenode-standard
* tripleo-ci-base-standalone-standard
* tripleo-ci-base-standalone-upgrade-standard
* tripleo-ci-base-ovb-standard
* tripleo-ci-base-containers-standard
* tripleo-ci-base-images-standard
* tripleo-ci-content-provider-standard

tripleo-ci-base-common-required-projects
----------------------------------------

It contains a list of common required projects and ansible roles
which are needed to start the deployment. It is used in
upstream, RDO and Downstream.
If a new project is needed in all types of deployment
(upstream, RDO and Downstream) it can be added here.

tripleo-ci-base-standard
------------------------

It contains a set of ansible variables and playbooks used in
most deployments.

tripleo-ci-base-multinode-standard
----------------------------------
It contains a set of ansible variables and playbooks used in
most containers multinode and scenarios job.

It is used in those jobs where the user needs to deploy
OpenStack using one undercloud and one controller.

tripleo-ci-base-singlenode-standard
-----------------------------------
It contains a set of ansible variables and playbooks used in
most single node jobs.

It is used in those jobs where user needs to build containers
and overcloud images which later can be used in another deployment.

It can also be used for undercloud deployment.

tripleo-ci-base-standalone-standard
-----------------------------------
It contains a set of ansible variables and playbooks used in
vanilla standalone and standalone based scenario jobs.

The standalone job consists of single node overcloud deployment.

tripleo-ci-base-standalone-upgrade-standard
-------------------------------------------
It contains a set of ansible variables and playbooks used in
the standalone upgrade job.

The singlenode job consists of single node overcloud deployment
where we upgrade a deployment from an older release to a newer one.

tripleo-ci-base-ovb-standard
----------------------------
It contains a set of ansible variables and playbooks used in
the virtual baremetal deployment.

The ovb job consists of one undercloud and four overcloud
nodes (one compute and multiple controllers) deployed as
virtual baremetal nodes. It is a replica of
real world customer deployments.

It is used in RDO and downstream jobs.

tripleo-ci-base-containers-standard
-----------------------------------
It contains a set of ansible variables and playbooks used
during build containers and pushing it to specific registry.

tripleo-ci-base-images-standard
-------------------------------
It contains a set of ansible variables and playbooks used
during build overcloud images and pushing it to image server.

tripleo-ci-content-provider-standard
------------------------------------
It contains a set of ansible variables and playbooks used for
building containers and pushing them to a local registry.
Depends-on patches are built into respective rpm packages via DLRN and
served by a local yum repos.

The job is `paused`_ to serve container registry and yum repos which
can be used later in dependent jobs.

Currently these jobs are running in Upstream and Downstream.

Required Project Jobs
+++++++++++++++++++++

It contains the list of required projects needed for specific type
of deployment.

Upstream job `tripleo-ci-build-containers-required-projects-upstream`_
requires projects like ansible-role-container-registry,
kolla, python-tripleoclient, tripleo-ansible to build containers.

In case of RDO `tripleo-ci-build-containers-required-projects-rdo`_ serves the
same purpose.

Many Upstream OpenStack projects are forked downstream and have different
branches.

To accommodate the downstream namespace and branches we use the downstream
specific required project job (*required-projects-downstream*)
as a base job with proper branches and override-checkout.

tripleo-ci-base-required-projects-multinode-internal job defined in the
examples are perfect example for the same.

Below is one of the examples of container multinode required projects job.

`Upstream`_ ::

    - job:
        name: tripleo-ci-base-required-projects-multinode-upstream
        description: |
                    Base abstract job to add required-projects for Upstream Multinode Jobs
        abstract: true
        parent: tripleo-ci-base-multinode-standard
        required-projects:
          - opendev.org/openstack/tripleo-ansible
          - opendev.org/openstack/tripleo-common
          - opendev.org/openstack/tripleo-operator-ansible
          - name: opendev.org/openstack/ansible-config_template
            override-checkout: master

`RDO`_ ::

    - job:
        name: tripleo-ci-base-required-projects-multinode-rdo
        abstract: true
        description: |
            Base abstract job for multinode in RDO CI zuulv3 jobs
        parent: tripleo-ci-base-multinode-standard
        pre-run:
          - playbooks/tripleo-rdo-base/pre.yaml
          - playbooks/tripleo-rdo-base/container-login.yaml
        roles:
          - zuul: opendev.org/openstack/ansible-role-container-registry
          - zuul: opendev.org/openstack/tripleo-ansible
        required-projects:
          - opendev.org/openstack/ansible-role-container-registry
          - opendev.org/openstack/tripleo-ansible
        secrets:
          - rdo_registry
        vars:
          registry_login_enabled: true


Downstream ::

    - job:
        name: tripleo-ci-base-required-projects-multinode-internal
        description: |
            Base abstract job to add required-projects for multinode downstream job
        abstract: true
        override-checkout: <downstream branch name>
        parent: tripleo-ci-base-multinode-standard
        required-projects:
          - name: tripleo-ansible
            branch: <downstream-branch>
          - ansible-config_template
          - tripleo-operator-ansible
          - rdo-jobs
          - tripleo-environments
        roles:
          - zuul: rdo-jobs
        pre-run:
          - playbooks/configure-mirrors.yaml
          - playbooks/tripleo-rdo-base/cert-install.yaml
          - playbooks/tripleo-rdo-base/pre-keys.yaml
        vars:
          mirror_locn: <downstream mirror address>
          featureset_override:
            artg_repos_dir: /home/zuul/src/<downstream-url>/openstack

Distribution Jobs
+++++++++++++++++

The TripleO deployment is supported on multiple distro versions.
Here is the current supported matrix in RDO, Downstream and Upstream.

+----------+------------------------------+-------------+
| Release  | CentOS/CentOS Stream Version |RHEL Version |
+==========+==============================+=============+
| Master   | 9-Stream                     |-            |
+----------+------------------------------+-------------+
| Wallaby  | 8-Stream, 9-Stream           |8.x, 9       |
+----------+------------------------------+-------------+
| Victoria | 8-Stream                     |-            |
+----------+------------------------------+-------------+
| Ussuri   | 8-Stream                     |-            |
+----------+------------------------------+-------------+
| Train    | 7, 8-Stream                  |8.x          |
+----------+------------------------------+-------------+

Each of these distros have different settings which are used in deployment.
It's easier to maintain separate variables based on distributions.

Below is an example of distro jobs for containers multinode at different levels.

`Upstream Distro Jobs`_ ::


    - job:
        name: tripleo-ci-base-multinode
        abstract: true
        description: |
                    Base abstract job for multinode TripleO CI C7 zuulv3 jobs
        parent: tripleo-ci-base-required-projects-multinode-upstream
        nodeset: two-centos-7-nodes


    - job:
        name: tripleo-ci-base-multinode-centos-8
        abstract: true
        description: |
                    Base abstract job for multinode TripleO CI centos-8 zuulv3 jobs
        parent: tripleo-ci-base-required-projects-multinode-upstream
        nodeset: two-centos-8-nodes

    - job:
        name: tripleo-ci-base-multinode-centos-9
        abstract: true
        description: |
                    Base abstract job for multinode TripleO CI centos-9 zuulv3 jobs
        parent: tripleo-ci-base-required-projects-multinode-upstream
        nodeset: two-centos-9-nodes

`RDO Distro Jobs`_ ::

    - job:
        name: tripleo-ci-base-multinode-periodic
        parent: tripleo-ci-base-multinode-rdo
        pre-run: playbooks/tripleo-ci-periodic-base/pre.yaml
        post-run: playbooks/tripleo-ci-periodic-base/post.yaml
        required-projects:
          - config
          - rdo-infra/ci-config
        roles:
          - zuul: rdo-infra/ci-config
        secrets:
          - dlrnapi

    - job:
        name: tripleo-ci-base-multinode-periodic-centos-8
        parent: tripleo-ci-base-multinode-rdo-centos-8
        pre-run: playbooks/tripleo-ci-periodic-base/pre.yaml
        post-run: playbooks/tripleo-ci-periodic-base/post.yaml
        required-projects:
          - config
          - rdo-infra/ci-config
        roles:
          - zuul: rdo-infra/ci-config
        vars:
          promote_source: tripleo-ci-testing
        secrets:
          - dlrnapi

    - job:
        name: tripleo-ci-base-multinode-periodic-centos-9
        parent: tripleo-ci-base-multinode-rdo-centos-9
        pre-run: playbooks/tripleo-ci-periodic-base/pre.yaml
        post-run: playbooks/tripleo-ci-periodic-base/post.yaml
        required-projects:
          - config
          - rdo-infra/ci-config
        roles:
          - zuul: rdo-infra/ci-config
        vars:
          promote_source: tripleo-ci-testing
        secrets:
          - dlrnapi

Zuul Job Inheritance Order
++++++++++++++++++++++++++

Here is an example of Upstream inheritance of tripleo-ci-centos-9-containers-multinode_ job.::

    tripleo-ci-base-common-required-projects
       |
       v
    tripleo-ci-base-standard
       |
       v
    tripleo-ci-base-multinode-standard
       |
       v
    tripleo-ci-base-required-projects-multinode-upstream
       |
       v
    tripleo-ci-base-multinode-centos-9
       |
       v
    tripleo-ci-centos-9-containers-multinode


Here is the another example of RDO job periodic-tripleo-ci-centos-8-containers-multinode-master_ ::

    tripleo-ci-base-multinode-standard
       |
       v
    tripleo-ci-base-required-projects-multinode-rdo
       |
       v
    tripleo-ci-base-multinode-rdo-centos-8
       |
       v
    tripleo-ci-base-multinode-periodic-centos-8
       |
       v
    periodic-tripleo-ci-centos-8-containers-multinode-master


TripleO CI Zuul Job Repos
+++++++++++++++++++++++++

Below is the list of repos where tripleo-ci related Zuul jobs are defined.

Upstream
--------
* `tripleo-ci <https://opendev.org/openstack/tripleo-ci/src/branch/master/zuul.d>`_

RDO
---
* `config <https://github.com/rdo-infra/review.rdoproject.org-config/tree/master/zuul.d>`_: Jobs which needs secrets are defined here.
* `rdo-jobs <https://github.com/rdo-infra/rdo-jobs/tree/master/zuul.d>`_

FAQs regarding TripleO CI jobs
++++++++++++++++++++++++++++++

* If we have a new project, which needs to be tested at all places
  and installed from source but

  - cloned from upstream source, then it must be added under required-projects
    at tripleo-ci-base-common-required-projects job.

  - the project namespace is different in Upstream and downstream, then it must be
    added under required-projects at
    Downstream (tripleo-ci-base-required-projects-multinode-internal) or
    Upstream (tripleo-ci-base-required-projects-multinode-upstream) specific
    required-projects parent job.

  - if the project is only developed at downstream or RDO or Upstream, then it must
    be added under required project at downstream or RDO or Upstream required-projects
    parent job.

* In order to add support for new distros, please use required-projects job as a
  parent and then create distro version specific child job with required nodeset.

* If a project with different branch is re-added in child job required-projects,
  then the child job project will be used in the deployment.

* If a playbook (which calls another role, exists in different repo) is called at
  pre-run step in Zuul job, then role specific required projects and roles needs
  to be added at that job level. For example: In `tripleo-ci-containers-rdo-upstream-pre`_
  job, ansible-role-container-registry and triple-ansible is needed for pre.yaml playbook.
  So both projects are added in roles and required-projects.

* If a job having pre/post run playbook needs zuul secrets and playbook depends on
  distros, then the job needs to be defined in config repo.

* We should not use branches `attributes`_ in Zuul Distro jobs or options jobs.

.. _`zuul.d`: https://opendev.org/openstack/tripleo-ci/src/branch/master/zuul.d
.. _`parent`: https://zuul-ci.org/docs/zuul/latest/config/job.html#attr-job.parent
.. _`zuul.d/base.yaml`: https://opendev.org/openstack/tripleo-ci/src/branch/master/zuul.d/base.yaml
.. _`tripleo-ci-build-containers-required-projects-rdo`: https://github.com/rdo-infra/rdo-jobs/commit/86e7e63ce6da27c2815afa845a6878cf96acdb47#diff-4897e02c92e2979a54f09d6eb383dba74c9a9211b065a52f9ecc4efbcce19637R17
.. _`paused`: https://zuul-ci.org/docs/zuul/latest/job-content.html#pausing-the-job
.. _`tripleo-ci-build-containers-required-projects-upstream`: https://opendev.org/openstack/tripleo-ci/commit/1d640d09fd808caa33b82f0bdd5622120cebef09
.. _`Upstream`: https://opendev.org/openstack/tripleo-ci/src/commit/9e270ea7f8c19fc3902a38d87a7ea4ace8219cd9/zuul.d/multinode-jobs.yaml#L17
.. _`RDO`: https://github.com/rdo-infra/review.rdoproject.org-config/commit/b96b916fb2446171f5040ba8168c470a79f1befa#diff-80b60a19d10a7b56e22da7bfc1926e4e8d2143670b3ec3f26d009bda8e8910bfR527
.. _`Upstream Distro Jobs`: https://github.com/openstack/tripleo-ci/commit/9e270ea7f8c19fc3902a38d87a7ea4ace8219cd9#diff-7653508e44c2cd8de8b5140648d7583c5efb27f0012155ff21f83c22edad69a3R29-R57
.. _`RDO Distro Jobs`: https://github.com/rdo-infra/review.rdoproject.org-config/commit/b96b916fb2446171f5040ba8168c470a79f1befa#diff-80b60a19d10a7b56e22da7bfc1926e4e8d2143670b3ec3f26d009bda8e8910bfR574-R616
.. _`periodic-tripleo-ci-centos-8-containers-multinode-master`: https://review.rdoproject.org/zuul/job/periodic-tripleo-ci-centos-8-containers-multinode-master
.. _`tripleo-ci-centos-9-containers-multinode`: https://zuul.openstack.org/job/tripleo-ci-centos-9-containers-multinode
.. _`tripleo-ci-containers-rdo-upstream-pre`: https://opendev.org/openstack/tripleo-ci/commit/05366af2930d76b4791a0fcb1f8ed9fddb132721
.. _`attributes`: https://opendev.org/openstack/tripleo-ci/commit/bda6e1a61a846890c9cc39d0bc91952e9c6deb8f
