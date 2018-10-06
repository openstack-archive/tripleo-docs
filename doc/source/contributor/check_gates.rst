How to add a TripleO job to your projects check pipeline
========================================================

To ensure a non-TripleO project's changes work with TripleO an additional
check job can be added to the project's job definitions in OpenStack's
`project config <https://github.com/openstack-infra/project-config>`_

Project Config Example
----------------------

In this case we'll use openstack/neutron as an example to understand how
this works. Note that this is only an example and this job may not be appropriate
for your project, we will cover how to pick a job later on in this documentation.
Browse through the `layout.yaml
<https://github.com/openstack-infra/project-config/blob/master/zuul/layout.yaml>`_
file in the project-config repository until you find::

    - name: openstack/neutron
      template:
        - name: merge-check
        - ...
        - ...
      check:
        - ...
        - ...
        - gate-tripleo-ci-centos-7-nonha-multinode-oooq-nv

The above configuration will run the TripleO job
``gate-tripleo-ci-centos-7-nonha-multinode-oooq-nv`` without voting (nv).
This type of job is used to inform the reviewers of the patch whether or not
the change under review works with TripleO.


How to pick which job to execute for any given OpenStack project
----------------------------------------------------------------

TripleO can deploy a number of different OpenStack services. To best utilize
the available upstream CI resources TripleO uses the same concept as the
`puppet-openstack-integration project
<https://github.com/openstack/puppet-openstack-integration>`_ to define how
services are deployed.  The TripleO documentation regarding services can be found
`here. <https://github.com/openstack/tripleo-heat-templates/blob/master/README.rst#service-testing-matrix>`_
Review the TripleO documentation and find a scenario that includes the services
that your project requires to be tested.  Once you have determined which
scenario to use you are ready to pick a TripleO check job.

The following is a list of available check jobs::

    gate-tripleo-ci-centos-7-scenario001-multinode-oooq
    gate-tripleo-ci-centos-7-scenario001-multinode-oooq-puppet
    gate-tripleo-ci-centos-7-scenario001-multinode-oooq-container
    gate-tripleo-ci-centos-7-scenario002-multinode-oooq
    gate-tripleo-ci-centos-7-scenario002-multinode-oooq-puppet
    gate-tripleo-ci-centos-7-scenario002-multinode-oooq-container
    gate-tripleo-ci-centos-7-scenario003-multinode-oooq
    gate-tripleo-ci-centos-7-scenario003-multinode-oooq-puppet
    gate-tripleo-ci-centos-7-scenario003-multinode-oooq-container
    gate-tripleo-ci-centos-7-scenario004-multinode-oooq
    gate-tripleo-ci-centos-7-scenario004-multinode-oooq-puppet
    gate-tripleo-ci-centos-7-scenario004-multinode-oooq-container
    gate-tripleo-ci-centos-7-nonha-multinode-oooq
    gate-tripleo-ci-centos-7-containers-multinode

**Note** over time additional scenarios will be added and will follow the same
pattern as the job names listed above.

Adding a new non-voting check job
---------------------------------

Find your project in `layout.yaml
<https://github.com/openstack-infra/project-config/blob/master/zuul/layout.yaml>`_.
An example of a project will look like the following example::

    - name: openstack/$project
      template:
        - ...
        - ...

**Note** ``$project`` is the name of your project.

Under the section named ``check``, add the job that best suits your project.
Be sure to add ``-nv`` to the job name to ensure the job does not vote::

      check:
        - ...
        - ...
        - $job-nv

Enabling voting jobs
--------------------

If your project is interested in gating your project with a voting version
of a TripleO job, you can follow the openstack/mistral project's example in
`layout.yaml
<https://github.com/openstack-infra/project-config/blob/master/zuul/layout.yaml>`_

For example::

    - name: openstack/mistral
      template:
        -name: merge-check
        - ...
        - ...
      check:
        - ...
        - ...
        - gate-tripleo-ci-centos-7-scenario003-multinode-oooq-puppet
      gate:
        - gate-tripleo-ci-centos-7-scenario003-multinode-oooq-puppet

**Note** the example does **not** append ``-nv`` as a suffix to the job name

Troubleshooting a failed job
----------------------------

When your newly added job fails, you may want to download its logs for a local
inspection and root cause analysis. Use the
`tripleo-ci gethelogs script
<https://github.com/openstack-infra/tripleo-ci/blob/master/scripts/getthelogs>`_
for that.

Enabling tempest tests notification
-----------------------------------

There is a way to get notifications by email when a job finishes to running
tempest.
People interested to receive these notifications can submit a patch to add
their email address in `this config file
<https://github.com/openstack/tripleo-quickstart-extras/blob/master/roles/validate-tempest/files/tempestmail/config.yaml>`_.
Instructions can be found `here
<https://github.com/openstack/tripleo-quickstart-extras/blob/master/roles/validate-tempest/files/tempestmail/README.md>`_.

featureset override
-------------------

The set of tempest tests that run for a given TripleO CI job is defined in the
`featureset config files
<https://github.com/openstack/tripleo-quickstart/tree/master/config/general_config>`_.
You may want to run a popular TripleO CI job with a custom set of Tempest
tests and override the default Tempest run. This can be accomplished through
`featureset_override` group of vars in Zuul job config. This setting allows
projects to override featureset post deployment configuration. The overridable
settings are:

 - `run_tempest`: To run tempest or not (true|false).
 - `tempest_whitelist`: List of tests you want to be executed.
 - `test_black_regex`: Set of tempest tests to skip.

For a given job `tripleo-ci-centos-7-scenario001-multinode-oooq-container`, you
can create a new abstract layer job and overrides the tempest tests::

    - job:
        name: scn001-multinode-oooq-container-custom-tempest
        parent: tripleo-ci-centos-7-scenario001-multinode-oooq-container
        abstract: true
        ...
        vars:
          featureset_override:
            run_tempest: true
            tempest_whitelist:
              - 'tempest.scenario.test_volume_boot_pattern.TestVolumeBootPattern.test_volume_boot_pattern'

In a similar way, for skipping Tempest run for the scenario001 job, you can do
something like::

    - job:
        name: scn001-multinode-oooq-container-skip-tempest
        parent: tripleo-ci-centos-7-scenario001-multinode-oooq-container
        abstract: true
        ...
        vars:
          featureset_override:
            run_tempest: false

