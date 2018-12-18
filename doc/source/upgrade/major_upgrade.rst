Upgrading to a Next Major Release
=================================

Upgrading a TripleO deployment to the next major release is done by first
upgrading the undercloud and using it to upgrade the overcloud.

Note that there are version specific caveats and notes which are pointed out
as below:

.. note::

   You can use the "Limit Environment Specific Content" in the left hand nav
   bar to restrict content to the upgrade you are performing.

.. note::

   Generic upgrade testing cannot cover all possible deployment
   configurations. Before performing the upgrade in production, test
   it in a matching staging environment, and create a backup of the
   production environment.

.. Undercloud upgrade section
.. include:: undercloud.rst

Upgrading the Overcloud to Queens and later
-------------------------------------------

The overcloud upgrade workflow is mainly delivered through the
`openstack overcloud upgrade` command, in particular one of its
subcommands: **prepare**, **run** and **converge**. Each subcommand
has its own set of options which you can explore with ``--help``:

.. code-block:: bash

       source /home/stack/stackrc
       openstack overcloud upgrade run --help

The upgrade workflow essentially consists of the following steps:

#. `Prepare your environment files`_.
   Generate any environment files you need for the upgrade such as the
   references to the latest container images or commands used to
   switch repos.

#. `openstack overcloud upgrade prepare`_.
   Run a heat stack update to generate the upgrade playbooks.

#. `openstack overcloud external-upgrade run (for container images)`_.
   Generate any environment files you need for the upgrade such as the
   references to the latest container images or commands used to switch repos.

#. `openstack overcloud upgrade run`_.
   Run the upgrade on specific nodes or groups of nodes. Repeat until all nodes
   are successfully upgraded.

#. `openstack overcloud external-upgrade run (for services)`_. (optional)
   This step is only necessary if your deployment contains services
   which are managed using external installers, e.g. Ceph.

#. `openstack overcloud external-upgrade run (for online upgrades)`_
   Run the part of service upgrades which can run while the cloud is
   fully operational, e.g. online data migrations.

#. `openstack overcloud upgrade converge`_.
   Finally run a heat stack update, unsetting any upgrade specific variables
   and leaving the heat stack in a healthy state for future updates.

Detailed infromation and pointers can be found in the relevant the
queens-upgrade-dev-docs_.

.. _queens-upgrade-dev-docs: https://docs.openstack.org/tripleo-docs/latest/install/developer/upgrades/major_upgrade.html # WIP @ https://review.openstack.org/#/c/569443/

Prepare your environment files
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

First we prepare an environment file for new container images:

.. admonition:: Pike to Queens
   :class: ptoq

   As part of the upgrade to Queens, the container images for the
   target release should be downloaded to the Undercloud. Please see
   the `openstack overcloud container image prepare`.
   :doc:`../install/containers_deployment/overcloud` for more information.

   The output of this step will be a Heat environment file that contains
   references to the latest container images. You will need to pass the path to
   this file into the **upgrade prepare** command using the -e option as you would
   any other environment file.

.. admonition:: Queens to Rocky
   :class: qtor

   In Rocky we only generate a new environment file with
   ``ContainerImagePrepare`` parameter at this point in the workflow. See
   :doc:`container image preparation documentation<../install/advanced_deployment/container_image_prepare>`.
   for details how to generate this environment file.

   The file is then passed to the `upgrade prepare` command, and
   images will be uploaded to the local registry in a separate
   `external-upgrade run` step afterwards.

You will also need to create an environment file to override the
UpgradeInitCommand_ tripleo-heat-templates parameter, that can be used to
switch the yum repos in use by the nodes during the upgrade. This will likely
be the same commands that were used to switch repositories on the undercloud.

.. code-block:: bash

   cat <<EOF > init-repo.yaml
     parameter_defaults:
     UpgradeInitCommand: |
       set -e
       #  -- REPLACE LINES WITH YOUR REPO SWITCH COMMANDS --
       curl -L -o /etc/yum.repos.d/delorean.repo https://trunk.rdoproject.org/centos7-queens/current/delorean.repo
       curl -L -o /etc/yum.repos.d/delorean-deps.repo https://trunk.rdoproject.org/centos7-queens/delorean-deps.repo
       yum clean all
     EOF

