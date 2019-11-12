.. _distributed_compute_node:

Distributed Compute Node deployment
===================================

Introduction
------------
Additional groups of compute nodes can be deployed and integrated with an
existing deployment of a control plane stack. These compute nodes are deployed
in separate stacks from the main control plane (overcloud) stack, and they
consume exported data from the overcloud stack to reuse as
configuration data.

Deploying these additional nodes in separate stacks provides for separation of
management between the control plane stack and the stacks for additional compute
nodes. The stacks can be managed, scaled, and updated separately.

Using separate stacks also creates smaller failure domains as there are less
baremetal nodes in each invidiual stack. A failure in one baremetal node only
requires that management operations to address that failure need only affect
the single stack that contains the failed node.

A routed spine and leaf networking layout can be used to deploy these
additional groups of compute nodes in a distributed nature. Not all nodes need
to be co-located at the same physical location or datacenter. See
:ref:`routed_spine_leaf_network` for more details.

Such an architecture is referred to as "Distributed Compute Node" or "DCN" for
short.

Supported failure modes and High Availability recommendations
-------------------------------------------------------------

Handling negative scenarios for DCN starts from the deployment planning, like
choosing some particular SDN solution over provider networks to meet the
expected SLA.

Loss of control plane connectivity
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A failure of the central control plane affects all DCN edge sites. There is no
autonomous control planes at the edge. No OpenStack control plane API or CLI
operations can be executed locally in that case. For example, you cannot create
a snapshot of a Nova VM, or issue an auth token, nor can you delete an image or
a VM.

.. note:: A single Controller service failure normally induces
   no downtime for edge sites and should be handled as for usual HA deployments.

Loss of an edge site
^^^^^^^^^^^^^^^^^^^^

Running Nova VM instances will keep running. If stopped running, you need the
control plane back to recover the stopped or crashed workloads. If Neutron DHCP
agent is centralized, and we are forwarding DHCP requests to the central site,
any VMs that are trying to renew their IPs will eventually time out and lose
connectivity.

.. note:: A single Compute service failure normally affects only its edge site
   without additional downtime induced for neighbor edge sites or the central
   control plane.

OpenStack infrastructure services, like Nova Compute, will automatically
reconnect to MariaDB database cluster and RabbitMQ broker when the control
plane's uplink is back. No timed out operations can be resumed though and need
to be retried manually.

It is recommended to maintain each DCN edge site as a separate Availability Zone
(AZ) for Nova/Neutron and Cinder services.

Improving resiliency for N/S and E/W traffic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Reliability of the central control plane may be enhanced with L3 HA network,
which only provides North-South routing. The East-West routing effectiveness of
edge networks may be improved by using DVR or highly available Open Virtual
Network (OVN). There is also BGPVPN and its backend specific choices.

Network recommendations
^^^^^^^^^^^^^^^^^^^^^^^

Traditional or external provider networks with backbone routing at the edge may
fulfill or complement a custom distributed routing solution, like L3 Spine-Leaf
topology.

.. note:: Neutron SDN backends that involve tunnelling may be sub-optimal for
   Edge DCN cases because of the known issues 1808594_ and 1808062_.

   .. _1808594: https://bugs.launchpad.net/tripleo/+bug/1808594
   .. _1808062: https://bugs.launchpad.net/tripleo/+bug/1808062

For dynamic IPv4 and stateful IPv6 IPAM cases, you will also need DHCP on those
provider networks in order to assign IPs to VM instances. External provider
networks usually require no Neutron DHCP agents and handle IPAM (and
routing) on its own. While for traditional or
`Routed Provider Networks <https://docs.openstack.org/neutron/latest/admin/config-routed-networks.html>`_,
when there is no L2 connectivity to edge over WAN, and Neutron DHCP agents are
placed on controllers at the central site, you should have a DHCP relay on
every provider network. Alternatively, DHCP agents need to be moved to the edge.
Such setups also require highly reliable links between remote and central sites.

.. note::  Neither of DHCP relays/agents at compute nodes, nor routed/external
   provider networks are tested or automated via TripleO Heat Templates. You would
   have to have those configured manually for your DCN environments.

.. note:: OVN leverages DVR and does not require running Neutron DHCP/L3 agents,
  which might as well simplify particular DCN setups.

That said, when there is a network failure that disconnects the edge off the
central site, there is no SLA for recovery time but only what the provider
networks or a particular SDN choice can guarantee. For switched/routed/MPLS
provider networks, that may span from 10's of ms to a few seconds. With
the outage thresholds are typically considered to be a 15 seconds. These trace
back on various standards that are relevant here.

Config-drive/cloud-init details
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Config-drive uses virtual media capabilities of the BMC controller, so that no
DHCP is required for VMs to obtain IP addresses at edge sites. This is
the most straightforward solution. This does require that the WAN between the
remote site and central site is live during deployment of a VM, but after that
the VM can run independently without a connection to the central site.

.. note:: Config-drive may be a tricky for VMs that do not support
  cloud-init, like some appliance VMs. It may be that such ones (or other VMs
  that do not support config-drive) will have to be configured with a static IP
  that matches the Neutron port.

The simplest solution we recommend for DCN would involve only external provider
networks at the edge. For that case, it is also recommended to use either
config-drive, or IPv6 SLAAC, or another configuration mechanism other than
those requiring a `169.254.169.254/32` route for the provider routers to forward
data to the metadata service.

IPv6 details
^^^^^^^^^^^^

IPv6 for tenants' workloads and infrastructure tunnels interconnecting
the central site and the edge is a viable option as well. IPv6 cannot be used for
provisioning networks though. Key benefits IPv6 may provide for DCN are:

* SLAAC, which is a EUI-64 form of autoconfig that makes IPv6 addresses
  calculated based on MAC addresses and requires no DHCP services placed on the
  provider networks.
* Improved mobility for endpoints, like NFV APIs, to roam around different links
  and edge sites without losing its connections and IP addresses.
