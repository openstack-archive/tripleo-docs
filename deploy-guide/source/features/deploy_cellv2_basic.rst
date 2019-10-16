Example 1. - Basic Cell Architecture in Train release
=====================================================

.. warning::
  Multi cell support is only supported in Stein or later versions.
  This guide addresses Train release and later!

.. contents::
  :depth: 3
  :backlinks: none

This guide assumes that you are ready to deploy a new overcloud, or have
already installed an overcloud (min Train release).

.. note::

  Starting with CentOS 8 and TripleO Stein release, podman is the CONTAINERCLI
  to be used in the following steps.

.. _basic_cell_arch:

The following example uses six nodes and the split control plane method to
deploy a distributed cell deployment. The first Heat stack deploys a controller
cluster and a compute. The second Heat stack deploys a cell controller and a
compute node:

.. code-block:: bash

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

The above deployed overcloud shows the nodes from the first stack.

.. note::

  In this example the default cell and the additional cell uses the
  same network, When configuring another network scenario keep in
  mind that it will be necessary for the systems to be able to
  communicate with each other.

Extract deployment information from the overcloud stack
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Any additional cell stack requires information from the overcloud Heat stack
where the central OpenStack services are located. The extracted parameters are
needed as input for additional cell stacks. To extract these parameters
into separate files in a directory (e.g. DIR=cell1) run the following:

.. code-block:: bash

  source stackrc
  mkdir cell1
  export DIR=cell1

.. _cell_export_overcloud_info:

Export EndpointMap, HostsEntry, AllNodesConfig, GlobalConfig and passwords information
______________________________________________________________________________________
The tripleo-client in Train provides an `openstack overcloud cell export`
functionality to export the required data from the control plane stack which
then is used as an environment file passed to the cell stack.

.. code-block:: bash

  openstack overcloud cell export cell1 -o cell1/cell1-cell-input.yaml

`cell1` is the chosen name for the new cell. This parameter is used to
set the default export file name, which is then stored on the current
directory.
In this case a dedicated export file was set via `-o`.

.. note::

  If the export file already exists it can be forced to be overwritten using
  `--force-overwrite` or `-f`.

.. note::

  The services from the cell stacks use the same passwords services as the
  control plane services.

.. _cell_create_roles_file:

Create roles file for the cell stack
____________________________________
Different roles are provided within tripleo-heat-templates, depending on
the configuration and desired services to be deployed.

The default compute role at roles/Compute.yaml can be used for cell computes
if that is sufficient for the use case.

A dedicated role, `roles/CellController.yaml` is provided. This role includes
the necessary roles for the cell controller, where the main services are
galera database, rabbitmq, nova-conductor, nova novnc proxy and nova metadata
in case `NovaLocalMetadataPerCell` is enabled.

Create the roles file for the cell:

.. code-block:: bash

  openstack overcloud roles generate --roles-path \
  /usr/share/openstack-tripleo-heat-templates/roles \
  -o $DIR/cell_roles_data.yaml Compute CellController

.. _cell_parameter_file:

Create cell parameter file for additional customization (e.g. cell1/cell1.yaml)
_______________________________________________________________________________
Each cell has some mandatory parameters which need to be set using an
environment file.
Add the following content into a parameter file for the cell, e.g. `cell1/cell1.yaml`:

.. code-block:: yaml

  resource_registry:
    # since the same networks are used in this example, the
    # creation of the different networks is omitted
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
    CloudName: cell1.ooo.test
    CloudNameInternal: cell1.internalapi.ooo.test
    CloudNameStorage: cell1.storage.ooo.test
    CloudNameStorageManagement: cell1.storagemgmt.ooo.test
    CloudNameCtlplane: cell1.ctlplane.ooo.test

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

The above file disables creating networks as the networks from the overcloud stack
are reused. It also specifies that this will be an additional cell using parameter
`NovaAdditionalCell`.

