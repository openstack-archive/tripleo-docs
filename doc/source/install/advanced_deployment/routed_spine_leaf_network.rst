Deploying Overcloud with L3 routed networking
=============================================

Layer 3 Routed spine and leaf architectures is gaining in popularity due to the
benefits, such as high-performance, increased scalability and reduced failure
domains.

The below diagram is an example L3 routed
`Clos <https://en.wikipedia.org/wiki/Clos_network>`_ architecture. In this
example each server is connected to top-of-rack leaf switches. Each leaf switch
is attached to each spine switch. Within each rack, all servers share a layer 2
domain. The layer 2 network segments are local to the rack. Layer 3 routing via
the spine switches permits East-West traffic between the racks:

.. image:: ../_images/spine_and_leaf.svg

.. Note:: Typically Dynamic Routing is implemented in such an architecture.
          Often also
          `ECMP <https://en.wikipedia.org/wiki/Equal-cost_multi-path_routing>`_
          (Equal-cost multi-path routing) and
          `BFD <https://en.wikipedia.org/wiki/Bidirectional_Forwarding_Detection>`_
          (Bidirectional Forwarding Detection) are used to provide non-blocking
          forwarding and fast convergence times in case of failures.
          Configuration of the underlying network architecture is not in the
          scope of this document.

Layer 3 routed Requirements
---------------------------

For TripleO to deploy the ``overcloud`` on a network with a layer 3 routed
architecture the following requirements must be met:

* **Layer 3 routing**:
  The network infrastructure must have *routing* configured to enable traffic
  between the different layer 2 segments. This can be statically or dynamically
  configured.

* **DHCP-Relay**:
  Each layer 2 segment that is not local to the ``undercloud`` must provide
  *dhcp-relay*. DHCP requests must be forwarded to the Undercloud on the
  provisioning network segment where the ``undercloud`` is connected.

  .. Note:: The ``undercloud`` uses two DHCP servers. One for baremetal node
            introspection, and another for deploying overcloud nodes.

            Make sure to read `DHCP relay configuration`_ to understand the
            requirements when configuring *dhcp-relay*.

Layer 3 routed Limitations
--------------------------

* Some roles, such as the Controller role, use virtual IP addresses and
  clustering. The mechanism behind this functionality requires layer-2 network
  connectivity between these nodes. These nodes must all be placed within the
  same leaf.

* Similar restrictions apply to networker nodes. The Network service implements
  highly-available default paths in the network using Virtual Router Redundancy
  Protocol (VRRP). Since VRRP uses a virtual router ip address, master and
  backup nodes must be connected to the same L2 network segment.

* When using tenant or provider networks with VLAN segmentation, the particular
  VLANs used must be shared between all networker and compute nodes.

  .. Note:: It is possible to configure the Network service with multiple sets
            of networker nodes. Each set would share routes for their networks,
            and VRRP would be used within each set of networker nodes to
            provide highly-available default paths. In such configuration all
            networker nodes sharing networks must be on the same L2 network
            segment.

Create undercloud configuration
-------------------------------

To deploy the ``overcloud`` on a L3 routed architecture the ``undercloud``
needs to be configured with multiple neutron network segments and subnets on
the ``ctlplane`` network.

#. In the ``[DEFAULT]`` section of ``undercloud.conf`` enable the routed
   networks feature by setting ``enable_routed_networks`` to ``true``. For
   example::

     enable_routed_networks = true

#. In the ``[DEFAULT]`` section of ``undercloud.conf`` add a comma separated
   list of control plane subnets. Define one subnet for each layer 2 segment in
   the routed spine and leaf. For example::

     subnets = leaf0,leaf1,leaf2

#. In the ``[DEFAULT]`` section of ``undercloud.conf`` specify the subnet that
   is associated with the physical layer 2 segment that is *local* to the
   ``undercloud``. For example::

     local_subnet = leaf0

#. For each of the control plane subnets specified in ``[DEFAULT]\subnets``
   add an additional section in ``undercloud.conf``, for example::

     [leaf0]
     cidr = 192.168.10.0/24
     dhcp_start = 192.168.10.10
     dhcp_end = 192.168.10.90
     inspection_iprange = 192.168.10.100,192.168.10.190
     gateway = 192.168.10.1
     masquerade = False

     [leaf1]
     cidr = 192.168.11.0/24
     dhcp_start = 192.168.11.10
     dhcp_end = 192.168.11.90
     inspection_iprange = 192.168.11.100,192.168.11.190
     gateway = 192.168.11.1
     masquerade = False

     [leaf2]
     cidr = 192.168.12.0/24
     dhcp_start = 192.168.12.10
     dhcp_end = 192.168.12.90
     inspection_iprange = 192.168.12.100,192.168.12.190
     gateway = 192.168.12.1
     masquerade = False

