Deploy an additional nova cell v2 in Stein release
==================================================

.. warning::
   Multi cell support is only supported in Stein or later versions.
   This guide addresses only the Stein release!

.. contents::
   :depth: 3
   :backlinks: none

This guide assumes that you are ready to deploy a new overcloud, or have
already installed an overcloud (min Stein release).

.. note::

   Starting with CentOS 8 and TripleO Stein release, podman is the CONTAINERCLI
   to be used in the following steps.

Initial Deploy
--------------

.. note::

   Right now the current implementation does not support running nova metadata
   API per cell as explained in the cells v2 layout section `Local per cell
   <https://docs.openstack.org/nova/latest/user/cellsv2-layout.html#nova-metadata-api-service>`_

The following example uses six nodes and the split control plane method to
simulate a distributed cell deployment. The first Heat stack deploys a controller
cluster and a compute. The second Heat stack deploys a cell controller and a
compute node::

    openstack overcloud status
    +-----------+---------------------+---------------------+-------------------+
    | Plan Name |       Created       |       Updated       | Deployment Status |
    +-----------+---------------------+---------------------+-------------------+
    | overcloud | 2019-02-12 09:00:27 | 2019-02-12 09:00:27 |   DEPLOY_SUCCESS  |
    +-----------+---------------------+---------------------+-------------------+

    openstack server list -c Name -c Status -c Networks
    +----------------------------+--------+------------------------+
    | Name                       | Status | Networks               |
    +----------------------------+--------+------------------------+
    | overcloud-controller-1     | ACTIVE | ctlplane=192.168.24.19 |
    | overcloud-controller-2     | ACTIVE | ctlplane=192.168.24.11 |
    | overcloud-controller-0     | ACTIVE | ctlplane=192.168.24.29 |
    | overcloud-novacompute-0    | ACTIVE | ctlplane=192.168.24.15 |
    +----------------------------+--------+------------------------+

.. note::

  In this example the default cell and the additional cell uses the
  same network, When configuring another network scenario keep in
  mind that it will be necessary for the systems to be able to
  communicate with each other.

Extract deployment information from the overcloud stack
-------------------------------------------------------

Any additional cell stack requires information from the overcloud Heat stack
where the central OpenStack services are located. The extracted parameters are
needed as input for additional cell stacks. To extract these parameters
into separate files in a directory (e.g. DIR=cell1) run the following::

    source stackrc
    mkdir cell1
    export DIR=cell1

#. Export the default cell EndpointMap

    .. code::

        openstack stack output show overcloud EndpointMap --format json \
        | jq '{"parameter_defaults": {"EndpointMapOverride": .output_value}}' \
        > $DIR/endpoint-map.json

#. Export the default cell HostsEntry

    .. code::

        openstack stack output show overcloud HostsEntry -f json \
        | jq -r '{"parameter_defaults":{"ExtraHostFileEntries": .output_value}}' \
        > $DIR/extra-host-file-entries.json

