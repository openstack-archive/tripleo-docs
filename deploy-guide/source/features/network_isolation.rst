.. _network_isolation:

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
* External (Public OpenStack APIs, Horizon dashboard, optionally floating IPs)
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
* Tenant
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

#. Create and edit network_data.yaml file for the cluster
#. Generate templates from Jinja2
#. Create network environment overrides file (e.g. ~/network-environment-overrides.yaml)
#. Make a copy of the appropriate sample network interface configurations
#. Edit the network interface configurations to match local environment
#. Deploy overcloud with the proper parameters to include network isolation

The next section will walk through the elements that need to be added to
the network-environment.yaml to enable network isolation. The sections
after that deal with configuring the network interface templates. The final step
will deploy the overcloud with network isolation and a custom environment.

Create and Edit network_data.yaml file for the Cluster
------------------------------------------------------

Copy the default ``network_data.yaml`` file and customize the networks, IP
subnets, VLANs, etc., as per the cluster requirements::

  $ cp /usr/share/openstack-tripleo-heat-templates/network_data.yaml ~/templates/network_data.yaml


Generate Templates from Jinja2
------------------------------

With Queens cycle, the network configuration templates have been converted to
Jinja2 templates, so that templates can be generated for each role with
customized network data. A utility script is available to generate the
templates based on the provided ``roles_data.yaml`` and ``network_data.yaml``
inputs.

Before generating the templates, ensure that the ``roles_data.yaml`` is
generated as per the cluster requirements using the command::

  $ openstack overcloud roles generate -o ~/templates/roles_data.yaml Controller Compute \
        BlockStorage ObjectStorage CephStorage

.. note::
  If the default ``roles_data.yaml`` or ``network_data.yaml`` file suits the
  needs of the cluster, then there is no need to generate/customize the files,
  the default files can be used as is for generating the templates.

To generate the templates, run::

  $ /usr/share/openstack-tripleo-heat-templates/tools/process-templates.py \
        -p /usr/share/openstack-tripleo-heat-templates \
        -r ~/templates/roles_data.yaml \
        -n ~/templates/network_data.yaml \
        -o ~/generated-openstack-tripleo-heat-templates --safe

Now the temporary directory ``~/generated-openstack-tripleo-heat-templates``
contains the generated template files according to provided role and network
data. Copy the required templates to a user specific template directory
``~/templates`` to modify the content to suit the cluster needs. Some of the
specific use of generated templates are explained by some of the below
sections.

Create Network Environment Overrides File
-----------------------------------------

The environment file will describe the network environment and will point to
the network interface configuration files to use for the overcloud nodes.

Earlier method of generating network interface configurations with heat has
been deprecated since victoria. To use a custom network configuration copy
an appropriate sample network interface configuration file from
`tripleo-ansible <tripleo_ansible_>`_  and make necessary changes.

Then copy the generated
``net-single-nic-with-vlans.yaml`` file to apply the required cluster specific
changes, which overrides the defaults::

  $ cp ~/generated-openstack-tripleo-heat-templates/environments/net-single-nic-with-vlans.yaml \
        ~/templates/network-environment-overrides.yaml

Add any other parameters which should be overridden from the defaults to this
environment file. It is important for the ``ExternalInterfaceDefaultRoute`` to
be reachable on the subnet that is used for ``ExternalNetCidr``. This will
allow the OpenStack Public APIs and the Horizon Dashboard to be reachable.
Without a valid default route, the post-deployment steps cannot be performed.

.. note::

   The ``parameter_defaults`` section of the ``network-environment-overrides.yaml``
   contains pointers to the network interface configuration files for the deployed
   roles. These files must exist at the path referenced here.