Install the undercloud
----------------------

Once the ``undercloud.conf`` is updated with the desired configuration, install
the undercloud by running the following command::

  openstack undercloud install

Once the ``undercloud`` is installed complete the post-install tasks such as
uploading images and registering baremetal nodes. (For addition details
regarding the post-install tasks, see
:doc:`../basic_deployment/basic_deployment_cli`.)

DHCP relay configuration
------------------------

The TripleO Undercloud uses two DHCP servers on the provisioning network, one
for ``introspection`` and another one for ``provisioning``. When configuring
*dhcp-relay* make sure that DHCP requests are forwarded to both DHCP servers on
the Undercloud.

For devices that support it, UDP *broadcast* can be used to relay DHCP requests
to the L2 network segment where the Undercloud provisioning network is
connected. Alternatively UDP *unicast* can be can be used, in this case DHCP
requests are relayed to specific ip addresses.

.. Note:: Configuration of *dhcp-relay* on specific devices types is beyond the
          scope of this document. As a reference
          `DHCP relay configuration (Example)`_ using the implementation in
          `ISC DHCP software <https://www.isc.org/downloads/dhcp/>`_ is
          available below. (Please refer to manual page
          `dhcrelay(8) <https://linux.die.net/man/8/dhcrelay>`_ for further
          details on how to use this implementation.)


Broadcast DHCP relay
~~~~~~~~~~~~~~~~~~~~

DHCP requests are relayed onto the L2 network segment where the DHCP server(s)
reside using UDP *broadcast* traffic. All devices on the network segment will
receive the broadcast traffic. When using UDP *broadcast* both DHCP servers on
the Undercloud will receive the relayed DHCP request.

Depending on implementation this is typically configured by specifying either
*interface* or *ip network address*:

* **Interface**:
  Specifying an interface connected to the L2 network segment where the DHCP
  requests will be relayed.
* **IP network address**:
  Specifying the network address of the IP network where the DHCP request will
  be relayed.

Unicast DHCP relay
~~~~~~~~~~~~~~~~~~

DHCP requests are relayed to specific DHCP servers using UDP *unicast* traffic.
When using UDP *unicast* the device configured to provide *dhcp-relay* must be
configured to relay DHCP requests to both the IP address assigned to the
interface used for *introspection* on the Undercloud and the IP address of the
network namespace created by the Network service to host the DHCP service for
the ``ctlplane`` network.

The interface used for *introspection* is the one defined as
``inspection_interface`` in ``undercloud.conf``.

.. Note:: It is common to use the ``br-ctlplane`` interface for introspection,
          the IP address defined as ``local_ip`` in ``undercloud.conf`` will be
          on the ``br-ctlplane`` interface.

The IP address allocated to the neutron DHCP namespace will typically be the
first address available in the IP range configured for the ``local_subnet`` in
``undercloud.conf``. (The first address in the IP range is the one defined as
``dhcp_start`` in the configuration.) For example: ``172.20.0.10`` would be the
IP address when the following configuration is used::

  [DEFAULT]
  local_subnet = leaf0
  subnets = leaf0,leaf1,leaf2

  [leaf0]
  cidr = 172.20.0.0/26
  dhcp_start = 172.20.0.10
  dhcp_end = 172.20.0.19
  inspection_iprange = 172.20.0.20,172.20.0.29
  gateway = 172.20.0.62
  masquerade = False

.. Warning:: The IP address for the DHCP namespace is automatically allocated,
             it will in most cases be the first address in the IP range, but
             do make sure to verify that this is the case by running the
             following commands on the Undercloud::

               $ openstack port list --device-owner network:dhcp -c "Fixed IP Addresses"
               +----------------------------------------------------------------------------+
               | Fixed IP Addresses                                                         |
               +----------------------------------------------------------------------------+
               | ip_address='172.20.0.10', subnet_id='7526fbe3-f52a-4b39-a828-ec59f4ed12b2' |
               +----------------------------------------------------------------------------+
               $ openstack subnet show 7526fbe3-f52a-4b39-a828-ec59f4ed12b2 -c name
               +-------+--------+
               | Field | Value  |
               +-------+--------+
               | name  | leaf0  |
               +-------+--------+

