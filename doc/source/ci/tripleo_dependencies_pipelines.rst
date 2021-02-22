TripleO Dependency Pipeline
+++++++++++++++++++++++++++++

This section introduces the TripleO Dependency Pipeline. The dependency
pipeline is what the TripleO CI team calls the series of zuul CI jobs
that aim to catch problems in deployment *dependencies*.

A dependency is any package that is not directly related to the deployment
of OpenStack itself, such as OpenvSwitch, podman, buildah, pacemaker and ansible.
Each time, these projects release a newer version, it breaks the OpenStack
deployment and CI.

Currently we have `promotion and component pipeline`_ set up to detect
OpenStack projects related issues early.

In order to detect the breakages from non-openstack projects, TripleO
dependency pipeline comes into existance. we have added two types of
pipeline:

* packages coming from specific module
* packages coming from specific repo

The configurations for each pipeline can be found under
tripleo-quickstart/src/branch/master/config/release/dependency_ci/<module/repo name>/repo_config.yaml.

Current OpenStack Dependency Pipeline jobs
------------------------------------------
* openstack-dependencies-containertools - for testing container tools dependencies
* openstack-dependencies-centos8stream - for testing base operating system dependencies coming from CentOS-8 stream repo.
* openstack-dependencies-openvswitch - for testing OVS and OVN dependencies coming from NFV sig repo.

UnderStanding Module dependency pipeline
----------------------------------------

Under openstack-dependencies-containertools pipeline,
We test the latest podman/buildah version comes from container tools RHEL-8 module.

The release specific configuration for `containertools`_ pipeline can be found here:

.. code-block:: yaml

    dependency_modules:
      - module_name: container-tools
        control_version: 2.0
        test_version: rhel8
    dep_repo_cmd_after: |
      sudo dnf repolist;
      sudo dnf module list;
      {% for item in dependency_modules %}
      sudo dnf module disable {{ item.module_name }}:{{ item.control_version }} -y;
      sudo dnf module enable {{ item.module_name }}:{{ item.test_version }} -y;
      {% endfor %}
      sudo dnf clean metadata
      sudo dnf clean all;
      sudo dnf update -y;


What do the above terms mean:
* `module_name`: Name of the DNF module to pull packages from
* `control_version`: Name of the version to disable for the particular module
* `test_version`: Name of the version to enable for the particular module

In the above cases, it is going to test container-tools module in which
it will disable container-tools:2.0 and enable container-tools:rhel8 module.
All the container-tools packages will be coming here.

The above config is used by `get-dependency-module-content.yaml`_ to enable or
disable the above module during the deployment.

- jobs running in `openstack-dependencies-containertools`_ pipeline on review.rdoproject.org.
  * periodic-tripleo-ci-centos-8-standalone-container-tools-master
  * periodic-tripleo-ci-centos-8-standalone-container-tools-container-build-master

The job definitions are defined under `rdo-jobs/zuul.d/dependencies-jobs.yaml`_ and
their triggers are defined here: `rdo-jobs/zuul.d/project-templates-dependencies.yaml`_.

UnderStanding package dependency pipeline
-----------------------------------------

openstack-dependencies-openvswitch is a package dependency pipeline where we
tests OVS and OVN packages coming from NFV sig.

Here is the config for the `openvswitch dependency pipeline`_:

.. code-block:: yaml

    add_repos:
      - type: generic
        reponame: openvswitch-next
        filename: "openvswitch-next.repo"
        baseurl: "https://buildlogs.centos.org/centos/8/nfv/x86_64/openvswitch-2/"
        update_container: false
    dependency_override_repos:
      - centos-nfv-ovs,http://mirror.centos.org/centos/8/nfv/x86_64/openvswitch-2/
    dep_repo_cmd_after: |
      {% if dependency_override_repos is defined %}
      {% for item in dependency_override_repos %}
      sudo dnf config-manager --set-disabled {{ item.split(',')[0] }}
      {% endfor %}
      sudo dnf clean metadata;
      sudo dnf clean all;
      sudo dnf update -y;
      {% endif %}


