Example 3. - Advanced example using split cell controller/compute architecture and routed networks in Train release
===================================================================================================================

.. warning::
  Multi cell support is only supported in Stein or later versions.
  This guide addresses Train release and later!

.. contents::
  :depth: 3
  :backlinks: none

This guide assumes that you are ready to deploy a new overcloud, or have
already installed an overcloud (min Train release).

.. note::

  Starting with CentOS 8 and the TripleO Stein release, podman is the CONTAINERCLI
  to be used in the following steps.

In this example we use the :doc:`deploy_cellv2_advanced` using a routed spine and
leaf networking layout to deploy an additional cell. Not all nodes need
to be co-located at the same physical location or datacenter. See
:ref:`routed_spine_leaf_network` for more details.

The nodes deployed to the control plane, which are part of the overcloud stack,
use different networks then the cell stacks which are separated in a cell
controller stack and a cell compute stack. The cell controller and cell compute
stack use the same networks,

.. note::

  In this example the routing for the different VLAN subnets is done by
  the undercloud, which must _NOT_ be done in a production environment
  as it is a single point of failure!

Used networks
^^^^^^^^^^^^^
The following provides and overview of the used networks and subnet
details for this example:

.. code-block:: yaml

  InternalApi
    internal_api_subnet
      vlan: 20
      net: 172.16.2.0/24
      route: 172.17.2.0/24 gw: 172.16.2.254
    internal_api_cell1
      vlan: 21
      net: 172.17.2.0/24
      gateway: 172.17.2.254
  Storage
    storage_subnet
      vlan: 30
      net: 172.16.1.0/24
      route: 172.17.1.0/24 gw: 172.16.1.254
    storage_cell1
      vlan: 31
      net: 172.17.1.0/24
      gateway: 172.17.1.254
  StorageMgmt
    storage_mgmt_subnet
      vlan: 40
      net: 172.16.3.0/24
      route: 172.17.3.0/24 gw: 172.16.3.254
    storage_mgmt_cell1
      vlan: 41
      net: 172.17.3.0/24
      gateway: 172.17.3.254
  Tenant
    tenant_subnet
      vlan: 50
      net: 172.16.0.0/24
  External
    external_subnet
      vlan: 10
      net: 10.0.0.0/24
    external_cell1
      vlan: 11
      net: 10.0.1.0/24
      gateway: 10.0.1.254

Prepare control plane for cell network routing
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash

  openstack overcloud status
  +-----------+-------------------+
  | Plan Name | Deployment Status |
  +-----------+-------------------+
  | overcloud |   DEPLOY_SUCCESS  |
  +-----------+-------------------+

  openstack server list -c Name -c Status -c Networks
  +-------------------------+--------+------------------------+
  | Name                    | Status | Networks               |
  +-------------------------+--------+------------------------+
  | overcloud-controller-2  | ACTIVE | ctlplane=192.168.24.29 |
  | overcloud-controller-0  | ACTIVE | ctlplane=192.168.24.18 |
  | overcloud-controller-1  | ACTIVE | ctlplane=192.168.24.20 |
  | overcloud-novacompute-0 | ACTIVE | ctlplane=192.168.24.16 |
  +-------------------------+--------+------------------------+

Overcloud stack for the control planed deployed using a `routes.yaml`
environment file to add the routing information for the new cell
subnets.

.. code-block:: yaml

  parameter_defaults:
    InternalApiInterfaceRoutes:
      - destination: 172.17.2.0/24
        nexthop: 172.16.2.254
    StorageInterfaceRoutes:
      - destination: 172.17.1.0/24
        nexthop: 172.16.1.254
    StorageMgmtInterfaceRoutes:
      - destination: 172.17.3.0/24
        nexthop: 172.16.3.254

Reuse networks and adding cell subnets
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
To prepare the  `network_data` parameter file for the cell controller stack
the file from the control plane is used as base:

.. code-block:: bash

  cp /usr/share/openstack-tripleo-heat-templates/network_data.yaml cell1/network_data-ctrl.yaml

When deploying a cell in separate stacks it may be necessary to reuse networks,
subnets, segments, and VIP resources between stacks. Only a single Heat stack
can own a resource and be responsible for its creation and deletion, however
the resources can be reused in other stacks.