DHCP relay configuration (Example)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the following example ``dhcrelay`` from
`ISC DHCP software <https://www.isc.org/downloads/dhcp/>`_ is started using
configuration parameters to relay incoming DHCP request on interfaces:
``eth1``, ``eth2`` and ``eth3``. The undercloud DHCP servers are on the network
segment connected to the ``eth0`` interface. The DHCP server used for
``introspection`` is listening on ip address: ``172.20.0.1`` and the DHCP
server used for ``provisioning`` is listening on ip address: ``172.20.0.10``::

  dhcrelay -d --no-pid 172.20.0.10 172.20.0.1 \
           -iu eth0 -id eth1 -id eth2 -id eth3

Map bare metal node ports to control plane network segments
-----------------------------------------------------------

To enable deployment onto a L3 routed network the baremetal ports must have
its ``physical_network`` field configured. Each baremetal port is associated
with a baremetal node in the Bare Metal service. The physical network names are
the ones used in the ``subnets`` option in the undercloud configuration.

.. Note:: The physical network name of the subnet specified as ``local_subnet``
          in ``undercloud.conf`` is special. It is **always** named
          ``ctlplane``.

#. Make sure the baremetal nodes are in one of the following states: *enroll*,
   or *manageable*. If the baremetal node is not in one of these states the
   command used to set the ``physical_network`` property on the baremetal port
   will fail. (For additional details regarding node states see
   :doc:`../advanced_deployment/node_states`.)

   To set all nodes to ``manageable`` state run the following command::

       for node in $(openstack baremetal node list -f value -c Name); do \
           openstack baremetal node manage $node --wait; done

#. Use ``openstack baremetal port list --node <node-uuid>`` command to find out
   which baremetal ports are associated with which baremetal node. Then set the
   ``physical-network`` for the ports.

   In the example below three subnets where defined in the configuration,
   *leaf0*, *leaf1* and *leaf2*. Notice that the ``local_subnet`` is ``leaf0``,
   since the physical network for the ``local_subnet`` is always ``ctlplane``
   the baremetal port connected to ``leaf0`` use ``ctlplane``. The remaining
   ports use the ``leafX`` names::

     openstack baremetal port set --physical-network ctlplane <port-uuid>

     openstack baremetal port set --physical-network leaf1 <port-uuid>
     openstack baremetal port set --physical-network leaf2 <port-uuid>
     openstack baremetal port set --physical-network leaf2 <port-uuid>

#. Make sure the nodes are in ``available`` state before deploying the
   overcloud::

    openstack overcloud node provide --all-manageable

Create roles specific to each leaf (layer 2 segment)
----------------------------------------------------

To aid in scheduling and to allow override of leaf specific parameters in
``tripleo-heat-templates`` create new roles for each l2 leaf. The following is
an example with one controller role, and two compute roles. Please refer to
:doc:`custom_roles` for details on configuring custom roles.

