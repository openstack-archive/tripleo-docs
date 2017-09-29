Updating Undercloud Components
------------------------------

You can upgrade any packages that are installed on the undercloud machine.

#. Remove all Delorean repositories:

   .. note::

     You may wish to backup your current repos before disabling them::

         mkdir /home/stack/REPOBACKUP
         sudo mv /etc/yum.repos.d/delorean* /home/stack/REPOBACKUP

   ::

     sudo rm /etc/yum.repos.d/delorean*


#. Enable new Delorean repositories:

   .. include:: ../repositories.txt

.. We need to manually continue our list numbering here since the above
  "include" directive breaks the numbering.

3. Clean the yum cache to ensure only the new repos are used::

    sudo yum clean all

#. Update required package:

   .. admonition:: Validations
      :class: validations

      It is strongly recommended that you validate the state of your undercloud
      before starting any upgrade operations. The tripleo-validations_ repo has
      some 'pre-upgrade' validations that you can execute by following the
      instructions at validations_ to execute the "pre-upgrade" group::

          mistral execution-get-output $(openstack workflow execution create -f value -c ID tripleo.validations.v1.run_groups '{"group_names": ["pre-upgrade"]}')

   .. admonition:: Newton to Ocata
      :class: ntoo

      The following commands need to be run before the undercloud upgrade::

         sudo systemctl stop openstack-*
         sudo systemctl stop neutron-*
         sudo systemctl stop openvswitch
         sudo systemctl stop httpd
         sudo yum -y update instack-undercloud openstack-puppet-modules openstack-tripleo-common

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

            sudo yum -y install --enablerepo=extras centos-release-ceph-jewel
            sudo yum -y install ceph-ansible


   Update TripleO CLI package::

      sudo yum -y update python-tripleoclient


#. Run the undercloud upgrade command. This command will upgrade all packages
   and use puppet to apply new configuration and restart all OpenStack
   services::

      openstack undercloud upgrade

   .. note::

      You may wish to use time and capture the output to a file for any debug::

        time openstack undercloud upgrade 2>&1 | tee undercloud_upgrade.log

   .. note::

      If you added custom OVS ports to the undercloud (e.g. in a virtual
      testing environment) you may need to re-add them at this point.

   .. _validations: ../validations/validations.html#running-a-group-of-validations
   .. _tripleo-validations: https://github.com/openstack/tripleo-validations/tree/master/validations