To reuse network related resources between stacks, the following parameters have
been added to the network definitions in the network_data.yaml file format:

.. code-block:: yaml

  external_resource_network_id: Existing Network UUID
  external_resource_subnet_id: Existing Subnet UUID
  external_resource_segment_id: Existing Segment UUID
  external_resource_vip_id: Existing VIP UUID

.. note:

  The cell controllers use virtual IPs, therefore the existing VIPs from the
  central overcloud stack should not be referenced. In case cell controllers
  and cell computes get split into separate stacks, the cell compute stack
  network_data file need an external_resource_vip_id reference to the cell
  controllers VIP resource.

These parameters can be set on each network definition in the `network_data-ctrl.yaml`
file used for the deployment of the separate stack.

Not all networks need to be reused or shared across stacks. The `external_resource_*`
parameters can be set for only the networks that are meant to be shared, while
the other networks can be newly created and managed.

In this example we reuse all networks, except the management network as it is
not being used at all.

The resulting storage network here looks like this:

.. code-block::

  - name: Storage
      external_resource_network_id: 30e9d52d-1929-47ed-884b-7c6d65fa2e00
      external_resource_subnet_id: 11a3777a-8c42-4314-a47f-72c86e9e6ad4
      vip: true
      vlan: 30
      name_lower: storage
      ip_subnet: '172.16.1.0/24'
      allocation_pools: [{'start': '172.16.1.4', 'end': '172.16.1.250'}]
      ipv6_subnet: 'fd00:fd00:fd00:3000::/64'
      ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:3000::10', 'end': 'fd00:fd00:fd00:3000:ffff:ffff:ffff:fffe'}]
      mtu: 1500
      subnets:
        storage_cell1:
          vlan: 31
          ip_subnet: '172.17.1.0/24'
          allocation_pools: [{'start': '172.17.1.10', 'end': '172.17.1.250'}]
          gateway_ip: '172.17.1.254'

We added the `external_resource_network_id` and `external_resource_subnet_id` of
the control plane stack as we want to reuse those resources:

.. code-block:: bash

  openstack network show storage -c id -f value
  openstack subnet show storage_subnet -c id -f value

In addition a new `storage_cell1` subnet is now added to the `subnets` section
to get it created in the cell controller stack for cell1:

.. code-block::

  subnets:
    storage_cell1:
      vlan: 31
      ip_subnet: '172.17.1.0/24'
      allocation_pools: [{'start': '172.17.1.10', 'end': '172.17.1.250'}]
      gateway_ip: '172.17.1.254'

.. note::

  In this example no Management network is used, therefore it was removed.

Full networks data example:

.. code-block::

  - name: Storage
    external_resource_network_id: 30e9d52d-1929-47ed-884b-7c6d65fa2e00
    external_resource_subnet_id: 11a3777a-8c42-4314-a47f-72c86e9e6ad4
    vip: true
    vlan: 30
    name_lower: storage
    ip_subnet: '172.16.1.0/24'
    allocation_pools: [{'start': '172.16.1.4', 'end': '172.16.1.250'}]
    ipv6_subnet: 'fd00:fd00:fd00:3000::/64'
    ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:3000::10', 'end': 'fd00:fd00:fd00:3000:ffff:ffff:ffff:fffe'}]
    mtu: 1500
    subnets:
      storage_cell1:
        vlan: 31
        ip_subnet: '172.17.1.0/24'
        allocation_pools: [{'start': '172.17.1.10', 'end': '172.17.1.250'}]
        gateway_ip: '172.17.1.254'
  - name: StorageMgmt
    name_lower: storage_mgmt
    external_resource_network_id: 29e85314-2177-4cbd-aac8-6faf2a3f7031
    external_resource_subnet_id: 01c0a75e-e62f-445d-97ad-b98a141d6082
    vip: true
    vlan: 40
    ip_subnet: '172.16.3.0/24'
    allocation_pools: [{'start': '172.16.3.4', 'end': '172.16.3.250'}]
    ipv6_subnet: 'fd00:fd00:fd00:4000::/64'
    ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:4000::10', 'end': 'fd00:fd00:fd00:4000:ffff:ffff:ffff:fffe'}]
    mtu: 1500
    subnets:
      storage_mgmt_cell1:
        vlan: 41
        ip_subnet: '172.17.3.0/24'
        allocation_pools: [{'start': '172.17.3.10', 'end': '172.17.3.250'}]
        gateway_ip: '172.17.3.254'
  - name: InternalApi
    name_lower: internal_api
    external_resource_network_id: 5eb79743-7ff4-4f68-9904-6e9c36fbaaa6
    external_resource_subnet_id: dbc24086-0aa7-421d-857d-4e3956adec10
    vip: true
    vlan: 20
    ip_subnet: '172.16.2.0/24'
    allocation_pools: [{'start': '172.16.2.4', 'end': '172.16.2.250'}]
    ipv6_subnet: 'fd00:fd00:fd00:2000::/64'
    ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:2000::10', 'end': 'fd00:fd00:fd00:2000:ffff:ffff:ffff:fffe'}]
    mtu: 1500
    subnets:
      internal_api_cell1:
        vlan: 21
        ip_subnet: '172.17.2.0/24'
        allocation_pools: [{'start': '172.17.2.10', 'end': '172.17.2.250'}]
        gateway_ip: '172.17.2.254'
  - name: Tenant
    external_resource_network_id: ee83d0fb-3bf1-47f2-a02b-ef5dc277afae
    external_resource_subnet_id: 0b6030ae-8445-4480-ab17-dd4c7c8fa64b
    vip: false  # Tenant network does not use VIPs
    name_lower: tenant
    vlan: 50
    ip_subnet: '172.16.0.0/24'
    allocation_pools: [{'start': '172.16.0.4', 'end': '172.16.0.250'}]
    ipv6_subnet: 'fd00:fd00:fd00:5000::/64'
    ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:5000::10', 'end': 'fd00:fd00:fd00:5000:ffff:ffff:ffff:fffe'}]
    mtu: 1500
  - name: External
    external_resource_network_id: 89b7b481-f609-45e7-ad5e-e006553c1d3a
    external_resource_subnet_id: dd84112d-2129-430c-a8c2-77d2dee05af2
    vip: true
    name_lower: external
    vlan: 10
    ip_subnet: '10.0.0.0/24'
    allocation_pools: [{'start': '10.0.0.4', 'end': '10.0.0.250'}]
    gateway_ip: '10.0.0.1'
    ipv6_subnet: '2001:db8:fd00:1000::/64'
    ipv6_allocation_pools: [{'start': '2001:db8:fd00:1000::10', 'end': '2001:db8:fd00:1000:ffff:ffff:ffff:fffe'}]
    gateway_ipv6: '2001:db8:fd00:1000::1'
    mtu: 1500
    subnets:
      external_cell1:
        vlan: 11
        ip_subnet: '10.0.1.0/24'
        allocation_pools: [{'start': '10.0.1.10', 'end': '10.0.1.250'}]
        gateway_ip: '10.0.1.254'

.. note:

  When not sharing networks between stacks, each network defined in `network_data*.yaml`
  must have a unique name across all deployed stacks. This requirement is necessary
  since regardless of the stack, all networks are created in the same tenant in
  Neutron on the undercloud.

Export EndpointMap, HostsEntry, AllNodesConfig, GlobalConfig and passwords information
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Follow the steps as explained in :ref:`cell_export_overcloud_info` on how to
export the required data from the overcloud stack.

Cell roles
^^^^^^^^^^
Modify the cell roles file to use new subnets for `InternalApi`, `Storage`,
`StorageMgmt` and `External` for cell controller and compute:

.. code-block:: bash

  openstack overcloud roles generate --roles-path \
  /usr/share/openstack-tripleo-heat-templates/roles \
  -o $DIR/cell_roles_data.yaml Compute CellController

For each role modify the subnets to match what got defined in the previous step
in `cell1/network_data-ctrl.yaml`:

.. code-block::

  - name: Compute
    description: |
      Basic Compute Node role
    CountDefault: 1
    # Create external Neutron bridge (unset if using ML2/OVS without DVR)
    tags:
      - external_bridge
    networks:
      InternalApi:
        subnet: internal_api_cell1
      Tenant:
        subnet: tenant_subnet
      Storage:
        subnet: storage_cell1
  ...
  - name: CellController
      description: |
        CellController role for the nova cell_v2 controller services
      CountDefault: 1
      tags:
        - primary
        - controller
      networks:
        External:
          subnet: external_cell1
        InternalApi:
          subnet: internal_api_cell1
        Storage:
          subnet: storage_cell1
        StorageMgmt:
          subnet: storage_mgmt_cell1
        Tenant:
          subnet: tenant_subnet

