Managing the cell
-----------------

.. _cell_host_discovery:

Add a compute to a cell
~~~~~~~~~~~~~~~~~~~~~~~

To increase resource capacity of a running cell, you can start more servers of
a selected role. For more details on how to add nodes see :doc:`../post_deployment/scale_roles`.

After the node got deployed, login to one of the overcloud controllers and run
the cell host discovery:

.. code-block:: bash

  CTRL=overcloud-controller-0
  CTRL_IP=$(openstack server list -f value -c Networks --name $CTRL | sed 's/ctlplane=//')

  # CONTAINERCLI can be either docker or podman
  export CONTAINERCLI='docker'

  # run cell host discovery
  ssh heat-admin@${CTRL_IP} sudo ${CONTAINERCLI} exec -i -u root nova_api \
  nova-manage cell_v2 discover_hosts --by-service --verbose

  # verify the cell hosts
  ssh heat-admin@${CTRL_IP} sudo ${CONTAINERCLI} exec -i -u root nova_api \
  nova-manage cell_v2 list_hosts

  # add new node to the availability zone
  source overcloudrc
  (overcloud) $ openstack aggregate add host <cell name> <compute host>

.. note::

  Optionally the cell uuid cal be specified to the `discover_hosts` and
  `list_hosts` command to only target against a specific cell.

Delete a compute from a cell
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* As initial step migrate all instances off the compute.

* From one of the overcloud controllers, delete the computes from the cell:

  .. code-block:: bash

    source stackrc
    CTRL=overcloud-controller-0
    CTRL_IP=$(openstack server list -f value -c Networks --name $CTRL | sed 's/ctlplane=//')

    # CONTAINERCLI can be either docker or podman
    export CONTAINERCLI='docker'

    # list the cell hosts
    ssh heat-admin@${CTRL_IP} sudo ${CONTAINERCLI} exec -i -u root nova_api \
    nova-manage cell_v2 list_hosts

    # delete a node from a cell
    ssh heat-admin@${CTRL_IP} sudo ${CONTAINERCLI} exec -i -u root nova_api \
    nova-manage cell_v2 delete_host --cell_uuid <uuid> --host <compute>

* Delete the node from the cell stack

  See :doc:`../post_deployment/delete_nodes`.

* Delete the resource providers from placement

  This step is required as otherwise adding a compute node with the same hostname
  will make it to fail to register and update the resources with the placement
  service.:

  .. code-block:: bash

    sudo yum install python2-osc-placement
    openstack resource provider list
    +--------------------------------------+---------------------------------------+------------+
    | uuid                                 | name                                  | generation |
    +--------------------------------------+---------------------------------------+------------+
    | 9cd04a8b-5e6c-428e-a643-397c9bebcc16 | computecell1-novacompute-0.site1.test |         11 |
    +--------------------------------------+---------------------------------------+------------+

    openstack resource provider delete 9cd04a8b-5e6c-428e-a643-397c9bebcc16

Delete a cell
~~~~~~~~~~~~~

* As initial step delete all instances from the cell.

* From one of the overcloud controllers, delete all computes from the cell:

  .. code-block:: bash

    CTRL=overcloud-controller-0
    CTRL_IP=$(openstack server list -f value -c Networks --name $CTRL | sed 's/ctlplane=//')

    # CONTAINERCLI can be either docker or podman
    export CONTAINERCLI='docker'

    # list the cell hosts
    ssh heat-admin@${CTRL_IP} sudo ${CONTAINERCLI} exec -i -u root nova_api \
    nova-manage cell_v2 list_hosts

    # delete a node from a cell
    ssh heat-admin@${CTRL_IP} sudo ${CONTAINERCLI} exec -i -u root nova_api \
    nova-manage cell_v2 delete_host --cell_uuid <uuid> --host <compute>

* On the cell controller delete all deleted instances from the database:

  .. code-block:: bash

    CELL_CTRL=cell1-cellcontrol-0
    CELL_CTRL_IP=$(openstack server list -f value -c Networks --name $CELL_CTRL | sed 's/ctlplane=//')

    # CONTAINERCLI can be either docker or podman
    export CONTAINERCLI='docker'

    ssh heat-admin@${CELL_CTRL_IP} sudo ${CONTAINERCLI} exec -i -u root nova_conductor \
    nova-manage db archive_deleted_rows --until-complete --verbose

* From one of the overcloud controllers, delete the cell:

  .. code-block:: bash

    CTRL=overcloud-controller-0
    CTRL_IP=$(openstack server list -f value -c Networks --name $CTRL | sed 's/ctlplane=//')

    # CONTAINERCLI can be either docker or podman
    export CONTAINERCLI='docker'

    # list the cells
    ssh heat-admin@${CTRL_IP} sudo ${CONTAINERCLI} exec -i -u root nova_api \
    nova-manage cell_v2 list_cells

    # delete the cell
    ssh heat-admin@${CTRL_IP} sudo ${CONTAINERCLI} exec -i -u root nova_api \
    nova-manage cell_v2 delete_cell --cell_uuid <uuid>

* Delete the cell stack:

  .. code-block:: bash

    openstack stack delete <stack name> --wait --yes && openstack overcloud plan delete <stack name>

  .. note::

    If the cell consist of a controller and compute stack, delete as a first step the
    compute stack and then the controller stack.

* From a system which can reach the placement endpoint, delete the resource providers from placement

    This step is required as otherwise adding a compute node with the same hostname
    will make it to fail to register as a resource with the placement service.
    In case of Centos/RHEL 8 the required packages is `python3-osc-placement`:

  .. code-block:: bash

    sudo yum install python2-osc-placement
    source overcloudrc
    openstack resource provider list
    +--------------------------------------+---------------------------------------+------------+
    | uuid                                 | name                                  | generation |
    +--------------------------------------+---------------------------------------+------------+
    | 9cd04a8b-5e6c-428e-a643-397c9bebcc16 | computecell1-novacompute-0.site1.test |         11 |
    +--------------------------------------+---------------------------------------+------------+

    openstack resource provider delete 9cd04a8b-5e6c-428e-a643-397c9bebcc16

Updating a cell
~~~~~~~~~~~~~~~
Each stack in a multi-stack cell deployment must be updated to perform a full minor
update across the entire deployment.

Cells can be updated just like the overcloud nodes following update procedure described
in :ref:`package_update` and using  appropriate stack name for update commands.

The control plane and cell controller stack should be updated first by completing all
the steps from the minor update procedure.

Once the control plane stack is updated, re-run the export command to recreate the
required input files for each separate cell stack.

.. note::

  Before re-running the export command, backup the previously used input file so that
  the previous versions are not overwritten. In the event that a separate cell stack
  needs a stack update operation performed prior to the minor update procedure, the
  previous versions of the exported files should be used.
