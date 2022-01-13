.. _deployed_server:

Using Already Deployed Servers
==============================

TripleO can be used with servers that have already been deployed and
provisioned with a running operating system.

In this deployment scenario, Ironic from the Undercloud is not used
to do any server deployment, installation, or power management. An external to
TripleO and already existing provisioning tool is expected to have already
installed an operating system on the servers that are intended to be used as
nodes in the Overcloud.

Additionally, Neutron can be optionally used or not.

.. note::
   It's an all or nothing approach when using already deployed servers. Mixing
   using deployed servers with servers provisioned with Nova and Ironic is not
   currently possible.

Benefits to using this feature include not requiring a dedicated provisioning
network, and being able to use a custom partitioning scheme on the already
deployed servers.

Deployed Server Requirements
----------------------------

Networking
^^^^^^^^^^

Network interfaces
__________________

It's recommended that each server have a dedicated management NIC with
externally configured connectivity so that the servers are reachable outside of
any networking configuration done by the OpenStack deployment.

A separate interface, or set of interfaces should then be used for the
OpenStack deployment itself, configured in the typical fashion with a set of
NIC config templates during the Overcloud deployment. See
:doc:`../features/network_isolation` for more information on configuring networking.

.. note::

  When configuring network isolation be sure that the configuration does not
  result in a loss of network connectivity from the deployed servers to the
  undercloud. The interface(s) that are being used for this connectivity should
  be excluded from the NIC config templates so that the configuration does not
  unintentionally drop all networking access to the deployed servers.


Undercloud
__________

Neutron in the Undercloud is not used for providing DHCP services for the
Overcloud nodes, hence a dedicated provisioning network with L2 connectivity is
not a requirement in this scenario. Neutron is however still used for IPAM for
the purposes of assigning IP addresses to the port resources created by
tripleo-heat-templates.

Network L3 connectivity is still a requirement between the Undercloud and
Overcloud nodes. The undercloud will need to be able to connect over a routable
IP to the overcloud nodes for software configuration with ansible.

Overcloud
_________

Configure the deployed servers that will be used as nodes in the overcloud with
L3 connectivity from the Undercloud as needed. The configuration could be done
via static or DHCP IP assignment.

Further networking configuration of Overcloud nodes is the same as in a typical
TripleO deployment, except for:

* Initial configuration of L3 connectivity from the undercloud to the
  overcloud.
* No requirement for dedicating a separate L2 network for provisioning

Testing Connectivity
____________________

Test connectivity from the undercloud to the overcloud nodes using SSH over the configured IP
address on the deployed servers. This should be the IP address that is
configured on ``--overcloud-ssh-network`` as passed to the ``openstack overcloud
deploy`` command. The key and user to use with the test should be the same as
used with ``--overcloud-ssh-key`` and ``--overcloud-ssh-user`` with the
deployment command.

Package repositories
^^^^^^^^^^^^^^^^^^^^

The servers will need to already have the appropriately enabled yum repositories
as packages will be installed on the servers during the Overcloud deployment.
The enabling of repositories on the Overcloud nodes is the same as it is for
other areas of TripleO, such as Undercloud installation. See
:doc:`../repositories` for the detailed steps on how to
enable the standard repositories for TripleO.

Deploying the Overcloud
-----------------------

Provision networks and ports if using Neutron
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
If using Neutron for resource managment, Network resources for the deployment
still must be provisioned with the ``openstack overcloud network provision``
command as documented in :ref:`custom_networks`.

Port resources for the deployment still must be provisioned with the
``openstack overcloud node provision`` command as documented in
:ref:`baremetal_provision`.

Set the ``managed`` key to false in either the ``defaults`` dictionary for each
role, or on each instances dictionary in the baremetal provision configuration
file.

The generated file must then be passed to the ``openstack overcloud deploy``
command.

Deployment Command
^^^^^^^^^^^^^^^^^^

With generated baremetal and network environments
_________________________________________________
Include the generated environment files with the deployment command::

  openstack overcloud deploy \
    --deployed-server \
    -e ~/overcloud-networks-deployed.yaml \
    -e ~/overcloud-baremetal-deployed.yaml \
    <other arguments>