The resulting init-repo.yaml will then be passed into the upgrade prepare using
the -e option.

.. _Upgradeinitcommand: https://github.com/openstack/tripleo-heat-templates/blob/1d9629ec0b3320bcbc5a4150c8be19c6eb4096eb/puppet/role.role.j2.yaml#L468-L493

openstack overcloud upgrade prepare
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

   Before running the overcloud upgrade prepare ensure you have a valid backup
   of the current state, including the **undercloud** since there will be a
   Heat stack update performed here.

.. note::

   If you have enabled neutron_DVR_ in your deployment you must ensure that
   compute nodes are connected to the External network via the
   roles_data.yaml that you will pass using the -r parameter to upgrade prepare.
   This is necessary to allow floating IP connectivity via the external api network.

.. note::

   After running the upgrade prepare and until successful completion
   of the upgrade converge operation, stack updates to the deployment Heat
   stack are expected to fail. That is, operations such as scaling to add
   a new node or to apply any new TripleO configuration via Heat stack
   update **must not** be performed on a Heat stack that has been prepared
   for upgrade with the 'prepare' command and only consider doing so after
   running the converge step. See the queens-upgrade-dev-docs_ for more.

Run **overcloud upgrade prepare**. This command expects the full set
of environment files that were passed into the deploy command, as well
as the roles_data.yaml and network_data.yaml, if you've customized
those. Be sure to include environment files with the new container
image parameter and Yum repository switch parameter.

.. note::

   It is especially important to remember that you **must** include all
   environment files that were used to deploy the overcloud including the
   container image references file for the target version container images

.. code-block:: bash

   openstack overcloud upgrade prepare --templates \
     -r /path/to/roles_data.yaml \
     -n /path/to/network_data.yaml \
     -e <ALL Templates from overcloud-deploy.sh> \
     -e init-repo.yaml \
     -e containers-prepare-parameter.yaml

This will begin an update on the overcloud Heat stack but without
applying any of the TripleO configuration. Once this `upgrade prepare`
operation has successfully completed the heat stack will be in the
UPDATE_COMPLETE state. At that point you can use `config download` to
download and inspect the configuration ansible playbooks that will be
used to deliver the upgrade in the next step:

.. code-block:: bash

   openstack overcloud config download --config-dir SOMEDIR
   # playbooks will be downloaded to SOMEDIR directory

.. _neutron_DVR: https://specs.openstack.org/openstack/neutron-specs/specs/juno/neutron-ovs-dvr.html


openstack overcloud external-upgrade run (for container images)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. admonition:: Rocky
   :class: qtor

   In Rocky and beyond, we'll need to upload the container images to
   the local registry after we've run `upgrade prepare`. Run:

   .. code-block:: bash

      openstack overcloud external-update run --tags container_image_prepare

.. _openstack-overcloud-upgrade-run:

openstack overcloud upgrade run
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `upgrade run` command runs the Ansible playbooks to deliver the upgrade configuration.
By default, 3 playbooks are executed: the upgrade_steps_playbook, then the
deploy_steps_playbook and finally the post_upgrade_steps_playbook. These
playbooks are invoked on those overcloud nodes specified by the ``--roles`` or
``--nodes`` parameters, which are mutually exclusive. You are expected to use
``--roles`` for controlplane nodes, since these need to be upgraded in the same
step. For non controlplane nodes, such as Compute or Storage, you can use
``--nodes`` to specify a single node or list of nodes to upgrade.

.. code-block:: bash

   openstack overcloud upgrade run --roles Controller

**Optionally** specify ``--playbook`` to manually step through the upgrade
playbooks: You need to run all three in this order and as specified below
(no path) for a full upgrade.

.. code-block:: bash

   openstack overcloud upgrade run --roles Controller --playbook upgrade_steps_playbook.yaml
   openstack overcloud upgrade run --roles Controller --playbook deploy_steps_playbook.yaml
   openstack overcloud upgrade run --roles Controller --playbook post_upgrade_steps_playbook.yaml

