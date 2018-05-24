Deploying with Custom Networks
==============================

TripleO offers the option of deploying with a user-defined list of networks,
where each network can be enabled (or not) for each role (group of servers) in
the deployment.

Default networks
----------------

TripleO offers a default network topology when deploying with network isolation
enabled, and this is reflected in the network_data.yaml_ file in
tripleo-heat-templates_.

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

Each network is defined in the ``network_data.yaml`` file. There is a sample
file in ``/usr/share/openstack-tripleo-heat-templates``, or the
tripleo-heat-templates_ git repository which can be copied and modified
as needed.

The ``network_data.yaml`` file contains a list of networks, with definitions
like::

  - name: CustomNetwork
    vip: false
    name_lower: custom_network
    ip_subnet: '172.16.6.0/24'
    allocation_pools: [{'start': '172.16.6.4', 'end': '172.16.6.250'}]
    gateway_ip: '172.16.6.1'

The data in ``network_data.yaml`` is used to perform templating with jinja2_
such that arbitrary user-defined networks may be added, and the default
networks may be modified or removed.

The steps to define your custom networks are:

1. Copy the default ``network_data.yaml`` provided by tripleo-heat-templates_::

       cp /usr/share/openstack-tripleo-heat-templates/network_data.yaml custom_network_data.yaml

2. Modify the ``custom_network_data.yaml`` file as required. The network data
   is a list of networks, where each network contains at least the
   following items:

   name
    Name of the network (mandatory)
   vip
    Enable creation of a virtual IP on this network
   ip_subnet
    IP/CIDR, e.g. ``'10.0.0.0/24'``
   allocation_pools
    IP range list, e.g. ``[{'start':'10.0.0.4', 'end':'10.0.0.250'}]``
   gateway_ip
    gateway for the network
   vlan  (supported in Queens and later)
    Vlan ID for this network.

   Other options are supported, see the documentation in the default
   network_data.yaml_ for details.

   .. warning::
      Currently there is no validation of the network subnet and
      allocation_pools, so care must be take to ensure these are consistent,
      and do not conflict with any existing networks, otherwise your deployment
      may fail or result in unexpected results.

3. Copy network configuration templates, add new networks.

   Prior to Queens the nic config templates are not dynamically generated, so it is
   necessary to copy those that are in use, and add parameters for any
   additional networks, for example::

      cp -r /usr/share/openstack-tripleo-heat-templates/network/config/single-nic-vlans custom-single-nic-vlans

   Each file in ``single-nic-vlans`` will require update to add parameters for
   each custom network. Copy those that exist for the default networks, and
   rename to match the *name* field in ``custom_network_data.yaml``.

  .. note::
     In Queens and later the NIC config templates are dynamically generated so
     this step is only necessary when creating custom NIC config templates,
     not when just adding a custom network.


4. Adjust your network-environment to reference the modified nic templates.

   It is necessary to adjust the environment paths to match the location of the
   copied nic templates above::

      cp /usr/share/openstack-tripleo-heat-templates//environments/net-single-nic-with-vlans.yaml custom-net-single-nic-with-vlans.yaml

   Edit the paths for each role to reference ``custom-single-nic-vlans``
   directory created above.

5. To deploy you pass the ``custom_network_data.yaml`` file via the ``-n``
   option to the overcloud deploy, for example::

      openstack overcloud deploy --templates -n custom_network_data.yaml -e custom-net-single-nic-with-vlans.yaml

   .. note::
     It is also possible to copy the entire tripleo-heat-templates tree, and
     modify the ``network_data.yaml`` file in place, then deploy via
     ``--templates <copy of tht>``.

.. _tripleo-heat-templates: https://git.openstack.org/cgit/openstack/tripleo-heat-templates
.. _network_data.yaml: https://git.openstack.org/cgit/openstack/tripleo-heat-templates/tree/network_data.yaml
.. _jinja2: http://jinja.pocoo.org/docs/dev/