#. Export AllNodesConfig and GlobalConfig information

    In addition to the ``GlobalConfig``, which contains the RPC information (port,
    ssl, scheme, user and password), additional information from the ``AllNodesConfig``
    is required to point components to the default cell service instead of the
    service served by the cell controller. These are

    * ``oslo_messaging_notify_short_bootstrap_node_name`` - default cell overcloud
      messaging notify bootstrap node information
    * ``oslo_messaging_notify_node_names`` - default cell overcloud messaging notify
      node information
    * ``oslo_messaging_rpc_node_names`` - default cell overcloud messaging rpc node
      information as e.g. neutron agent needs to point to the overcloud messaging
      cluster
    * ``memcached_node_ips`` - memcached node information used by the cell services.

    .. code::

        ALLNODESCFG=$(openstack stack output show overcloud AllNodesConfig --format json)
        GLOBALCFG=$(openstack stack output show overcloud GlobalConfig --format json)
        (echo $ALLNODESCFG | jq '.output_value |
        {oslo_messaging_notify_short_bootstrap_node_name:
        .oslo_messaging_notify_short_bootstrap_node_name,
        oslo_messaging_notify_node_names: .oslo_messaging_notify_node_names,
        oslo_messaging_rpc_node_names: .oslo_messaging_rpc_node_names,
        memcached_node_ips: .memcached_node_ips}';\
        echo $GLOBALCFG | jq '.output_value') |\
        jq -s '.[0] * .[1]| {"parameter_defaults":
        {"AllNodesExtraMapData": .}}' > $DIR/all-nodes-extra-map-data.json

    An example of a ``all-nodes-extra-map-data.json`` file::

        {
          "parameter_defaults": {
            "AllNodesExtraMapData": {
              "oslo_messaging_notify_short_bootstrap_node_name": "overcloud-controller-0",
              "oslo_messaging_notify_node_names": [
                "overcloud-controller-0.internalapi.site1.test",
                "overcloud-controller-1.internalapi.site1.test",
                "overcloud-controller-2.internalapi.site1.test"
              ],
              "oslo_messaging_rpc_node_names": [
                "overcloud-controller-0.internalapi.site1.test",
                "overcloud-controller-1.internalapi.site1.test",
                "overcloud-controller-2.internalapi.site1.test"
              ],
              "memcached_node_ips": [
                "172.16.2.232",
                "172.16.2.29",
                "172.16.2.49"
              ],
              "oslo_messaging_rpc_port": 5672,
              "oslo_messaging_rpc_use_ssl": "False",
              "oslo_messaging_notify_scheme": "rabbit",
              "oslo_messaging_notify_use_ssl": "False",
              "oslo_messaging_rpc_scheme": "rabbit",
              "oslo_messaging_rpc_password": "7l4lfamjPp6nqJgBMqb1YyM2I",
              "oslo_messaging_notify_password": "7l4lfamjPp6nqJgBMqb1YyM2I",
              "oslo_messaging_rpc_user_name": "guest",
              "oslo_messaging_notify_port": 5672,
              "oslo_messaging_notify_user_name": "guest"
            }
          }
        }

#. Export passwords

    .. code::

        openstack object save --file - overcloud plan-environment.yaml \
        | python -c 'import yaml as y, sys as s; \
        s.stdout.write(y.dump({"parameter_defaults": \
        y.load(s.stdin.read())["passwords"]}));' > $DIR/passwords.yaml

    The same passwords are used for the cell services.

#. Create roles file for cell stack

    .. code::

        openstack overcloud roles generate --roles-path \
        /usr/share/openstack-tripleo-heat-templates/roles \
        -o $DIR/cell_roles_data.yaml Compute CellController

    .. note::

        In case a different default heat stack name or compute role name is used,
        modify the above commands.

#. Create cell parameter file for additional customization (e.g. cell1/cell1.yaml)

    Add the following content into a parameter file for the cell, e.g. ``cell1/cell1.yaml``::

        resource_registry:
          # since the same network is used, the creation of the
          # different kind of networks is omitted for additional
          # cells
          OS::TripleO::Network::External: OS::Heat::None
          OS::TripleO::Network::InternalApi: OS::Heat::None
          OS::TripleO::Network::Storage: OS::Heat::None
          OS::TripleO::Network::StorageMgmt: OS::Heat::None
          OS::TripleO::Network::Tenant: OS::Heat::None
          OS::TripleO::Network::Management: OS::Heat::None

        parameter_defaults:
          # new CELL Parameter to reflect that this is an additional CELL
          NovaAdditionalCell: True

          # The DNS names for the VIPs for the cell
          CloudName: computecell1.ooo.test
          CloudNameInternal: computecell1.internalapi.ooo.test
          CloudNameStorage: computecell1.storage.ooo.test
          CloudNameStorageManagement: computecell1.storagemgmt.ooo.test
          CloudNameCtlplane: computecell1.ctlplane.ooo.test

          # Flavors used for the cell controller and computes
          OvercloudCellControllerFlavor: cellcontroller
          OvercloudComputeFlavor: compute

          # number of controllers/computes in the cell
          CellControllerCount: 1
          ComputeCount: 1

          # default gateway
          ControlPlaneStaticRoutes:
            - ip_netmask: 0.0.0.0/0
              next_hop: 192.168.24.1
              default: true
          DnsServers:
            - x.x.x.x

    The above file disables creating networks as the same as the overcloud stack
    created are used. It also specifies that this will be an additional cell using
    parameter `NovaAdditionalCell`.

#. Create the network configuration for `cellcontroller` and add to environment file.

    .. code::

        resource_registry:
          OS::TripleO::BlockStorage::Net::SoftwareConfig: three-nics-vlans//cinder-storage.yaml
          OS::TripleO::CephStorage::Net::SoftwareConfig: three-nics-vlans//ceph-storage.yaml
          OS::TripleO::Compute::Net::SoftwareConfig: three-nics-vlans//compute.yaml
          OS::TripleO::Controller::Net::SoftwareConfig: three-nics-vlans//controller.yaml
          OS::TripleO::CellController::Net::SoftwareConfig: three-nics-vlans//cellcontroller.yaml
          OS::TripleO::ObjectStorage::Net::SoftwareConfig: three-nics-vlans//swift-storage.yaml

    .. note::

             For details on network configuration consult :ref:`network_isolation` guide, chapter *Customizing the Interface Templates*.

Deploy the cell
---------------

#. Create new flavor used to tag the cell controller

    .. code::

        openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 cellcontroller
        openstack flavor set --property "cpu_arch"="x86_64" \
        --property "capabilities:boot_option"="local" \
        --property "capabilities:profile"="cellcontroller" \
        --property "resources:CUSTOM_BAREMETAL=1" \
        --property "resources:DISK_GB=0" \
        --property "resources:MEMORY_MB=0" \
        --property "resources:VCPU=0" \
        cellcontroller

    The properties need to be modified to the needs of the environment.

#. Tag node into the new flavor using the following command

    .. code::

        openstack baremetal node set --property \
        capabilities='profile:cellcontroller,boot_option:local' <node id>

    Verify the tagged cellcontroller::

        openstack overcloud profiles list

#. Deploy the cell

    To deploy the overcloud we can use use the same ``overcloud deploy`` command as
    it was used to deploy the ``overcloud`` stack and add the created export
    environment files::

        openstack overcloud deploy --override-ansible-cfg \
          /home/stack/custom_ansible.cfg \
          --stack computecell1 \
          --templates /usr/share/openstack-tripleo-heat-templates \
          -e ... additional environment files used for overcloud stack, like container
            prepare parameters, or other specific parameters for the cell
          ...
          -r $HOME/$DIR/cell_roles_data.yaml \
          -e $HOME/$DIR/passwords.yaml \
          -e $HOME/$DIR/endpoint-map.json \
          -e $HOME/$DIR/all-nodes-extra-map-data.json \
          -e $HOME/$DIR/extra-host-file-entries.json \
          -e $HOME/$DIR/cell1.yaml

    Wait for the deployment to finish::

        openstack stack list
        +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+
        | ID                                   | Stack Name   | Project                          | Stack Status    | Creation Time        | Updated Time         |
        +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+
        | 890e4764-1606-4dab-9c2f-6ed853e3fed8 | computecell1 | 2b303a97f4664a69ba2dbcfd723e76a4 | CREATE_COMPLETE | 2019-02-12T08:35:32Z | None                 |
        | 09531653-1074-4568-b50a-48a7b3cc15a6 | overcloud    | 2b303a97f4664a69ba2dbcfd723e76a4 | UPDATE_COMPLETE | 2019-02-09T09:52:56Z | 2019-02-11T08:33:37Z |
        +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+

Create the cell and discover compute nodes
------------------------------------------

#. Add cell information to overcloud controllers

    On all central controllers add information on how to reach the messaging cell
    controller endpoint (usually internalapi) to ``/etc/hosts``, from the undercloud::

        API_INFO=$(ssh heat-admin@<cell controlle ip> grep cellcontrol-0.internalapi /etc/hosts)
        ansible -i /usr/bin/tripleo-ansible-inventory Controller -b \
        -m lineinfile -a "dest=/etc/hosts line=\"$API_INFO\""

    .. note::

        Do this outside the ``HEAT_HOSTS_START`` .. ``HEAT_HOSTS_END`` block, or
        add it to an `ExtraHostFileEntries` section of an environment file for the
        central overcloud controller. Add the environment file to the next
        `overcloud deploy` run.

#. Extract transport_url and database connection

    Get the ``transport_url`` and database ``connection`` endpoint information
    from the cell controller. This information is used to create the cell in the
    next step::

        ssh heat-admin@<cell controller ip> sudo crudini --get \
        /var/lib/config-data/nova/etc/nova/nova.conf DEFAULT transport_url
        ssh heat-admin@<cell controller ip> sudo crudini --get \
        /var/lib/config-data/nova/etc/nova/nova.conf database connection

#. Create the cell

    Login to one of the central controllers create the cell with reference to
    the IP of the cell controller in the ``database_connection`` and the
    ``transport_url`` extracted from previous step, like::

        ssh heat-admin@<ctlplane ip overcloud-controller-0>

        # CONTAINERCLI can be either docker or podman
        export CONTAINERCLI='docker'

        sudo $CONTAINERCLI exec -it -u root nova_api /bin/bash
        nova-manage cell_v2 create_cell --name computecell1 \
        --database_connection \
        '{scheme}://{username}:{password}@172.16.2.102/nova?{query}' \
        --transport-url \
        'rabbit://guest:7l4lfamjPp6nqJgBMqb1YyM2I@computecell1-cellcontrol-0.internalapi.cell1.test:5672/?ssl=0'

    .. note::

        Templated transport cells URLs could be used if the same amount of controllers
        are in the default and add on cell.

    .. code::

        nova-manage cell_v2 list_cells --verbose

    After the cell got created the nova services on all central controllers need to
    be restarted.

    Docker::

        ansible -i /usr/bin/tripleo-ansible-inventory Controller -b -a \
        "docker restart nova_api nova_scheduler nova_conductor"

    Podman::

        ansible -i /usr/bin/tripleo-ansible-inventory Controller -b -a \
        "systemctl restart tripleo_nova_api tripleo_nova_conductor tripleo_nova_scheduler"


#. Perform cell host discovery

    Login to one of the overcloud controllers and run the cell host discovery::

        ssh heat-admin@<ctlplane ip overcloud-controller-0>

        # CONTAINERCLI can be either docker or podman
        export CONTAINERCLI='docker'

        sudo $CONTAINERCLI exec -it -u root nova_api /bin/bash
        nova-manage cell_v2 discover_hosts --by-service --verbose
        nova-manage cell_v2 list_hosts

        +--------------+--------------------------------------+---------------------------------------+
        |  Cell Name   |              Cell UUID               |                Hostname               |
        +--------------+--------------------------------------+---------------------------------------+
        | computecell1 | 97bb4ee9-7fe9-4ec7-af0d-72b8ef843e3e | computecell1-novacompute-0.site1.test |
        |   default    | f012b67d-de96-471d-a44f-74e4a6783bca |   overcloud-novacompute-0.site1.test  |
        +--------------+--------------------------------------+---------------------------------------+

    The cell is now deployed and can be used.