* End-to-end IPv6 has been shown to have better performance by large content
  networks. This is largely due to the presence of NAT in most end-to-end IPv4
  connections that slows them down.

Storage recommendations
^^^^^^^^^^^^^^^^^^^^^^^

DCN with only ephemeral storage is available for Nova Compute services.
That is up to the edge cloud applications to be designed to provide enhanced
data availability, locality awareness and/or replication mechanisms.

Deploying DCN
-------------

Deploying the DCN architecture requires consideration as it relates to the
undercloud, roles, networks, and availability zones configuration. This section
will document on how to approach the DCN deployment.

The deployment will make use of specific undercloud configuration, and then
deploying multiple stacks, typically one stack per distributed location,
although this is not a strict requirement.

At the central site, stack separation can still be used to deploy separate
stacks for control plane and compute services if compute services are desired
at the central site. See deploy_control_plane_ for more information.

Each distributed site will be a separate stack as well. See deploy_dcn_ for
more information.

.. _undercloud_dcn:

Undercloud configuration
^^^^^^^^^^^^^^^^^^^^^^^^
This section describes the steps required to configure the undercloud for DCN.

Using direct deploy instead of iSCSI
____________________________________

In a default undercloud configuration, ironic deploys nodes using the `iscsi`
deploy interface. When using the `iscsi` deploy interface, the deploy ramdisk
publishes the node’s disk as an iSCSI target, and the `ironic-conductor`
service then copies the image to this target.

For a DCN deployment, network latency is often a concern between the undercloud
and the distributed compute nodes. Considering the potential for latency, the
distributed compute nodes should be configured to use the `direct` deploy
interface in the undercloud. This process is described later in this guide
under :ref:`configure-deploy-interface`.

When using the `direct` deploy interface, the deploy ramdisk will download the
image over HTTP from the undercloud's Swift service, and copy it to the node’s
disk. HTTP is more resilient when dealing with network latency than iSCSI, so
using the `direct` deploy interface provides a more stable node deployment
experience for distributed compute nodes.

Configure the Swift temporary URL key
_____________________________________

Images are served by Swift and are made available to nodes using an HTTP URL,
over the `direct` deploy interface. To allow Swift to create temporary URLs, it
must be configured with a temporary URL key. The key value is used for
cryptographic signing and verification of the temporary URLs created by Swift.

The following commands demonstrate how to configure the setting. In this
example, `uuidgen` is used to randomly create a key value. You should choose a
unique key value that is a difficult to guess value. For example::

    source ~/stackrc
    openstack role add --user admin --project service ResellerAdmin
    openstack --os-project-name service object store account set --property Temp-URL-Key=$(uuidgen | sha1sum | awk '{​print $1}')

.. _configure-deploy-interface:

Configure nodes to use the deploy interface
___________________________________________

This section describes how to configure the deploy interface for new and
existing nodes.

For new nodes, the deploy interface can be specified directly in the JSON
structure for each node. For example, see the `“deploy_interface”: “direct”`
setting below::

    {
       "nodes":[
           {
               "mac":[
                   "bb:bb:bb:bb:bb:bb"
               ],
               "name":"node01",
               "cpu":"4",
               "memory":"6144",
               "disk":"40",
               "arch":"x86_64",
               "pm_type":"ipmi",
               "pm_user":"admin",
               "pm_password":"p@55w0rd!",
               "pm_addr":"192.168.24.205",
               “deploy_interface”: “direct”
           },
           {
               "mac":[
                   "cc:cc:cc:cc:cc:cc"
               ],
               "name":"node02",
               "cpu":"4",
               "memory":"6144",
               "disk":"40",
               "arch":"x86_64",
               "pm_type":"ipmi",
               "pm_user":"admin",
               "pm_password":"p@55w0rd!",
               "pm_addr":"192.168.24.206"
               “deploy_interface”: “direct”
           }
       ]
    }

Existing nodes can be updated to use the `direct` deploy interface. For
example::

    openstack baremetal node set --deploy-interface direct 4b64a750-afe3-4236-88d1-7bb88c962666

.. _deploy_control_plane:

Deploying the control plane
^^^^^^^^^^^^^^^^^^^^^^^^^^^
The main overcloud control plane stack should be deployed as needed for the
desired cloud architecture layout. This stack contains nodes running the
control plane and infrastructure services needed for the cloud. For the
purposes of this documentation, this stack is referred to as the
``control-plane`` stack.

No specific changes or deployment configuration is necessary to deploy just the
control plane services.

It's recommended that the ``control-plane`` stack contain only control plane
services, and no compute or storage services. If compute and storage services
are desired at the same geographical site as the ``control-plane`` stack then
they should be deployed in a separate stack just like a edge site specific stack,
but using nodes at the same geographical location. In such a scenario, the
stack with compute and storage services could be called ``central`` and
deploying it in a separate stack allows for separation of management and
operations.

It is suggested to give each stack an explicit name. For example, the control
plane stack could be called `control-plane` and set by passing `--stack
control-plane` to the `openstack overcloud deploy` command.

.. _deploy_dcn:

Deploying a DCN site
^^^^^^^^^^^^^^^^^^^^
Once the control plane is deployed, separate deployments can be done for each
DCN site. This section will document how to perform such a deployment.

.. _export_dcn:

Saving configuration from the overcloud
_______________________________________
Once the overcloud control plane has been deployed, data needs to be retrieved
from the overcloud Heat stack and plan to pass as input values into the
separate DCN deployment.

Extract the needed data from the control plane stack:

.. code-block:: bash

  # Pass --help to see a full list of options
  openstack overcloud export \
    --stack control-plane \
    --output-file control-plane-export.yaml

.. note::

  The `control-plane-export.yaml` generated in the previous command contains
  sensitive security data such as passwords and TLS certificates that are used
  in the overcloud deployment. Some passwords in the file may be removed if
  they are not needed by DCN. For example, the passwords for RabbitMQ, MySQL,
  Keystone, Nova and Neutron should be sufficient to launch an instance.  When
  the export comman is run, the Ceph passwords are excluded so that DCN
  deployments which include Ceph do not reuse the same Ceph password and
  instead new ones are generated per DCN deployment.

  Care should be taken to keep the file as secured as possible.

