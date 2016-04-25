Configuring Network Isolation
=============================

Introduction
------------

|project| provides configuration of isolated overcloud networks. Using
this approach it is possible to host traffic for specific types of network
traffic (tenants, storage, API/RPC, etc.) in isolated networks. This allows
for assigning network traffic to specific network interfaces or bonds. Using
bonds provides fault tolerance, and may provide load sharing, depending on the
bonding protocols used. When isolated networks are configured, the OpenStack
services will be configured to use the isolated networks. If no isolated
networks are configured, all services run on the provisioning network.

There are two parts to the network configuration: the parameters that apply
to the network as a whole, and the templates which configure the network
interfaces on the deployed hosts.

Architecture
------------

The following VLANs will be used in the final deployment:

* IPMI* (IPMI System controller, iLO, DRAC)
* Provisioning* (Undercloud control plane for deployment and management)
* Internal API (OpenStack internal API, RPC, and DB)
* Tenant (Tenant tunneling network for GRE/VXLAN networks)
* Storage (Access to storage resources from Compute and Controller nodes)
* Storage Management (Replication, Ceph back-end services)
* External (Public OpenStack APIs, Horzizon dashboard, optionally floating IPs)
* Floating IP (Optional, can be combined with External)

.. note::
  Networks marked with '*' are usually native VLANs, others may be trunked.

The External network should have a gateway router address. This will be used
in the subnet configuration of the network environment.

If floating IPs will be hosted on a separate VLAN from External, that VLAN will
need to be trunked to the controller hosts. It will not be included in the
network configuration steps for the deployment, the VLAN will be added via
Neutron and Open vSwitch. There can be multiple floating IP networks, and they
can be attached to multiple bridges. The VLANs will be trunked, but not
configured as interfaces. Instead, Neutron will create an OVS port with the
VLAN segmentation ID on the chosen bridge for each floating IP network.

The Provisioning network will usually be delivered on a dedicated interface.
DHCP+PXE is used to initially deploy, then the IP will be converted to static.
By default, PXE boot must occur on the native VLAN, although some system
controllers will allow booting from a VLAN. The Provisioning interface is
also used by the Compute and Storage nodes as their default gateway, in order
to contact DNS, NTP, and for system maintenance. The Undercloud can be used
as a default gateway, but in that case all traffic will be behind an IP
masquerade NAT, and will not be reachable from the rest of the network. The
Undercloud is also a single point of failure for the overcloud default route.
If there is an external gateway on a router device on the Provisioning network,
the Undercloud Neutron DHCP server can offer that instead. If the
``network_gateway`` was not set properly in undercloud.conf, it can be set
manually after installing the Undercloud::

  neutron subnet-show     # Copy the UUID from the provisioning subnet
  neutron subnet-update <UUID> --gateway_ip <IP_ADDRESS>

Often, the number of VLANs will exceed the number of physical Ethernet ports,
so some VLANs are delivered with VLAN tagging to separate the traffic. On an
Ethernet bond, typically all VLANs are trunked, and there is no traffic on the
native VLAN (native VLANs on bonds are supported, but will require customizing
the NIC templates).

.. note::
  It is recommended to deploy a Tenant VLAN (which is used for tunneling GRE
  and/or VXLAN) even if Neutron VLAN mode is chosen and tunneling is disabled
  at deployment time. This requires the least customization at deployment time,
  and leaves the option available to use tunnel networks as utility networks,
  or for network function virtualization in the future. Tenant networks will
  still be created using VLANs, but the operator can create VXLAN tunnels for
  special use networks without consuming tenant VLANs. It is possible to add
  VXLAN capability to a deployment with a Tenant VLAN, but it is not possible
  to add a Tenant VLAN to an already deployed set of hosts without disruption.

The networks are connected to the roles as follows:

Controller:

* Provisioning
* Internal API
* Storage
* Storage Management
* External

Compute:

* Provisioning
* Internal API
* Storage
* Tenant

Ceph Storage:

* Provisioning
* Storage
* Storage Management

Cinder Storage:

* Provisioning
* Internal API
* Storage
* Storage Management

Swift Storage:

* Provisioning
* Internal API
* Storage
* Storage Management

Workflow
--------

The procedure for enabling network isolation is this:

#. Create network environment file (e.g. /home/stack/network-environment.yaml)
#. Edit IP subnets and VLANs in the environment file to match local environment
#. Make a copy of the appropriate sample network interface configurations
#. Edit the network interface configurations to match local environment
#. Deploy overcloud with the proper parameters to include network isolation

The next section will walk through the elements that need to be added to
the network-environment.yaml to enable network isolation. The sections
after that deal with configuring the network interface templates. The final step
will deploy the overcloud with network isolation and a custom environment.

Create Network Environment File
-------------------------------
The environment file will describe the network environment and will point to
the network interface configuration files to use for the overcloud nodes.
The subnets that will be used for the isolated networks need to be defined,
along with the IP address ranges that should be used for IP assignment. These
values must be customized for the local environment.

It is important for the ExternalInterfaceDefaultRoute to be reachable on the
subnet that is used for ExternalNetCidr. This will allow the OpenStack Public
APIs and the Horizon Dashboard to be reachable. Without a valid default route,
the post-deployment steps cannot be performed.

.. note::
  The ``resource_registry`` section of the network-environment.yaml contains
  pointers to the network interface configurations for the deployed roles.
  These files must exist at the path referenced here, and will be copied
  later in this guide.

Example::

  resource_registry:
    OS::TripleO::BlockStorage::Net::SoftwareConfig: /home/stack/nic-configs/cinder-storage.yaml
    OS::TripleO::Compute::Net::SoftwareConfig: /home/stack/nic-configs/compute.yaml
    OS::TripleO::Controller::Net::SoftwareConfig: /home/stack/nic-configs/controller.yaml
    OS::TripleO::ObjectStorage::Net::SoftwareConfig: /home/stack/nic-configs/swift-storage.yaml
    OS::TripleO::CephStorage::Net::SoftwareConfig: /home/stack/nic-configs/ceph-storage.yaml

  parameter_defaults:
    # Customize all these values to match the local environment
    InternalApiNetCidr: 172.17.0.0/24
    StorageNetCidr: 172.18.0.0/24
    StorageMgmtNetCidr: 172.19.0.0/24
    TenantNetCidr: 172.16.0.0/24
    ExternalNetCidr: 10.1.2.0/24
    # CIDR subnet mask length for provisioning network
    ControlPlaneSubnetCidr: '24'
    InternalApiAllocationPools: [{'start': '172.17.0.10', 'end': '172.17.0.200'}]
    StorageAllocationPools: [{'start': '172.18.0.10', 'end': '172.18.0.200'}]
    StorageMgmtAllocationPools: [{'start': '172.19.0.10', 'end': '172.19.0.200'}]
    TenantAllocationPools: [{'start': '172.16.0.10', 'end': '172.16.0.200'}]
    # Use an External allocation pool which will leave room for floating IPs
    ExternalAllocationPools: [{'start': '10.1.2.10', 'end': '10.1.2.50'}]
    # Set to the router gateway on the external network
    ExternalInterfaceDefaultRoute: 10.1.2.1
    # Gateway router for the provisioning network (or Undercloud IP)
    ControlPlaneDefaultRoute: 192.0.2.254
    # Generally the IP of the Undercloud
    EC2MetadataIp: 192.0.2.1
    # Define the DNS servers (maximum 2) for the overcloud nodes
    DnsServers: ["8.8.8.8","8.8.4.4"]
    InternalApiNetworkVlanID: 201
    StorageNetworkVlanID: 202
    StorageMgmtNetworkVlanID: 203
    TenantNetworkVlanID: 204
    ExternalNetworkVlanID: 100
    # May set to br-ex if using floating IPs only on native VLAN on bridge br-ex
    NeutronExternalNetworkBridge: "''"
    # Customize bonding options if required (ignored if bonds are not used)
    BondInterfaceOvsOptions:
        "bond_mode=balance-tcp lacp=active other-config:lacp-fallback-ab=true"

Configure IP Subnets
--------------------
Each environment will have its own IP subnets for each network. This will vary
by deployment, and should be tailored to the environment. We will set the
subnet information for all the networks inside our environment file. Each
subnet will have a range of IP addresses that will be used for assigning IP
addresses to hosts and virtual IPs.