After all three playbooks have been executed without error on all nodes of
the controller role the controlplane will have been fully upgraded to Queens.
At a minimum an operator should check the health of the pacemaker cluster.

.. code-block:: bash

   [root@overcloud-controller-0 ~]# pcs status | grep -C 10 -i "error\|fail"

The operator may also want to confirm that openstack and related service
containers are all in a good state and using the target version (new) images
passed during upgrade prepare.

.. code-block:: bash

   [root@overcloud-controller-0 ~]# docker ps -a

For non controlplane nodes, such as Compute or ObjectStorage, you can use
``--nodes overcloud-compute-0`` to upgrade particular nodes, or even
"compute0,compute1,compute3" for multiple nodes. Note these are again
upgraded in parallel. Also note that you can still use the ``--roles`` parameter
with non controlplane roles if that is preferred.

.. code-block:: bash

   openstack overcloud upgrade run --nodes overcloud-compute-0

Use of ``--nodes`` allows the operator to upgrade some subset, perhaps just one,
compute or other non controlplane node and verify that the upgrade is
successful. One may even migrate workloads onto the newly upgraded node and
confirm there are no problems, before deciding to proceed with upgrading the
remaining nodes that are still on Pike.

Again you can optionally step through the upgrade playbooks if you prefer. Be
sure to run upgrade_steps_playbook.yaml then deploy_steps_playbook.yaml and
finally post_upgrade_steps_playbook.yaml in that order.

.. code-block:: bash

   openstack overcloud upgrade run --nodes overcloud-compute-1 \
      --playbook upgrade_steps_playbook.yaml
   # etc for the other 2 as above example for controller

For re-run, you can specify ``--skip-tags`` validation to skip those step 0
ansible tasks that check if services are running, in case you can't or
don't want to start them all.

.. code-block:: bash

   openstack overcloud upgrade run --roles Controller --skip-tags validation

openstack overcloud external-upgrade run (for services)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This step is only necessary a service using an external installer was
deployed in the Overcloud. Most typically this is the case of
overclouds with Ceph.

.. admonition:: Pike to Queens
   :class: ptoq

   Among the services with external installers, only upgrade of Ceph
   is supported in the Queens release cycle. It has a specific
   `ceph-upgrade` command. Run it as follows:

   .. note::

      It is especially important to remember that you **must** include all
      environment files that were used to deploy the overcloud.

   .. code-block:: bash

      openstack overcloud ceph-upgrade run --templates \
        -r /path/to/roles_data.yaml \
        -n /path/to/network_data.yaml \
        -e <ALL Templates from overcloud-deploy.sh> \
        -e containers-prepare-parameter.yaml

.. admonition:: Queens to Rocky
   :class: qtor

   More services with external installers can be upgraded to
   Rocky. The `external-upgrade run` command accepts a ``--tags``
   parameter which allows to limit the scope of the upgrade to
   particular services. It is recommended to always use this
   parameter for accurately scoping the upgrade.

   For example, to upgrade Ceph, run the following command:

   .. code-block:: bash

      openstack overcloud external-upgrade run --tags ceph

   .. note::

      The `external-upgrade run` command does not update the Heat
      stack, and as such it does not accept any environment files as
      parameters. It uses playbooks generated during `upgrade
      prepare`.

openstack overcloud external-upgrade run (for online upgrades)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. admonition:: Queens to Rocky
   :class: qtor

   The offline (downtime inducing) part of upgrade has finished at this
   point, and the cloud should be fully operational. Some services have
   an online component to their upgrade procedure -- operations which
   don't induce downtime and can run while the cloud operates
   normally. For OpenStack services these are e.g. online data
   migrations. Run all these online upgrade operations by executing the
   following command:

   .. code-block:: bash

      openstack overcloud external-upgrade run --tags online_upgrade

   .. note::

      If desired, the online upgrades can be run per-service. E.g. to run
      only Nova online data migrations, execute:

      .. code-block:: bash

         openstack overcloud external-upgrade run --tags online_upgrade_nova

      However, when executing online upgrades in selective parts like
      this, extra care must be taken to not miss any necessary online
      upgrade operations.