Create the cell parameter file
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Each cell has some mandatory parameters which need to be set using an
environment file.
Add the following content into a parameter file for the cell, e.g. `cell1/cell1.yaml`:

.. code-block:: yaml

  parameter_defaults:
    # new CELL Parameter to reflect that this is an additional CELL
    NovaAdditionalCell: True

    # The DNS names for the VIPs for the cell
    CloudName: cell1.ooo.test
    CloudNameInternal: cell1.internalapi.ooo.test
    CloudNameStorage: cell1.storage.ooo.test
    CloudNameStorageManagement: cell1.storagemgmt.ooo.test
    CloudNameCtlplane: cell1.ctlplane.ooo.test

    # Flavors used for the cell controller and computes
    OvercloudCellControllerFlavor: cellcontroller
    OvercloudComputeFlavor: compute

    # number of controllers/computes in the cell
    CellControllerCount: 3
    ComputeCount: 0

    # Compute names need to be unique, make sure to have a unique
    # hostname format for cell nodes
    ComputeHostnameFormat: 'cell1-compute-%index%'

    # default gateway
    ControlPlaneStaticRoutes:
      - ip_netmask: 0.0.0.0/0
        next_hop: 192.168.24.1
        default: true
    DnsServers:
      - x.x.x.x

Virtual IP addresses
^^^^^^^^^^^^^^^^^^^^
The cell controller is hosting VIP’s (Virtual IP addresses) and is not using
the base subnet of one or more networks, therefore additional overrides to the
`VipSubnetMap` are required to ensure VIP’s are created on the subnet associated
with the L2 network segment the controller nodes is connected to.

Add a `VipSubnetMap` to the `cell1/cell1.yaml` or a new parameter file to
point the VIPs to the correct subnet:

.. code-block:: yaml

  parameter_defaults:
    VipSubnetMap:
      InternalApi: internal_api_cell1
      Storage: storage_cell1
      StorageMgmt: storage_mgmt_cell1
      External: external_cell1

Create the network configuration for `cellcontroller` and add to environment file
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Depending on the network configuration of the used hardware and network
architecture it is required to register a resource for the `CellController`
role in `cell1/cell1.yaml`.

.. code-block:: yaml

  resource_registry:
    OS::TripleO::CellController::Net::SoftwareConfig: cell1/single-nic-vlans/controller.yaml
    OS::TripleO::Compute::Net::SoftwareConfig: cell1/single-nic-vlans/compute.yaml

.. note::

  For details on network configuration consult :ref:`network_isolation` guide, chapter *Customizing the Interface Templates*.

Deploy the cell controllers
^^^^^^^^^^^^^^^^^^^^^^^^^^^
Create new flavor used to tag the cell controller
_________________________________________________
Follow the instructions in :ref:`cell_create_flavor_and_tag` on how to create
a new flavor and tag the cell controller.

Run cell deployment
___________________
To deploy the overcloud we can use the same `overcloud deploy` command as
it was used to deploy the `overcloud` stack and add the created export
environment files:

.. code-block:: bash

    openstack overcloud deploy \
      --templates /usr/share/openstack-tripleo-heat-templates \
      -e ... additional environment files used for overcloud stack, like container
        prepare parameters, or other specific parameters for the cell
      ...
      --stack cell1-ctrl \
      -n $HOME/$DIR/network_data-ctrl.yaml \
      -r $HOME/$DIR/cell_roles_data.yaml \
      -e $HOME/$DIR/cell1-ctrl-input.yaml \
      -e $HOME/$DIR/cell1.yaml

Wait for the deployment to finish:

.. code-block:: bash

  openstack stack list

  +--------------------------------------+------------+----------------------------------+-----------------+----------------------+----------------------+
  | ID                                   | Stack Name | Project                          | Stack Status    | Creation Time        | Updated Time         |
  +--------------------------------------+------------+----------------------------------+-----------------+----------------------+----------------------+
  | 6403ed94-7c8f-47eb-bdb8-388a5ac7cb20 | cell1-ctrl | f7736589861c47d8bbf1ecd29f02823d | CREATE_COMPLETE | 2019-08-15T14:46:32Z | None                 |
  | 925a2875-fbbb-41fd-bb06-bf19cded2510 | overcloud  | f7736589861c47d8bbf1ecd29f02823d | UPDATE_COMPLETE | 2019-08-13T10:43:20Z | 2019-08-15T10:13:41Z |
  +--------------------------------------+------------+----------------------------------+-----------------+----------------------+----------------------+

Create the cell
^^^^^^^^^^^^^^^
As in :ref:`cell_create_cell` create the cell, but we can skip the final host
discovery step as the computes are note yet deployed.


Extract deployment information from the cell controller stack
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Follow the steps explained in :ref:`cell_export_cell_controller_info` on
how to export the required input data from the cell controller stack.

Create cell compute parameter file for additional customization
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Create the `cell1/cell1-cmp.yaml` parameter file to overwrite settings
which are different from the cell controller stack.

.. code-block:: yaml

  parameter_defaults:
    # number of controllers/computes in the cell
    CellControllerCount: 0
    ComputeCount: 1

The above file overwrites the values from `cell1/cell1.yaml` to not deploy
a controller in the cell compute stack. Since the cell compute stack uses
the same role file the default `CellControllerCount` is 1.

Reusing networks from control plane and cell controller stack
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
For the cell compute stack we reuse the networks from the control plane
stack and the subnet from the cell controller stack. Therefore references
to the external resources for network, subnet, segment and vip are required:

.. code-block:: bash

  cp cell1/network_data-ctrl.yaml cell1/network_data-cmp.yaml

The storage network definition in `cell1/network_data-cmp.yaml` looks
like this:

.. code-block::

  - name: Storage
    external_resource_network_id: 30e9d52d-1929-47ed-884b-7c6d65fa2e00
    external_resource_subnet_id: 11a3777a-8c42-4314-a47f-72c86e9e6ad4
    external_resource_vip_id: 4ed73ea9-4cf6-42c1-96a5-e32b415c738f
    vip: true
    vlan: 30
    name_lower: storage
    ip_subnet: '172.16.1.0/24'
    allocation_pools: [{'start': '172.16.1.4', 'end': '172.16.1.250'}]
    ipv6_subnet: 'fd00:fd00:fd00:3000::/64'
    ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:3000::10', 'end': 'fd00:fd00:fd00:3000:ffff:ffff:ffff:fffe'}]
    mtu: 1500
    subnets:
      storage_cell1:
        vlan: 31
        ip_subnet: '172.17.1.0/24'
        allocation_pools: [{'start': '172.17.1.10', 'end': '172.17.1.250'}]
        gateway_ip: '172.17.1.254'
        external_resource_subnet_id: 7930635d-d1d5-4699-b318-00233c73ed6b
        external_resource_segment_id: 730769f8-e78f-42a3-9dd4-367a212e49ff

Previously we already added the `external_resource_network_id` and `external_resource_subnet_id`
for the network in the upper level hierarchy.

In addition we add the `external_resource_vip_id` of the VIP of the stack which
should be reused for this network (Storage).

Important is that the `external_resource_vip_id` for the InternalApi points
the VIP of the cell controller stack!

.. code-block:: bash

  openstack port show <id storage_virtual_ip overcloud stack> -c id -f value

In the `storage_cell1` subnet section we add the `external_resource_subnet_id`
and `external_resource_segment_id` of the cell controller stack:

.. code-block:: yaml

  storage_cell1:
    vlan: 31
    ip_subnet: '172.17.1.0/24'
    allocation_pools: [{'start': '172.17.1.10', 'end': '172.17.1.250'}]
    gateway_ip: '172.17.1.254'
    external_resource_subnet_id: 7930635d-d1d5-4699-b318-00233c73ed6b
    external_resource_segment_id: 730769f8-e78f-42a3-9dd4-367a212e49ff

.. code-block:: bash

  openstack subnet show storage_cell1 -c id -f value
  openstack network segment show storage_storage_cell1 -c id -f value

