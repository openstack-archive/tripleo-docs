.. _custom_networks:

Deploying with Custom Networks
==============================

TripleO offers the option of deploying with a user-defined list of networks,
where each network can be enabled (or not) for each role (group of servers) in
the deployment.

Default networks
----------------

TripleO offers a default network topology when deploying with network isolation
enabled, and this is reflected in the default-network-isolation_ file in
tripleo-heat-templates_.

.. admonition:: Victoria and prior releases

  In Victoria and prior releases the default network topology is reflected in
  the network_data.yaml_ file in tripleo-heat-templates_.

These default networks are as follows:

* ``External`` - External network traffic (disabled by default for
  Compute/Storage nodes)

* ``InternalApi`` - Internal API traffic, most intra-service traffic uses this
  network by default

* ``Storage`` - Storage traffic

* ``StorageMgmt`` - Storage management traffic (such as replication traffic
  between storage nodes)

* ``Tenant`` - Tenant networks for compute workloads running on the cloud

Deploying with custom networks
------------------------------

Each network is defined in the ``network_data`` YAML file. There are sample
files in ``/usr/share/openstack-tripleo-heat-templates/network-data-samples``,
or the tripleo-heat-templates_ git repository which can be copied and modified
as needed.

The ``network_data`` YAML file contains a list of networks, with definitions
like:

.. code-block:: yaml

  - name: CustomNetwork
    vip: false
    name_lower: custom_network
    subnets:
      custom_network_subnet:
        ip_subnet: 172.16.6.0/24
        allocation_pools:
          - start: 172.16.6.4
          - end: 172.16.6.250
        gateway_ip: 172.16.6.1

.. admonition:: Victoria and prior releases

  Victoria and releases prior to it used a slightly different ``network_data``
  YAML.

  .. code-block:: yaml

    - name: CustomNetwork
      vip: false
      name_lower: custom_network
      ip_subnet: '172.16.6.0/24'
      allocation_pools: [{'start': '172.16.6.4', 'end': '172.16.6.250'}]
      gateway_ip: '172.16.6.1'

The data in the ``network_data`` YAML definition is used to create and update
the network and subnet API resources in Neutron on the undercloud. It is also
used to perform templating with jinja2_ such that arbitrary user-defined
networks may be added, and the default networks may be modified or removed.

The steps to define your custom networks are:

#. Copy one of the sample ``network_data`` YAML definitions provided by
   tripleo-heat-templates_, for example::

    cp /usr/share/openstack-tripleo-heat-templates/network-data-samples/default-network-isolation.yaml \
      custom_network_data.yaml


   .. admonition:: Victoria and prior releases

     In Victoria and earlier releases the sample network data YAML was in a
     different location.

     ::

       cp /usr/share/openstack-tripleo-heat-templates/network_data.yaml custom_network_data.yaml

#. Modify the ``custom_network_data.yaml`` file as required. The network data
   is a list of networks, where each network contains at least the
   following items:

   :name: Name of the network (mandatory)
   :vip: Enable creation of a virtual IP on this network
   :subnets: Dictionary's, one or more subnet definition items keyed by the
     subnet name.

     :subnet_name: Name of the subnet

       :ip_subnet: IP/CIDR, e.g. ``'10.0.0.0/24'``

       :allocation_pools: IP range list, e.g. ``[{'start':'10.0.0.4', 'end':'10.0.0.250'}]``

       :gateway_ip: Gateway for the network

       :vlan: Vlan ID for this network. (supported in Queens and later)

   See `Network data YAML options`_ for a list of all documented options for
   the ``network_data`` YAML network definition.

   .. admonition:: Victoria and prior releases

     Victoria and earlier releases requires the first subnet definition **not**
     to be in the *subnets* dictionary.

     :name: Name of the network (mandatory)
     :vip: Enable creation of a virtual IP on this network
     :vlan: Vlan ID for this network. (supported in Queens and later)
     :ip_subnet: IP/CIDR, e.g. ``'10.0.0.0/24'``
     :allocation_pools: IP range list, e.g. ``[{'start':'10.0.0.4', 'end':'10.0.0.250'}]``
     :gateway_ip: Gateway for the network

     Other options are supported, see the documentation in the default
     network_data.yaml_ for details.

   .. warning::
      Currently there is no validation of the network subnet and
      allocation_pools, so care must be take to ensure these are consistent,
      and do not conflict with any existing networks, otherwise your deployment
      may fail or result in unexpected results.