Without generated environments (no Neutron)
___________________________________________
The following command would be used when the ``openstack overcloud network
provision`` and ``openstack overcloud node provision`` commands were not used.
Additional environment files need to be passed to the deployment command::

  openstack overcloud deploy \
    --deployed-server \
    -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-environment.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-networks.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-ports.yaml \
    -e ~/hostnamemap.yaml \
    -e ~/deployed-server-network-environment.yaml \
    <other arguments>

The environment file ``deployed-server-environment.yaml`` contains the necessary
``resource_registry`` mappings to disable Nova management of overcloud servers
so that deployed servers are used instead.

``deployed-networks.yaml`` and ``deployed-ports.yaml`` enable the necessary
mappings to disable the Neutron management of network resources.

``hostnamemap.yaml`` is optional and should define the ``HostnameMap``
parameter if the actual server hostnames do not match the default role hostname
format. For example::

  parameter_defaults:
    HostnameMap:
      overcloud-controller-0: controller-00-rack01
      overcloud-controller-1: controller-01-rack02
      overcloud-controller-2: controller-02-rack03
      overcloud-novacompute-0: compute-00-rack01
      overcloud-novacompute-1: compute-01-rack01
      overcloud-novacompute-2: compute-02-rack01

``deployed-server-network-environment.yaml`` should define at a minimum the
following parameters::

  NodePortMap
  DeployedNetworkEnvironment
  ControlPlaneVipData
  VipPortMap
  OVNDBsVirtualFixedIPs
  RedisVirtualFixedIPs
  EC2MetadataIp
  ControlPlaneDefaultRoute

The following is a sample environment file that shows setting these values