openstack overcloud upgrade converge
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Finally, run the upgrade converge step. This will re-apply all
configuration across all nodes and unset all variables that were used
during the upgrade. Successful completion of this step is required to
assert that the overcloud state is in sync with the latest TripleO
Heat templates, which is a prerequisite for any further overcloud
management (e.g. scaling).

.. note::

   It is especially important to remember that you **must** include
   all environment files that were used to deploy the overcloud,
   including the new container image parameter file. You should
   omit any repo switch commands and ensure that none of the
   environment files you are about to use is specifying a value for
   UpgradeInitCommand.

.. code-block:: bash

   openstack overcloud upgrade converge --templates
     -r /path/to/roles_data.yaml \
     -n /path/to/network_data.yaml \
     -e <ALL Templates from overcloud-deploy.sh> \
     -e containers-prepare-parameter.yaml

Successful completion of the `upgrade converge` command concludes the
major version upgrade.

Upgrading the Overcloud to Ocata or Pike
----------------------------------------

As of the Ocata release, the upgrades workflow in tripleo has changed
significantly to accommodate the operators' new ability to deploy custom roles
with the Newton release (see the Composable Service Upgrade spec_ for more
info). The new workflow uses ansible upgrades tasks to define the upgrades
workflow on a per-service level. The Pike release upgrade uses a similar
mechanism and the steps are invoked with the same cli. A big difference however
is that after upgrading to Pike most of the overcloud services will be running
in containers.

.. note::

   Upgrades to Pike and further will only be tested with containers. Baremetal
   deployments, which don't use containers, will be deprecated in Queens and
   have full support removed in Rocky.

The operator starts the upgrade with a ``openstack overcloud deploy`` that
includes the major-upgrade-composable-steps.yaml_ environment file (or the
docker variant for the `containerized upgrade to Pike`__)
as well as all environment files used on the initial deployment. This will
collect the ansible upgrade tasks for all roles, except those that have the
``disable_upgrade_deployment`` flag set ``True`` in roles_data.yaml_. The
tasks will be executed in a series of steps, for example (and not limited to):
step 0 for validations or other pre-upgrade tasks, step 1 to stop the
pacemaker cluster, step 2 to stop services, step 3 for package updates,
step 4 for cluster startup, step 5 for any special case db syncs or post
package update migrations. The Pike upgrade tasks are in general much simpler
than those used in Ocata since for Pike these tasks are mainly for stopping
and disabling the systemd services, since they will be containerized as part
of the upgrade.

After the ansible tasks have run the puppet (or docker, for Pike containers)
configuration is also applied in the 'normal' manner we do on an initial
deploy, to complete the upgrade and bring services back up, or start the
service containers, as the case may be for Ocata or Pike.

For those roles with the ``disable_upgrade_deployment`` flag set True, the
operator will upgrade the corresponding nodes with the
upgrade-non-controller.sh_. The operator uses that script to invoke the
tripleo_upgrade_node.sh_ which is delivered during the
major-upgrade-composable-steps that come first, as described above.

