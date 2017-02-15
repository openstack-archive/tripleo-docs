Updating Undercloud Components
------------------------------

You can upgrade any packages that are installed on the undercloud machine.

#. Remove all Delorean repositories::

       sudo rm /etc/yum.repos.d/delorean*

#. Enable new Delorean repositories:

.. include:: ../repositories.txt

.. We need to manually continue our list numbering here since the above
  "include" directive breaks the numbering.

3. Clean the yum cache to ensure only the new repos are used::

    sudo yum clean all

#. Stop services so that they are not restarted by packaging scripts
   when they are updated. The service restarts will be handled by the
   undercloud upgrade command after new configuration has been applied.::

    sudo systemctl stop openstack-*
    sudo systemctl stop neutron-*
    sudo systemctl stop openvswitch
    sudo systemctl stop httpd

#. Update the TripleO CLI package::

    sudo yum -y update python-tripleoclient

#. Run the undercloud upgrade command. This command will upgrade all packages
   and use puppet to apply new configuration and restart all OpenStack
   services.::

    openstack undercloud upgrade

#. Proceed with :ref:`package_update`.