.. _reuse_networks_dcn:

Reusing networks from the overcloud
___________________________________
When deploying separate stacks it may be necessary to reuse networks, subnets,
and VIP resources between stacks if desired. Only a single Heat stack can own a
resource and be responsible for its creation and deletion, however the
resources can be reused in other stacks.

To reuse network related resources between stacks, the following parameters
have been added to the network definitions in the `network_data.yaml` file
format::

      external_resource_network_id: Existing Network UUID
      external_resource_subnet_id: Existing Subnet UUID
      external_resource_segment_id: Existing Segment UUID
      external_resource_vip_id: Existing VIP UUID

These parameters can be set on each network definition in the
`network_data.yaml` file used for the deployment of the separate stack.

Not all networks need to be reused or shared across stacks. The
`external_resource_*` parameters can be set for only the networks that are
meant to be shared, while the other networks can be newly created and managed.

For example, to reuse the `internal_api` network from the control plane stack
in a separate stack, run the following commands to show the UUIDs for the
related network resources:

.. code-block:: bash

      openstack network show internal_api -c id -f value
      openstack subnet show internal_api_subnet -c id -f value
      openstack port show internal_api_virtual_ip -c id -f value

Save the values shown in the output of the above commands and add them to the
network definition for the `internal_api` network in the `network_data.yaml`
file for the separate stack. An example network definition would look like:

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


DCN related roles
_________________
Different roles are provided within `tripleo-heat-templates`, depending on the
configuration and desired services to be deployed at each distributed site.

The default compute role at `roles/Compute.yaml` can be used if that is
sufficient for the use case.

Two additional roles are also available for deploying compute nodes with
co-located persistent storage at the distributed site.

The first is `roles/DistributedCompute.yaml`. This role includes the default
compute services, but also includes the cinder volume service. The cinder
volume service would be configured to talk to storage that is local to the
distributed site for persistent storage.

The second is `roles/DistributedComputeHCI.yaml`. This role includes the
default computes services, the cinder volume service, and also includes the
Ceph services for deploying a Ceph cluster at the distributed site. Using this
role, both the compute services and ceph services are deployed on the same
nodes, enabling a hyperconverged infrastructure for persistent storage at the
distributed site.

Configuring Availability Zones (AZ)
___________________________________
Each edge site must be configured as a separate availability zone (AZ). When
you deploy instances to this AZ, you can expect it to run on the remote Compute
node. In addition, the central site must also be within a specific AZ (or
multiple AZs), rather than the default AZ.

When also deploying persistent storage at each site, the storage backend
availability zone must match the compute availability zone name.

AZs are configured differently for compute (Nova) and storage (Cinder).
Configuring AZs are documented in the next sections.

Configuring AZs for Nova (compute)
##################################
The Nova AZ configuration for compute nodes in the stack can be set with the
``NovaComputeAvailabilityZone`` parameter during the deployment.

The value of the parameter is the name of the AZ where compute nodes in that
stack will be added.

For example, the following environment file would be used to add compute nodes
in the ``edge-0`` stack to the ``edge-0`` AZ::

      parameter_defaults:
         NovaComputeAvailabilityZone: edge-0

Additionally, the ``OS::TripleO::NovaAZConfig`` service must be enabled by
including the following ``resource_registry`` mapping::

      resource_registry:
        OS::TripleO::Services::NovaAZConfig: tripleo-heat-templates/deployment/nova/nova-az-config.yaml

Or, the following environment can be included which sets the above mapping::

      environments/nova-az-config.yaml

It's also possible to configure the AZ for a compute node by adding it to a
host aggregate after the deployment is completed. The following commands show
creating a host aggregate, an associated AZ, and adding compute nodes to a
``edge-0`` AZ::

    openstack aggregate create edge-0 --zone edge-0
    openstack aggregate add host edge-0 hostA
    openstack aggregate add host edge-0 hostB

.. note::

    The above commands are run against the deployed overcloud, not the
    undercloud. Make sure the correct rc file for the control plane stack of
    the overcloud is sourced for the shell before running the commands.


Configuring AZs for Cinder (storage)
####################################
Each site that uses consistent storage is configured with its own cinder
backend(s). Cinder backends are not shared between sites. Each backend is also
configured with an AZ that should match the configured Nova AZ for the compute
nodes that will make use of the storage provided by that backend.

The ``CinderStorageAvailabilityZone`` parameter can be used to configure the AZ
for a given backend. Parameters are also available for different backend types,
such as ``CinderISCSIAvailabilityZone``, ``CinderRbdAvailabilityZone``, and
``CinderNfsAvailabilityZone``. When set, the backend type specific parameter
will take precedence over ``CinderStorageAvailabilityZone``.

This example shows an environment file setting the AZ for the backend in the
``central`` site::

      parameter_defaults:
         CinderStorageAvailabilityZone: central

This example shows an environment file setting the AZ for the backend in the
``edge0`` site::

      parameter_defaults:
         CinderStorageAvailabilityZone: edge0

Deploying Ceph with HCI
########################
When deploying Ceph while using the ``DistributedComputeHCI`` roles, the
environment file to enable ceph should be used::

      environments/ceph-ansible/ceph-ansible.yaml

Sample environments
###################

There are sample environments that are included in ``tripleo-heat-templates``
for setting many of the parameter values and ``resource_registry`` mappings. These
environments are located within the ``tripleo-heat-templates`` directory at::

      environments/dcn.yaml
      environments/dcn-hci.yaml

The environments are not all-inclusive and do not set all needed values and
mappings, but can be used as a guide when deploying DCN.

Example: DCN deployment with pre-provisioned nodes, shared networks, and multiple stacks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This example shows the deployment commands and associated environment files
of a example of a DCN deployment. The deployment uses pre-provisioned nodes.
All networks are shared between the multiple stacks. The example illustrates
the deployment workflow of deploying multiple stacks for a real
world DCN deployment.

