Example 2. - Split Cell controller/compute Architecture in Train release
========================================================================

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

.. _advanced_cell_arch:

In this scenario the cell computes get split off in its own stack, e.g. to
manage computes from each edge site in its own stack.

This section only explains the differences to the :doc:`deploy_cellv2_basic`.

Like before the following example uses six nodes and the split control plane method
to deploy a distributed cell deployment. The first Heat stack deploys the controller
cluster. The second Heat stack deploys the cell controller. The computes will then
again be split off in its own stack.

.. _cell_export_cell_controller_info:

Extract deployment information from the overcloud stack
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Again like in :ref:`cell_export_overcloud_info` information from the control
plane stack needs to be exported:

.. code-block:: bash

  source stackrc
  mkdir cell1
  export DIR=cell1

  openstack overcloud cell export cell1-ctrl -o cell1/cell1-ctrl-input.yaml


Create roles file for the cell stack
____________________________________

The same roles get exported as in :ref:`cell_create_roles_file`.

Create cell parameter file for additional customization (e.g. cell1/cell1.yaml)
_______________________________________________________________________________

The cell parameter file remains the same as in :ref:`cell_parameter_file` with
the only difference that the `ComputeCount` gets set to 0. This is required as
we use the roles file contain both `CellController` and `Compute` role and the
default count for the `Compute` role is 1 (e.g. `cell1/cell1.yaml`):

.. code-block:: yaml

  parameter_defaults:
    ...
    # number of controllers/computes in the cell
    CellControllerCount: 1
    ComputeCount: 0
    ...

Create the network configuration for `cellcontroller` and add to environment file
_________________________________________________________________________________
Depending on the network configuration of the used hardware and network
architecture it is required to register a resource for the `CellController`
role.

.. code-block:: yaml

  resource_registry:
    OS::TripleO::CellController::Net::SoftwareConfig: single-nic-vlans/controller.yaml

.. note::

  For details on network configuration consult :ref:`network_isolation` guide, chapter *Customizing the Interface Templates*.

Deploy the cell
^^^^^^^^^^^^^^^

Create new flavor used to tag the cell controller
_________________________________________________

Follow the instructions in :ref:`cell_create_flavor_and_tag` on how to create
a new flavor and tag the cell controller.

Run cell deployment
___________________
To deploy the cell controller stack we use use the same `overcloud deploy`
command as it was used to deploy the `overcloud` stack and add the created
export environment files:

.. code-block:: bash

    openstack overcloud deploy \
      --templates /usr/share/openstack-tripleo-heat-templates \
      -e ... additional environment files used for overcloud stack, like container
        prepare parameters, or other specific parameters for the cell
      ...
      --stack cell1-ctrl \
      -r $HOME/$DIR/cell_roles_data.yaml \
      -e $HOME/$DIR/cell1-ctrl-input.yaml \
      -e $HOME/$DIR/cell1.yaml

Wait for the deployment to finish:

.. code-block:: bash

  openstack stack list
  +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+
  | ID                                   | Stack Name   | Project                          | Stack Status    | Creation Time        | Updated Time         |
  +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+
  | 890e4764-1606-4dab-9c2f-6ed853e3fed8 | cell1-ctrl   | 2b303a97f4664a69ba2dbcfd723e76a4 | CREATE_COMPLETE | 2019-02-12T08:35:32Z | None                 |
  | 09531653-1074-4568-b50a-48a7b3cc15a6 | overcloud    | 2b303a97f4664a69ba2dbcfd723e76a4 | UPDATE_COMPLETE | 2019-02-09T09:52:56Z | 2019-02-11T08:33:37Z |
  +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+

Create the cell
^^^^^^^^^^^^^^^
As in :ref:`cell_create_cell` create the cell, but we can skip the final host
discovery step as the computes are note yet deployed.

Extract deployment information from the cell controller stack
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The cell compute stack again requires input information from both the control
plane stack (`overcloud`) and the cell controller stack (`cell1-ctrl`):

.. code-block:: bash

  source stackrc
  export DIR=cell1

Export EndpointMap, HostsEntry, AllNodesConfig, GlobalConfig and passwords information
______________________________________________________________________________________
As before the `openstack overcloud cell export` functionality of the tripleo-client
is used to export the required data from the cell controller stack.

.. code-block:: bash

  openstack overcloud cell export cell1-cmp -o cell1/cell1-cmp-input.yaml -e cell1-ctrl

`cell1-cmp` is the chosen name for the new compute stack. This parameter is used to
set the default export file name, which is then stored on the current directory.
In this case a dedicated export file was set via `-o`.
In addition it is required to use the `--cell-stack <cell stack>` or `-e <cell stack>`
parameter to point the export command to the cell controller stack and indicate
that this is a compute child stack. This is required as the input information for
the cell controller and cell compute stack is not the same.

.. note::

  If the export file already exists it can be forced to be overwritten using
  `--force-overwrite` or `-f`.

.. note::

  The services from the cell stacks use the same passwords services as the
  control plane services.

Create cell compute parameter file for additional customization
_______________________________________________________________
A new parameter file is used to overwrite, or customize settings which are
different from the cell controller stack. Add the following content into
a parameter file for the cell compute stack, e.g. `cell1/cell1-cmp.yaml`:

.. code-block:: yaml

  parameter_defaults:
    # number of controllers/computes in the cell
    CellControllerCount: 0
    ComputeCount: 1

The above file overwrites the values from `cell1/cell1.yaml` to not deploy
a controller in the cell compute stack. Since the cell compute stack uses
the same role file the default `CellControllerCount` is 1.
If there are other differences, like network config, parameters,  ... for
the computes, add them here.

Deploy the cell computes
^^^^^^^^^^^^^^^^^^^^^^^^

Run cell deployment
___________________
To deploy the overcloud we can use use the same `overcloud deploy` command as
it was used to deploy the `cell1-ctrl` stack and add the created export
environment files:

.. code-block:: bash

    openstack overcloud deploy \
      --templates /usr/share/openstack-tripleo-heat-templates \
      -e ... additional environment files used for overcloud stack, like container
        prepare parameters, or other specific parameters for the cell
      ...
      --stack cell1-cmp \
      -n $HOME/$DIR/cell1-cmp/network_data.yaml \
      -r $HOME/$DIR/cell_roles_data.yaml \
      -e $HOME/$DIR/cell1-ctrl-input.yaml \
      -e $HOME/$DIR/cell1-cmp-input.yaml \
      -e $HOME/$DIR/cell1.yaml \
      -e $HOME/$DIR/cell1-cmp.yaml

Wait for the deployment to finish:

.. code-block:: bash

  openstack stack list
  +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+
  | ID                                   | Stack Name   | Project                          | Stack Status    | Creation Time        | Updated Time         |
  +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+
  | 790e4764-2345-4dab-7c2f-7ed853e7e778 | cell1-cmp    | 2b303a97f4664a69ba2dbcfd723e76a4 | CREATE_COMPLETE | 2019-02-12T08:35:32Z | None                 |
  | 890e4764-1606-4dab-9c2f-6ed853e3fed8 | cell1-ctrl   | 2b303a97f4664a69ba2dbcfd723e76a4 | CREATE_COMPLETE | 2019-02-12T08:35:32Z | None                 |
  | 09531653-1074-4568-b50a-48a7b3cc15a6 | overcloud    | 2b303a97f4664a69ba2dbcfd723e76a4 | UPDATE_COMPLETE | 2019-02-09T09:52:56Z | 2019-02-11T08:33:37Z |
  +--------------------------------------+--------------+----------------------------------+-----------------+----------------------+----------------------+

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