.. code-block:: yaml

    parameter_defaults:

      NodePortMap:
        controller0:
          ctlplane:
            ip_address: 192.168.100.2
            ip_address_uri: 192.168.100.2
            ip_subnet: 192.168.100.0/24
          external:
            ip_address: 10.0.0.10
            ip_address_uri: 10.0.0.10
            ip_subnet: 10.0.0.10/24
          internal_api:
            ip_address: 172.16.2.10
            ip_address_uri: 172.16.2.10
            ip_subnet: 172.16.2.10/24
          management:
            ip_address: 192.168.1.10
            ip_address_uri: 192.168.1.10
            ip_subnet: 192.168.1.10/24
          storage:
            ip_address: 172.16.1.10
            ip_address_uri: 172.16.1.10
            ip_subnet: 172.16.1.10/24
          storage_mgmt:
            ip_address: 172.16.3.10
            ip_address_uri: 172.16.3.10
            ip_subnet: 172.16.3.10/24
          tenant:
            ip_address: 172.16.0.10
            ip_address_uri: 172.16.0.10
            ip_subnet: 172.16.0.10/24

        compute0:
          ctlplane:
            ip_address: 192.168.100.3
            ip_address_uri: 192.168.100.3
            ip_subnet: 192.168.100.0/24
          external:
            ip_address: 10.0.0.110
            ip_address_uri: 10.0.0.110
            ip_subnet: 10.0.0.110/24
          internal_api:
            ip_address: 172.16.2.110
            ip_address_uri: 172.16.2.110
            ip_subnet: 172.16.2.110/24
          management:
            ip_address: 192.168.1.110
            ip_address_uri: 192.168.1.110
            ip_subnet: 192.168.1.110/24
          storage:
            ip_address: 172.16.1.110
            ip_address_uri: 172.16.1.110
            ip_subnet: 172.16.1.110/24
          storage_mgmt:
            ip_address: 172.16.3.110
            ip_address_uri: 172.16.3.110
            ip_subnet: 172.16.3.110/24
          tenant:
            ip_address: 172.16.0.110
            ip_address_uri: 172.16.0.110
            ip_subnet: 172.16.0.110/24

      ControlPlaneVipData:
        fixed_ips:
        - ip_address: 192.168.100.1
        name: control_virtual_ip
        network:
          tags: []
        subnets:
        - ip_version: 4

      VipPortMap:
        external:
          ip_address: 10.0.0.100
          ip_address_uri: 10.0.0.100
          ip_subnet: 10.0.0.100/24
        internal_api:
          ip_address: 172.16.2.100
          ip_address_uri: 172.16.2.100
          ip_subnet: 172.16.2.100/24
        storage:
          ip_address: 172.16.1.100
          ip_address_uri: 172.16.1.100
          ip_subnet: 172.16.1.100/24
        storage_mgmt:
          ip_address: 172.16.3.100
          ip_address_uri: 172.16.3.100
          ip_subnet: 172.16.3.100/24

      RedisVirtualFixedIPs:
        - ip_address: 192.168.100.10
          use_neutron: false
      OVNDBsVirtualFixedIPs:
        - ip_address: 192.168.100.11
          use_neutron: false

      DeployedNetworkEnvironment:
        net_attributes_map:
          external:
            network:
              dns_domain: external.tripleodomain.
              mtu: 1400
              name: external
              tags:
              - tripleo_network_name=External
              - tripleo_net_idx=0
              - tripleo_vip=true
            subnets:
              external_subnet:
                cidr: 10.0.0.0/24
                dns_nameservers: []
                gateway_ip: null
                host_routes: []
                ip_version: 4
                name: external_subnet
                tags:
                - tripleo_vlan_id=10
          internal_api:
            network:
              dns_domain: internalapi.tripleodomain.
              mtu: 1400
              name: internal_api
              tags:
              - tripleo_net_idx=1
              - tripleo_vip=true
              - tripleo_network_name=InternalApi
            subnets:
              internal_api_subnet:
                cidr: 172.16.2.0/24
                dns_nameservers: []
                gateway_ip: null
                host_routes: []
                ip_version: 4
                name: internal_api_subnet
                tags:
                - tripleo_vlan_id=20
          management:
            network:
              dns_domain: management.tripleodomain.
              mtu: 1400
              name: management
              tags:
              - tripleo_net_idx=5
              - tripleo_network_name=Management
            subnets:
              management_subnet:
                cidr: 192.168.1.0/24
                dns_nameservers: []
                gateway_ip: 192.168.1.1
                host_routes: []
                ip_version: 4
                name: management_subnet
                tags:
                - tripleo_vlan_id=60
          storage:
            network:
              dns_domain: storage.tripleodomain.
              mtu: 1400
              name: storage
              tags:
              - tripleo_net_idx=3
              - tripleo_vip=true
              - tripleo_network_name=Storage
            subnets:
              storage_subnet:
                cidr: 172.16.1.0/24
                dns_nameservers: []
                gateway_ip: null
                host_routes: []
                ip_version: 4
                name: storage_subnet
                tags:
                - tripleo_vlan_id=30
          storage_mgmt:
            network:
              dns_domain: storagemgmt.tripleodomain.
              mtu: 1400
              name: storage_mgmt
              tags:
              - tripleo_net_idx=4
              - tripleo_vip=true
              - tripleo_network_name=StorageMgmt
            subnets:
              storage_mgmt_subnet:
                cidr: 172.16.3.0/24
                dns_nameservers: []
                gateway_ip: null
                host_routes: []
                ip_version: 4
                name: storage_mgmt_subnet
                tags:
                - tripleo_vlan_id=40
          tenant:
            network:
              dns_domain: tenant.tripleodomain.
              mtu: 1400
              name: tenant
              tags:
              - tripleo_net_idx=2
              - tripleo_network_name=Tenant
            subnets:
              tenant_subnet:
                cidr: 172.16.0.0/24
                dns_nameservers: []
                gateway_ip: null
                host_routes: []
                ip_version: 4
                name: tenant_subnet
                tags:
                - tripleo_vlan_id=50
        net_cidr_map:
          external:
          - 10.0.0.0/24
          internal_api:
          - 172.16.2.0/24
          management:
          - 192.168.1.0/24
          storage:
          - 172.16.1.0/24
          storage_mgmt:
          - 172.16.3.0/24
          tenant:
          - 172.16.0.0/24
        net_ip_version_map:
          external: 4
          internal_api: 4
          management: 4
          storage: 4
          storage_mgmt: 4
          tenant: 4