Four stacks are deployed:

control-plane
   All control plane services. Shares the same geographical location as the
   central stack.
central
   Compute, Cinder, Ceph deployment. Shares the same geographical location as
   the control-plane stack.
edge0
   Compute, Cinder, Ceph deployment. Separate geographical location from any
   other stack.
edge1
   Compute, Cinder, Ceph deployment. Separate geographical location from any
   other stack.

Notice how the ``central`` stack will contain only compute and storage
services. It is really just another instance of an edge site, but just happens
to be deployed at the same geographical location as the ``control-plane``
stack. ``control-plane`` and ``central`` could instead be deployed in the same
stack, however for easier manageability and separation, they are deployed in
separate stacks.

This example also uses pre-provisioned nodes as documented at
:ref:`deployed_server`.

Undercloud
__________
Since this example uses pre-provisioned nodes, no additional undercloud
configuration is needed. The steps in undercloud_dcn_ are not specifically
applicable when using pre-provisioned nodes.

Deploy the control-plane stack
______________________________
The ``control-plane`` stack is deployed with the following command::

    openstack overcloud deploy \
      --verbose \
      --stack control-plane \
      --disable-validations \
      --templates /home/centos/tripleo-heat-templates \
      -r roles-data.yaml \
      -e role-counts.yaml \
      -n network_data.yaml \
      -e /home/centos/tripleo-heat-templates/environments/docker-ha.yaml \
      -e /home/centos/tripleo/environments/containers-prepare-parameter.yaml \
      -e /home/centos/tripleo-heat-templates/environments/deployed-server-environment.yaml \
      -e /home/centos/tripleo-heat-templates/environments/deployed-server-bootstrap-environment-centos.yaml \
      -e /home/centos/tripleo-heat-templates/environments/network-isolation.yaml \
      -e /home/centos/tripleo-heat-templates/environments/net-multiple-nics.yaml \
      -e hostnamemap.yaml \
      -e network-environment.yaml \
      -e deployed-server-port-map.yaml \
      -e az.yaml


Many of the specified environments and options are not specific to DCN. The
ones that relate to DCN are as follows.

``--stack control-plane`` sets the stack name to ``control-plane``.

The ``roles-data.yaml`` file contains only the Controller role from the
templates directory at ``roles/Controller.yaml``.

``role-counts.yaml`` contains::

    parameter_defaults:
      ControllerCount: 1

``network_data.yaml`` contains the default contents from the templates
directory.

``az.yaml`` contains::

    parameter_defaults:
      CinderStorageAvailabilityZone: 'central'
      NovaComputeAvailabilityZone: 'central'
      NovaAZAttach: false

When the deployment completes, a single stack is deployed::

    (undercloud) [centos@scale ~]$ openstack stack list
    +--------------------------------------+---------------+----------------------------------+-----------------+----------------------+----------------------+
    | ID                                   | Stack Name    | Project | Stack Status    | Creation Time        | Updated Time         |
    +--------------------------------------+---------------+----------------------------------+-----------------+----------------------+----------------------+
    | 5f172fd8-97a5-4b9b-8d4c-2c931fd048e7 | control-plane | c117a9b489384603b2f45185215e9728 | CREATE_COMPLETE | 2019-03-13T18:51:08Z | 2019-03-13T19:44:27Z |
    +--------------------------------------+---------------+----------------------------------+-----------------+----------------------+----------------------+

.. _example_export_dcn:

Export configuration from the ``control-plane`` stack
_____________________________________________________
As documented in export_dcn_, the following command is run to save needed
configuration data from the ``control-plane`` stack::

  openstack overcloud export \
    --stack control-plane \
    --output-file control-plane-export.yaml

Deploy the central stack
________________________
The ``central`` stack deploys compute and storage services to be co-located
at the same site where the ``control-plane`` stack was deployed.

Before the deployment command is run, a new ``networks_data.yaml`` file needs
to be created and updated with the UUIDs of the existing network resources
that are reused from the ``control-plane`` stack in the ``central``
stack as documented in reuse_networks_dcn_.

