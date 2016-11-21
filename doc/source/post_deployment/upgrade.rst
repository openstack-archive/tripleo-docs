Upgrading to a Next Major Release
=================================

Upgrading a TripleO deployment to a next major release is done by
first upgrading the undercloud, and then upgrading the overcloud.

Note that there are version specific caveats and notes which are pointed out as below:

.. admonition:: Liberty to Mitaka
   :class: ltom

   liberty to mitaka specific note

.. admonition:: Mitaka to Newton
   :class: mton

   mitaka to newton specific note

.. note::

   You can use the "Limit Environment Specific Content" in the left hand nav
   bar to restrict content to the upgrade you are performing.

.. note::

   Generic upgrade testing cannot cover all possible deployment
   configurations. Before performing the upgrade in production, test
   it in a matching staging environment, and create a backup of the
   production environment.


Upgrading the Undercloud
------------------------

1. Disable the old OpenStack release repositories and enable new
   release repositories on the undercloud:

  .. admonition:: Liberty to Mitaka
     :class: ltom

      ::

            export CURRENT_VERSION=liberty
            export NEW_VERSION=mitaka

  .. admonition:: Mitaka to Newton
     :class: mton

      ::

            export CURRENT_VERSION=mitaka
            export NEW_VERSION=newton

  Backup and disable current repos. Note that the repository files might be
  named differently depending on your installation::

        mkdir /home/stack/REPOBACKUP
        sudo mv /etc/yum.repos.d/delorean* /home/stack/REPOBACKUP/

  Get and enable new repos for `NEW_VERSION`:

.. include:: ../repositories.txt

2. Run undercloud upgrade:

   .. admonition:: Liberty to Mitaka
      :class: ltom

       For liberty to mitaka upgrades we need to manually update
       mariadb including a database backup/restore::

          mysqldump -u root --flush-privileges --single-transaction --all-databases > /home/stack/backup.sql
          sudo systemctl stop mariadb
          sudo mv /var/lib/mysql /home/stack/mysql-backup

          sudo yum -y update mariadb

          sudo systemctl start mariadb
          mysql -u root < /home/stack/backup.sql

   .. admonition:: Mitaka to Newton
      :class: mton

       In the first release of instack-undercloud newton(5.0.0), the undercloud
       telemetry services are **disabled** by default. In order to maintain the
       telemetry services during the mitaka to newton upgrade the operator must
       explicitly enable them **before** running the undercloud upgrade. This
       is done by adding::

          enable_telemetry = true

       in the [DEFAULT] section of the undercloud.conf configuration file.

       If you are using any newer newton release, this option is switched back
       to **enabled** by default to make upgrade experience better. Hence, if
       you are using a later newton release you don't need to explicitly enable
       this option.

   The following command will upgrade the undercloud::

      sudo systemctl stop openstack-*
      sudo systemctl stop neutron-*
      sudo yum -y update instack-undercloud openstack-puppet-modules openstack-tripleo-common python-tripleoclient
      openstack undercloud upgrade

   Once the undercloud upgrade is fully completed you may
   remove the older mysql backup folder /home/stack/mysql-backup

.. note::

            You may wish to use time and capture the output to a file for any debug::

                time openstack undercloud upgrade 2>&1 | tee undercloud_upgrade.log

.. note::

   If you added custom OVS ports to the undercloud (e.g. in a virtual
   testing environment) you may need to re-add them at this point.


Upgrading the Overcloud
-----------------------

.. note::

   The `openstack overcloud deploy` calls in upgrade steps below are
   non-blocking. Make sure that the overcloud is `UPDATE_COMPLETE` in
   `openstack stack list` and `sudo pcs status` on a controller
   reports everything running fine before proceeding to the next step.

.. admonition:: Liberty to Mitaka
   :class: ltom

    **Create the new CephClientKey**

    If using a TripleO managed Ceph deployment, a new key for the
    "client.openstack" CephX user needs to be provided. A sample
    environment file would look like the following::

      parameter_defaults:
        CephClientKey: 'my_cephx_key'

    A proper value for the key parameter can be generated from any of
    the overcloud nodes with::

      $ ceph-authtool --gen-print-key

.. admonition:: Liberty to Mitaka
   :class: ltom


    **Deliver the aodh migration.**

    For Liberty to Mitaka we need to run an extra step in the upgrades workflow
    after the upgrade initialisation.

    This is to deliver the migration from ceilometer-alarms to aodh. To execute
    this step run `overcloud deploy`, passing in the full set of environment
    files plus `major-upgrade-aodh.yaml`. Note that the `--force-postconfig`
    switch is needed in order to add the newly created aodh endpoint::

      openstack overcloud deploy --templates \
          -e <full environment> \
          -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-aodh.yaml \
          --force-postconfig

.. admonition:: Liberty to Mitaka
   :class: ltom


    **Deliver the migration for keystone to run under httpd.**

    For Liberty to Mitaka we need to run an extra step in the upgrades workflow
    after the aodh migration.

    This is to deliver the migration for keystone to be run under httpd (apache)
    rather than eventlet as was the case before. To execute this step run
    `overcloud deploy`, passing in the full set of environment files plus
    `major-upgrade-keystone-liberty-mitaka.yaml`::

      openstack overcloud deploy --templates \
          -e <full environment> \
          -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-keystone-liberty-mitaka.yaml

.. admonition:: Mitaka to Newton
   :class: mton


    **Deliver the migration for ceilometer to run under httpd.**

    This is to deliver the migration for ceilometer to be run under httpd (apache)
    rather than eventlet as was the case before. To execute this step run
    `overcloud deploy`, passing in the full set of environment files plus
    `major-upgrade-ceilometer-wsgi-mitaka-newton.yaml`::

      openstack overcloud deploy --templates \
          -e <full environment> \
          -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-ceilometer-wsgi-mitaka-newton.yaml