Example::

  parameter_defaults:
    ControllerNetworkConfigTemplate: 'templates/single_nic_vlans/single_nic_vlans.j2'
    ComputeNetworkConfigTemplate: 'templates/single_nic_vlans/single_nic_vlans.j2'
    BlockStorageNetworkConfigTemplate: 'templates/single_nic_vlans/single_nic_vlans_storage.j2'

    # May set to br-ex if using floating IPs only on native VLAN on bridge br-ex
    NeutronExternalNetworkBridge: "''"
    NeutronNetworkType: 'vxlan,vlan'
    NeutronTunnelTypes: 'vxlan'
    # Customize bonding options if required (ignored if bonds are not used)
    BondInterfaceOvsOptions: "lacp=active other-config:lacp-fallback-ab=true"


Users can still use the old network interface configuration heat templates
for custom network configuration. Set ``NetworkConfigWithAnsible`` parameter
to ``false`` to use them::

  parameter_defaults:
    NetworkConfigWithAnsible: false


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

Configure Bonding Options
-----------------------------------

The example bonding options will try to negotiate LACP, but will fallback to
active-backup if LACP cannot be established::

  BondInterfaceOvsOptions:
    "lacp=active other-config:lacp-fallback-ab=true"

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

Start by copying the existing templates in `tripleo-ansible <tripleo_ansible_>`_.
The first example copies the templates which include network bonding. The second
example copies the templates which use a single network interface with multiple
VLANs (this configuration is mostly intended for testing).

.. _tripleo_ansible: https://opendev.org/openstack/tripleo-ansible/src/branch/master/tripleo_ansible/roles/tripleo_network_config/templates