#. Run the major upgrade composable ansible steps

   This step will upgrade the nodes of all roles that do not explicitly set the
   ``disable_upgrade_deployment`` flag to ``True`` in the roles_data.yaml_
   (this is an operator decision, and the current default is for the **Compute**
   and **ObjectStorage** roles to have this set).

   The ansible upgrades tasks are collected from all service manifests_ and
   executed in a series of steps as described in the introduction above.
   Even before the invocation of these ansible tasks however, this upgrade
   step also delivers the tripleo_upgrade_node.sh_ and role specific puppet
   manifest to allow the operator to upgrade those nodes after this step has
   completed.

   From Ocata to Pike, the Overcloud will be upgraded to a containerized
   environment. All OpenStack related services will run in containers.

   If you deploy TripleO with custom roles, you want to synchronize them with
   `roles_data.yaml` visible in default roles and make sure parameters and new
   services are present in your roles.

   .. admonition:: Newton
      :class: newton

      Newton roles_data.yaml is available here:
      https://github.com/openstack/tripleo-heat-templates/blob/stable/newton/roles_data.yaml

   .. admonition:: Ocata
      :class: ocata

      Ocata roles_data.yaml is available here:
      https://github.com/openstack/tripleo-heat-templates/blob/stable/ocata/roles_data.yaml

   .. admonition:: Pike
      :class: pike

      Pike roles_data.yaml is available here:
      https://github.com/openstack/tripleo-heat-templates/blob/stable/pike/roles_data.yaml

   .. admonition:: Queens
      :class: queens

      Queens roles_data.yaml is available here:
      https://github.com/openstack/tripleo-heat-templates/blob/stable/queens/roles_data.yaml


   Create an environment file with commands to switch OpenStack repositories to
   a new release. This will likely be the same commands that were used to switch
   repositories on the undercloud

    .. code-block:: bash

       cat > overcloud-repos.yaml <<EOF
       parameter_defaults:
         UpgradeInitCommand: |
           set -e
           # REPOSITORY SWITCH COMMANDS GO HERE
       EOF

   .. admonition:: Newton to Ocata
      :class: ntoo

      Run ``overcloud deploy``, passing in full set of environment files plus
      `major-upgrade-composable-steps.yaml` and ``overcloud-repos.yaml``

      .. code-block:: bash

         openstack overcloud deploy --templates \
           -e <full environment> \
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-composable-steps.yaml \
           -e overcloud-repos.yaml

   .. note::

      Before upgrading your deployment to containers, you must perform the
      actions mentioned here to prepare your environment. In particular
      *image prepare* to generate the docker registry which you must include
      as one of the environment files specified below:
      * :doc:`../install/containers_deployment/overcloud`

   .. __:

   Run `overcloud deploy`, passing in full set of environment
   files plus `major-upgrade-composable-steps-docker.yaml` and
   `overcloud-repos.yaml` (and docker registry if upgrading to containers)

   .. code-block:: bash

      openstack overcloud deploy --templates \
        -e <full environment> \
        -e /usr/share/openstack-tripleo-heat-templates/environments/docker.yaml \
        -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-composable-steps-docker.yaml \
        -e overcloud-repos.yaml

   .. note::

      It is especially important to remember that you **must** include all
      environment files that were used to deploy the overcloud that you are about
      to upgrade.

   .. note::

      If the Overcloud has been deployed with Pacemaker, then add the
      `docker-ha.yaml` environment file to the upgrade command

      .. code-block:: bash

         openstack overcloud deploy --templates \
           -e <full environment> \
           -e /usr/share/openstack-tripleo-heat-templates/environments/docker.yaml \
           -e /usr/share/openstack-tripleo-heat-templates/environments/docker-ha.yaml \
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-composable-steps-docker.yaml \
           -e overcloud-repos.yaml

   .. admonition:: Ceph
      :class: ceph

      When upgrading to Pike, if Ceph has been deployed in the Overcloud, then
      use the `ceph-ansible.yaml` environment file **instead of**
      `storage-environment.yaml`. Make sure to move any customization into
      `ceph-ansible.yaml` (or a copy of ceph-ansible.yaml)

      .. code-block:: bash

          openstack overcloud deploy --templates \
            -e <full environment> \
            -e /usr/share/openstack-tripleo-heat-templates/environments/docker.yaml \
            -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
            -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-composable-steps-docker.yaml \
            -e overcloud-repos.yaml

      Customizations for the Ceph deployment previously passed as hieradata
      via \*ExtraConfig should be removed as they are ignored, specifically
      the deployment will stop if ``ceph::profile::params::osds`` is found to
      ensure the devices list has been migrated to the format expected by
      ceph-ansible. It is possible to use the ``CephAnsibleExtraConfig`` and
      ``CephAnsibleDisksConfig`` parameters to pass arbitrary variables to
      ceph-ansible, like ``devices`` and ``dedicated_devices``.  See the
      `ceph-ansible scenarios`_ or the :doc:`TripleO Ceph config guide
      <../install/advanced_deployment/ceph_config>`

      The other parameters (for example ``CinderRbdPoolName``,
      ``CephClientUserName``, ...) will behave as they used to with puppet-ceph
      with the only exception of ``CephPools``. This can be used to create
      additional pools in the Ceph cluster but the two tools expect the list
      to be in a different format. Specifically while puppet-ceph expected it
      in this format::

        {
         "mypool": {
          "size": 1,
          "pg_num": 32,
          "pgp_num": 32
         }
        }

      with ceph-ansible that would become::

        [{"name": "mypool", "pg_num": 32, "rule_name": ""}]

   .. note::

      The first step of the ansible tasks is to validate that the deployment is
      in a good state before performing any other upgrade operations. Each
      service manifest in the tripleo-heat-templates includes a check that it is
      running and if any of those checks fail the upgrade will exit early at
      ansible step 0.

      If you are re-running the upgrade after an initial failed attempt, you may
      need to disable these checks in order to allow the upgrade to proceed with
      services down. This is done with the SkipUpgradeConfigTags parameter to
      specify that tasks with the 'validation' tag should be skipped. You can
      include this in any of the environment files you are using::

         SkipUpgradeConfigTags: [validation]