#. Upgrade initialization

   The initialization step switches to new repositories on overcloud
   nodes, and it delivers upgrade scripts to nodes which are going to
   be upgraded one-by-one (this means non-controller nodes, except any
   stand-alone block storage nodes).

   Create an environment file with commands to switch OpenStack
   repositories to a new release. This will likely be the same
   commands that were used to switch repositories on the undercloud::

      cat > overcloud-repos.yaml <<EOF
      parameter_defaults:
        UpgradeInitCommand: |
          set -e
          # REPOSITORY SWITCH COMMANDS GO HERE
      EOF

   And run `overcloud deploy`, passing in full set of environment
   files plus `major-upgrade-pacemaker-init.yaml` and
   `overcloud-repos.yaml`::

      openstack overcloud deploy --templates \
          -e <full environment> \
          -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-pacemaker-init.yaml \
          -e overcloud-repos.yaml


#. Object storage nodes upgrade

   If the deployment has any standalone object storage nodes, upgrade
   them one-by-one using the `upgrade-non-controller.sh` script on the
   undercloud node::

      upgrade-non-controller.sh --upgrade <nova-id of object storage node>

   This is ran before controller node upgrade because swift storage
   services should be upgraded before swift proxy services.

#. Upgrade controller and block storage nodes


   .. admonition:: Mitaka to Newton
      :class: mton

       **Explicitly disable sahara services if so desired:**
       As discussed at bug1630247_  sahara services are disabled by default
       in the Newton overcloud deployment. This special case is handled for
       the duration of the upgrade by defaulting to 'keep sahara-*'.

       That is by default sahara services are restarted after the mitaka to
       newton upgrade of controller nodes and sahara config is re-applied
       during the final upgrade converge step.

       If an operator wishes to **disable** sahara services as part of the mitaka
       to newton upgrade they need to include the major-upgrade-remove-sahara.yaml_
       environment file during the controller upgrade step as well as during
       the converge step later::

          openstack overcloud deploy --templates \
           -e <full environment> \
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-pacemaker.yaml
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-remove-sahara.yaml

   All controllers will be upgraded in sync in order to make services
   only talk to DB schema versions they expect. Services will be
   unavailable during this operation. Standalone block storage nodes
   are automatically upgraded in this step too, in sync with
   controllers, because block storage services don't have a version
   pinning mechanism.

   Run the deploy command with `major-upgrade-pacemaker.yaml`::

      openstack overcloud deploy --templates \
          -e <full environment> \
          -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-pacemaker.yaml

   Services of the compute component on the controller nodes are now
   pinned to communicate like the older release, ensuring that they
   can talk to the compute nodes which haven't been upgraded yet.

   .. note::

      If this step fails, it may leave the pacemaker cluster stopped
      (together with all OpenStack services on the controller
      nodes). The root cause and restoration procedure may vary, but
      in simple cases the pacemaker cluster can be started by logging
      into one of the controllers and running `sudo pcs cluster start
      --all`.

#. Upgrade ceph storage nodes

   If the deployment has any ceph storage nodes, upgrade them
   one-by-one using the `upgrade-non-controller.sh` script on the
   undercloud node::

      upgrade-non-controller.sh --upgrade <nova-id of ceph storage node>

#. Upgrade compute nodes

   Upgrade compute nodes one-by-one using the
   `upgrade-non-controller.sh` script on the undercloud node::

      upgrade-non-controller.sh --upgrade <nova-id of compute node>

#. Apply configuration from upgraded tripleo-heat-templates

   .. admonition:: Mitaka to Newton
      :class: mton

       **Explicitly disable sahara services if so desired:**
       As discussed at bug1630247_  sahara services are disabled by default
       in the Newton overcloud deployment. This special case is handled for
       the duration of the upgrade by defaulting to 'keep sahara-*'.

       That is by default sahara services are restarted after the mitaka to
       newton upgrade of controller nodes and sahara config is re-applied
       during the final upgrade converge step.

       If an operator wishes to **disable** sahara services as part of the mitaka
       to newton upgrade they need to include the major-upgrade-remove-sahara.yaml_
       environment file during the controller upgrade earlier and converge
       step here::

          openstack overcloud deploy --templates \
           -e <full environment> \
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-pacemaker-converge.yaml
           -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-remove-sahara.yaml

   .. _bug1630247: https://bugs.launchpad.net/tripleo/+bug/1630247
   .. _major-upgrade-remove-sahara.yaml: https://github.com/openstack/tripleo-heat-templates/blob/2e6cc07c1a74c2dd7be70568f49834bace499937/environments/major-upgrade-remove-sahara.yaml



   This step unpins compute services communication (upgrade level) on
   controller and compute nodes, and it triggers configuration
   management tooling to converge the overcloud configuration
   according to the new release of `tripleo-heat-templates`.

   Make sure that all overcloud nodes have been upgraded to the new
   release, and then run the deploy command with
   `major-upgrade-pacemaker-converge.yaml`::


      openstack overcloud deploy --templates \
          -e <full environment> \
          -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-pacemaker-converge.yaml

.. admonition:: Mitaka to Newton
   :class: mton


    **Deliver the data migration for aodh.**

    This is to deliver the data migration for aodh. In Newton, aodh uses its
    own mysql backend. This step migrates all the existing alarm data from
    mongodb to the new mysql backend. To execute this step run
    `overcloud deploy`, passing in the full set of environment files plus
    `major-upgrade-aodh-migration.yaml`::

      openstack overcloud deploy --templates \
          -e <full environment> \
          -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-aodh-migration.yaml