Create the network configuration for `cellcontroller` and add to environment file
_________________________________________________________________________________
Depending on the network configuration of the used hardware and network
architecture it is required to register a resource for the `CellController`
role.

.. code-block:: yaml

  resource_registry:
    OS::TripleO::CellController::Net::SoftwareConfig: single-nic-vlans/controller.yaml
    OS::TripleO::Compute::Net::SoftwareConfig: single-nic-vlans/compute.yaml

.. note::

  This example just reused the exiting network configs as it is a shared L2
  network. For details on network configuration consult :ref:`network_isolation` guide,
  chapter *Customizing the Interface Templates*.

Deploy the cell
^^^^^^^^^^^^^^^

.. _cell_create_flavor_and_tag:

Create new flavor used to tag the cell controller
_________________________________________________
Depending on the hardware create a flavor and tag the node to be used.

.. code-block:: bash

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

Tag node into the new flavor using the following command


.. code-block:: bash

  openstack baremetal node set --property \
  capabilities='profile:cellcontroller,boot_option:local' <node id>

Verify the tagged cellcontroller:

.. code-block:: bash

  openstack overcloud profiles list

Run cell deployment
___________________
To deploy the overcloud we can use use the same `overcloud deploy` command as
it was used to deploy the `overcloud` stack and add the created export
environment files:

.. code-block:: bash

    openstack overcloud deploy \
      --templates /usr/share/openstack-tripleo-heat-templates \
      -e ... additional environment files used for overcloud stack, like container
        prepare parameters, or other specific parameters for the cell
      ...
      --stack cell1 \
      -r $HOME/$DIR/cell_roles_data.yaml \
      -e $HOME/$DIR/cell1-cell-input.yaml \
      -e $HOME/$DIR/cell1.yaml

Wait for the deployment to finish:

.. code-block:: bash

  openstack stack list
  +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+
  | ID                                   | Stack Name   | Project                          | Stack Status    | Creation Time        | Updated Time         |
  +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+
  | 890e4764-1606-4dab-9c2f-6ed853e3fed8 | cell1        | 2b303a97f4664a69ba2dbcfd723e76a4 | CREATE_COMPLETE | 2019-02-12T08:35:32Z | None                 |
  | 09531653-1074-4568-b50a-48a7b3cc15a6 | overcloud    | 2b303a97f4664a69ba2dbcfd723e76a4 | UPDATE_COMPLETE | 2019-02-09T09:52:56Z | 2019-02-11T08:33:37Z |
  +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+

.. _cell_create_cell:

Create the cell and discover compute nodes (ansible playbook)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
An ansible role and playbook is available to automate the one time tasks
to create a cell after the deployment steps finished successfully. In
addition :ref:`cell_create_cell_manual` explains the tasks being automated
by this ansible way.

.. code-block:: bash

    source stackrc
    mkdir inventories
    for i in $(openstack stack list -f value -c 'Stack Name'); do \
      /usr/bin/tripleo-ansible-inventory \
      --static-yaml-inventory inventories/${i}.yaml --stack ${i}; \
    done

    ansible-playbook -i inventories \
      /usr/share/ansible/tripleo-playbooks/create-nova-cell-v2.yaml \
      -e tripleo_cellv2_cell_name=cell1 \
      -e tripleo_cellv2_containercli=docker

The playbook requires two parameters `tripleo_cellv2_cell_name` to provide
the name of the new cell and until docker got dropped `tripleo_cellv2_containercli`
to specify either if podman or docker is used.

.. _cell_create_cell_manual:

Create the cell and discover compute nodes (manual way)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The following describes the manual needed steps to finalize the cell
deployment of a new cell. These are the steps automated in the ansible
playbook mentioned in :ref:`cell_create_cell`.

Get control plane and cell controller IPs:

.. code-block:: bash

  CTRL_IP=$(openstack server list -f value -c Networks --name overcloud-controller-0 | sed 's/ctlplane=//')
  CELL_CTRL_IP=$(openstack server list -f value -c Networks --name cellcontrol-0 | sed 's/ctlplane=//')