The following commands are used to show the UUIDs::

    (undercloud) [centos@scale ~]$ openstack network list
    +--------------------------------------+--------------+--------------------------------------+
    | ID                                   | Name         | Subnets                              |
    +--------------------------------------+--------------+--------------------------------------+
    | 0fcb505b-81c8-483e-93f6-0da574e4acd5 | tenant       | e6544a7f-ec00-4b33-b7b0-a02e1c0f503a |
    | 40ed54c0-1c85-4bcb-b244-0764f83d2ca8 | management   | 9ca595f9-aa92-415a-9e13-0ed8b9f78e68 |
    | 447fd403-e977-436d-ba21-7d1ac258dd81 | internal_api | 3449c5f3-ebb0-4e77-b671-eb6ea209a10e |
    | 47a73786-4066-49ac-9e6a-49fb5d1f964a | storage_mgmt | eb78ae43-c575-4fdd-8c3f-405f4bdd0ca5 |
    | bf1fbe99-08f9-4f12-9af5-57a4f396b894 | ctlplane     | 5d366b80-a360-4b3d-be5f-c5dbd13fd7eb |
    | cf19bf6e-1ed5-428b-9aab-727d43e88f3a | external     | 6fc8578c-8028-450a-b83e-bf92cfda61bc |
    | ef89c994-5b8d-4a5d-aa53-ef02452665d0 | storage      | d6c975db-8943-4261-abf1-f7d2b482d88c |
    +--------------------------------------+--------------+--------------------------------------+
    (undercloud) [centos@scale ~]$ openstack subnet list
    +--------------------------------------+---------------------+--------------------------------------+----------------+
    | ID                                   | Name                | Network                              | Subnet         |
    +--------------------------------------+---------------------+--------------------------------------+----------------+
    | 3449c5f3-ebb0-4e77-b671-eb6ea209a10e | internal_api_subnet | 447fd403-e977-436d-ba21-7d1ac258dd81 | 172.16.2.0/24  |
    | 5d366b80-a360-4b3d-be5f-c5dbd13fd7eb | ctlplane-subnet     | bf1fbe99-08f9-4f12-9af5-57a4f396b894 | 192.168.0.0/24 |
    | 6fc8578c-8028-450a-b83e-bf92cfda61bc | external_subnet     | cf19bf6e-1ed5-428b-9aab-727d43e88f3a | 10.0.0.0/24    |
    | 9ca595f9-aa92-415a-9e13-0ed8b9f78e68 | management_subnet   | 40ed54c0-1c85-4bcb-b244-0764f83d2ca8 | 10.0.1.0/24    |
    | d6c975db-8943-4261-abf1-f7d2b482d88c | storage_subnet      | ef89c994-5b8d-4a5d-aa53-ef02452665d0 | 172.16.1.0/24  |
    | e6544a7f-ec00-4b33-b7b0-a02e1c0f503a | tenant_subnet       | 0fcb505b-81c8-483e-93f6-0da574e4acd5 | 172.16.0.0/24  |
    | eb78ae43-c575-4fdd-8c3f-405f4bdd0ca5 | storage_mgmt_subnet | 47a73786-4066-49ac-9e6a-49fb5d1f964a | 172.16.3.0/24  |
    +--------------------------------------+---------------------+--------------------------------------+----------------+
    (undercloud) [centos@scale ~]$ openstack port list
    +--------------------------------------+-------------------------+-------------------+-----------------------------------------------------------------------------+--------+
    | ID                                   | Name                    | MAC Address       | Fixed IP Addresses                                                          | Status |
    +--------------------------------------+-------------------------+-------------------+-----------------------------------------------------------------------------+--------+
    | 06603164-6fc0-4ca9-b480-5b73736dec01 | openstack-0_Storage     | fa:16:3e:8c:5e:8a | ip_address='172.16.1.200', subnet_id='d6c975db-8943-4261-abf1-f7d2b482d88c' | DOWN   |
    | 3b2244e4-0bf2-4675-a88f-3c149a5ab634 | openstack-0_External    | fa:16:3e:67:49:95 | ip_address='10.0.0.137', subnet_id='6fc8578c-8028-450a-b83e-bf92cfda61bc'   | DOWN   |
    | 7ed9ac55-fec0-4320-9ba6-d95bb5207680 | openstack-0_InternalApi | fa:16:3e:df:46:7e | ip_address='172.16.2.36', subnet_id='3449c5f3-ebb0-4e77-b671-eb6ea209a10e'  | DOWN   |
    | 824081da-9205-4ed9-9a94-047dccceb8ff | storage_mgmt_virtual_ip | fa:16:3e:f9:ff:5a | ip_address='172.16.3.222', subnet_id='eb78ae43-c575-4fdd-8c3f-405f4bdd0ca5' | DOWN   |
    | 894b834f-b911-42eb-a4b8-08e3b0084825 | public_virtual_ip       | fa:16:3e:d9:d2:f6 | ip_address='10.0.0.136', subnet_id='6fc8578c-8028-450a-b83e-bf92cfda61bc'   | DOWN   |
    | 9daa4ac1-c7f0-4e25-a6d1-1f00e2f0ee72 | openstack-0_Tenant      | fa:16:3e:eb:b4:f7 | ip_address='172.16.0.107', subnet_id='e6544a7f-ec00-4b33-b7b0-a02e1c0f503a' | DOWN   |
    | b140c67e-3755-4068-9c61-0349cee5695a | openstack-0_StorageMgmt | fa:16:3e:bc:9e:d7 | ip_address='172.16.3.49', subnet_id='eb78ae43-c575-4fdd-8c3f-405f4bdd0ca5'  | DOWN   |
    | b9299348-d761-410a-b81d-4d78b2d985a9 | internal_api_virtual_ip | fa:16:3e:9f:fb:fa | ip_address='172.16.2.244', subnet_id='3449c5f3-ebb0-4e77-b671-eb6ea209a10e' | DOWN   |
    | cdf8edac-55b0-4321-98fd-0201ec554c33 | storage_virtual_ip      | fa:16:3e:35:a6:55 | ip_address='172.16.1.147', subnet_id='d6c975db-8943-4261-abf1-f7d2b482d88c' | DOWN   |
    | d2d6a257-b43d-4a1c-ab13-cd91aa05d6fe |                         | fa:16:3e:a3:a5:b1 | ip_address='192.168.0.5', subnet_id='5d366b80-a360-4b3d-be5f-c5dbd13fd7eb'  | ACTIVE |
    +--------------------------------------+-------------------------+-------------------+-----------------------------------------------------------------------------+--------+

A copy of the default ``networks_data.yaml`` file is created::

    cp /home/centos/tripleo-heat-templates/networks_data.yaml site_networks_data.yaml

``site_networks_data.yaml`` is updated the external resource ids for each
network resource are added. For example, the ``InternalApi`` network definition
looks like::

    - name: InternalApi
      external_resource_id: 447fd403-e977-436d-ba21-7d1ac258dd81
      external_resource_subnet_id: 3449c5f3-ebb0-4e77-b671-eb6ea209a10e
      external_resource_vip_id: b9299348-d761-410a-b81d-4d78b2d985a9
      name_lower: internal_api
      vip: true
      ip_subnet: '172.16.2.0/24'
      allocation_pools: [{'start': '172.16.2.4', 'end': '172.16.2.250'}]
      ipv6_subnet: 'fd00:fd00:fd00:2000::/64'
      ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:2000::10', 'end': 'fd00:fd00:fd00:2000:ffff:ffff:ffff:fffe'}]
      mtu: 1400

