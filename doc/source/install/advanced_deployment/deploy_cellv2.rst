Deploy an additional nova cell v2
=================================

.. warning::
   This currently is only supported in Stein or newer versions.

.. contents::
   :depth: 3
   :backlinks: none

This guide assumes that you are ready to deploy a new overcloud, or have
already installed an overcloud (min Stein release).

Initial Deploy
--------------

The minimum requirement for having multiple cells is to have a central OpenStack
controller cluster running all controller services. Additional cells will
have cell controllers running the cell DB, cell MQ and a nova cell conductor
service. In addition there are 1..n compute nodes. The central nova conductor
service acts as a super conductor of the whole environment.

For more details on the cells v2 layout check `Cells Layout (v2)
<https://docs.openstack.org/nova/latest/user/cellsv2-layout.html>`_

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

The above deployed overcloud shows the nodes from the first stach and should be
configured with Swift as Glance backend, using the following parameter, that
images are pulled by the remote cell compute node over HTTP::

    parameter_defaults:
      GlanceBackend: swift

.. note::

    In this example the default cell and the additional cell uses the same
    network, When configuring another network scenario keep in mind that it
    will be necessary for the systems to be able to communicate with each
    other. E.g. the IPs for the default cell controller node will be in the
    endpoint map that later will be extracted from the overcloud stack and
    passed as a parameter to the second cell stack for it to access its
    endpoints. In this example both cells share an L2 network. In a production
    deployment it may be necessary instead to route.

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

        (openstack stack output show overcloud AllNodesConfig --format json \
        | jq '.output_value | {oslo_messaging_notify_short_bootstrap_node_name: \
        .oslo_messaging_notify_short_bootstrap_node_name, \
        oslo_messaging_notify_node_names: .oslo_messaging_notify_node_names, \
        oslo_messaging_rpc_node_names: .oslo_messaging_rpc_node_names, \
        memcached_node_ips: .memcached_node_ips}'; \
        openstack stack output show overcloud GlobalConfig --format json \
        | jq '.output_value') |jq -s '.[0] * .[1]| {"parameter_defaults": \
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

          # In case of an tls-everywhere environment the CloudName*
          # parameters need to be set for the cell as connection to
          # endpoints are done via DNS names, like MySQL and MQ  endpoints.
          # This is optional for non tls-everywhere environments.
          #CloudName: computecell1.ooo.test
          #CloudNameInternal: computecell1.internalapi.ooo.test
          #CloudNameStorage: computecell1.storage.ooo.test
          #CloudNameStorageManagement: computecell1.storagemgmt.ooo.test
          #CloudNameCtlplane: computecell1.ctlplane.ooo.test

          # CloudDomain is the same as in the default cell.
          #CloudDomain: ooo.test

          # Flavors used for the cell controller and computes
          OvercloudControllerFlavor: cellcontroller
          OvercloudComputeFlavor: compute

          # number of controllers/computes in the cell
          ControllerCount: 1
          ComputeCount: 1
          CephStorageCount: 0

          # default gateway
          ControlPlaneStaticRoutes:
            - ip_netmask: 0.0.0.0/0
              next_hop: 192.168.24.1
              default: true
          Debug: true
          DnsServers:
            - x.x.x.x

    The above file disables creating networks as the same as the overcloud stack
    created are used. It also specifies that this will be an additional cell using
    parameter `NovaAdditionalCell`.

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
          -e /usr/share/openstack-tripleo-heat-templates/environments/docker-ha.yaml \
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

        # CONTAINERCLI can be either /bin/docker or /bin/podman
        export CONTAINERCLI='/bin/docker'

        ssh heat-admin@<ctlplane ip overcloud-controller-0>
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
    be restarted::

        ansible -i /usr/bin/tripleo-ansible-inventory Controller -b -a \
        "$CONTAINERCLI restart nova_api nova_scheduler nova_conductor"

