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
:doc:`../provisioning/introspect_single_node`. If the `undercloud.conf` does not have
`inspection_extras = true` prior to undercloud installation/upgrade
and introspection, then the machine unique UUID will not be in the
Ironic database.

Then extract the machine unique UUID for the target node with a command like::

  openstack baremetal introspection data save NODE-ID | jq .extra.system.product.uuid | tr '[:upper:]' '[:lower:]'

where `NODE-ID` is the target node Ironic UUID. The value returned by the above
command will be a unique and immutable machine UUID which isn't related to the
Ironic node UUID. For the next step, we'll assume the output was
`32e87b4c-c4a7-41be-865b-191684a6883b`.

Creating the Heat environment file
----------------------------------

Assuming we want to use `/dev/sdc` as a data disk for `ceph-osd` on our target
node, we'll create a yaml file, e.g. `my-node-settings.yaml`, with the
following content depending on if either ceph-ansible (Pike and newer)
or puppet-ceph (Ocata and older).

For ceph-ansible use::

  parameter_defaults:
    NodeDataLookup: {"32e87b4c-c4a7-41be-865b-191684a6883b": {"devices": ["/dev/sdc"]}}

For puppet-ceph use::

  resource_registry:
    OS::TripleO::CephStorageExtraConfigPre: /path/to/tripleo-heat-templates/puppet/extraconfig/pre_deploy/per_node.yaml

  parameter_defaults:
    NodeDataLookup: {"32e87b4c-c4a7-41be-865b-191684a6883b": {"ceph::profile::params::osds": {"/dev/sdc": {}}}}

In the above example we're customizing only a single key for a single node, but
the structure is that of a UUID-mapped hash so it is possible to customize
multiple and different keys for multiple nodes.

Generating the Heat environment file for Ceph devices
-----------------------------------------------------

The tools directory of tripleo-heat-templates
(`/usr/share/openstack-tripleo-heat-templates/tools/`) contains a
utility called `make_ceph_disk_list.py` which can be used to create
a valid JSON Heat environment file automatically from Ironic's
introspection data.

Export the introspection data from Ironic for the Ceph nodes to be
deployed::

  openstack baremetal introspection data save oc0-ceph-0 > ceph0.json
  openstack baremetal introspection data save oc0-ceph-1 > ceph1.json
  ...

Copy the utility to the stack user's home directory on the undercloud
and then use it to generate a `node_data_lookup.json` file which may
be passed during openstack overcloud deployment::

  ./make_ceph_disk_list.py -i ceph*.json -o node_data_lookup.json -k by_path

Pass the introspection data file from `openstack baremetal
introspection data save` for all nodes hosting Ceph OSDs to the
utility as you may only define `NodeDataLookup` once during a
deployment. The `-i` option can take an expression like `*.json` or a
list of files as input.

The `-k` option defines the key of ironic disk data structure to use
to identify the disk to be used as an OSD. Using `name` is not
recommended as it will produce a file of devices like `/dev/sdd` which
may not always point to the same device on reboot. Thus, `by_path` is
recommended and is the default if `-k` is not specified.

Ironic will have one of the available disks on the system reserved as
the root disk. The utility will always exclude the root disk from the
list of devices generated.

Use `./make_ceph_disk_list.py --help` to see other available options.

Deploying with NodeDataLookup
-----------------------------

Add the environment file described in the previous section to the
deploy commandline::

  openstack overcloud deploy [other overcloud deploy options] -e ~/my-node-settings.yaml

or::

  openstack overcloud deploy [other overcloud deploy options] -e ~/node_data_lookup.json

JSON is the recommended format (instead of JSON embedded in YAML)
because you may use `jq` to validate the entire file before deployment.