The ``central`` stack is then deployed with the following command::

    openstack overcloud deploy \
      --verbose \
      --stack central \
      --templates /home/centos/tripleo-heat-templates \
      -r distributed-roles-data.yaml \
      -n site_network_data.yaml \
      --disable-validations \
      -e /home/centos/tripleo-heat-templates/environments/docker-ha.yaml \
      -e /home/centos/tripleo/environments/containers-prepare-parameter.yaml \
      -e /home/centos/tripleo-heat-templates/environments/deployed-server-environment.yaml \
      -e /home/centos/tripleo-heat-templates/environments/deployed-server-bootstrap-environment-centos.yaml \
      -e /home/centos/tripleo-heat-templates/environments/network-isolation.yaml \
      -e /home/centos/tripleo-heat-templates/environments/net-multiple-nics.yaml \
      -e /home/centos/tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
      -e /home/centos/tripleo-heat-templates/environments/low-memory-usage.yaml \
      -e role-counts.yaml \
      -e hostnamemap.yaml \
      -e network-environment.yaml \
      -e deployed-server-port-map.yaml \
      -e ceph-environment.yaml \
      -e az.yaml \
      -e /home/centos/control-plane-export.yaml

``--stack central`` sets the stack name to ``central``.

``distributed-roles-data.yaml`` contains a single role called ``DistributedComputeHCI``
which contains Nova, Cinder, and Ceph services. The example role is from the
templates directory at ``roles/DistributedComputeHCI.yaml``.

``role-counts.yaml`` contains::

  parameter_defaults:
    DistributedComputeHCICount: 1

``az.yaml`` contains the same content as was used in the ``control-plane``
stack::

    parameter_defaults:
      CinderStorageAvailabilityZone: 'central'
      NovaComputeAvailabilityZone: 'central'
      NovaAZAttach: false

The ``control-plane-export.yaml`` file was generated from the command from
example_export_dcn_.

When the deployment completes, 2 stacks are deployed::

      +--------------------------------------+---------------+----------------------------------+-----------------+----------------------+----------------------+
      | ID                                   | Stack Name    | Project                          | Stack Status    | Creation Time        | Updated Time         |
      +--------------------------------------+---------------+----------------------------------+-----------------+----------------------+----------------------+
      | 0bdade63-4645-4490-a540-24be48527e10 | central       | c117a9b489384603b2f45185215e9728 | CREATE_COMPLETE | 2019-03-25T21:35:49Z | None                 |
      | 5f172fd8-97a5-4b9b-8d4c-2c931fd048e7 | control-plane | c117a9b489384603b2f45185215e9728 | CREATE_COMPLETE | 2019-03-13T18:51:08Z | None                 |
      +--------------------------------------+---------------+----------------------------------+-----------------+----------------------+----------------------+

The AZ and aggregate configuration for Nova can be checked and verified with
these commands. Note that the ``rc`` file for the ``control-plane`` stack must be
sourced as these commands talk to overcloud APIs::

      (undercloud) [centos@scale ~]$ source control-planerc
      (control-plane) [centos@scale ~]$ openstack aggregate list
      +----+---------+-------------------+
      | ID | Name    | Availability Zone |
      +----+---------+-------------------+
      |  9 | central | central           |
      +----+---------+-------------------+
      (control-plane) [centos@scale ~]$ openstack aggregate show central
      +-------------------+----------------------------+
      | Field             | Value                      |
      +-------------------+----------------------------+
      | availability_zone | central                    |
      | created_at        | 2019-03-25T22:23:25.000000 |
      | deleted           | False                      |
      | deleted_at        | None                       |
      | hosts             | [u'compute-0.localdomain'] |
      | id                | 9                          |
      | name              | central                    |
      | properties        |                            |
      | updated_at        | None                       |
      +-------------------+----------------------------+
      (control-plane) [centos@scale ~]$ nova availability-zone-list
      +----------------------------+----------------------------------------+
      | Name                       | Status                                 |
      +----------------------------+----------------------------------------+
      | internal                   | available                              |
      | |- openstack-0.localdomain |                                        |
      | | |- nova-conductor        | enabled :-) 2019-03-27T18:21:29.000000 |
      | | |- nova-scheduler        | enabled :-) 2019-03-27T18:21:31.000000 |
      | | |- nova-consoleauth      | enabled :-) 2019-03-27T18:21:34.000000 |
      | central                    | available                              |
      | |- compute-0.localdomain   |                                        |
      | | |- nova-compute          | enabled :-) 2019-03-27T18:21:32.000000 |
      +----------------------------+----------------------------------------+
      (control-plane) [centos@scale ~]$ openstack compute service list
      +----+------------------+-------------------------+----------+---------+-------+----------------------------+
      | ID | Binary           | Host                    | Zone     | Status  | State | Updated At                 |
      +----+------------------+-------------------------+----------+---------+-------+----------------------------+
      |  1 | nova-scheduler   | openstack-0.localdomain | internal | enabled | up    | 2019-03-27T18:23:31.000000 |
      |  2 | nova-consoleauth | openstack-0.localdomain | internal | enabled | up    | 2019-03-27T18:23:34.000000 |
      |  3 | nova-conductor   | openstack-0.localdomain | internal | enabled | up    | 2019-03-27T18:23:29.000000 |
      |  7 | nova-compute     | compute-0.localdomain   | central  | enabled | up    | 2019-03-27T18:23:32.000000 |
      +----+------------------+-------------------------+----------+---------+-------+----------------------------+


Note how a new aggregate and AZ called ``central`` has been automatically
created. The newly deployed ``nova-compute`` service from the ``compute-0`` host in
the ``central`` stack is automatically added to this aggregate and zone.

The AZ configuration for Cinder can be checked and verified with these
commands::

      (control-plane) [centos@scale ~]$ openstack volume service list
      +------------------+-------------------------+---------+---------+-------+----------------------------+
      | Binary           | Host                    | Zone    | Status  | State | Updated At                 |
      +------------------+-------------------------+---------+---------+-------+----------------------------+
      | cinder-scheduler | openstack-0.rdocloud    | central | enabled | up    | 2019-03-27T21:17:44.000000 |
      | cinder-volume    | compute-0@tripleo_ceph  | central | enabled | up    | 2019-03-27T21:17:44.000000 |
      +------------------+-------------------------+---------+---------+-------+----------------------------+

