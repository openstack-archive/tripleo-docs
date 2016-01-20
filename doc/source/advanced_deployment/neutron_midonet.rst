Configuring MidoNet as a Neutron Backend
========================================

This guide assumes that your undercloud is already installed and ready to deploy
an overcloud

Deploying the overcloud
-----------------------

.. note::
  You need to build the overcloud image with the `overcloud-network-midonet`_
  element to have the midonet packages installed on the ``overcloud-full.qcow2``
  image. If you have that image, you won't be able to run overcloud without
  MidoNet, since MidoNet packages uninstall openvswitch packages.

.. _`overcloud-network-midonet`: ../basic_deployment/basic_deployment_cli.html#build-a-single-image

#. Copy the MidoNet configuration file to your home directory::

    sudo cp /usr/share/openstack-tripleo-heat-templates/environments/neutron-midonet.yaml ~

#. Edit the permissions (user is typically ``stack``)::

    sudo chown $USER ~/neutron-midonet.yaml
    sudo chmod 644 ~/neutron-midonet.yaml

#. Edit the file. There are several commented options that you can configure.
   `DO NOT EDIT THE UNCOMMENTED ONES`

#. Continue following the TripleO instructions for deploying an overcloud.
   Before entering the command to deploy the overcloud, add the environment
   file that you just configured as an argument::

    openstack overcloud deploy --templates -e ~/neutron-midonet.yaml

#. Wait for the completion of the overcloud deployment process.

Using Network Isolation
-----------------------

MidoNet is compatible with `network isolation`_, but you can not configure the
NICs using openvswitch, but with linux bridges. So an example of deployment with
Network Isolation could be::

    openstack overcloud deploy --templates -e ~/neutron-midonet.yaml \
        -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
        -e /usr/share/openstack-tripleo-heat-templates/environments/net-single-nic-linux-bridge-with-vlans.yaml

.. _`network isolation`: ./network_isolation.html
