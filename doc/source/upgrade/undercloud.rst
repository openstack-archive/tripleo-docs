Updating Undercloud Components
------------------------------

.. note::
   Instack-undercloud is deprecated in Rocky cycle. Instack undercloud can
   only be upgraded to containerized undercloud. See
   :doc:`../install/containers_deployment/undercloud`
   for backward compatibility related information.

.. note::
   When updating the existing containerized undercloud installation,
   keep in mind the special cases described in :ref:`notes-for-stack-updates`.

#. Before upgrading the undercloud, it is highly suggested to perform
   a :doc:`backup <../install/controlplane_backup_restore/01_undercloud_backup>`
   of the undercloud and validate that a restore works fine.

#. Remove all Delorean repositories:

   .. note::

      You may wish to backup your current repos before disabling them

      .. code-block:: bash

         mkdir -p /home/stack/REPOBACKUP
         sudo mv /etc/yum.repos.d/delorean* /home/stack/REPOBACKUP

   .. code-block:: bash

      sudo rm /etc/yum.repos.d/delorean*


#. Enable new Delorean repositories:

   .. include:: ../install/repositories.txt

.. We need to manually continue our list numbering here since the above
  "include" directive breaks the numbering.

#. Clean the yum cache to ensure only the new repos are used

   .. code-block:: bash

      sudo yum clean all
      sudo rm -rf /var/cache/yum

#. Update required package:

   .. admonition:: Validations
      :class: validations

      It is strongly recommended that you validate the state of your undercloud
      before starting any upgrade operations. The tripleo-validations_ repo has
      some 'pre-upgrade' validations that you can execute by following the
      instructions at validations_ to execute the "pre-upgrade" group

      .. code-block:: bash

         mistral execution-get-output $(openstack workflow execution create -f value -c ID tripleo.validations.v1.run_groups '{"group_names": ["pre-upgrade"]}')

   .. admonition:: Newton to Ocata
      :class: ntoo

      The following commands need to be run before the undercloud upgrade::

         sudo systemctl stop openstack-*
         sudo systemctl stop neutron-*
         sudo systemctl stop openvswitch
         sudo systemctl stop httpd
         sudo yum update instack-undercloud openstack-puppet-modules openstack-tripleo-common

   .. admonition:: Ocata to Pike
      :class: otop

      .. admonition:: Ceph
         :class: ceph

         Prior to Pike, TripleO deployed Ceph with puppet-ceph. With the
         Pike release it is possible to use TripleO to deploy Ceph with
         either ceph-ansible or puppet-ceph, though puppet-ceph is
         deprecated. To use ceph-ansible, the CentOS Storage SIG Ceph
         repository must be enabled on the undercloud and the
         ceph-ansible package must then be installed::

            sudo yum install --enablerepo=extras centos-release-ceph-jewel
            sudo yum install ceph-ansible

         Ceph clusters deployed with Ocata via puppet-ceph will be migrated
         so that all of the existing Ceph services are run inside of containers.
         This migration will be managed not by puppet-ceph, but by ceph-ansible,
         which TripleO will use to control updates to the same ceph cluster after
         the Ocata to Pike upgrade.


   Update TripleO CLI and dependencies

   .. code-block:: bash

      sudo yum update python-tripleoclient* openstack-tripleo-common openstack-tripleo-heat-templates

#. As part of the undercloud install, an image registry is configured on port
   `8787`.  This is used to increase reliability of overcloud image pulls, and
   minimise overall network transfers. First it is highly suggested to perform
   a backup of the initial `containers-prepare-parameter.yaml` file. Then
   update the new `containers-prepare-parameter.yaml` file with the same
   modifications made in the initial one::

      openstack tripleo container image prepare default \
        --local-push-destination \
        --output-env-file ~/containers-prepare-parameter.yaml

   .. note::
      This command is available since Rocky.

#. Run the undercloud upgrade command. This command will upgrade all packages
   and use puppet to apply new configuration and restart all OpenStack
   services

   .. code-block:: bash

      openstack undercloud upgrade

   .. note::
       The undercloud is containerized by default as of Rocky. Therefore,
       an undercloud deployed on Queens (non-containerized) will be upgraded
       to a containerized undercloud on Rocky, by default.
       To upgrade with instack undercloud, you'll need to upgrade with
       ``--use-heat=False`` option. Note this isn't tested and not supported.

   .. note::
       It's possible to enable verbose logging with ``--verbose`` option.
       To cleanup an undercloud after its upgrade, you'll need to set
       upgrade_cleanup to True in undercloud.conf. It'll remove the rpms
       that were deployed by instack-undercloud, after the upgrade to a
       containerized undercloud.

   .. note::

      If you added custom OVS ports to the undercloud (e.g. in a virtual
      testing environment) you may need to re-add them at this point.

   .. _validations: ../install/validations/validations.html#running-a-group-of-validations
   .. _tripleo-validations: https://github.com/openstack/tripleo-validations/tree/master/validations