In the example above, the Allocation Pool for the Internal API network starts
at .10 and continues to .200. This results in the static IPs and virtual IPs
that are assigned starting at .10, and will be assigned upwards with .200 being
the highest assigned IP. The External network hosts the Horizon dashboard and
the OpenStack public API. If the External network will be used for both cloud
administration and floating IPs, we need to make sure there is room for a pool
of IPs to use as floating IPs for VM instances. Alternately, the floating IPs
can be placed on a separate VLAN (which is configured by the operator
post-deployment).

Configure VLANs and Bonding Options
-----------------------------------
The VLANs will need to be customized to match the environment. The values
entered in the ``network-environment.yaml`` will be used in the network
interface configuration templates covered below. For example::

  # Customize the VLAN IDs to match the local environment
  InternalApiNetworkVlanID: 10
  StorageNetworkVlanID: 20
  StorageMgmtNetworkVlanID: 30
  TenantNetworkVlanID: 40
  ExternalNetworkVlanID: 50

The example bonding options will try to negotiate LACP, but will fallback to
active-backup if LACP cannot be established::

  BondInterfaceOvsOptions:
    "bond_mode=balance-tcp lacp=active other-config:lacp-fallback-ab=true"

The BondInterfaceOvsOptions parameter will pass the options to Open vSwitch
when setting up bonding (if used in the environment). The value above will
enable fault-tolerance and load balancing if the switch supports (and is
configured to use) LACP bonding. If LACP cannot be established, the bond will
fallback to active/backup mode, with fault tolerance, but where only one link
in the bond will be used at a time.

If the switches do not support LACP, then do not configure a bond on the
upstream switch. Instead, OVS can use ``balance-slb`` mode to enable using
two interfaces on the same VLAN as a bond::

  # Use balance-slb for bonds configured on a switch without LACP support
  "bond_mode=balance-slb lacp=off"

Bonding with balance-slb allows a limited form of load balancing without the
remote switch's knowledge or cooperation. The basics of SLB are simple. SLB
assigns each source MAC+VLAN pair to a link and transmits all packets
from that MAC+VLAN through that link. Learning in the remote switch causes it
to send packets to that MAC+VLAN through the same link.

OVS will balance traffic based on source MAC and destination VLAN. The
switch will only see a given MAC address on one link in the bond at a time, and
OVS will use special filtering to prevent packet duplication across the links.

In addition, the following options may be added to the options string to tune
the bond::

  # Force bond to use active-backup, e.g. for connecting to 2 different switches
  "bond_mode=active-backup"

  # Set the LACP heartbeat to 1 second or 30 seconds (default)
  "other_config:lacp-time=[fast|slow]"

  # Set the link detection to use miimon heartbeats or monitor carrier (default)
  "other_config:bond-detect-mode=[miimon|carrier]"

  # If using miimon, heartbeat interval in milliseconds (100 is usually good)
  "other_config:bond-miimon-interval=100"

  # Number of milliseconds a link must be up to be activated (to prevent flapping)
  "other_config:bond_updelay=1000"

  # Milliseconds between rebalancing flows between bond members, zero to disable
  "other_config:bond-rebalance-interval=10000"

Creating Custom Interface Templates
-----------------------------------

In order to configure the network interfaces on each node, the network
interface templates may need to be customized.

Start by copying the configurations from one of the example directories. The
first example copies the templates which include network bonding. The second
example copies the templates which use a single network interface with
multiple VLANs (this configuration is mostly intended for testing).

