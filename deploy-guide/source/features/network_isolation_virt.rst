Configuring Network Isolation in Virtualized Environments
=========================================================

Introduction
------------

This document describes how to configure a virtualized development
environment for use with network isolation. To make things as easy as
possible we will use the ``single-nic-with-vlans`` network isolation
templates to create isolated VLANs on top of the single NIC already
used for the provisioning/``ctlplane``.

The ``single_nic_vlans.j2`` template work well for many virtualized environments
because they do not require adding any extra NICs. Additionally, Open vSwitch
automatically trunks VLANs for us, so there is no extra switch configuration
required.

Create an External VLAN on Your Undercloud
------------------------------------------

By default all instack undercloud machines have a ``br-ctlplane`` which
is used as the provisioning network. We want to add an interface
on the 10.0.0.0/24 network which is used as the default "external"
(public) network for the overcloud. The default VLAN for the external
network is ``vlan10`` so we create an interface file to do this. Create
the following file ``/etc/sysconfig/network-scripts/ifcfg-vlan10``::

  DEVICE=vlan10
  ONBOOT=yes
  HOTPLUG=no
  TYPE=OVSIntPort
  OVS_BRIDGE=br-ctlplane
  OVS_OPTIONS="tag=10"
  BOOTPROTO=static
  IPADDR=10.0.0.1
  PREFIX=24
  NM_CONTROLLED=no

And then run ``ifup vlan10`` on your undercloud.

Create a Custom Environment File
--------------------------------

When using network isolation most of the network/config templates configure
static IPs for the ``ctlplane``. To ensure connectivity with Heat and Ec2
metadata, we need to specify a couple of extra Heat parameters. Create a file
called ``/home/stack/custom.yaml`` with the following contents::

  parameter_defaults:
    EC2MetadataIp: 192.168.24.1
    ControlPlaneDefaultRoute: 192.168.24.1

Note that the specified IP addresses ``192.168.24.1`` are the same as the
undercloud IP address.

Modify Your Overcloud Deploy to Enable Network Isolation
--------------------------------------------------------

At this point we are ready to create the overcloud using the network
isolation defaults. The example command below demonstrates how to enable
network isolation by using Heat templates for network isolation, a
custom set of network config templates (single NIC VLANs), and our
``custom.yaml`` config file from above::

  TEMPLATES=/path/to/openstack-tripleo-heat-templates
  openstack overcloud deploy \
  --templates=$TEMPLATES \
  -e $TEMPLATES/environments/network-isolation.yaml \
  -e $TEMPLATES/environments/net-single-nic-with-vlans.yaml \
  -e /home/stack/custom.yaml

After creating the stack you should now have a working virtualized
development environment with network isolation enabled.