#. Copy one of the sample ``vip_data`` YAML definitions provided by
   tripleo-heat-templates_, for example::

     cp /usr/share/openstack-tripleo-heat-templates/network-data-samples/vip-data-default-network-isolation.yaml  \
       custom_vip_data.yaml

   .. admonition:: Victoria and prior releases

     For Victoria and prior releases the Virtual IP resources are created as
     part of the overcloud heat stack. This step is not valid for these
     releases.

#. Modify the ``custom_vip_data.yaml`` file as required. The Virtual IP data
   is a list of Virtual IP address definitions, each containing at a minimum
   the name of the network where the IP address should be allocated.

   See `Network Virtual IPs data YAML options`_ for a list of all documented
   options for the ``vip_data`` YAML network Virtual IPs definition.

   .. admonition:: Victoria and prior releases

     For Victoria and prior releases the Virtual IP resources are created as
     part of the overcloud heat stack. This step is not valid for these
     releases.

#. Copy network configuration templates, add new networks.

   Prior to Victoria, Heat templates were used to define nic configuration
   templates. With the Victoria release, Ansible jinja2_ templates were
   introduced, and replaced the heat templates.

   The nic configuration examples in tripleo-heat-templates_ was ported to
   Ansible jinja2_ templates located in the tripleo_network_config role in
   tripleo-ansible_.

   If one of the shipped examples match, use it! If not, be inspired by the
   shipped examples and create a set of custom Ansible jinja2 templates. Please
   refer to the :ref:`creating_custom_interface_templates` documentation page
   which provide a detailed guide on how to create custom Ansible jinja2
   nic config templates.

   For example, copy a sample template to a custom location::

     cp -r /usr/share/ansible/roles/tripleo_network_config/templates/single_nic_vlans custom-single-nic-vlans

   Modify the templates in custom-single-nic-vlans to match your needs.

   .. admonition:: Ussuri and prior releases

      Prior to Queens, the nic config templates were not dynamically generated,
      so it was necessary to copy those that were in use, and add parameters for
      any additional networks, for example::

        cp -r /usr/share/openstack-tripleo-heat-templates/network/config/single-nic-vlans custom-single-nic-vlans

      Each file in ``single-nic-vlans`` will require updating to add
      parameters for each custom network. Copy those that exist for the
      default networks, and rename to match the *name* field in
      ``custom_network_data.yaml``.

      .. note::
         Since Queens, the NIC config templates are dynamically
         generated so this step is only necessary when creating custom NIC
         config templates, not when just adding a custom network.


#. Set your environment overrides to enable your nic config templates. 
   
   Create or update an existing environment file and set the parameter values
   to enable your custom nic config templates, for example create a file
   ``custom-net-single-nic-with-vlans.yaml`` with these parameter settings::

     parameter_defaults:
       ControllerNetworkConfigTemplate: '/path/to/custom-single-nic-vlans/single_nic_vlans.j2'
       CephStorageNetworkConfigTemplate: '/path/to/custom-single-nic-vlans/single_nic_vlans_storage.j2'
       ComputeNetworkConfigTemplate: '/path/to/custom-single-nic-vlans/single_nic_vlans.j2'

#. Create the networks on the undercloud and generate the
   ``networks-deployed-environment.yaml`` which will be used as an environment
   file when deploying the overcloud.

   ::

     openstack overcloud network provision \
       --output networks-deployed-environment.yaml \
       custom_network_data.yaml

   .. admonition:: Victoria and prior releases

     For Victoria and earlier releases *skip* this step.

     There was no command ``openstack overcloud network provision`` in these
     releases. Network resources was created as part of the overcloud heat
     stack.

   .. note:: This step is optional when using the ``--baremetal-deployment``
             and ``--vip-data`` options with the ``overcloud deploy`` command.
             The deploy command will detect the new format of the network data
             YAML definition, run the workflow to create the networks and
             include the ``networks-deployed-environment.yaml`` automatically.