#. Upgrade remaining nodes for roles with ``disable_upgrade_deployment: True``

   It is expected that the operator will want to upgrade the roles that have the
   ``openstack-nova-compute`` and ``openstack-swift-object`` services deployed
   to allow for pre-upgrade migration of workloads. For this reason the default
   ``Compute`` and ``ObjectStorage`` roles in the roles_data.yaml_ have the
   ``disable_upgrade_deployment`` set ``True``.

   Note that unlike in previous releases, this operator driven upgrade step
   includes a full puppet configuration run as happens after the ansible
   steps on the roles those are executed on. The significance is that nodes
   are 'fully' upgraded after each step completes, rather than having to wait
   for the final converge step as has previously been the case. In the case of
   Ocata to Pike the full puppet/docker config is applied to bring up the
   overclod services in containers.

   The tripleo_upgrade_node.sh_ script and puppet configuration are delivered to
   the nodes with ``disable_upgrade_deployment`` set ``True`` during the initial
   major upgrade composable steps in step 1 above.

   For Ocata to Pike, the tripleo_upgrade_node.sh_ is still delivered to the
   ``disable_upgrade_deployment`` nodes but is now empty. Instead, the
   `upgrade_non_controller.sh` downloads ansible playbooks and those are
   executed to deliver the upgrade. See the Queens-upgrade-spec_ for more
   information on this mechanism.

   To upgrade remaining roles (at your convenience)

   .. code-block:: bash

      upgrade-non-controller.sh --upgrade overcloud-compute-0

      for i in $(seq 0 2); do
        upgrade-non-controller.sh --upgrade overcloud-objectstorage-$i &
      done

#. Converge to unpin Nova RPC

   The final step is required to unpin Nova RPC version. Unlike in previous
   releases, for Ocata the puppet configuration has already been applied to
   nodes as part of each upgrades step, i.e. after the ansible tasks or when
   invoking the tripleo_upgrade_node.sh_ script to upgrade compute nodes. Thus
   the significance of this step is somewhat diminished compared to previously.
   However a re-application of puppet configuration across all nodes here will
   also serve as a sanity check and hopefully show any issues that an operator
   may have missed during any of the previous upgrade steps.

   To converge, run the deploy command with `major-upgrade-converge-docker.yaml`

   .. code-block:: bash

      openstack overcloud deploy --templates \
        -e <full environment> \
        -e /usr/share/openstack-tripleo-heat-templates/environments/docker.yaml \
        -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-converge-docker.yaml

   .. admonition:: Newton to Ocata
      :class: ntoo

      For Newton to Ocata, run the deploy command with
      `major-upgrade-pacemaker-converge.yaml`

      .. code-block:: bash

         openstack overcloud deploy --templates \
           -e <full environment> \
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-pacemaker-converge.yaml

   .. note::

      If the Overcloud has been deployed with Pacemaker, then add the
      `docker-ha.yaml` environment file to the upgrade command

      .. code-block:: bash

         openstack overcloud deploy --templates \
           -e <full environment> \
           -e /usr/share/openstack-tripleo-heat-templates/environments/docker.yaml \
           -e /usr/share/openstack-tripleo-heat-templates/environments/docker-ha.yaml \
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-converge-docker.yaml

         openstack overcloud deploy --templates \
           -e <full environment> \
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-converge.yaml

   .. note::

      It is especially important to remember that you **must** include all
      environment files that were used to deploy the overcloud.