Add cell information to overcloud controllers
_____________________________________________
On all central controllers add information on how to reach the cell controller
endpoint (usually internalapi) to `/etc/hosts`, from the undercloud:

.. code-block:: bash

  CELL_INTERNALAPI_INFO=$(ssh heat-admin@${CELL_CTRL_IP} egrep \
  cellcontrol.*\.internalapi /etc/hosts)
  ansible -i /usr/bin/tripleo-ansible-inventory Controller -b \
  -m lineinfile -a "dest=/etc/hosts line=\"$CELL_INTERNALAPI_INFO\""

.. note::

  Do this outside the `HEAT_HOSTS_START` .. `HEAT_HOSTS_END` block, or
  add it to an `ExtraHostFileEntries` section of an environment file for the
  central overcloud controller. Add the environment file to the next
  `overcloud deploy` run.

Extract transport_url and database connection
_____________________________________________
Get the `transport_url` and database `connection` endpoint information
from the cell controller. This information is used to create the cell in the
next step:

.. code-block:: bash

  CELL_TRANSPORT_URL=$(ssh heat-admin@${CELL_CTRL_IP} sudo \
  crudini --get /var/lib/config-data/nova/etc/nova/nova.conf DEFAULT transport_url)
  CELL_MYSQL_VIP=$(ssh heat-admin@${CELL_CTRL_IP} sudo \
  crudini --get /var/lib/config-data/nova/etc/nova/nova.conf database connection \
  | perl -nle'/(\d+\.\d+\.\d+\.\d+)/ && print $1')

Create the cell
_______________
Login to one of the central controllers create the cell with reference to
the IP of the cell controller in the `database_connection` and the
`transport_url` extracted from previous step, like:

.. code-block:: bash

  ssh heat-admin@${CTRL_IP} sudo ${CONTAINERCLI} exec -i -u root nova_api \
  nova-manage cell_v2 create_cell --name computecell1 \
  --database_connection "{scheme}://{username}:{password}@$CELL_MYSQL_VIP/nova?{query}" \
  --transport-url "$CELL_TRANSPORT_URL"

.. note::

  Templated transport cells URLs could be used if the same amount of controllers
  are in the default and add on cell. For further information about templated
  URLs for cell mappings check: `Template URLs in Cell Mappings
  <https://docs.openstack.org/nova/stein/user/cells.html#template-urls-in-cell-mappings>`_

.. code-block:: bash

  ssh heat-admin@${CTRL_IP} sudo ${CONTAINERCLI} exec -i -u root nova_api \
  nova-manage cell_v2 list_cells --verbose

After the cell got created the nova services on all central controllers need to
be restarted.

Docker:

.. code-block:: bash

  ansible -i /usr/bin/tripleo-ansible-inventory Controller -b -a \
  "docker restart nova_api nova_scheduler nova_conductor"

Podman:

.. code-block:: bash

  ansible -i /usr/bin/tripleo-ansible-inventory Controller -b -a \
  "systemctl restart tripleo_nova_api tripleo_nova_conductor tripleo_nova_scheduler"

We now see the cell controller services registered:

.. code-block:: bash

  (overcloud) [stack@undercloud ~]$ nova service-list

Perform cell host discovery
___________________________
The final step is to discover the computes deployed in the cell. Run the host discovery
as explained in :ref:`cell_host_discovery`.

Create and add the node to an Availability Zone
_______________________________________________
After a cell got provisioned, it is required to create an availability zone for the
cell to make sure an instance created in the cell, stays in the cell when performing
a migration. Check :ref:`cell_availability_zone` on more about how to create an
availability zone and add the node.

After that the cell is deployed and can be used.

.. note::

  Migrating instances between cells is not supported. To move an instance to
  a different cell it needs to be re-created in the new target cell.