#. Create the overcloud network Virtual IPs and generate the
   ``vip-deployed-environment.yaml`` which will be used as an environment file
   when deploying the overcloud.

   .. code-block:: bash

     $ openstack overcloud network vip provision  \
         --output ~/templates/vip-deployed-environment.yaml \
         ~/templates/custom_vip_data.yaml

   .. note:: This step is optional if using the ``--vip-data`` options with the
             ``overcloud deploy`` command. In that case workflow to create the
             Virtual IPs and including the environment is automated.

#. To deploy you pass the ``custom_network_data.yaml`` file via the ``-n``
   option to the overcloud deploy, for example:

   .. code-block:: bash

      openstack overcloud deploy --templates \
        -n custom_network_data.yaml \
        -e networks-deployed-environment.yaml \
        -e vip-deployed-environment.yaml \
        -e custom-net-single-nic-with-vlans.yaml

   Alternatively include the network, Virtual  IPs and baremetal provisioning
   in the ``overcloud deploy`` command to do it all in one:

   .. code-block:: bash

      openstack overcloud deploy --templates \
        --networks-file custom_network_data.yaml \
        --vip-file custom_vip_data.yaml \
        --baremetal-deployment baremetal_deployment.yaml \
        --network-config \
        -e custom-net-single-nic-with-vlans.yaml

   .. note:: Please refer to :doc:`../provisioning/baremetal_provision`
             document page for a reference on the ``baremetal_deployment.yaml``
             used in the above example.


   .. admonition:: Victoria and prior releases

     ::

       openstack overcloud deploy --templates \
         -n custom_network_data.yaml \
         -e custom-net-single-nic-with-vlans.yaml

   .. note::
     It is also possible to copy the entire tripleo-heat-templates tree, and
     modify the ``network_data.yaml`` file in place, then deploy via
     ``--templates <copy of tht>``.


.. _network_definition_opts:

Network data YAML options
-------------------------

:name:
  Name of the network

  type: *string*

:name_lower:
  *(optional)* Lower case name of the network

  type: *string*

  default: *name.lower()*

:dns_domain:
  *(optional)* Dns domain name for the network

  type: *string*

:mtu:
  *(optional)* Set the maximum transmission unit (MTU) that is guaranteed to
  pass through the data path of the segments in the network.

  type: *number*

  default: 1600

:service_net_map_replace:
  *(optional)* if name_lower is set to a custom name this should be set to
  original default (optional). This field is only necessary when changing the
  default network names, not when adding a new custom network.

  type: *string*

  .. warning:: Please avoid using this option, the correct solution when
               changing a *name_lower* of one of the default networks is to
               also update the ``ServiceNetMap`` parameter to use the same
               custom *name_lower*.

:ipv6:
  *(optional)*

  type: *boolean*

  default: *false*

:vip:
  *(optional)* Enable creation of a virtual IP on this network

  type: *boolean*

  default: *false*

:subnets:
  A map of subnets for the network. The collection should contain keys which
  define the subnet name. The value for each item is a subnet definition.

  Example:

  .. code-block:: yaml

    subnets:
      subnet_name_a:
        ip_subnet: 192.0.2.0/24
        allocation_pools:
          - start: 192.0.2.50
            end: 192.0.2.99
        gateway_ip: 192.0.2.1
        vlan: 102
      subnet_name_b:
        ip_subnet: 198.51.100.0/24
        allocation_pools:
          - start: 198.51.100.50
            end: 198.51.100.99
        gateway_ip: 198.51.100.1
        vlan: 101

  See `Options for network data YAML subnet definitions`_ for a list of all
  documented sub-options for the subnet definitions.

  type: *dictonary*


Options for network data YAML subnet definitions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:ip_subnet:
  IPv4 CIDR block notation for this subnet. For example: ``192.0.2.0/24``

  type: *string*

  .. note:: Optional if ``ipv6_subnet`` is specified.