#. Perform cell host discovery

    Login to one of the overcloud controllers and run the cell host discovery::

        ssh heat-admin@<ctlplane ip overcloud-controller-0>
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

Managing the cell
-----------------

Add a compute to a cell
~~~~~~~~~~~~~~~~~~~~~~~

To increase resource capacity of a running cell, you can start more servers of
a selected role. For more details on how to add nodes see :doc:`../post_deployment/scale_roles`.

After the node got deployed, login to one of the overcloud controllers and run
the cell host discovery::

    # CONTAINERCLI can be either /bin/docker or /bin/podman
    export CONTAINERCLI='/bin/docker'

    ssh heat-admin@<ctlplane ip overcloud-controller-0>
    sudo $CONTAINERCLI exec -it -u root nova_api /bin/bash
    nova-manage cell_v2 discover_hosts --by-service --verbose
    nova-manage cell_v2 list_hosts

Delete a compute from a cell
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As initial step migrate all instances off the compute.

#. From one of the overcloud controllers, delete the computes from the cell::

    # CONTAINERCLI can be either /bin/docker or /bin/podman
    export CONTAINERCLI='/bin/docker'

    ssh heat-admin@<ctlplane ip overcloud-controller-0>
    sudo $CONTAINERCLI exec -it -u root nova_api /bin/bash
    nova-manage cell_v2 delete_host --cell_uuid <uuid> --host <compute>

#. Delete the resource providers from placement

    This step is required as otherwise adding a compute node with the same hostname
    will make it to fail to register and update the resources with the placement
    service.::

        sudo yum install python2-osc-placement
        openstack resource provider list
        +--------------------------------------+---------------------------------------+------------+
        | uuid                                 | name                                  | generation |
        +--------------------------------------+---------------------------------------+------------+
        | 9cd04a8b-5e6c-428e-a643-397c9bebcc16 | computecell1-novacompute-0.site1.test |         11 |
        +--------------------------------------+---------------------------------------+------------+

    openstack resource provider delete 9cd04a8b-5e6c-428e-a643-397c9bebcc16

#. Delete the node from the cell stack

    See :doc:`../post_deployment/delete_nodes`.

Deleting a cell
~~~~~~~~~~~~~~~

As initial step delete all instances from cell

#. From one of the overcloud controllers, delete all computes from the cell::

    # CONTAINERCLI can be either /bin/docker or /bin/podman
    export CONTAINERCLI='/bin/docker'

    ssh heat-admin@<ctlplane ip overcloud-controller-0>
    sudo $CONTAINERCLI exec -it -u root nova_api /bin/bash
    nova-manage cell_v2 delete_host --cell_uuid <uuid> --host <compute>

#. On the cell controller delete all deleted instances::

    ssh heat-admin@<ctlplane ip cell control>
    sudo $CONTAINERCLI exec -it -u root nova_conductor /bin/bash
    nova-manage db archive_deleted_rows --verbose

#. From one of the overcloud controllers, delete the cell::

    ssh heat-admin@<ctlplane ip overcloud-controller-0>
    sudo $CONTAINERCLI exec -it -u root nova_api /bin/bash
    nova-manage cell_v2 delete_cell --cell_uuid <uuid>

#. From a system which can reach the placement endpoint, delete the resource providers from placement

    This step is required as otherwise adding a compute node with the same hostname
    will make it to fail to register and update the resources with the placement
    service.::

        sudo yum install python2-osc-placement
        openstack resource provider list
        +--------------------------------------+---------------------------------------+------------+
        | uuid                                 | name                                  | generation |
        +--------------------------------------+---------------------------------------+------------+
        | 9cd04a8b-5e6c-428e-a643-397c9bebcc16 | computecell1-novacompute-0.site1.test |         11 |
        +--------------------------------------+---------------------------------------+------------+

    openstack resource provider delete 9cd04a8b-5e6c-428e-a643-397c9bebcc16


#. Delete the cell stack::

    openstack stack delete computecell1 --wait --yes && openstack overcloud plan delete computecell1
