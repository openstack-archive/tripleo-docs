Standalone Scenario jobs
========================

This section gives an overview and some details on the standalone scenario ci
jobs. The standalone deployment is intended as a one node development
environment for TripleO. - see the `Standalone Deploy Guide <standalone_deploy_guide_>`_
for more information on setting up a standalone environment.

A 'scenario' is a concept used in TripleO
to describe a collection of services - see the service-testing-matrix_ for more
information about each scenario and the services deployed there. We combine the
two to define the standalone scenario jobs.

These are intended to give developers faster feedback (the jobs are relatively
fast to complete) and allow us to have better coverage across services by defining a
number of scenarios. Crucially the standalone scenario jobs allow us to increase
coverage without further increasing our resource usage footprint with eachjob only taking
a single node. See this openstack-dev-thread_ for background around the move from
the multinode jobs to the more resource friendly standalone versions.

.. _service-testing-matrix: https://github.com/openstack/tripleo-heat-templates/blob/master/README.rst#service-testing-matrix
.. _openstack-dev-thread: http://lists.openstack.org/pipermail/openstack-dev/2018-October/136192.html
.. _standalone_deploy_guide: https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/standalone.html


Where
-----

The standalone scenario jobs (hereafter referred to as just 'standalone' in this
document), are defined in the `tripleo-ci/zuul.d/standalone.yaml`_ file. Besides
the definitions for each of the scenario00X-standalone jobs, this file also
carries the tripleo-standalone-scenarios-full_project-template_ which defines
the zuul layout and files: sections for the standalone jobs in a central location.

Thus, the jobs are consumed by other projects across tripleo by inclusion of
the template in their respective zuul layout file, for example
tripleo-heat-templates_ and tripleo-common_.

Besides the job definitions in the tripleo-ci repo, the other main part of the
standalone jobs is a service environment file, which lives in the
`tripleo-heat-templates-ci/environments`_. As you can see in scenario001-env_,
scenario002-env_, scenario003-env_ and scenario004-env_ that is where we define the
services and parameters that are part of a given scenario.

.. _`tripleo-ci/zuul.d/standalone.yaml`: https://github.com/openstack-infra/tripleo-ci/blob/master/zuul.d/standalone-jobs.yaml
.. _tripleo-standalone-scenarios-full_project-template: https://github.com/openstack-infra/tripleo-ci/blob/75ff68608baab31f6ac9e5395a9841c08c62e092/zuul.d/standalone-jobs.yaml#L78-L80
.. _tripleo-heat-templates: https://github.com/openstack/tripleo-heat-templates/blob/d5298e2f7936bcb5ca7d41466d024fe6958ce177/zuul.d/layout.yaml#L8
.. _tripleo-common: https://github.com/openstack/tripleo-common/blob/026ed7d9e041c92956aa9db59e881f6632eed2f2/zuul.d/layout.yaml#L14
.. _`tripleo-heat-templates-ci/environments`: https://github.com/openstack/tripleo-heat-templates/tree/master/ci/environments
.. _scenario001-env: https://github.com/openstack/tripleo-heat-templates/blob/1c46d1850a8de89daeecd96f2f5288336e3778f8/ci/environments/scenario001-standalone.yaml#L1
.. _scenario002-env: https://github.com/openstack/tripleo-heat-templates/blob/1c46d1850a8de89daeecd96f2f5288336e3778f8/ci/environments/scenario002-standalone.yaml#L1
.. _scenario003-env: https://github.com/openstack/tripleo-heat-templates/blob/1c46d1850a8de89daeecd96f2f5288336e3778f8/ci/environments/scenario003-standalone.yaml#L1
.. _scenario004-env: https://github.com/openstack/tripleo-heat-templates/blob/1c46d1850a8de89daeecd96f2f5288336e3778f8/ci/environments/scenario004-standalone.yaml#L1

How
---

The standalone jobs are special in that they differ from 'traditional' multinode
jobs by having a shared featureset rather than requiring a dedicated featureset
for each job. Some of the standalone scenarios, notably scenario012_ will end up
having a dedicated-featureset_ however in most cases the base standalone-featureset052_
can be re-used for the different scenarios. Notably you can see that scenario001-job_,
scenario002-job_, scenario003-job_ and scenario004-job_ job definitions are all
using the same standalone-featureset052_.

Given that we use the same featureset the main differentiator between these
standalone jobs is the scenario environment file, which we pass using
featureset_override (see :doc:`../ci/check_gates`).
For example in the scenario001 job we point to the scenario001-standalone.yaml
(scenario001-env_)::

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
             - 'environments/low-memory-usage.yaml'
             - 'ci/environments/scenario001-standalone.yaml'
      ...

Finally we use a task in the tripleo-ci-run-test-role_ to pass the scenario
environment file into the standalone deployment command using the standalone
role standalone_custom_env_files_ parameter.

.. _scenario012: https://review.opendev.org/634723
.. _dedicated-featureset: https://review.opendev.org/636355
.. _standalone-featureset052: https://github.com/openstack/tripleo-quickstart/blob/6585d6320ca4f0c37ae62dfc60fe2eb0cd42647c/config/general_config/featureset052.yml#L2
.. _scenario001-job: https://github.com/openstack-infra/tripleo-ci/blob/1d890565feeeea6ce637cf0384da822926480f07/zuul.d/standalone-jobs.yaml#L376
.. _scenario002-job: https://github.com/openstack-infra/tripleo-ci/blob/1d890565feeeea6ce637cf0384da822926480f07/zuul.d/standalone-jobs.yaml#L401
.. _scenario003-job: https://github.com/openstack-infra/tripleo-ci/blob/1d890565feeeea6ce637cf0384da822926480f07/zuul.d/standalone-jobs.yaml#L426
.. _scenario004-job: https://github.com/openstack-infra/tripleo-ci/blob/1d890565feeeea6ce637cf0384da822926480f07/zuul.d/standalone-jobs.yaml#L448
.. _tripleo-ci-run-test-role: https://github.com/openstack-infra/tripleo-ci/blob/1d890565feeeea6ce637cf0384da822926480f07/roles/run-test/tasks/main.yaml#L26-L36
.. _standalone_custom_env_files: https://github.com/openstack/tripleo-quickstart-extras/blob/def233448d2ae8ed5bcc6d286f5cf8378f7cf7ec/roles/standalone/templates/standalone.sh.j2#L9
