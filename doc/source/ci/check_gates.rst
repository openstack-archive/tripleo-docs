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
`tripleo-ci getthelogs script
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

In TripleO CI, we test each patchset using different jobs. These jobs
are defined using `featureset config files
<https://opendev.org/openstack/tripleo-quickstart/src/branch/master/config/general_config>`_.
Each featureset config file is mapped to a job template that is defined in
`tripleo-ci <https://opendev.org/openstack-infra/tripleo-ci/src/branch/master/zuul.d>`_.
Tempest tests are basically triggered in scenario jobs in order to post validate the
a particular scenario deployment.
The set of tempest tests that run for a given TripleO CI job is defined in the
`featureset config files
<https://opendev.org/openstack/tripleo-quickstart/src/branch/master/config/general_config>`_.
You may want to run a popular TripleO CI job with a custom set of Tempest
tests and override the default Tempest run. This can be accomplished through
adding the `featureset_overrides` var to zuul job config `vars:` section.
The allowed featureset_override are defined in the `tripleo-ci run-test role
<https://opendev.org/openstack/tripleo-ci/src/commit/5a902b351f3728a95e4a989527178c66815bdc54/roles/run-test/tasks/main.yaml#L8>`_.
This setting allows projects to override featureset post deployment configuration.
Some of the overridable settings are:

 - `run_tempest`: To run tempest or not (true|false).
 - `tempest_whitelist`: List of tests you want to be executed.
 - `test_black_regex`: Set of tempest tests to skip.
 - `tempest_format`: To run tempest using different format (packages, containers, venv).
 - `tempest_extra_config`: A dict of additional tempest config to be overridden.
 - `tempest_plugins`: A list of tempest plugins needs to be installed.
 - `standalone_environment_files`: List of environment files to be overridden
   by the featureset configuration on standalone deployment. The environment
   file should exist in tripleo-heat-templates repo.
 - `test_white_regex`: Regex to be used by tempest
 - `tempest_workers`: Numbers of parallel workers to run
 - `standalone_container_cli`: Container cli to use
 - `tempest_private_net_provider_type`: The Neutron type driver that should be
   used by tempest tests.

For a given job `tripleo-ci-centos-7-scenario001-multinode-oooq-container`, you
can create a new abstract layer job and overrides the tempest tests::

    - job:
        name: scn001-multinode-oooq-container-custom-tempest
        parent: tripleo-ci-centos-7-scenario001-multinode-oooq-container
        ...
        vars:
          featureset_override:
            run_tempest: true
            tempest_whitelist:
              - 'tempest.scenario.test_volume_boot_pattern.TestVolumeBootPattern.test_volume_boot_pattern'
            test_black_regex:
              - 'keystone_tempest_plugin'
            tempest_format: 'containers'
            tempest_extra_config: {'compute-feature-enabled.attach_encrypted_volume': 'True',
                                   'auth.tempest_roles': '"Member"'}
            tempest_plugins:
              - 'python2-keystone-tests-tempest'
              - 'python2-cinder-tests-tempest'
            tempest_workers: 1
            test_white_regex:
              - 'tempest.api.identity'
              - 'keystone_tempest_plugin'
            standalone_environment_files:
              - 'environments/low-memory-usage.yaml'
              - 'ci/environments/scenario003-standalone.yaml'
            standalone_container_cli: docker

In a similar way, for skipping Tempest run for the scenario001 job, you can do
something like::

    - job:
        name: scn001-multinode-oooq-container-skip-tempest
        parent: tripleo-ci-centos-7-scenario001-multinode-oooq-container
        ...
        vars:
          featureset_override:
            run_tempest: false

Below is the list of jobs based on `tripleo-puppet-ci-centos-7-standalone` which uses
featureset_override and run specific tempest tests against puppet projects:

* puppet-nova

  - job name: puppet-nova-tripleo-standalone
  - tempest_test: compute

* puppet-horizon

  - job name: puppet-horizon-tripleo-standalone
  - tempest_test: horizon

* puppet-keystone

  - job name: puppet-keystone-tripleo-standalone
  - tempest_test: keystone_tempest_plugin & identity

* puppet-glance

  - job name: puppet-glance-tripleo-standalone
  - tempest_test: image

* puppet-cinder

  - job name: puppet-cinder-tripleo-standalone
  - tempest_test: volume & cinder_tempest_tests

* puppet-neutron

  - job name: puppet-neutron-tripleo-standalone
  - tempest_test: neutron_tempest_tests & network

* puppet-swift

  - job name: puppet-swift-tripleo-standalone
  - tempest_test: object_storage