Full networks data example for the compute stack:
 
.. code-block::

  - name: Storage
    external_resource_network_id: 30e9d52d-1929-47ed-884b-7c6d65fa2e00
    external_resource_subnet_id: 11a3777a-8c42-4314-a47f-72c86e9e6ad4
    external_resource_vip_id: 4ed73ea9-4cf6-42c1-96a5-e32b415c738f
    vip: true
    vlan: 30
    name_lower: storage
    ip_subnet: '172.16.1.0/24'
    allocation_pools: [{'start': '172.16.1.4', 'end': '172.16.1.250'}]
    ipv6_subnet: 'fd00:fd00:fd00:3000::/64'
    ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:3000::10', 'end': 'fd00:fd00:fd00:3000:ffff:ffff:ffff:fffe'}]
    mtu: 1500
    subnets:
      storage_cell1:
        vlan: 31
        ip_subnet: '172.17.1.0/24'
        allocation_pools: [{'start': '172.17.1.10', 'end': '172.17.1.250'}]
        gateway_ip: '172.17.1.254'
        external_resource_subnet_id: 7930635d-d1d5-4699-b318-00233c73ed6b
        external_resource_segment_id: 730769f8-e78f-42a3-9dd4-367a212e49ff
  - name: StorageMgmt
    name_lower: storage_mgmt
    external_resource_network_id: 29e85314-2177-4cbd-aac8-6faf2a3f7031
    external_resource_subnet_id: 01c0a75e-e62f-445d-97ad-b98a141d6082
    external_resource_segment_id: 4b4f6f83-f031-4495-84c5-7422db1729d5
    vip: true
    vlan: 40
    ip_subnet: '172.16.3.0/24'
    allocation_pools: [{'start': '172.16.3.4', 'end': '172.16.3.250'}]
    ipv6_subnet: 'fd00:fd00:fd00:4000::/64'
    ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:4000::10', 'end': 'fd00:fd00:fd00:4000:ffff:ffff:ffff:fffe'}]
    mtu: 1500
    subnets:
      storage_mgmt_cell1:
        vlan: 41
        ip_subnet: '172.17.3.0/24'
        allocation_pools: [{'start': '172.17.3.10', 'end': '172.17.3.250'}]
        gateway_ip: '172.17.3.254'
        external_resource_subnet_id: de9233d4-53a3-485d-8433-995a9057383f
        external_resource_segment_id: 2400718d-7fbd-4227-8318-245747495241
  - name: InternalApi
    name_lower: internal_api
    external_resource_network_id: 5eb79743-7ff4-4f68-9904-6e9c36fbaaa6
    external_resource_subnet_id: dbc24086-0aa7-421d-857d-4e3956adec10
    external_resource_vip_id: 1a287ad7-e574-483a-8288-e7c385ee88a0
    vip: true
    vlan: 20
    ip_subnet: '172.16.2.0/24'
    allocation_pools: [{'start': '172.16.2.4', 'end': '172.16.2.250'}]
    ipv6_subnet: 'fd00:fd00:fd00:2000::/64'
    ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:2000::10', 'end': 'fd00:fd00:fd00:2000:ffff:ffff:ffff:fffe'}]
    mtu: 1500
    subnets:
      internal_api_cell1:
        external_resource_subnet_id: 16b8cf48-6ca1-4117-ad90-3273396cb41d
        external_resource_segment_id: b310daec-7811-46be-a958-a05a5b0569ef
        vlan: 21
        ip_subnet: '172.17.2.0/24'
        allocation_pools: [{'start': '172.17.2.10', 'end': '172.17.2.250'}]
        gateway_ip: '172.17.2.254'
  - name: Tenant
    external_resource_network_id: ee83d0fb-3bf1-47f2-a02b-ef5dc277afae
    external_resource_subnet_id: 0b6030ae-8445-4480-ab17-dd4c7c8fa64b
    vip: false  # Tenant network does not use VIPs
    name_lower: tenant
    vlan: 50
    ip_subnet: '172.16.0.0/24'
    allocation_pools: [{'start': '172.16.0.4', 'end': '172.16.0.250'}]
    ipv6_subnet: 'fd00:fd00:fd00:5000::/64'
    ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:5000::10', 'end': 'fd00:fd00:fd00:5000:ffff:ffff:ffff:fffe'}]
    mtu: 1500
  - name: External
    external_resource_network_id: 89b7b481-f609-45e7-ad5e-e006553c1d3a
    external_resource_subnet_id: dd84112d-2129-430c-a8c2-77d2dee05af2
    external_resource_vip_id: b7a0606d-f598-4dc6-9e85-e023c64fd20b
    vip: true
    name_lower: external
    vlan: 10
    ip_subnet: '10.0.0.0/24'
    allocation_pools: [{'start': '10.0.0.4', 'end': '10.0.0.250'}]
    gateway_ip: '10.0.0.1'
    ipv6_subnet: '2001:db8:fd00:1000::/64'
    ipv6_allocation_pools: [{'start': '2001:db8:fd00:1000::10', 'end': '2001:db8:fd00:1000:ffff:ffff:ffff:fffe'}]
    gateway_ipv6: '2001:db8:fd00:1000::1'
    mtu: 1500
    subnets:
      external_cell1:
        vlan: 11
        ip_subnet: '10.0.1.0/24'
        allocation_pools: [{'start': '10.0.1.10', 'end': '10.0.1.250'}]
        gateway_ip: '10.0.1.254'
        external_resource_subnet_id: 81ac9bc2-4fbe-40be-ac0e-9aa425799626
        external_resource_segment_id: 8a877c1f-cb47-40dd-a906-6731f042e544