What do the above terms mean?
* `add_repos`: This is the 'test' repo i.e. the one that is bringing us a newer
than 'normal' version of the package we are testing, OpenvSwitch in this case.
* `dependency_override_repos`: It is used to disable or override a particular repo.

In the above case, openvswitch-next.repo repo will get generated due to repo setup
and will disables the centos-nfv-ovs repo.

Before the deployment, `rdo-jobs/dependency/get-dependency-repo-content.yaml` playbook
is used to set particular release file (in this case it is
config/release/dependency_ci/openvswitch/repo_config.yaml) and then generate a diff
of packages from dependency_override_repos and new repos added by add_repos option.

Below are the jobs running in `openstack-dependencies-openvswitch`_ pipeline on review.rdoproject.org.

.. code-block:: yaml

    openstack-dependencies-openvswitch:
      jobs:
        - periodic-tripleo-ci-centos-8-standalone-openvswitch-container-build-master:
            dependencies:
              - periodic-tripleo-ci-centos-8-standalone-master
        - periodic-tripleo-ci-centos-8-scenario007-standalone-openvswitch-container-build-master:
            dependencies:
              - periodic-tripleo-ci-centos-8-scenario007-standalone-master
        - periodic-tripleo-ci-centos-8-standalone-master:
            vars:
              force_periodic: false
        - periodic-tripleo-ci-centos-8-scenario007-standalone-master:
            vars:
              force_periodic: false


Ensuring correct module or repo is used
---------------------------------------

Once a jobs runs and finishes in the dependency pipeline, we need to navigate
to job log url. Under `logs/undercloud/home/zuul` directory, we can see
two log files:

* control_repoquery_list.log.txt.gz - Contains a list of new packages coming from newly added repos.
* control_test_diff_table.log.txt.gz - contains a diff of the packages coming from new repo and overridden repo

All the above operation is done `rdo-jobs/playbooks/dependency/diff-control-test.yaml`_ playbook which uses
`compare_rpms`_ project from ci-config/ci-scripts/infra-setup/roles/rrcockpit/files.

.. _`promotion and component pipeline`: https://docs.openstack.org/tripleo-docs/latest/ci/stages-overview.html
.. _`openvswitch dependency pipeline`: https://opendev.org/openstack/tripleo-quickstart/src/branch/master/config/release/dependency_ci/openvswitch/repo_config.yaml
.. _`containertools`: https://opendev.org/openstack/tripleo-quickstart/src/branch/master/config/release/dependency_ci/container-tools/repo_config.yaml
.. _`openstack-dependencies-containertools`: https://review.rdoproject.org/zuul/builds?pipeline=openstack-dependencies-containertools
.. _`openstack-dependencies-openvswitch`: https://review.rdoproject.org/zuul/builds?pipeline=openstack-dependencies-openvswitch
.. _`rdo-jobs/zuul.d/dependencies-jobs.yaml`: https://github.com/rdo-infra/rdo-jobs/blob/master/zuul.d/dependencies-jobs.yaml
.. _`rdo-jobs/zuul.d/project-templates-dependencies.yaml`: https://github.com/rdo-infra/rdo-jobs/blob/master/zuul.d/project-templates-dependencies.yaml
.. _`rdo-jobs/playbooks/dependency/diff-control-test.yaml`: https://github.com/rdo-infra/rdo-jobs/blob/master/playbooks/dependency/diff-control-test.yaml
.. _`get-dependency-module-content.yaml`: https://github.com/rdo-infra/rdo-jobs/blob/master/playbooks/dependency/get-dependency-module-content.yaml
.. _`rdo-jobs/dependency/get-dependency-repo-content.yaml`: https://github.com/rdo-infra/rdo-jobs/blob/master/playbooks/dependency/get-dependency-repo-content.yaml
.. _`compare_rpms`: https://github.com/rdo-infra/ci-config/tree/master/ci-scripts/infra-setup/roles/rrcockpit/files/compare_rpms
