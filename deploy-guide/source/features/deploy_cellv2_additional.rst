Additional cell considerations and features
===========================================

.. warning::
  Multi cell support is only supported in Stein or later versions.

.. contents::
  :depth: 3
  :backlinks: none

.. _cell_availability_zone:

Availability Zones (AZ)
-----------------------
A nova AZ must be configured for each cell to make sure instances stay in the
cell when performing migration and to be able to target a cell when an instance
gets created. The central cell must also be configured as a specific AZs
(or multiple AZs) rather than the default.

Configuring AZs for Nova (compute)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
It's also possible to configure the AZ for a compute node by adding it to a
host aggregate after the deployment is completed. The following commands show
creating a host aggregate, an associated AZ, and adding compute nodes to a
`cell-1` AZ:

.. code-block:: bash

  source overcloudrc
  openstack aggregate create cell1 --zone cell1
  openstack aggregate add host cell1 hostA
  openstack aggregate add host cell1 hostB

.. note::

  Right now we can not use `OS::TripleO::Services::NovaAZConfig` to auto
  create the AZ during the deployment as at this stage the initial cell
  creation is not complete. Further work is needed to fully automate the
  post cell creation steps before `OS::TripleO::Services::NovaAZConfig`
  can be used.


Routed networks
---------------

A routed spine and leaf networking layout can be used to deploy the additional
cell nodes in a distributed nature. Not all nodes need to be co-located at the
same physical location or datacenter. See :ref:`routed_spine_leaf_network` for
more details.

Reusing networks from an already deployed stack
-----------------------------------------------
When deploying separate stacks it may be necessary to reuse networks, subnets,
and VIP resources between stacks if desired. Only a single Heat stack can own a
resource and be responsible for its creation and deletion, however the
resources can be reused in other stacks.

Usually the internal api network in case of split cell controller and cell
compute stacks are shared.

To reuse network related resources between stacks, the following parameters
have been added to the network definitions in the `network_data.yaml` file
format:

.. code-block:: bash

  external_resource_network_id: Existing Network UUID
  external_resource_subnet_id: Existing Subnet UUID
  external_resource_segment_id: Existing Segment UUID
  external_resource_vip_id: Existing VIP UUID

These parameters can be set on each network definition in the
`network_data.yaml` file used for the deployment of the separate stack.

Not all networks need to be reused or shared across stacks. The
`external_resource_*` parameters can be set for only the networks that are
meant to be shared, while the other networks can be newly created and managed.

For example, to reuse the `internal_api` network from the cell controller stack
in the compute stack, run the following commands to show the UUIDs for the
related network resources:

.. code-block:: bash

  openstack network show internal_api -c id -f value
  openstack subnet show internal_api_subnet -c id -f value
  openstack port show internal_api_virtual_ip -c id -f value

Save the values shown in the output of the above commands and add them to the
network definition for the `internal_api` network in the `network_data.yaml`
file for the separate stack.

In case the overcloud and the cell controller stack uses the same internal
api network there are two ports with the name `internal_api_virtual_ip`.
In this case it is required to identify the correct port and use the id
instead of the name in the `openstack port show` command.

An example network definition would look like:

.. code-block:: bash

  - name: InternalApi
    external_resource_network_id: 93861871-7814-4dbc-9e6c-7f51496b43af
    external_resource_subnet_id: c85c8670-51c1-4b17-a580-1cfb4344de27
    external_resource_vip_id: 8bb9d96f-72bf-4964-a05c-5d3fed203eb7
    name_lower: internal_api
    vip: true
    ip_subnet: '172.16.2.0/24'
    allocation_pools: [{'start': '172.16.2.4', 'end': '172.16.2.250'}]
    ipv6_subnet: 'fd00:fd00:fd00:2000::/64'
    ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:2000::10', 'end': 'fd00:fd00:fd00:2000:ffff:ffff:ffff:fffe'}]
    mtu: 1400

.. note::

  When *not* sharing networks between stacks, each network defined in
  `network_data.yaml` must have a unique name across all deployed stacks.
  This requirement is necessary since regardless of the stack, all networks are
  created in the same tenant in Neutron on the undercloud.

  For example, the network name `internal_api` can't be reused between
  stacks, unless the intent is to share the network between the stacks.
  The network would need to be given a different `name` and `name_lower`
  property such as `InternalApiCompute0` and `internal_api_compute_0`.

Configuring nova-metadata API per-cell
--------------------------------------

.. note::
  Deploying nova-metadata API per-cell is only supported in Train
  and later.

.. note::

  NovaLocalMetadataPerCell is only tested with ovn metadata agent to
  automatically forward requests to the nova metadata api.

It is possible to configure the nova-metadata API service local per-cell.
In this situation the cell controllers also host the nova-metadata API
service. The `NovaLocalMetadataPerCell` parameter, which defaults to
`false` need to be set to `true`.
Using nova-metadata API service per-cell can have better performance and
data isolation in a multi-cell deployment. Users should consider the use
of this configuration depending on how neutron is setup. If networks span
cells, you might need to run nova-metadata API service centrally.
If your networks are segmented along cell boundaries, then you can
run nova-metadata API service per cell.

.. code-block:: yaml

  parameter_defaults:
     NovaLocalMetadataPerCell: True

See also information on running nova-metadata API per cell as explained
in the cells v2 layout section `Local per cell <https://docs.openstack.org/nova/latest/user/cellsv2-layout.html#nova-metadata-api-service>`_
