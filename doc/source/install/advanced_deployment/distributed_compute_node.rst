.. _distributed_compute_node:

Distributed Compute Node deployment
===================================

Introduction
------------
Additional groups of compute nodes can be deployed and integrated with an
existing deployment of a control plane stack. These compute nodes are deployed
in separate stacks from the main control plane (overcloud) stack, and they
consume some of the stack outputs from the overcloud stack to reuse as
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

Deploying from a centralized undercloud
---------------------------------------
The main overcloud control plane stack should be deployed as needed for the
desired cloud architecture layout. This stack contains nodes running the
control plane and infrastructure services needed for the cloud. For the
purposes of this documentation, this stack is referred to as the overcloud
stack.

The overcloud stack may or may not contain compute nodes. It may be a user
requirement that compute services are available within the overcloud stack,
however it is not strictly required.

Undercloud configuration
^^^^^^^^^^^^^^^^^^^^^^^^
TODO

Saving configuration from the overcloud
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Once the overcloud has been deployed, data needs to be retrieved
from the overcloud Heat stack and plan to pass as input values into the
separate DCN deployment.

Extract the needed data from the stack outputs:

.. code-block:: bash

  # EndpointMap: Cloud service to URL mapping
  openstack stack output show standalone EndpointMap --format json \
    | jq '{"parameter_defaults": {"EndpointMapOverride": .output_value}}' \
    > endpoint-map.json

  # AllNodesConfig: Node specific hieradata (hostnames, etc) set on all nodes
  openstack stack output show standalone AllNodesConfig --format json \
    | jq '{"parameter_defaults": {"AllNodesExtraMapData": .output_value}}' \
    > all-nodes-extra-map-data.json

  # GlobalConfig: Service specific hieradata set on all nodes
  openstack stack output show $STACK GlobalConfig --format json \
    | jq '{"parameter_defaults": {"GlobalConfigExtraMapData": .output_value}}' \
    > $DIR/global-config-extra-map-data.json

  # HostsEntry: Entries for /etc/hosts set on all nodes
  openstack stack output show standalone HostsEntry -f json \
    | jq -r '{"parameter_defaults":{"ExtraHostFileEntries": .output_value}}' \
    > extra-host-file-entries.json

The same passwords and secrets should be reused when deploying the additional
compute stacks. These values can be saved from the existing control plane stack
deployment with the following command::

.. code-block:: bash

  openstack object save overcloud plan-environment.yaml
  python -c "import yaml; data=yaml.safe_load(open('plan-environment.yaml').read()); print yaml.dump(dict(parameter_defaults=data['passwords']))" > passwords.yaml

Use the passwords.yaml enviornment file generated by the previous command, or
reuse the environment file used to set the values for the control plane stack.

.. note::

  The `passwords.yaml` generated in the previous command contains sensitive
  security data such as passwords and TLS certificates that are used in the
  overcloud deployment.

  Care should be taken to keep the file as secured as possible.

Create an environment file for setting necessary oslo messaging configuration
overrides:

.. code-block:: bash

  parameter_defaults:
    ComputeExtraConfig:
      oslo_messaging_notify_use_ssl: false
      oslo_messaging_rpc_use_ssl: false


Reusing networks from the overcloud
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
TODO

Spine and Leaf configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
TODO


Standalone deployment
---------------------
TODO