.. note::

    Beginning in Wallaby, the above parameter values from
    ``deployed-server-network-environment.yaml`` and the
    ``deployed-networks.yaml`` and ``deployed-ports.yaml`` environments replace the use of the
    ``DeployedServerPortMap`` parameter, the
    ``environments/deployed-server-deployed-neutron-ports.yaml`` environment, and
    the ``deployed-neutron-port.yaml`` template.

    The previous parameters and environments can still be used with the
    exception that no resources can be mapped to any Neutron native Heat
    resources (resources starting with ``OS::Neutron::*``) when using
    :doc:`ephemeral Heat <../deployment/ephemeral_heat>` as there is no Heat
    and Neutron API communication.

    Note that the following resources may be mapped to ``OS::Neutron::*``
    resources in environment files used prior to Wallaby, and these mappings
    should be removed from Wallaby onward::

        OS::TripleO::Network::Ports::ControlPlaneVipPort
        OS::TripleO::Network::Ports::RedisVipPort
        OS::TripleO::Network::Ports::OVNDBsVipPort

  .. admonition:: Victoria and prior releases

    The ``DeployedServerPortMap`` parameter can be used to assign fixed IP's
    from either the ctlplane network or the IP address range for the
    overcloud.

    If the deployed servers were preconfigured with IP addresses from the ctlplane
    network for the initial undercloud connectivity, then the same IP addresses can
    be reused during the overcloud deployment. Add the following to a new
    environment file and specify the environment file as part of the deployment
    command::

        resource_registry:
          OS::TripleO::DeployedServer::ControlPlanePort: ../deployed-server/deployed-neutron-port.yaml
        parameter_defaults:
          DeployedServerPortMap:
            controller0-ctlplane:
              fixed_ips:
                - ip_address: 192.168.24.9
              subnets:
                - cidr: 192.168.24.0/24
              network:
                tags:
                  - 192.168.24.0/24
            compute0-ctlplane:
              fixed_ips:
                - ip_address: 192.168.24.8
              subnets:
                - cidr: 192.168.24..0/24
              network:
                tags:
                  - 192.168.24.0/24

    The value of the DeployedServerPortMap variable is a map. The keys correspond
    to the ``<short hostname>-ctlplane`` of the deployed servers. Specify the ip
    addresses and subnet CIDR to be assigned under ``fixed_ips``.

    In the case where the ctlplane is not routable from the deployed
    servers, the virtual IPs on the ControlPlane, as well as the virtual IPs
    for services (Redis and OVNDBs) must be statically assigned.

    Use ``DeployedServerPortMap`` to assign an IP address from any CIDR::

      resource_registry:
        OS::TripleO::DeployedServer::ControlPlanePort: /usr/share/openstack-tripleo-heat-templates/deployed-server/deployed-neutron-port.yaml
        OS::TripleO::Network::Ports::ControlPlaneVipPort: /usr/share/openstack-tripleo-heat-templates/deployed-server/deployed-neutron-port.yaml

        # Set VIP's for redis and OVN to noop to default to the ctlplane VIP
        # The ctlplane VIP is set with control_virtual_ip in
        # DeployedServerPortMap below.
        #
        # Alternatively, these can be mapped to deployed-neutron-port.yaml as
        # well and redis_virtual_ip and ovn_dbs_virtual_ip added to the
        # DeployedServerPortMap value to set fixed IP's.
        OS::TripleO::Network::Ports::RedisVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/noop.yaml
        OS::TripleO::Network::Ports::OVNDBsVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/noop.yaml

      parameter_defaults:
        NeutronPublicInterface: eth1
        EC2MetadataIp: 192.168.100.1
        ControlPlaneDefaultRoute: 192.168.100.1

        DeployedServerPortMap:
          control_virtual_ip:
            fixed_ips:
              - ip_address: 192.168.100.1
            subnets:
              - cidr: 192.168.100.0/24
            network:
              tags:
                - 192.168.100.0/24
          controller0-ctlplane:
            fixed_ips:
              - ip_address: 192.168.100.2
            subnets:
              - cidr: 192.168.100.0/24
            network:
              tags:
                - 192.168.100.0/24
          compute0-ctlplane:
            fixed_ips:
              - ip_address: 192.168.100.3
            subnets:
              - cidr: 192.168.100.0/24
            network:
              tags:
                - 192.168.100.0/24

    In the above example, notice how ``RedisVipPort`` and ``OVNDBsVipPort`` are
    mapped to ``network/ports/noop.yaml``. This mapping is due to the fact that
    these VIP IP addresses comes from the ctlplane by default, and they will use
    the same VIP address that is used for ``ControlPlanePort``. Alternatively
    these VIP's can be mapped to their own fixed IP's, in which case a VIP will
    be created for each. In this case, the following mappings and values would be
    added to the above example::

        resource_registry:
          OS::TripleO::Network::Ports::RedisVipPort: /usr/share/openstack-tripleo-heat-templates/deployed-server/deployed-neutron-port.yaml
          OS::TripleO::Network::Ports::OVNDBsVipPort: /usr/share/openstack-tripleo-heat-templates/deployed-server/deployed-neutron-port.yaml

        parameter_defaults:

          DeployedServerPortMap:
            redis_virtual_ip:
              fixed_ips:
                - ip_address: 192.168.100.10
              subnets:
                - cidr: 192.168.100.0/24
              network:
                tags:
                  - 192.168.100.0/24
            ovn_dbs_virtual_ip:
              fixed_ips:
                - ip_address: 192.168.100.11
              subnets:
                - cidr: 192.168.100.0/24
              network:
                tags:
                  - 192.168.100.0/24


    Use ``DeployedServerPortMap`` to assign an ControlPlane Virtual IP address from
    any CIDR, and the ``RedisVirtualFixedIPs`` and ``OVNDBsVirtualFixedIPs``
    parameters to assing the ``RedisVip`` and ``OVNDBsVip``::

      resource_registry:
        OS::TripleO::DeployedServer::ControlPlanePort: /usr/share/openstack-tripleo-heat-templates/deployed-server/deployed-neutron-port.yaml
        OS::TripleO::Network::Ports::ControlPlaneVipPort: /usr/share/openstack-tripleo-heat-templates/deployed-server/deployed-neutron-port.yaml

      parameter_defaults:
        NeutronPublicInterface: eth1
        EC2MetadataIp: 192.168.100.1
        ControlPlaneDefaultRoute: 192.168.100.1

        # Set VIP's for redis and OVN
        RedisVirtualFixedIPs:
          - ip_address: 192.168.100.10
            use_neutron: false
        OVNDBsVirtualFixedIPs:
          - ip_address: 192.168.100.11
            use_neutron: false

        DeployedServerPortMap:
          control_virtual_ip:
            fixed_ips:
              - ip_address: 192.168.100.1
            subnets:
              - cidr: 192.168.100.0/24
            network:
              tags:
                - 192.168.100.0/24
          controller0-ctlplane:
            fixed_ips:
              - ip_address: 192.168.100.2
            subnets:
              - cidr: 192.168.100.0/24
            network:
              tags:
                - 192.168.100.0/24
          compute0-ctlplane:
            fixed_ips:
              - ip_address: 192.168.100.3
            subnets:
              - cidr: 192.168.100.0/24
            network:
              tags:
                - 192.168.100.0/24