The Cinder AZ configuration shows the ceph backend in the ``central`` zone
which was deployed by the ``central`` stack.

Deploy the edge-0 and edge-1 stacks
___________________________________
Now that the ``control-plane`` and ``central`` stacks are deployed, we'll deploy an
``edge-0`` and ``edge-1`` stack. These stacks are similar to the ``central`` stack in that they
deploy the same roles with the same services. It differs in that the nodes
will be managed in a separate stack and it illustrates the separation of
deployment and management between edge sites.

The AZs will be configured differently in these stacks as the nova and
cinder services will belong to an AZ unique to each the site.

The ``edge-0`` stack is deployed with the following command::

    openstack overcloud deploy \
      --verbose \
      --stack edge-0 \
      --templates /home/centos/tripleo-heat-templates \
      -r distributed-roles-data.yaml \
      -n site_network_data.yaml \
      --disable-validations \
      -e /home/centos/tripleo-heat-templates/environments/docker-ha.yaml \
      -e /home/centos/tripleo/environments/containers-prepare-parameter.yaml \
      -e /home/centos/tripleo-heat-templates/environments/deployed-server-environment.yaml \
      -e /home/centos/tripleo-heat-templates/environments/deployed-server-bootstrap-environment-centos.yaml \
      -e /home/centos/tripleo-heat-templates/environments/network-isolation.yaml \
      -e /home/centos/tripleo-heat-templates/environments/net-multiple-nics.yaml \
      -e /home/centos/tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
      -e /home/centos/tripleo-heat-templates/environments/low-memory-usage.yaml \
      -e role-counts.yaml \
      -e hostnamemap.yaml \
      -e network-environment.yaml \
      -e deployed-server-port-map.yaml \
      -e ceph-environment.yaml \
      -e az.yaml \
      -e /home/centos/control-plane-export.yaml

``--stack edge-0`` sets the stack name to ``edge-0``.

``distributed-roles-data.yaml`` contains a single role called ``DistributedComputeHCI``
which contains Nova, Cinder, and Ceph services. The example role is from the
templates directory at ``roles/DistributedComputeHCI.yaml``. This file is the
same as was used in the ``central`` stack.

``role-counts.yaml`` contains::

  parameter_defaults:
    DistributedComputeHCICount: 1

``az.yaml`` contains specific content for the ``edge-0`` stack::

    parameter_defaults:
      CinderStorageAvailabilityZone: 'edge-0'
      NovaComputeAvailabilityZone: 'edge-0'
      NovaAZAttach: false

The ``CinderStorageAvailabilityZone`` and ``NovaDefaultAvailabilityZone``
parameters are set to ``edge-0`` to match the stack name.

The ``control-plane-export.yaml`` file was generated from the command from
example_export_dcn_, and is the same file that was used with the ``central``
stack.

The ``edge-1`` stack is deployed with a similar command. The stack is given a
different name with ``--stack edge-1`` and ``az.yaml`` contains::

    parameter_defaults:
      CinderStorageAvailabilityZone: 'edge-1'
      NovaComputeAvailabilityZone: 'edge-1'
      NovaAZAttach: false

When the deployment completes, there are now 4 stacks are deployed::

    (undercloud) [centos@scale ~]$ openstack stack list
    +--------------------------------------+---------------+----------------------------------+-----------------+----------------------+----------------------+
    | ID                                   | Stack Name    | Project                          | Stack Status    | Creation Time        | Updated Time         |
    +--------------------------------------+---------------+----------------------------------+-----------------+----------------------+----------------------+
    | 203dc480-3b0b-4cd9-9f70-f79898461c17 | edge-0        | c117a9b489384603b2f45185215e9728 | CREATE_COMPLETE | 2019-03-29T17:30:15Z | None                 |
    | 0bdade63-4645-4490-a540-24be48527e10 | central       | c117a9b489384603b2f45185215e9728 | CREATE_COMPLETE | 2019-03-25T21:35:49Z | None                 |
    | 5f172fd8-97a5-4b9b-8d4c-2c931fd048e7 | control-plane | c117a9b489384603b2f45185215e9728 | UPDATE_COMPLETE | 2019-03-13T18:51:08Z | 2019-03-13T19:44:27Z |
    +--------------------------------------+---------------+----------------------------------+-----------------+----------------------+----------------------+