:ipv6_subnet:
  IPv6 CIDR block notation for this subnet. For example:
  ``2001:db8:fd00:1000::/64``

  type: *string*

  .. note:: Optional if ``ip_subnet`` is specified.

:gateway_ip:
  *(optional)* The gateway IPv4 address

  type: *string*

:gateway_ipv6:
  *(optional)* The gateway IPv6 address

:allocation_pools:
  *(optional)* The start and end addresses for the subnets IPv4 allocation
  pools.

  type: *list*

  elements: *dictonary*

  :suboptions:

    :start: Start address for the allocation pool.

      type: *string*

    :end: End address for the allocation pool.

      type: *string*

  Example:

  .. code-block:: yaml

    allocation_pools:
      - start: 192.0.2.50
        end: 192.0.2.99
      - start: 192.0.2.150
        end: 192.0.2.199

:ipv6_allocation_pools:
  *(optional)* The start and end addresses for the subnets IPv6 allocation
  pools.

  type: *list*

  elements: *dictonary*

  :suboptions:

    :start: Start address for the allocation pool.

      type: *string*

    :end: End address for the allocation pool.

      type: *string*

  Example:

  .. code-block:: yaml

    allocation_pools:
      - start: 2001:db8:fd00:1000:100::1
        end: 2001:db8:fd00:1000:199::1
      - start: 2001:db8:fd00:1000:300::1
        end: 2001:db8:fd00:1000:399::1

:routes:
  *(optional)* List of networks that should be routed via network gateway. A
  single /16 supernet route could be used for 255 smaller /24 subnets.

  type: *list*

  elements: *dictonary*

  :suboptions:

    :destination: Destination network,
      for example: ``198.51.100.0/24``

      type: *string*

    :nexthop: IP address of the router to use for the destination network,
      for example: ``192.0.2.1``

      type: *string*

  Example:

  .. code-block:: yaml

    routes:
      - destination: 198.51.100.0/24
        nexthop: 192.0.2.1
      - destination: 203.0.113.0/24
        nexthost: 192.0.2.1

:routes_ipv6:
  *(optional)* List of IPv6 networks that should be routed via network gateway.

  type: *list*

  elements: *dictonary*

  :suboptions:

    :destination: Destination network,
      for example: ``2001:db8:fd00:2000::/64``

      type: *string*

    :nexthop: IP address of the router to use for the destination network,
      for example: ``2001:db8:fd00:1000::1``

      type: *string*

  Example:

  .. code-block:: yaml

    routes:
      - destination: 2001:db8:fd00:2000::/64
        nexthop: 2001:db8:fd00:1000:100::1
      - destination: 2001:db8:fd00:3000::/64
        nexthost: 2001:db8:fd00:1000:100::1

:vlan:
  *(optional)* vlan ID for the network

  type: *number*


.. _virtual_ips_definition_opts:

Network Virtual IPs data YAML options
-------------------------------------

:network:
  Neutron Network name

  type: *string*

:ip_address:
  *(optional)* IP address, a pre-defined fixed IP address.

  type: *string*

:subnet:
  *(optional)* Neutron Subnet name, used to specify the subnet to use when
  creating the Virtual IP neutron port.

  This is required for deployments using routed networks, to ensure the Virtual
  IP is allocated on the subnet where controller nodes are attached.

  type: *string*

:dns_name:
  *(optional)* Dns Name, the hostname part of the FQDN (Fully Qualified Domain
  Name)

  type: *string*

  default: overcloud

:name:
  *(optional)* Virtual IP name

  type: *string*
  default: $network_name_virtual_ip

.. _tripleo-heat-templates: https://opendev.org/openstack/tripleo-heat-templates
.. _default-network-isolation: https://opendev.org/openstack/tripleo-heat-templates/network-data-samples/default-network-isolation.yaml
.. _network_data.yaml: https://opendev.org/openstack/tripleo-heat-templates/src/branch/master/network_data.yaml
.. _jinja2: http://jinja.pocoo.org/docs/dev/
.. _tripleo-ansible: https://opendev.org/openstack/tripleo-ansible/src/branch/master/tripleo_ansible/roles/tripleo_network_config/templates