Scaling the Overcloud
---------------------

Scaling Up
^^^^^^^^^^
When scaling out compute nodes, the steps to be completed by the
user are as follows:

#. Prepare the new deployed server(s) as shown in `Deployed Server
   Requirements`_.
#. Start the scale out command. See :doc:`../post_deployment/scale_roles` for reference.

Scaling Down
^^^^^^^^^^^^


Starting in Train and onward, `openstack overcloud node delete` can take
a list of server hostnames instead of instance ids. However they can't be
mixed while running the command. Example: if you use hostnames, it would
have to be for all the nodes to delete.

.. admonition:: Victoria and prior releases
    :class: victoria

    The following instructions should be used when the cloud is deployed on
    Victoria or a prior release.

    When scaling down the Overcloud, follow the scale down instructions as normal
    as shown in :doc:`../post_deployment/delete_nodes`, however use the following
    command to get the uuid values to pass to `openstack overcloud node delete`
    instead of using `nova list`::

        openstack stack resource list overcloud -n5 --filter type=OS::TripleO::<RoleName>Server

    Replace `<RoleName>` in the above command with the actual name of the role that
    you are scaling down. The `stack_name` column in the command output can be used
    to identify the uuid associated with each node. The `stack_name` will include
    the integer value of the index of the node in the Heat resource group. For
    example, in the following sample output::

        $ openstack stack resource list overcloud -n5 --filter type=OS::TripleO::ComputeDeployedServerServer
        +-----------------------+--------------------------------------+------------------------------------------+-----------------+----------------------+-------------------------------------------------------------+
        | resource_name         | physical_resource_id                 | resource_type                            | resource_status | updated_time         | stack_name                                                  |
        +-----------------------+--------------------------------------+------------------------------------------+-----------------+----------------------+-------------------------------------------------------------+
        | ComputeDeployedServer | 66b1487c-51ee-4fd0-8d8d-26e9383207f5 | OS::TripleO::ComputeDeployedServerServer | CREATE_COMPLETE | 2017-10-31T23:45:18Z | overcloud-ComputeDeployedServer-myztzg7pn54d-0-pixawichjjl3 |
        | ComputeDeployedServer | 01cf59d7-c543-4f50-95df-6562fd2ed7fb | OS::TripleO::ComputeDeployedServerServer | CREATE_COMPLETE | 2017-10-31T23:45:18Z | overcloud-ComputeDeployedServer-myztzg7pn54d-1-ooCahg1vaequ |
        | ComputeDeployedServer | 278af32c-c3a4-427e-96d2-3cda7e706c50 | OS::TripleO::ComputeDeployedServerServer | CREATE_COMPLETE | 2017-10-31T23:45:18Z | overcloud-ComputeDeployedServer-myztzg7pn54d-2-xooM5jai2ees |
        +-----------------------+--------------------------------------+------------------------------------------+-----------------+----------------------+-------------------------------------------------------------+

    The index 0, 1, or 2 can be seen in the `stack_name` column. These indices
    correspond to the order of the nodes in the Heat resource group. Pass the
    corresponding uuid value from the `physical_resource_id` column to `openstack
    overcloud node delete` command.

The physical deployed servers that have been removed from the deployment need
to be powered off. In a deployment not using deployed servers, this would
typically be done with Ironic. When using deployed servers, it must be done
manually, or by whatever existing power management solution is already in
place. If the nodes are not powered down, they will continue to be operational
and could remain functional as part of the deployment, since there are no steps
to unconfigure, uninstall software, or stop services on nodes when scaling
down.

Once the nodes are powered down and all needed data has been saved from the
nodes, it is recommended that they be reprovisioned back to a base operating
system configuration so that they do not unintentionally join the deployment in
the future if they are powered back on.

.. note::

  Do not attempt to reuse nodes that were previously removed from the
  deployment without first reprovisioning them using whatever provisioning tool
  is in place.

Deleting the Overcloud
----------------------

When deleting the Overcloud, the Overcloud nodes need to be manually powered
off, otherwise, the cloud will still be active and accepting any user requests.

After archiving important data (log files, saved configurations, database
files), that needs to be saved from the deployment, it is recommended to
reprovision the nodes to a clean base operating system. The reprovision will
ensure that they do not start serving user requests, or interfere with future
deployments in the case where they are powered back on in the future.

.. note::

  As with scaling down, do not attempt to reuse nodes that were previously part
  of a now deleted deployment in a new deployment without first reprovisioning
  them using whatever provisioning tool is in place.