To copy the bonded example interface configurations, run::

    $ cp /usr/share/ansible/roles/tripleo_network_config/templates/bonds_vlans/* \
          ~/templates/nic-configs

To copy the single NIC with VLANs example interface configurations, run::

    $ cp /usr/share/ansible/roles/tripleo_network_config/templates/single_nic_vlans/* \
          ~/templates/nic-configs

Or, if you have custom NIC templates from another source, copy them to the
location referenced in the ``parameter_defaults`` section of the environment
file.

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

    ---
    {% set mtu_list = [ctlplane_mtu] %}
    {% for network in role_networks %}
    {{ mtu_list.append(lookup('vars', networks_lower[network] ~ '_mtu')) }}
    {%- endfor %}
    {% set min_viable_mtu = mtu_list | max %}
    network_config:
    - type: interface
      name: nic1
      mtu: {{ ctlplane_mtu }}
      use_dhcp: false
      addresses:
      - ip_netmask: {{ ctlplane_ip }}/{{ ctlplane_subnet_cidr }}
      routes: {{ ctlplane_host_routes }}
    - type: ovs_bridge
      name: {{ neutron_physical_bridge_name }}
      dns_servers: {{ ctlplane_dns_nameservers }}
      domain: {{ dns_search_domains }}
      members:
      - type: ovs_bond
        name: bond1
        mtu: {{ min_viable_mtu }}
        ovs_options: {{ bond_interface_ovs_options }}
        members:
        - type: interface
          name: nic3
          mtu: {{ min_viable_mtu }}
          primary: true
        - type: interface
          name: nic4
          mtu: {{ min_viable_mtu }}
    {% for network in role_networks %}
      - type: vlan
        mtu: {{ lookup('vars', networks_lower[network] ~ '_mtu') }}
        vlan_id: {{ lookup('vars', networks_lower[network] ~ '_vlan_id') }}
        addresses:
        - ip_netmask: {{ lookup('vars', networks_lower[network] ~ '_ip') }}/{{ lookup('vars', networks_lower[network] ~ '_cidr') }}
        routes: {{ lookup('vars', networks_lower[network] ~ '_host_routes') }}
    {%- endfor %}

.. note::
  If you are using old heat network interface configuration templates from versions
  prior to Victoria, you may need to make updates to the templates. See below.

Updating Existing Network Interface Configuration Templates
-----------------------------------------------------------

Prior to Victoria relase the network interface configuration file
used a ``OS::Heat::SoftwareConfig`` resource to configure interfaces::

  resources:
    OsNetConfigImpl:
      type: OS::Heat::SoftwareConfig
      properties:
        group: script
        config:
          str_replace:
            template:
              get_file: /usr/share/openstack-tripleo-heat-templates/network/scripts/run-os-net-config.sh
            params:
              $network_config:
                network_config:
                  [NETWORK INTERFACE CONFIGURATION HERE]

These templates are now expected to use ``OS::Heat::Value`` resource::

  resources:
    OsNetConfigImpl:
      type: OS::Heat::Value
      properties:
        value:
          network_config:
            [NETWORK INTERFACE CONFIGURATION HERE]
  outputs:
    config:
      value: get_attr[OsNetConfigImpl, value]



Old network inteface configuration heat templates can be converted using
the provided conversion `convert-nic-config.py <convert_nic_config_>`_ script.

.. _convert_nic_config: https://opendev.org/openstack/tripleo-heat-templates/src/branch/master/tools/convert_nic_config.py


Prior to the Ocata release, the network interface configuration files used
a different mechanism for running os-net-config. Ocata introduced the
run-os-net-config.sh script, and the old mechanism was deprecated. The
deprecated mechanism was removed in Queens, so older templates must be
updated. The resource definition must be changed, and {get_input: bridge_name} is
replaced with the special token "bridge_name", which will be replaced with
the value of the NeutronPhysicalBridge.

Old Header::

  resources:
    OsNetConfigImpl:
      type: OS::Heat::StructuredConfig
      properties:
        group: os-apply-config
        config:
          os_net_config:
            network_config:
              [NETWORK INTERFACE CONFIGURATION HERE]

New Header::

  resources:
    OsNetConfigImpl:
      type: OS::Heat::Value
      properties:
        value:
          network_config:
            [NETWORK INTERFACE CONFIGURATION HERE]

Old Bridge Definition::

  - type: ovs_bridge
    name: {get_input: bridge_name}

New Bridge Definition::

  - type: ovs_bridge
    name: bridge_name

Configuring Interfaces
----------------------
The individual interfaces may need to be modified. As an example, below are
the modifications that would be required to use the second NIC to connect to
an infrastructure network with DHCP addresses, and to use the third and fourth
NICs for the bond:

Example::

    network_config:
    - type: interface
      name: nic2
      mtu: {{ ctlplane_mtu }}
      use_dhcp: true
      defroute: no
    - type: ovs_bridge
      name: {{ neutron_physical_bridge_name }}
      members:
      - type: ovs_bond
        name: bond1
        mtu: {{ min_viable_mtu }}
        ovs_options: {{ bound_interface_ovs_options }}
        members:
        - type: interface
          name: nic3
          mtu: {{ min_viable_mtu }}
          primary: true
        - type: interface
          name: nic4
          mtu: {{ min_viable_mtu }}

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

    network_config:
    - type: interface
      name: nic1
      use_dhcp: true
      defroute: no
    - type: interface
      name: nic2
      use_dhcp: true

To set a static route on an interface with a static IP, specify a route to the
subnet. For instance, here is a hypothetical route to the 10.1.2.0/24 subnet
via the gateway at 172.17.0.1 on the Internal API network:

Example::

    - type: vlan
      device: bond1
      vlan_id: {{ internal_api_vlan_id }}
      addresses:
      - ip_netmask: {{ internal_api_ip ~ '/' ~ internal_api_cidr }}
      routes:
      - ip_netmask: 10.1.2.0/24
        next_hop: 172.17.0.1


Using a Dedicated Interface For Tenant VLANs
--------------------------------------------
When using a dedicated interface or bond for tenant VLANs, a bridge must be
created. Neutron will create OVS ports on that bridge with the VLAN tags for the
provider VLANs. For example, to use NIC 4 as a dedicated interface for tenant
VLANs, you would add the following to the Controller and Compute templates:

Example::

    - type: ovs_bridge
      name: br-vlan
      members:
      - type: interface
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

      - type: ovs_bridge
        name: bridge_name
        dns_servers: {{ ctlplane_dns_nameservers }}
        addresses:
        - ip_netmask: {{ external_ip ~ '/' ~ external_cidr }}
        routes: {{ external_host_routes }}
        members:
        - type: ovs_bond
          name: bond1
          ovs_options: {{ bond_interface_ovs_options }}
          members:
          - type: interface
            name: nic3
            primary: true
          - type: interface
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

                  - type: ovs_bond
                    name: bond1
                    mtu: 9000
                    ovs_options: {{ bond_interface_ovs_options }}
                    members:
                    - type: interface
                      name: nic3
                      mtu: 9000
                      primary: true
                    - type: interface
                      name: nic4
                      mtu: 9000
                  - type: vlan
                    device: bond1
                    vlan_id: {{ external_vlan_id }}
                    addresses:
                    - ip_netmask: {{ external_ip ~ '/' ~ external_cidr }}
                    routes: {{ external_host_routes }}
                  - type: vlan
                    device: bond1
                    mtu: 9000
                    vlan_id: {{ internal_api_vlan_id }}
                    addresses:
                    - ip_netmask: {{ internal_api_ip ~ '/' ~ internal_api_cidr }}

Assigning OpenStack Services to Isolated Networks
-------------------------------------------------
Each OpenStack service is assigned to a network using a default mapping. The
service will be bound to the host IP within the named network on each host.

.. note::
  The services will be assigned to the networks according to the
  ``ServiceNetMap`` in ``network/service_net_map.j2.yaml``. Unless these
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

Deploying the Overcloud With Network Isolation
----------------------------------------------

When deploying with network isolation, you should specify the NTP server for the
overcloud nodes. If the clocks are not synchronized, some OpenStack services may
be unable to start, especially when using HA. The NTP server should be reachable
from both the External and Provisioning subnets. The neutron network type should
be specified, along with the tunneling or VLAN parameters. Specify the libvirt
type if on bare metal, so that hardware virtualization will be used.

To deploy with network isolation and include the network environment file, use
the ``-e`` parameters with the ``openstack overcloud deploy`` command. The
following deploy command should work for all of the subsequent examples::

    openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/network-environment.yaml \
    -e ~/templates/network-environment-overrides.yaml \
    --ntp-server pool.ntp.org

To deploy VXLAN mode ``network-environment-overrides.yaml`` should contain the
following parameter values::

    NeutronNetworkType: vxlan
    NeutronTunnelTypes: vxlan

To deploy with VLAN mode, you should specify the range of VLANs that will be
used for tenant networks.  ``network-environment.yaml`` might contain the
following parameter values::

    NeutronNetworkType: vlan
    NeutronBridgeMappings: 'datacentre:br-ex'
    NeutronNetworkVLANRanges: 'datacentre:100:199'

If a dedicated interface or bridge is used for tenant VLANs or provider
networks, it should be included in the bridge mappings. For instance, if the
tenant VLANs were on a bridge named ``br-vlan``, then use these values in
``network-environment.yaml``::

    NeutronBridgeMappings: 'datacentre:br-ex,tenant:br-vlan'
    NeutronNetworkVLANRanges: 'tenant:200:299'

.. note::

    You must also pass the environment files (again using the ``-e`` or
    ``--environment-file`` option) whenever you make subsequent changes to the
    overcloud, such as :doc:`../post_deployment/scale_roles`,
    :doc:`../post_deployment/delete_nodes` or
    :doc:`../post_deployment/upgrade/minor_update`.

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
    --allocation-pool start=10.0.2.50,end=10.0.2.100 \
    --gateway 10.0.2.254 \
    ext-net 10.0.2.0/24

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
specify a tenant instead of specifying ``--shared``, and then that network will
only be available to that tenant. If a provider network is marked as external,
then only the operator may create ports on that network. A subnet can be added
to a provider network if Neutron is to provide DHCP services to tenant VMs::

    neutron subnet-create --name provider-subnet \
    --enable_dhcp=True \
    --allocation-pool start=10.0.3.50,end=10.0.3.100 \
    --gateway 10.0.3.254 \
    provider_network 10.0.3.0/24