.. _spec: https://specs.openstack.org/openstack/tripleo-specs/specs/ocata/tripleo-composable-upgrades.html
.. _major-upgrade-composable-steps.yaml: https://github.com/openstack/tripleo-heat-templates/blob/master/environments/major-upgrade-composable-steps.yaml
.. _roles_data.yaml: https://github.com/openstack/tripleo-heat-templates/blob/master/roles_data.yaml
.. _tripleo_upgrade_node.sh: https://github.com/openstack/tripleo-heat-templates/blob/master/extraconfig/tasks/tripleo_upgrade_node.sh
.. _upgrade-non-controller.sh: https://github.com/openstack/tripleo-common/blob/master/scripts/upgrade-non-controller.sh
.. _manifests: https://github.com/openstack/tripleo-heat-templates/tree/master/puppet/services
.. _Queens-upgrade-spec: https://specs.openstack.org/openstack/tripleo-specs/specs/queens/tripleo_ansible_upgrades_workflow.html
.. _ceph-ansible scenarios: https://github.com/ceph/ceph-ansible/blob/stable-3.0/docs/source/testing/scenarios.rst


Upgrading the Overcloud to Newton and earlier
---------------------------------------------

.. note::

   The `openstack overcloud deploy` calls in upgrade steps below are
   non-blocking. Make sure that the overcloud is `UPDATE_COMPLETE` in
   `openstack stack list` and `sudo pcs status` on a controller reports
   everything running fine before proceeding to the next step.

.. admonition:: Mitaka to Newton
   :class: mton

   **Deliver the migration for ceilometer to run under httpd.**

   This is to deliver the migration for ceilometer to be run under httpd (apache)
   rather than eventlet as was the case before. To execute this step run
   `overcloud deploy`, passing in the full set of environment files plus
   `major-upgrade-ceilometer-wsgi-mitaka-newton.yaml`

   .. code-block:: bash

      openstack overcloud deploy --templates \
        -e <full environment> \
        -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-ceilometer-wsgi-mitaka-newton.yaml

#. Upgrade initialization

   The initialization step switches to new repositories on overcloud nodes, and
   it delivers upgrade scripts to nodes which are going to be upgraded
   one-by-one (this means non-controller nodes, except any stand-alone block
   storage nodes).

   Create an environment file with commands to switch OpenStack repositories to
   a new release. This will likely be the same commands that were used to
   switch repositories on the undercloud

   .. code-block:: bash

      cat > overcloud-repos.yaml <<EOF
      parameter_defaults:
        UpgradeInitCommand: |
          set -e
          # REPOSITORY SWITCH COMMANDS GO HERE
      EOF


   And run `overcloud deploy`, passing in full set of environment files plus
   `major-upgrade-pacemaker-init.yaml` and `overcloud-repos.yaml`

   .. code-block:: bash

      openstack overcloud deploy --templates \
        -e <full environment> \
        -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-pacemaker-init.yaml \
        -e overcloud-repos.yaml

#. Object storage nodes upgrade

   If the deployment has any standalone object storage nodes, upgrade them
   one-by-one using the `upgrade-non-controller.sh` script on the undercloud
   node

   .. code-block:: bash

      upgrade-non-controller.sh --upgrade <nova-id of object storage node>

   This is ran before controller node upgrade because swift storage services
   should be upgraded before swift proxy services.

