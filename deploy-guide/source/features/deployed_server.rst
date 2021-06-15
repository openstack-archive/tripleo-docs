.. _deployed_server:

Using Already Deployed Servers
==============================

TripleO can be used with servers that have already been deployed and
provisioned with a running operating system.

In this deployment scenario, Nova and Ironic from the Undercloud are not used
to do any server deployment, installation, or power management. An external to
TripleO and already existing provisioning tool is expected to have already
installed an operating system on the servers that are intended to be used as
nodes in the Overcloud.

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

Deployment Command
^^^^^^^^^^^^^^^^^^

The functionality of using already deployed servers is enabled by passing
additional Heat environment files to the ``openstack overcloud deploy``
command.::

    openstack overcloud deploy \
      <other cli arguments> \
      -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-environment.yaml

.. admonition:: Until Victoria
   :class: stable

   Note that, until Victoria, you have to pass ``--disable-validations`` in
   order to disable the basics Nova, Ironic and Glance validations executed
   by python-tripleoclient. These validations are not necessary since those
   services will not be used to deploy the Overcloud.

The ``deployed-server.yaml`` environment takes advantage of the template
composition nature of Heat and tripleo-heat-templates to substitute
``OS::Heat::DeployedServer`` resources in place of ``OS::Nova::Server``.

.. note::

   Previously a custom roles file was needed when using deployed-server. The
   custom roles file was located in the templates directory at
   deployed-server/deployed-server-roles-data.yaml. The custom roles file
   addressed setting ``disable_constraints: true`` on each of the roles. This
   is no longer required starting in the train release.

.. note::

   Previously, environment files were used to enable bootstrap tasks on the
   deployed servers. These files were
   environments/deployed-server-bootstrap-environment-centos.yaml and
   environments/deployed-server-bootstrap-environment-rhel.yaml. Starting in
   the train release, these environment files are no longer required and they
   have been removed from tripleo-heat-templates.

.. note::
   Starting in the train release, support for setting DeploymentSwiftDataMap
   parameter and configuring deployed servers using heat has been removed.

deployed-server with config-download
____________________________________
When using :doc:`config-download <../deployment/ansible_config_download>` with
``deployed-server`` (pre-provisioned nodes), a ``HostnameMap`` parameter must
be provided. Create an environment file to define the parameter, and assign the
node hostnames in the parameter value. The following example shows a sample
value::

  parameter_defaults:
    HostnameMap:
      overcloud-controller-0: controller-00-rack01
      overcloud-controller-1: controller-01-rack02
      overcloud-controller-2: controller-02-rack03
      overcloud-novacompute-0: compute-00-rack01
      overcloud-novacompute-1: compute-01-rack01
      overcloud-novacompute-2: compute-02-rack01

Write the contents to an environment file such as ``hostnamemap.yaml``, and
pass it the environment as part of the deployment command. It's imperative that
the ``HostnameMap`` keys correspond to the ``HostnameFormatDefault`` for the
appropriate role. For example, using ``overcloud-controller-0`` matches
``HostnameFormatDefault: '%stackname%-controller-%index%'`` in the
``Controller`` role. Similarly, ``overcloud-novacompute-0`` matches
``HostnameFormatDefault: '%stackname%-novacompute-%index%'`` for the
``Compute`` role. If you decide to change the ``HostnameFormatDefault`` to a
different value, you'll need to account for this in your ``hostnamemap.yaml``
file. Mismatched values between the ``HostnameMap`` keys and
``HostnameFormatDefault`` causes failures during overcloud installation because
TripleO can't find the appropriate hosts, as it's using the wrong names.



Network Configuration
_____________________

The default network interface configuration mappings for the deployed-server
roles are::

  OS::TripleO::ControllerDeployedServer::Net::SoftwareConfig: net-config-static-bridge.yaml
  OS::TripleO::ComputeDeployedServer::Net::SoftwareConfig: net-config-static.yaml
  OS::TripleO::BlockStorageDeployedServer::Net::SoftwareConfig: net-config-static.yaml
  OS::TripleO::ObjectStorageDeployedServer::Net::SoftwareConfig: net-config-static.yaml
  OS::TripleO::CephStorageDeployedServer::Net::SoftwareConfig: net-config-static.yaml

The default NIC configs use static IP assignment instead of the default of
DHCP. This is due to there being no requirement of L2 connectivity between the
undercloud and overcloud.  However, the NIC config templates can be overridden
to use whatever configuration is desired (including DHCP).

As is the case when not using deployed-servers, the following parameters need
to also be specified::

    parameter_defaults:
      NeutronPublicInterface: eth1
      ControlPlaneDefaultRoute: 192.168.24.1
      EC2MetadataIp: 192.168.24.1

``ControlPlaneDefaultRoute`` and ``EC2MetadataIp`` are not necessarily
meaningful parameters depending on the network architecture in use with
deployed servers. However, they still must be specified as they are required
parameters for the template interface.

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

Beginning in Wallaby, the
``environments/deployed-server-deployed-neutron-ports.yaml`` environment, the
deployed-neutron-port.yaml template the DeployedServerPortMap parameter, are
deprecated in favor of using ``NodePortMap``, ``ControlPlaneVipData``, and
``VipPortMap`` with the generated ``environments/deployed-ports.yaml``
environment from ``environments/deployed-ports.j2.yaml``.

Using the previous example with ``DeployedServerPortMap``, that would be
migrated to use ``NodePortMap``, ``ControlPlaneVipData``, and ``VipPortMap`` as
follows. The example is expanded to also show parameter values as they would be
used when using network isolation::

    parameter_defaults:

      NodePortMap:
        controller0:
          ctlplane
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
          ctlplane
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

      VipPortMap
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


The environment file ``environments/deployed-ports.yaml`` must then be included
with the deployment command.

The ``EC2MetadataIp`` and ``ControlPlaneDefaultRoute`` parameters are set to
the value of the control virtual IP address. These parameters are required to
be set by the sample NIC configs, and must be set to a pingable IP address in
order to pass the validations performed during deployment. Alternatively, the
NIC configs could be further customized to not require these parameters.

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

.. admonition:: Train
   :class: train

   Starting in Train and onward, `openstack overcloud node delete` can take
   a list of server hostnames instead of instance ids. However they can't be
   mixed while running the command. Example: if you use hostnames, it would
   have to be for all the nodes to delete.

The following instructions are only useful when the cloud is deployed on Stein
or backward.
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
and could be part of the deployment, since there are no steps to unconfigure,
uninstall software, or stop services on nodes when scaling down.

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