Repeating the same commands that were run after the ``central`` stack was
deployed to show the AZ configuration, shows that the new AZs for ``edge-0``
and ``edge-1`` are created and available::

    (undercloud) [centos@scale ~]$ source control-planerc
    (control-plane) [centos@scale ~]$ openstack aggregate list
    +----+---------+-------------------+
    | ID | Name    | Availability Zone |
    +----+---------+-------------------+
    |  9 | central | central           |
    | 10 | edge-0  | edge-0            |
    | 11 | edge-1  | edge-1            |
    +----+---------+-------------------+
    (control-plane) [centos@scale ~]$ openstack aggregate show edge-0
    +-------------------+----------------------------+
    | Field             | Value                      |
    +-------------------+----------------------------+
    | availability_zone | edge-0                     |
    | created_at        | 2019-03-29T19:01:59.000000 |
    | deleted           | False                      |
    | deleted_at        | None                       |
    | hosts             | [u'compute-1.localdomain'] |
    | id                | 10                         |
    | name              | edge-0                     |
    | properties        |                            |
    | updated_at        | None                       |
    +-------------------+----------------------------+
    (control-plane) [centos@scale ~]$ nova availability-zone-list
    +----------------------------+----------------------------------------+
    | Name                       | Status                                 |
    +----------------------------+----------------------------------------+
    | internal                   | available                              |
    | |- openstack-0.localdomain |                                        |
    | | |- nova-conductor        | enabled :-) 2019-04-01T17:38:06.000000 |
    | | |- nova-scheduler        | enabled :-) 2019-04-01T17:38:13.000000 |
    | | |- nova-consoleauth      | enabled :-) 2019-04-01T17:38:09.000000 |
    | central                    | available                              |
    | |- compute-0.localdomain   |                                        |
    | | |- nova-compute          | enabled :-) 2019-04-01T17:38:07.000000 |
    | edge-0                     | available                              |
    | |- compute-1.localdomain   |                                        |
    | | |- nova-compute          | enabled :-) 2019-04-01T17:38:07.000000 |
    | edge-1                     | available                              |
    | |- compute-2.localdomain   |                                        |
    | | |- nova-compute          | enabled :-) 2019-04-01T17:38:06.000000 |
    +----------------------------+----------------------------------------+
    (control-plane) [centos@scale ~]$ openstack compute service list
    +----+------------------+-------------------------+----------+---------+-------+----------------------------+
    | ID | Binary           | Host                    | Zone     | Status  | State | Updated At                 |
    +----+------------------+-------------------------+----------+---------+-------+----------------------------+
    |  1 | nova-scheduler   | openstack-0.localdomain | internal | enabled | up    | 2019-04-01T17:38:23.000000 |
    |  2 | nova-consoleauth | openstack-0.localdomain | internal | enabled | up    | 2019-04-01T17:38:19.000000 |
    |  3 | nova-conductor   | openstack-0.localdomain | internal | enabled | up    | 2019-04-01T17:38:26.000000 |
    |  7 | nova-compute     | compute-0.localdomain   | central  | enabled | up    | 2019-04-01T17:38:27.000000 |
    | 16 | nova-compute     | compute-1.localdomain   | edge-0   | enabled | up    | 2019-04-01T17:38:27.000000 |
    | 17 | nova-compute     | compute-2.localdomain   | edge-1   | enabled | up    | 2019-04-01T17:38:26.000000 |
    +----+------------------+-------------------------+----------+---------+-------+----------------------------+
    (control-plane) [centos@scale ~]$ openstack volume service list
    +------------------+-------------------------+---------+---------+-------+----------------------------+
    | Binary           | Host                    | Zone    | Status  | State | Updated At                 |
    +------------------+-------------------------+---------+---------+-------+----------------------------+
    | cinder-scheduler | openstack-0.rdocloud    | central | enabled | up    | 2019-04-01T17:38:27.000000 |
    | cinder-volume    | hostgroup@tripleo_iscsi | central | enabled | up    | 2019-04-01T17:38:27.000000 |
    | cinder-volume    | compute-0@tripleo_ceph  | central | enabled | up    | 2019-04-01T17:38:30.000000 |
    | cinder-volume    | compute-1@tripleo_ceph  | edge-0  | enabled | up    | 2019-04-01T17:38:32.000000 |
    | cinder-volume    | compute-2@tripleo_ceph  | edge-1  | enabled | up    | 2019-04-01T17:38:28.000000 |
    +------------------+-------------------------+---------+---------+-------+----------------------------+
    (control-plane) [centos@scale ~]$


Updating DCN
------------

Each stack in a multi-stack DCN deployment must be updated to perform a full
minor update across the entire deployment.

The minor update procedure as detailed in :ref:`package_update` be run for
each stack in the deployment.

The control-plane or central stack should be updated first by completing all
the steps from the minor update procedure.

Once the central stack is updated, re-run the export command from
:ref:`export_dcn` to recreate the required input file for each separate
DCN stack.

.. note::

   When re-running the export command, save the generated file in a new
   directory so that the previous version is not overwritten. In the event
   that a separate DCN stack needs a stack update operation performed prior to
   the minor update procedure, the previous version of the exported file
   should be used.

Each separate DCN stack can then be updated individually as required. There is
no requirement as to the order of which DCN stack is updated first.

Running Ansible across multiple DCN stacks
------------------------------------------

.. warning::
   This currently is only supported in Train or newer versions.

Each DCN stack should usually be updated individually. However if you
need to run Ansible on nodes deployed from more than one DCN stack,
then the `tripleo-ansible-inventory` command's `--stack` option
supports being passed more than one stack. If more than one stack is
passed, then a single merged inventory will be generated which
contains the union of the nodes in those stacks. For example, if you
were to run the following::

  tripleo-ansible-inventory --static-yaml-inventory inventory.yaml --stack central,edge0

then you could use the genereated inventory.yaml as follows::

  (undercloud) [stack@undercloud ~]$ ansible -i inventory.yaml -m ping central
  central-controller-0 | SUCCESS => {
      "ansible_facts": {
          "discovered_interpreter_python": "/usr/bin/python"
      },
      "changed": false,
      "ping": "pong"
  }
  (undercloud) [stack@undercloud ~]$ ansible -i inventory.yaml -m ping edge0
  edge0-distributedcomputehci-0 | SUCCESS => {
      "ansible_facts": {
          "discovered_interpreter_python": "/usr/bin/python"
      },
      "changed": false,
      "ping": "pong"
  }
  (undercloud) [stack@undercloud ~]$ ansible -i inventory.yaml -m ping all
  undercloud | SUCCESS => {
      "changed": false,
      "ping": "pong"
  }
  edge0-distributedcomputehci-0 | SUCCESS => {
      "ansible_facts": {
          "discovered_interpreter_python": "/usr/bin/python"
      },
      "changed": false,
      "ping": "pong"
  }
  central-controller-0 | SUCCESS => {
      "ansible_facts": {
          "discovered_interpreter_python": "/usr/bin/python"
      },
      "changed": false,
      "ping": "pong"
  }
  (undercloud) [stack@undercloud ~]$

When multiple stacks are passed as input a host group is created
after each stack which refers to all of the nodes in that stack.
In the example above, edge0 has only one node from the
DistributedComputeHci role and central has only one node from the
Controller role.

The inventory will also have a host group created for every item in
the cross product of stacks and roles. For example,
central_Controller, edge0_Compute, edge1_Compute, etc. This is done
in order to avoid name collisions, e.g. Compute would refer to all
nodes in the Compute role, but when there's more than one stack
edge0_Compute and edge1_Compute refer to different Compute nodes
based on the stack from which they were deployed.