#. Upgrade controller and block storage nodes

   .. admonition:: Mitaka to Newton
      :class: mton

      **Explicitly disable sahara services if so desired:**
      As discussed at bug1630247_  sahara services are disabled by default in
      the Newton overcloud deployment. This special case is handled for the
      duration of the upgrade by defaulting to 'keep sahara-\*'.

      That is by default sahara services are restarted after the mitaka to
      newton upgrade of controller nodes and sahara config is re-applied during
      the final upgrade converge step.

      If an operator wishes to **disable** sahara services as part of the
      mitaka to newton upgrade they need to include the
      major-upgrade-remove-sahara.yaml_ environment file during the controller
      upgrade step as well as during the converge step later

      .. code-block:: bash

         openstack overcloud deploy --templates \
           -e <full environment> \
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-pacemaker.yaml
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-remove-sahara.yaml

   All controllers will be upgraded in sync in order to make services only talk
   to DB schema versions they expect. Services will be unavailable during this
   operation. Standalone block storage nodes are automatically upgraded in this
   step too, in sync with controllers, because block storage services don't
   have a version pinning mechanism.

   Run the deploy command with `major-upgrade-pacemaker.yaml`

   .. code-block:: bash

      openstack overcloud deploy --templates \
        -e <full environment> \
        -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-pacemaker.yaml

   Services of the compute component on the controller nodes are now pinned to
   communicate like the older release, ensuring that they can talk to the
   compute nodes which haven't been upgraded yet.

   .. note::

      If this step fails, it may leave the pacemaker cluster stopped (together
      with all OpenStack services on the controller nodes). The root cause and
      restoration procedure may vary, but in simple cases the pacemaker cluster
      can be started by logging into one of the controllers and running ``sudo
      pcs cluster start --all``.

   .. note::

      After this step, or if this step failed with the error: `ERROR: upgrade
      cannot start with some cluster nodes being offlineAfter`, it's possible
      that some pacemaker resources needs to be clean. Check the failed
      actions and clean them by running on `only one` controller node as root

      .. code-block:: bash

         pcs status
         pcs resource cleanup

      It can take few minutes for the cluster to go back to a “normal” state as
      displayed by `crm_mon`.  This is expected.

#. Upgrade ceph storage nodes

   If the deployment has any ceph storage nodes, upgrade them one-by-one using
   the `upgrade-non-controller.sh` script on the undercloud node

   .. code-block:: bash

      upgrade-non-controller.sh --upgrade <nova-id of ceph storage node>

#. Upgrade compute nodes

   Upgrade compute nodes one-by-one using the `upgrade-non-controller.sh`
   script on the undercloud node

   .. code-block:: bash

      upgrade-non-controller.sh --upgrade <nova-id of compute node>

#. Apply configuration from upgraded tripleo-heat-templates

   .. admonition:: Mitaka to Newton
      :class: mton

      **Explicitly disable sahara services if so desired:**
      As discussed at bug1630247_  sahara services are disabled by default in
      the Newton overcloud deployment. This special case is handled for the
      duration of the upgrade by defaulting to 'keep sahara-\*'.

      That is by default sahara services are restarted after the mitaka to
      newton upgrade of controller nodes and sahara config is re-applied during
      the final upgrade converge step.

      If an operator wishes to **disable** sahara services as part of the
      mitaka to newton upgrade they need to include the
      major-upgrade-remove-sahara.yaml_ environment file during the controller
      upgrade earlier and converge step here

      .. code-block:: bash

         openstack overcloud deploy --templates \
           -e <full environment> \
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-pacemaker-converge.yaml
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-remove-sahara.yaml

   .. _bug1630247: https://bugs.launchpad.net/tripleo/+bug/1630247
   .. _major-upgrade-remove-sahara.yaml: https://github.com/openstack/tripleo-heat-templates/blob/2e6cc07c1a74c2dd7be70568f49834bace499937/environments/major-upgrade-remove-sahara.yaml


   This step unpins compute services communication (upgrade level) on
   controller and compute nodes, and it triggers configuration management
   tooling to converge the overcloud configuration according to the new release
   of `tripleo-heat-templates`.

   Make sure that all overcloud nodes have been upgraded to the new release,
   and then run the deploy command with `major-upgrade-pacemaker-converge.yaml`

   .. code-block:: bash

      openstack overcloud deploy --templates \
        -e <full environment> \
        -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-pacemaker-converge.yaml


   .. note::

      After the converge step, it's possible that some pacemaker resources
      needs to be cleaned.  Check the failed actions and clean them by running
      on **only one** controller as root

      .. code-block:: bash

         pcs status
         pcs resource cleanup

      It can take few minutes for the cluster to go back to a “normal” state as
      displayed by ``crm_mon``. This is expected.