Deploy the cell computes
^^^^^^^^^^^^^^^^^^^^^^^^

Run cell deployment
___________________
To deploy the overcloud we can use the same `overcloud deploy` command as
it was used to deploy the `cell1-ctrl` stack and add the created export
environment files:

.. code-block:: bash

    openstack overcloud deploy \
      --templates /usr/share/openstack-tripleo-heat-templates \
      -e ... additional environment files used for overcloud stack, like container
        prepare parameters, or other specific parameters for the cell
      ...
      --stack cell1-cmp \
      -r $HOME/$DIR/cell_roles_data.yaml \
      -n $HOME/$DIR/network_data-cmp.yaml \
      -e $HOME/$DIR/cell1-ctrl-input.yaml \
      -e $HOME/$DIR/cell1-cmp-input.yaml \
      -e $HOME/$DIR/cell1.yaml \
      -e $HOME/$DIR/cell1-cmp.yaml

Wait for the deployment to finish:

.. code-block:: bash

  openstack stack list
  +--------------------------------------+------------+----------------------------------+--------------------+----------------------+----------------------+
  | ID                                   | Stack Name | Project                          | Stack Status       | Creation Time        | Updated Time         |
  +--------------------------------------+------------+----------------------------------+--------------------+----------------------+----------------------+
  | 12e86ea6-3725-482a-9b05-b283378dcf30 | cell1-cmp  | f7736589861c47d8bbf1ecd29f02823d | CREATE_COMPLETE    | 2019-08-15T15:57:19Z | None                 |
  | 6403ed94-7c8f-47eb-bdb8-388a5ac7cb20 | cell1-ctrl | f7736589861c47d8bbf1ecd29f02823d | CREATE_COMPLETE    | 2019-08-15T14:46:32Z | None                 |
  | 925a2875-fbbb-41fd-bb06-bf19cded2510 | overcloud  | f7736589861c47d8bbf1ecd29f02823d | UPDATE_COMPLETE    | 2019-08-13T10:43:20Z | 2019-08-15T10:13:41Z |
  +--------------------------------------+------------+----------------------------------+--------------------+----------------------+----------------------+

Perform cell host discovery
___________________________
The final step is to discover the computes deployed in the cell. Run the host discovery
as explained in :ref:`cell_host_discovery`.

Create and add the node to an Availability Zone
_______________________________________________
After a cell got provisioned, it is required to create an availability zone for the
compute stack, it is not enough to just create an availability zone for the complete
cell. In this used case we want to make sure an instance created in the compute group,
stays in it when performing a migration. Check :ref:`cell_availability_zone` on more
about how to create an availability zone and add the node.

After that the cell is deployed and can be used.

.. note::

  Migrating instances between cells is not supported. To move an instance to
  a different cell it needs to be re-created in the new target cell.
