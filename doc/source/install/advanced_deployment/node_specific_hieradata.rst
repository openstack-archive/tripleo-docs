Provisioning of node-specific Hieradata
=======================================

This guide assumes that your undercloud is already installed and ready to
deploy an overcloud.

It is possible to provide some node-specific hieradata via Heat environment
files and as such customize one or more settings for a specific node,
regardless of the Heat `ResourceGroup` to which it belongs.

As a sample use case, we will distribute a node-specific disks configuration
for a particular CephStorage node, which by default runs the `ceph-osd` service.

Collecting the node UUID
------------------------

The node-specific hieradata is provisioned based on the node UUID, which is
hardware dependent and immutable across reboots/reinstalls.

First make sure the introspection data is available for the target node, if it
isn't one may run introspection for a particular node as described in:
:doc:`introspect_single_node`. If the `undercloud.conf` does not have
`inspection_extras = true` prior to undercloud installation/upgrade
and introspection, then the machine unique UUID will not be in the
Ironic database.

Then extract the machine unique UUID for the target node with a command like::

  openstack baremetal introspection data save NODE-ID | jq .extra.system.product.uuid

where `NODE-ID` is the target node Ironic UUID. The value returned by the above
command will be a unique and immutable machine UUID which isn't related to the
Ironic node UUID. For the next step, we'll assume the output was
`32E87B4C-C4A7-418E-865B-191684A6883B`.

Creating the Heat environment file
----------------------------------

Assuming we want to use `/dev/sdc` as a data disk for `ceph-osd` on our target
node, we'll create a yaml file, e.g. `my-node-settings.yaml`, with the
following content depending on if either ceph-ansible (Pike and newer)
or puppet-ceph (Ocata and older).

For ceph-ansible use::

  parameter_defaults:
    NodeDataLookup: |
      {"32E87B4C-C4A7-418E-865B-191684A6883B": {"devices": ["/dev/sdc"]}}

For puppet-ceph use::

  resource_registry:
    OS::TripleO::CephStorageExtraConfigPre: /path/to/tripleo-heat-templates/puppet/extraconfig/pre_deploy/per_node.yaml

  parameter_defaults:
    NodeDataLookup: |
      {"32E87B4C-C4A7-418E-865B-191684A6883B": {"ceph::profile::params::osds": {"/dev/sdc": {}}}}

In the above example we're customizing only a single key for a single node, but
the structure is that of a UUID-mapped hash so it is possible to customize
multiple and different keys for multiple nodes.

Finally, add such an environment file to the deploy commandline::

  openstack overcloud deploy [other overcloud deploy options] -e ~/my-node-settings.yaml