To copy the bonded example interface configurations, run::

    $ cp /usr/share/openstack-tripleo-heat-templates/network/config/bond-with-vlans/* ~/nic-configs

To copy the single NIC with VLANs example interface configurations, run::

    $ cp /usr/share/openstack-tripleo-heat-templates/network/config/single-nic-vlans/* ~/nic-configs

Or, if you have custom NIC templates from another source, copy them to the location
referenced in the ``resource_registry`` section of the environment file.

Customizing the Interface Templates
-----------------------------------
The following example configures a bond on interfaces 3 and 4 of a system
with 4 interfaces. This example is based on the controller template from the
bond-with-vlans sample templates, but the bond has been placed on nic3 and nic4
instead of nic2 and nic3. The other roles will have a similar configuration,
but will have only a subset of the networks attached.

.. note::
  The nic1, nic2... abstraction considers only network interfaces which are
  connected to an Ethernet switch. If interfaces 1 and 4 are the only
  interfaces which are plugged in, they will be referred to as nic1 and nic2.

Example::

  heat_template_version: 2015-04-30

  description: >
    Software Config to drive os-net-config with 2 bonded nics on a bridge
    with a VLANs attached for the controller role.

  parameters:
    ControlPlaneIp:
      default: ''
      description: IP address/subnet on the ctlplane network
      type: string
    ExternalIpSubnet:
      default: ''
      description: IP address/subnet on the external network
      type: string
    InternalApiIpSubnet:
      default: ''
      description: IP address/subnet on the internal API network
      type: string
    StorageIpSubnet:
      default: ''
      description: IP address/subnet on the storage network
      type: string
    StorageMgmtIpSubnet:
      default: ''
      description: IP address/subnet on the storage mgmt network
      type: string
    TenantIpSubnet:
      default: ''
      description: IP address/subnet on the tenant network
      type: string
    BondInterfaceOvsOptions:
      default: ''
      description: The ovs_options string for the bond interface. Set things like
                   lacp=active and/or bond_mode=balance-slb using this option.
      type: string
    ExternalNetworkVlanID:
      default: 10
      description: Vlan ID for the external network traffic.
      type: number
    InternalApiNetworkVlanID:
      default: 20
      description: Vlan ID for the internal_api network traffic.
      type: number
    StorageNetworkVlanID:
      default: 30
      description: Vlan ID for the storage network traffic.
      type: number
    StorageMgmtNetworkVlanID:
      default: 40
      description: Vlan ID for the storage mgmt network traffic.
      type: number
    TenantNetworkVlanID:
      default: 50
      description: Vlan ID for the tenant network traffic.
      type: number
    ExternalInterfaceDefaultRoute:
      default: '10.0.0.1'
      description: Default route for the external network.
      type: string
    ControlPlaneSubnetCidr: # Override this via parameter_defaults
      default: '24'
      description: The subnet CIDR of the control plane network.
      type: string
    DnsServers: # Override this via parameter_defaults
      default: []
      description: A list of DNS servers (2 max) to add to resolv.conf.
      type: json
    EC2MetadataIp: # Override this via parameter_defaults
      description: The IP address of the EC2 metadata server.
      type: string

  resources:
    OsNetConfigImpl:
      type: OS::Heat::StructuredConfig
      properties:
        group: os-apply-config
        config:
          os_net_config:
            network_config:
              -
                type: interface
                name: nic1
                use_dhcp: false
                addresses:
                  -
                    ip_netmask:
                      list_join:
                        - '/'
                        - - {get_param: ControlPlaneIp}
                          - {get_param: ControlPlaneSubnetCidr}
                routes:
                  -
                    ip_netmask: 169.254.169.254/32
                    next_hop: {get_param: EC2MetadataIp}
              -
                type: ovs_bridge
                name: {get_input: bridge_name}
                dns_servers: {get_param: DnsServers}
                members:
                  -
                    type: ovs_bond
                    name: bond1
                    ovs_options: {get_param: BondInterfaceOvsOptions}
                    members:
                      -
                        type: interface
                        name: nic3
                        primary: true
                      -
                        type: interface
                        name: nic4
                  -
                    type: vlan
                    device: bond1
                    vlan_id: {get_param: ExternalNetworkVlanID}
                    addresses:
                      -
                        ip_netmask: {get_param: ExternalIpSubnet}
                    routes:
                      -
                        ip_netmask: 0.0.0.0/0
                        next_hop: {get_param: ExternalInterfaceDefaultRoute}
                  -
                    type: vlan
                    device: bond1
                    vlan_id: {get_param: InternalApiNetworkVlanID}
                    addresses:
                    -
                      ip_netmask: {get_param: InternalApiIpSubnet}
                  -
                    type: vlan
                    device: bond1
                    vlan_id: {get_param: StorageNetworkVlanID}
                    addresses:
                    -
                      ip_netmask: {get_param: StorageIpSubnet}
                  -
                    type: vlan
                    device: bond1
                    vlan_id: {get_param: StorageMgmtNetworkVlanID}
                    addresses:
                    -
                      ip_netmask: {get_param: StorageMgmtIpSubnet}
                  -
                    type: vlan
                    device: bond1
                    vlan_id: {get_param: TenantNetworkVlanID}
                    addresses:
                    -
                      ip_netmask: {get_param: TenantIpSubnet}

  outputs:
    OS::stack_id:
      description: The OsNetConfigImpl resource.
      value: {get_resource: OsNetConfigImpl}

Configuring Interfaces
----------------------
The individual interfaces may need to be modified. As an example, below are
the modifications that would be required to use the second NIC to connect to
an infrastructure network with DHCP addresses, and to use the third and fourth
NICs for the bond:

Example::

          network_config:
            # Add a DHCP infrastructure network to nic2
            -
              type: interface
              name: nic2
              use_dhcp: true
              defroute: false
            -
              type: ovs_bridge
              name: {get_input: bridge_name}
              members:
                -
                  type: ovs_bond
                  name: bond1
                  ovs_options: {get_param: BondInterfaceOvsOptions}
                  members:
                    # Modify bond NICs to use nic3 and nic4
                    -
                      type: interface
                      name: nic3
                      primary: true
                    -
                      type: interface
                      name: nic4

When using numbered interfaces ("nic1", "nic2", etc.) instead of named
interfaces ("eth0", "eno2", etc.), the network interfaces of hosts within
a role do not have to be exactly the same. For instance, one host may have
interfaces em1 and em2, while another has eno1 and eno2, but both hosts' NICs
can be referred to as nic1 and nic2.

The numbered NIC scheme only takes into account the interfaces that are live
(have a cable attached to the switch). So if you have some hosts with 4
interfaces, and some with 6, you should use nic1-nic4 and only plug in 4
cables on each host.

Configuring Routes and Default Routes
-------------------------------------
There are two ways that a host may have its default routes set. If the interface
is using DHCP, and the DHCP server offers a gateway address, the system will
install a default route for that gateway. Otherwise, a default route may be set
manually on an interface with a static IP.

Although the Linux kernel supports multiple default gateways, it will only use
the one with the lowest metric. If there are multiple DHCP interfaces, this can
result in an unpredictable default gateway. In this case, it is recommended that
defroute=no be set for the interfaces other than the one where we want the
default route. In this case, we want a DHCP interface (NIC 2) to be the default
route (rather than the Provisioning interface), so we disable the default route
on the provisioning interface (note that the defroute parameter only applies
to routes learned via DHCP):

Example::

            # No default route on the Provisioning network
            -
              type: interface
              name: nic1
              use_dhcp: true
              defroute: no
            # Instead use this DHCP infrastructure VLAN as the default route
            -
              type: interface
              name: nic2
              use_dhcp: true

To set a static route on an interface with a static IP, specify a route to the
subnet. For instance, here is a hypothetical route to the 10.1.2.0/24 subnet
via the gateway at 172.17.0.1 on the Internal API network:

Example::

            -
              type: vlan
              device: bond1
              vlan_id: {get_param: InternalApiNetworkVlanID}
              addresses:
              -
                ip_netmask: {get_param: InternalApiIpSubnet}
              routes:
                -
                  ip_netmask: 10.1.2.0/24
                  next_hop: 172.17.0.1

Using a Dedicated Interface For Tenant VLANs
--------------------------------------------
When using a dedicated interface or bond for tenant VLANs, a bridge must be
created. Neutron will create OVS ports on that bridge with the VLAN tags for the
provider VLANs. For example, to use NIC 4 as a dedicated interface for tenant
VLANs, you would add the following to the Controller and Compute templates:

Example::

            -
              type: ovs_bridge
              name: br-vlan
              members:
                -
                  type: interface
                  name: nic4
                  primary: true

A similar configuration may be used to define an interface or a bridge that
will be used for Provider VLANs. Provider VLANs are external networks which
are connected directly to the Compute hosts. VMs may be attached directly to
Provider networks to provide access to datacenter resources outside the cloud.

Using the Native VLAN for Floating IPs
--------------------------------------
By default, Neutron is configured with an empty string for the Neutron external
bridge mapping. This results in the physical interface being patched to br-int,
rather than using br-ex directly (as in previous versions). This model allows
for multiple floating IP networks, using either VLANs or multiple physical
connections.

Example::

  parameter_defaults:
    # May set to br-ex if using floating IPs only on native VLAN on bridge br-ex
    NeutronExternalNetworkBridge: "''"

When using only one floating IP network on the native VLAN of a bridge,
then you can optionally set the Neutron external bridge to e.g. "br-ex". This
results in the packets only having to traverse one bridge (instead of two),
and may result in slightly lower CPU when passing traffic over the floating
IP network.

The next section contains the changes to the NIC config that need to happen
to put the External network on the native VLAN (if the External network is on
br-ex, then that bridge may be used for floating IPs in addition to the Horizon
dashboard and Public APIs).

Using the Native VLAN on a Trunked Interface
--------------------------------------------
If a trunked interface or bond has a network on the native VLAN, then the IP
address will be assigned directly to the bridge and there will be no VLAN
interface.

For example, if the external network is on the native VLAN, the bond
configuration would look like this:

Example::

              -
                type: ovs_bridge
                name: {get_input: bridge_name}
                dns_servers: {get_param: DnsServers}
                addresses:
                  -
                    ip_netmask: {get_param: ExternalIpSubnet}
                routes:
                  -
                    ip_netmask: 0.0.0.0/0
                    next_hop: {get_param: ExternalInterfaceDefaultRoute}
                members:
                  -
                    type: ovs_bond
                    name: bond1
                    ovs_options: {get_param: BondInterfaceOvsOptions}
                    members:
                      -
                        type: interface
                        name: nic3
                        primary: true
                      -
                        type: interface
                        name: nic4

.. note::
  When moving the address (and possibly route) statements onto the bridge, be
  sure to remove the corresponding VLAN interface from the bridge. Make sure to
  make the changes to all applicable roles. The External network is only on the
  controllers, so only the controller template needs to be changed. The Storage
  network on the other hand is attached to all roles, so if the storage network
  were on the default VLAN, all roles would need to be edited.

Configuring Jumbo Frames
------------------------
The Maximum Transmission Unit (MTU) setting determines the maximum amount of
data that can be transmitted by a single Ethernet frame. Using a larger value
can result in less overhead, since each frame adds data in the form of a
header. The default value is 1500, and using a value higher than that will
require the switch port to be configured to support jumbo frames. Most switches
support an MTU of at least 9000, but many are configured for 1500 by default.

The MTU of a VLAN cannot exceed the MTU of the physical interface. Make sure to
include the MTU value on the bond and/or interface.

Storage, Storage Management, Internal API, and Tenant networking can all
benefit from jumbo frames. In testing, tenant networking throughput was
over 300% greater when using jumbo frames in conjunction with VXLAN tunnels.

.. note::
  It is recommended that the Provisioning interface, External interface, and
  any floating IP interfaces be left at the default MTU of 1500. Connectivity
  problems are likely to occur otherwise. This is because routers typically
  cannot forward jumbo frames across L3 boundaries.

Example::

                  -
                    type: ovs_bond
                    name: bond1
                    mtu: 9000
                    ovs_options: {get_param: BondInterfaceOvsOptions}
                    members:
                      -
                        type: interface
                        name: nic3
                        mtu: 9000
                        primary: true
                      -
                        type: interface
                        name: nic4
                        mtu: 9000
                  -
                    # The external interface should stay at default
                    type: vlan
                    device: bond1
                    vlan_id: {get_param: ExternalNetworkVlanID}
                    addresses:
                      -
                        ip_netmask: {get_param: ExternalIpSubnet}
                    routes:
                      -
                        ip_netmask: 0.0.0.0/0
                        next_hop: {get_param: ExternalInterfaceDefaultRoute}
                  -
                    # MTU 9000 for Internal API, Storage, and Storage Management
                    type: vlan
                    device: bond1
                    mtu: 9000
                    vlan_id: {get_param: InternalApiNetworkVlanID}
                    addresses:
                    -
                      ip_netmask: {get_param: InternalApiIpSubnet}

Assigning OpenStack Services to Isolated Networks
-------------------------------------------------
Each OpenStack service is assigned to a network using a default mapping. The
service will be bound to the host IP within the named network on each host.

.. note::
  The services will be assigned to the networks according to the
  ``ServiceNetMap`` in ``overcloud.yaml``. Unless these
  defaults need to be overridden, the ServiceNetMap does not need to be defined
  in the environment file.

A service can be assigned to an alternate network by overriding the service to
network map in an environment file. The defaults should generally work, but
can be overridden. To override these values, add the ServiceNetMap to the
``parameter_defaults`` section of the network environment.

Example::

  parameter_defaults:

    ServiceNetMap:
      NeutronTenantNetwork: tenant
      CeilometerApiNetwork: internal_api
      MongoDbNetwork: internal_api
      CinderApiNetwork: internal_api
      CinderIscsiNetwork: storage
      GlanceApiNetwork: storage
      GlanceRegistryNetwork: internal_api
      KeystoneAdminApiNetwork: internal_api
      KeystonePublicApiNetwork: internal_api
      NeutronApiNetwork: internal_api
      HeatApiNetwork: internal_api
      NovaApiNetwork: internal_api
      NovaMetadataNetwork: internal_api
      NovaVncProxyNetwork: internal_api
      SwiftMgmtNetwork: storage_mgmt
      SwiftProxyNetwork: storage
      HorizonNetwork: internal_api
      MemcachedNetwork: internal_api
      RabbitMqNetwork: internal_api
      RedisNetwork: internal_api
      MysqlNetwork: internal_api
      CephClusterNetwork: storage_mgmt
      CephPublicNetwork: storage
      # Define which network will be used for hostname resolution
      ControllerHostnameResolveNetwork: internal_api
      ComputeHostnameResolveNetwork: internal_api
      BlockStorageHostnameResolveNetwork: internal_api
      ObjectStorageHostnameResolveNetwork: internal_api
      CephStorageHostnameResolveNetwork: storage

.. note::
  If an entry in the ServiceNetMap points to a network which does not exist,
  that service will be placed on the Provisioning network. To avoid that,
  make sure that each entry points to a valid network.

Updating Existing Configuration Templates To Support New Parameters
-------------------------------------------------------------------

The most recent versions of TripleO include support for static Provisioning IPs.
The systems will boot via DHCP during deployment, and the DHCP address assigned
is converted to a static IP. The following parameters have been added to support
static IP addressing on the provisioning network:

* ControlPlaneIp
* ControlPlaneSubnetCidr
* DnsServers
* EC2MetadataIp

These changes require additional parameters for setting static IPs, routes,
and DNS servers. When using static Provisioning IPs, the network environment
file now needs to contain additional resource defaults (customize to match
the environment)::

  parameter_defaults:
    # CIDR subnet mask length for provisioning network
    ControlPlaneSubnetCidr: '24'
    # Gateway router for the provisioning network (or Undercloud IP)
    ControlPlaneDefaultRoute:10.8.146.254
    # Generally the IP of the Undercloud
    EC2MetadataIp: 10.8.146.1
    # Define the DNS servers (maximum 2) for the overcloud nodes
    DnsServers:['8.8.8.8','8.8.4.4']

The NIC config templates for each role now include additional parameters in the
parameters section. Whether the provisioning interface will use DHCP or static
IPs, these parameters are needed in any case::

  parameters:
    ControlPlaneIp:
      default: ''
      description: IP address/subnet on the ctlplane network
      type: string
    ControlPlaneSubnetCidr: # Override this via parameter_defaults
      default: '24'
      description: The subnet CIDR of the control plane network.
      type: string
    DnsServers: # Override this via parameter_defaults
      default: []
      description: A list of DNS servers (2 max) to add to resolv.conf.
      type: json
    EC2MetadataIp: # Override this via parameter_defaults
      description: The IP address of the EC2 metadata server.
      type: string

If you are customizing the templates in the ``network/config`` subdirectory of
the TripleO Heat Templates, you will find that they have been updated with
these parameters. If you have NIC configuration templates from an older version
of TripleO Heat Templates, then you will need to add these parameters and
modify the provisioning network to take advantage of static IP addresses.

Deploying the Overcloud With Network Isolation
----------------------------------------------

When deploying with network isolation, you should specify the NTP server for the
overcloud nodes. If the clocks are not synchronized, some OpenStack services may
be unable to start, especially when using HA. The NTP server should be reachable
from both the External and Provisioning subnets. The neutron network type should
be specified, along with the tunneling or VLAN parameters. Specify the libvirt
type if on bare metal, so that hardware virtualization will be used.

To deploy with network isolation and include the network environment file, use
the ``-e`` parameters with the ``openstack overcloud deploy`` command. For
instance, to deploy VXLAN mode, the deployment command might be::

    openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
    -e /home/stack/templates/network-environment.yaml \
    --ntp-server pool.ntp.org \
    --neutron-network-type vxlan \
    --neutron-tunnel-types vxlan

To deploy with VLAN mode, you should specify the range of VLANs that will be
used for tenant networks::

    openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
    -e /home/stack/templates/network-environment.yaml \
    --ntp-server pool.ntp.org \
    --neutron-network-type vlan \
    --neutron-bridge-mappings datacentre:br-ex \
    --neutron-network-vlan-ranges datacentre:30:100

If a dedicated interface or bridge is used for tenant VLANs or provider
networks, it should be included in the bridge mappings. For instance, if the
tenant VLANs were on a bridge named ``br-vlan``, then use these values in the
deployment command above::

    --neutron-bridge-mappings datacentre:br-ex,tenant:br-vlan \
    --neutron-network-vlan-ranges tenant:30:100

.. note::

    You must also pass the environment files (again using the ``-e`` or
    ``--environment-file`` option) whenever you make subsequent changes to the
    overcloud, such as :doc:`../post_deployment/scale_roles`,
    :doc:`../post_deployment/delete_nodes` or
    :doc:`../post_deployment/package_update`.

Creating Floating IP Networks
-----------------------------

In order to provide external connectivity and floating IPs to the VMs, an
external network must be created. The physical network is referred to by the
name used in the Neutron bridge mappings when deployed. The default bridge
mapping is ``datacentre:br-ex``, which maps the physical network name
``datacentre`` to the bridge ``br-ex`` which includes the physical network
link. For instance, to create a floating IP network on the br-ex bridge on
VLAN 104, this command is used::

    neutron net-create ext-net --router:external \
    --provider:physical_network datacentre \
    --provider:network_type vlan \
    --provider:segmentation_id 104

If the floating IP network is on the native VLAN of br-ex, then a different
command is used to create the external network::

    neutron net-create ext-net --router:external \
    --provider:physical_network datacentre \
    --provider:network_type flat

Floating IP networks do not have to use br-ex, they can use any bridge as
long as the NeutronExternalNetworkBridge is set to "''". If the floating IP
network were going to be placed on a bridge named "br-floating", and the
deployment command included the bridge mapping of
``datacenter:br-ex,floating:br-floating``, then following command would be used
to create a floating IP network on VLAN 105::

    neutron net-create ext-net --router:external \
        --provider:physical_network floating \
        --provider:network_type vlan \
        --provider:segmentation_id 105

Then a range of IP addresses must be assigned in the floating IP subnet and
assigned to the physical network. The Subnet will be associated with the network
name that was created in the previous step (``ext-net``)::

    neutron subnet-create --name ext-subnet \
    --enable_dhcp=False \
    --allocation-pool start=10.8.148.50,end=10.8.148.100 \
    --gateway 10.8.148.254 \
    ext-net 10.8.148.0/24

Creating Provider Networks
--------------------------

A Provider Network is a network which is attached physically to a datacenter
network that exists outside of the deployed overcloud. This can be an existing
infrastructure network, or a network which provides external access directly to
VMs via routing instead of floating IPs.

When a provider network is created, it is associated with a physical network
with a bridge mapping, similar to how floating IP networks are created. The
provider network being added must be attached to both the controller and the
compute nodes, since the compute node will attach a VM virtual network
interface directly to an attached network interface.

For instance, if the provider network being added is a VLAN on the br-ex
bridge, then this command would add a provider network on VLAN 201::

    neutron net-create --provider:physical_network datacentre \
    --provider:network_type vlan --provider:segmentation_id 201 \
    --shared provider_network

This command would create a shared network, but it is also possible to
specify a tenant instead of specifying --shared, and then that network will
only be available to that tenant. If a provider network is marked as external,
then only the operator may create ports on that network. A subnet can be added
to a provider network if Neutron is to provide DHCP services to tenant VMs::

    neutron subnet-create --name provider-subnet \
    --enable_dhcp=True \
    --allocation-pool start=10.9.101.50,end=10.9.101.100 \
    --gateway 10.9.101.254 \
    provider_network 10.9.101.0/24