Example ``roles_data``::

  #############################################################################
  # Role: Controller                                                          #
  #############################################################################
  - name: Controller
    description: |
      Controller role that has all the controler services loaded and handles
      Database, Messaging and Network functions.
    CountDefault: 1
    tags:
      - primary
      - controller
    networks:
      - External
      - InternalApi
      - Storage
      - StorageMgmt
      - Tenant
    HostnameFormatDefault: '%stackname%-controller-%index%'
    ServicesDefault:
      - OS::TripleO::Services::AodhApi
      - OS::TripleO::Services::AodhEvaluator
      - OS::TripleO::Services::AodhListener
      - OS::TripleO::Services::AodhNotifier
      - OS::TripleO::Services::AuditD
      - OS::TripleO::Services::BarbicanApi
      - OS::TripleO::Services::BarbicanBackendSimpleCrypto
      - OS::TripleO::Services::BarbicanBackendDogtag
      - OS::TripleO::Services::BarbicanBackendKmip
      - OS::TripleO::Services::BarbicanBackendPkcs11Crypto
      - OS::TripleO::Services::CACerts
      - OS::TripleO::Services::CeilometerAgentCentral
      - OS::TripleO::Services::CeilometerAgentNotification
      - OS::TripleO::Services::CephExternal
      - OS::TripleO::Services::CephMds
      - OS::TripleO::Services::CephMgr
      - OS::TripleO::Services::CephMon
      - OS::TripleO::Services::CephRbdMirror
      - OS::TripleO::Services::CephRgw
      - OS::TripleO::Services::CertmongerUser
      - OS::TripleO::Services::CinderApi
      - OS::TripleO::Services::CinderBackendDellPs
      - OS::TripleO::Services::CinderBackendDellSc
      - OS::TripleO::Services::CinderBackendDellEMCUnity
      - OS::TripleO::Services::CinderBackendDellEMCVMAXISCSI
      - OS::TripleO::Services::CinderBackendNetApp
      - OS::TripleO::Services::CinderBackendScaleIO
      - OS::TripleO::Services::CinderBackendVRTSHyperScale
      - OS::TripleO::Services::CinderBackup
      - OS::TripleO::Services::CinderHPELeftHandISCSI
      - OS::TripleO::Services::CinderScheduler
      - OS::TripleO::Services::CinderVolume
      - OS::TripleO::Services::Clustercheck
      - OS::TripleO::Services::Collectd
      - OS::TripleO::Services::Congress
      - OS::TripleO::Services::Docker
      - OS::TripleO::Services::Ec2Api
      - OS::TripleO::Services::Etcd
      - OS::TripleO::Services::ExternalSwiftProxy
      - OS::TripleO::Services::Fluentd
      - OS::TripleO::Services::GlanceApi
      - OS::TripleO::Services::GnocchiApi
      - OS::TripleO::Services::GnocchiMetricd
      - OS::TripleO::Services::GnocchiStatsd
      - OS::TripleO::Services::HAproxy
      - OS::TripleO::Services::HeatApi
      - OS::TripleO::Services::HeatApiCfn
      - OS::TripleO::Services::HeatEngine
      - OS::TripleO::Services::Horizon
      - OS::TripleO::Services::Ipsec
      - OS::TripleO::Services::IronicApi
      - OS::TripleO::Services::IronicConductor
      - OS::TripleO::Services::IronicPxe
      - OS::TripleO::Services::Iscsid
      - OS::TripleO::Services::Keepalived
      - OS::TripleO::Services::Kernel
      - OS::TripleO::Services::Keystone
      - OS::TripleO::Services::LoginDefs
      - OS::TripleO::Services::ManilaApi
      - OS::TripleO::Services::ManilaBackendCephFs
      - OS::TripleO::Services::ManilaBackendIsilon
      - OS::TripleO::Services::ManilaBackendNetapp
      - OS::TripleO::Services::ManilaBackendUnity
      - OS::TripleO::Services::ManilaBackendVNX
      - OS::TripleO::Services::ManilaBackendVMAX
      - OS::TripleO::Services::ManilaScheduler
      - OS::TripleO::Services::ManilaShare
      - OS::TripleO::Services::Memcached
      - OS::TripleO::Services::MongoDb
      - OS::TripleO::Services::MySQL
      - OS::TripleO::Services::MySQLClient
      - OS::TripleO::Services::NeutronApi
      - OS::TripleO::Services::NeutronBgpVpnApi
      - OS::TripleO::Services::NeutronSfcApi
      - OS::TripleO::Services::NeutronCorePlugin
      - OS::TripleO::Services::NeutronDhcpAgent
      - OS::TripleO::Services::NeutronL2gwAgent
      - OS::TripleO::Services::NeutronL2gwApi
      - OS::TripleO::Services::NeutronL3Agent
      - OS::TripleO::Services::NeutronLbaasv2Agent
      - OS::TripleO::Services::NeutronLinuxbridgeAgent
      - OS::TripleO::Services::NeutronMetadataAgent
      - OS::TripleO::Services::NeutronML2FujitsuCfab
      - OS::TripleO::Services::NeutronML2FujitsuFossw
      - OS::TripleO::Services::NeutronOvsAgent
      - OS::TripleO::Services::NeutronVppAgent
      - OS::TripleO::Services::NovaApi
      - OS::TripleO::Services::NovaConductor
      - OS::TripleO::Services::NovaConsoleauth
      - OS::TripleO::Services::NovaIronic
      - OS::TripleO::Services::NovaMetadata
      - OS::TripleO::Services::NovaPlacement
      - OS::TripleO::Services::NovaScheduler
      - OS::TripleO::Services::NovaVncProxy
      - OS::TripleO::Services::Ntp
      - OS::TripleO::Services::ContainersLogrotateCrond
      - OS::TripleO::Services::OctaviaApi
      - OS::TripleO::Services::OctaviaHealthManager
      - OS::TripleO::Services::OctaviaHousekeeping
      - OS::TripleO::Services::OctaviaWorker
      - OS::TripleO::Services::OpenDaylightApi
      - OS::TripleO::Services::OpenDaylightOvs
      - OS::TripleO::Services::OVNDBs
      - OS::TripleO::Services::OVNController
      - OS::TripleO::Services::Pacemaker
      - OS::TripleO::Services::PankoApi
      - OS::TripleO::Services::RabbitMQ
      - OS::TripleO::Services::Redis
      - OS::TripleO::Services::Rhsm
      - OS::TripleO::Services::RsyslogSidecar
      - OS::TripleO::Services::SaharaApi
      - OS::TripleO::Services::SaharaEngine
      - OS::TripleO::Services::Securetty
      - OS::TripleO::Services::SensuClient
      - OS::TripleO::Services::SkydiveAgent
      - OS::TripleO::Services::SkydiveAnalyzer
      - OS::TripleO::Services::Snmp
      - OS::TripleO::Services::Sshd
      - OS::TripleO::Services::SwiftProxy
      - OS::TripleO::Services::SwiftDispersion
      - OS::TripleO::Services::SwiftRingBuilder
      - OS::TripleO::Services::SwiftStorage
      - OS::TripleO::Services::Tacker
      - OS::TripleO::Services::Timezone
      - OS::TripleO::Services::TripleoFirewall
      - OS::TripleO::Services::TripleoPackages
      - OS::TripleO::Services::Tuned
      - OS::TripleO::Services::Vpp
      - OS::TripleO::Services::Zaqar
  #############################################################################
  # Role: ComputeLeaf0                                                        #
  #############################################################################
  - name: ComputeLeaf0
    description: |
      Basic Compute Node role
    CountDefault: 1
    networks:
      - InternalApi
      - Tenant
      - Storage
    HostnameFormatDefault: '%stackname%-compute-leaf0-%index%'
    disable_upgrade_deployment: True
    ServicesDefault:
      - OS::TripleO::Services::AuditD
      - OS::TripleO::Services::CACerts
      - OS::TripleO::Services::CephClient
      - OS::TripleO::Services::CephExternal
      - OS::TripleO::Services::CertmongerUser
      - OS::TripleO::Services::Collectd
      - OS::TripleO::Services::ComputeCeilometerAgent
      - OS::TripleO::Services::ComputeNeutronCorePlugin
      - OS::TripleO::Services::ComputeNeutronL3Agent
      - OS::TripleO::Services::ComputeNeutronMetadataAgent
      - OS::TripleO::Services::ComputeNeutronOvsAgent
      - OS::TripleO::Services::Docker
      - OS::TripleO::Services::Fluentd
      - OS::TripleO::Services::Ipsec
      - OS::TripleO::Services::Iscsid
      - OS::TripleO::Services::Kernel
      - OS::TripleO::Services::LoginDefs
      - OS::TripleO::Services::MySQLClient
      - OS::TripleO::Services::NeutronBgpVpnBagpipe
      - OS::TripleO::Services::NeutronLinuxbridgeAgent
      - OS::TripleO::Services::NeutronVppAgent
      - OS::TripleO::Services::NovaCompute
      - OS::TripleO::Services::NovaLibvirt
      - OS::TripleO::Services::NovaMigrationTarget
      - OS::TripleO::Services::Ntp
      - OS::TripleO::Services::ContainersLogrotateCrond
      - OS::TripleO::Services::OpenDaylightOvs
      - OS::TripleO::Services::Rhsm
      - OS::TripleO::Services::RsyslogSidecar
      - OS::TripleO::Services::Securetty
      - OS::TripleO::Services::SensuClient
      - OS::TripleO::Services::SkydiveAgent
      - OS::TripleO::Services::Snmp
      - OS::TripleO::Services::Sshd
      - OS::TripleO::Services::Timezone
      - OS::TripleO::Services::TripleoFirewall
      - OS::TripleO::Services::TripleoPackages
      - OS::TripleO::Services::Tuned
      - OS::TripleO::Services::Vpp
      - OS::TripleO::Services::OVNController
      - OS::TripleO::Services::OVNMetadataAgent
  #############################################################################
  # Role: ComputeLeaf1                                                        #
  #############################################################################
  - name: ComputeLeaf1
    description: |
      Basic Compute Node role
    CountDefault: 1
    networks:
      - Internal1
      - Tenant1
      - Storage1
    HostnameFormatDefault: '%stackname%-compute-leaf1-%index%'
    disable_upgrade_deployment: True
    ServicesDefault:
      - OS::TripleO::Services::AuditD
      - OS::TripleO::Services::CACerts
      - OS::TripleO::Services::CephClient
      - OS::TripleO::Services::CephExternal
      - OS::TripleO::Services::CertmongerUser
      - OS::TripleO::Services::Collectd
      - OS::TripleO::Services::ComputeCeilometerAgent
      - OS::TripleO::Services::ComputeNeutronCorePlugin
      - OS::TripleO::Services::ComputeNeutronL3Agent
      - OS::TripleO::Services::ComputeNeutronMetadataAgent
      - OS::TripleO::Services::ComputeNeutronOvsAgent
      - OS::TripleO::Services::Docker
      - OS::TripleO::Services::Fluentd
      - OS::TripleO::Services::Ipsec
      - OS::TripleO::Services::Iscsid
      - OS::TripleO::Services::Kernel
      - OS::TripleO::Services::LoginDefs
      - OS::TripleO::Services::MySQLClient
      - OS::TripleO::Services::NeutronBgpVpnBagpipe
      - OS::TripleO::Services::NeutronLinuxbridgeAgent
      - OS::TripleO::Services::NeutronVppAgent
      - OS::TripleO::Services::NovaCompute
      - OS::TripleO::Services::NovaLibvirt
      - OS::TripleO::Services::NovaMigrationTarget
      - OS::TripleO::Services::Ntp
      - OS::TripleO::Services::ContainersLogrotateCrond
      - OS::TripleO::Services::OpenDaylightOvs
      - OS::TripleO::Services::Rhsm
      - OS::TripleO::Services::RsyslogSidecar
      - OS::TripleO::Services::Securetty
      - OS::TripleO::Services::SensuClient
      - OS::TripleO::Services::SkydiveAgent
      - OS::TripleO::Services::Snmp
      - OS::TripleO::Services::Sshd
      - OS::TripleO::Services::Timezone
      - OS::TripleO::Services::TripleoFirewall
      - OS::TripleO::Services::TripleoPackages
      - OS::TripleO::Services::Tuned
      - OS::TripleO::Services::Vpp
      - OS::TripleO::Services::OVNController
      - OS::TripleO::Services::OVNMetadataAgent

