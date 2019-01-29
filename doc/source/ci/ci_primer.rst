TripleO CI jobs primer
======================

This primer aims to demonstrate where the Triple ci jobs are defined and
illustrate the difference between the check and gate queues and how jobs
are executed in them. Which queue a job is executed in also affects whether the
job is defined as voting or not. Generally:

* new jobs are run in check and are non voting
* once a job is voting in check, it needs to be added to gate too.
* once a job is voting in check and gate you should add it to the promotion
  jobs so that tripleo promotions (i.e. from tripleo-testing
  to current-tripleo) will depend on successful execution of that job.

Once a job becomes voting it must be added to the gate queue too. If it isn't
then we may end up with a situation where something passes the voting
check job and merges without being run in the gate queue. It could be that for
some reason it would have failed in the gate and thus not have merged. A common
occurrence is the check jobs run on a particular submission and pass on one day but
then not actually merge (and so run in the gate) until much later perhaps even after
some days.In the meantime some unrelated change merges in another project which would
cause the job to fail in the gate, but since we're not running it there the code
submission merges. This then means that the job is broken in subsequent check runs.

Non tripleo-projects are not gated in tripleo. The promotion jobs
represent the point at which we take the latest built tripleo packages and the
latest built non-tripleo projects packages (like nova, neutron etc) and test these together.
For more information about promotions refer to :doc:`Promotion Stages</ci/stages-overview>`

Where do tripleo-ci jobs live
-----------------------------

.. note::

  If you ever need to search for a particular job to see which file it is defined
  in or which tripleo project repos it is running for you can search by name in
  the openstack-codesearch_ (e.g. that is a search for the
  tripleo-ci-centos-7-scenario003-standalone job).

.. note::

  If you ever want to see the status for a particular job with respect to how
  often it is failing or passing, you can check the zuul_builds_ status and
  search by job name (again the linked example is for scenario003-standalone).

The tripleo ci jobs live in the tripleo-ci repo and specifically in various
files defined under the zuul.d_ directory. As an example we can examine one of
the scenario-standalone-jobs_::

    - job:
      name: tripleo-ci-centos-7-scenario001-standalone
      voting: true
      parent: tripleo-ci-base-standalone
      nodeset: single-centos-7-node
      branches: ^(?!stable/(newton|ocata|pike|queens|rocky)).*$
      vars:
        featureset: '052'
        standalone_ceph: true
        featureset_override:
          standalone_container_cli: docker
          standalone_environment_files:
            - 'ci/environments/scenario001-standalone.yaml'
            - 'environments/low-memory-usage.yaml'
          tempest_plugins:
            - python-telemetry-tests-tempest
            - python-heat-tests-tempest
          test_white_regex: ''
          tempest_workers: 1
          tempest_extra_config: {'telemetry.alarm_granularity': '60'}
          tempest_whitelist:
            - 'tempest.api.identity.v3'
            - 'tempest.scenario.test_volume_boot_pattern.TestVolumeBootPattern.test_volume_boot_pattern'
            - 'telemetry_tempest_plugin.scenario.test_telemetry_integration.TestTelemetryIntegration'

As you can see the job definition consists of the unique job name followed by
the rest of the zuul variables, including whether the job is voting and which
node layout (nodeset) should be used for that job. The unique job name is then
used in the zuul layout (discussed in the next section) to determine if the job
is run in check or gate or both. Since the job shown above is set as voting
we can expect it to be defined in both gate and check.

.. _zuul.d: https://github.com/openstack-infra/tripleo-ci/tree/master/zuul.d
.. _scenario-standalone-jobs: https://github.com/openstack-infra/tripleo-ci/blob/101074b2e804f97880440a3e62351844f390b2f2/zuul.d/standalone-jobs.yaml#L86-L88
.. _openstack-codesearch: http://codesearch.openstack.org/?q=tripleo-ci-centos-7-scenario003-standalone&i=nope&files=&repos=
.. _zuul_builds: http://zuul.openstack.org/builds?job_name=tripleo-ci-centos-7-scenario003-standalone

Zuul queues - gate vs check
---------------------------

As with all OpenStack projects there are two zuul queues to which jobs are
scheduled - the check jobs which are run each time a change is submitted and
then the gate jobs which are run before a change is merged. There is also
an experimental queue but that is invoked manually.

Which queue a given job is run in is determined by the zuul layout file for the
given project - e.g. here is tripleo-heat-templates-zuul-layout_. The layout
file has the following general format::

 - project:
    templates:
    .. list of templates
    check:
      jobs:
        .. list of job names and any options for each
    gate:
      queue: tripleo
      jobs:
        .. list of job names and any options for each

The templates: section in the outline above is significant because the layout
can also be defined in one of the included templates. For example the
scenario-standalone-layout_ defines the check/gate layout for the
tripleo-standalone-scenarios-full template which is then included by the
projects that want the jobs defined in that template to execute in the manner
it specifies.

.. _tripleo-heat-templates-zuul-layout: https://github.com/openstack/tripleo-heat-templates/blob/efe9b8fa1fff7ef1828777a95eee9fe4d901f9b9/zuul.d/layout.yaml#L9
.. _scenario-standalone-layout: https://github.com/openstack-infra/tripleo-ci/blob/7333a6fc8ff3990a971a661a817e30ae25e06374/zuul.d/standalone-jobs.yaml#L77-L79

Where do tripleo promotion jobs live
------------------------------------

.. note::
  If you even need to find the definition for a particular promotion job you can
  search for it by name using the rdo-codesearch_.

The tripleo promotions jobs are not defined in the tripleo-ci but instead live
in the rdo-jobs_ repository. For more information about the promotion pipeline
in TripleO refer to the :doc:`Promotion Stages</ci/stages-overview>`

Similar to the tripleo-ci jobs, they are defined in various files under the
rdo-jobs-zuul.d_ directory and the job definitions look very similar to the
tripleo-ci ones - for example the
periodic-tripleo-ci-centos-7-multinode-1ctlr-featureset010-master_::

  - job:
    name: periodic-tripleo-ci-centos-7-multinode-1ctlr-featureset010-master
    parent: tripleo-ci-base-multinode-periodic
    vars:
      nodes: 1ctlr
      featureset: '010'
      release: master

If you even need to find the definition for a particular promotion job you can
search for it by name using the rdo-codesearch_.

.. _rdo-jobs: https://github.com/rdo-infra/rdo-jobs
.. _rdo-jobs-zuul.d: https://github.com/rdo-infra/rdo-jobs/tree/master/zuul.d
.. _periodic-tripleo-ci-centos-7-multinode-1ctlr-featureset010-master: https://github.com/rdo-infra/rdo-jobs/blob/76daaff19a464614a002655bc85db4080607f1bf/zuul.d/multinode-jobs.yaml#L148
.. _rdo-codesearch: https://codesearch.rdoproject.org/?q=periodic-tripleo-ci-centos-7-multinode-1ctlr-featureset010-master&i=nope&files=&repos=