Configure node placement
------------------------

Use node placement to map the baremetal nodes to roles, with each role using a
different set of local layer 2 segments. Please refer to :doc:`node_placement`
for details on how to configure node placement.

Add configuration to parameters_default
---------------------------------------

Before deploying the ``overcloud`` create an environment file that contains the
required overrides. In the example below parameter overrides for the following
four roles and ``Controller``, ``ComputeLeaf0``, ``ComputeLeaf1`` and
``ComputeLeaf2``.

.. Note:: In TripleO templates role specific parameters are defined using
          variables. One of the variables used is ``{{role.name}}``. The
          templates have parameters such as ``{{role.name}}Count``,
          ``{{role.name}}Flavor``, ``{{role.name}}ControlPlaneSubnet`` and
          many more. This enables per-role values for these parameters, like in
          the example below where they are used to specify the
          *ControlPlaneSubnet* node *Count* and *Flavor* to use for the
          *per-leaf* roles.

Parameter override example::

  parameter_defaults:
    ControlPlaneSubnet: leaf0
    OvercloudComputeLeaf0Flavor: compute-leaf0
    OvercloudComputeLeaf1Flavor: compute-leaf1
    OvercloudComputeLeaf2Flavor: compute-leaf2
    ControllerCount: 3
    ComputeLeaf0Count: 5
    ComputeLeaf1Count: 5
    ComputeLeaf2Count: 5
    ControllerControlPlaneSubnet: leaf0
    ComputeLeaf0ControlPlaneSubnet: leaf0
    ComputeLeaf1ControlPlaneSubnet: leaf1
    ComputeLeaf2ControlPlaneSubnet: leaf2

Deploy the overcloud
--------------------

To deploy the overcloud, run the ``openstack overcloud deploy`` specifying the
roles data file and environment file. For example::

  openstack overcloud deploy --templates \
  -r /home/stack/roles_data.yaml \
  -e /home/stack/environments/node_data.yaml

.. Note:: Remember to include other environment files that you might want for
          configuration of the overcloud.
