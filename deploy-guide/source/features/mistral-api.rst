Mistral API
===========

.. warning::
    Mistral on the Undercloud has been deprecated in the Ussuri cycle and is
    soon being removed. You should consider Ansible playbooks instead of
    Mistral workflows.
    This page related to TripleO Train and backward.

The public API for TripleO uses the OpenStack Workflow service, `Mistral`_ to
provide its interface. This allows external systems to consume and use the same
Workflows used by python-tripleoclient and tripleo-ui.

Working with Mistral
--------------------

TripleO functionality can be accessed via Mistral Workflows and Actions.
Workflows define a set number of steps and typically use a number of actions.
There is a set of actions which are intended to be used directly. When actions
are called directly, Mistral executes them synchronously which is quicker for
simple actions.

Mistral can be used with the CLI interface or Python bindings which are both
provided by `python-mistralclient`_ or via the `REST API`_ directly. This
guide will use the Mistral CLI interface for brevity.

When using the CLI, all of the TripleO workflows can be viewed with the
command ``openstack workflow list``. All of the workflows provided by TripleO
will have a name starting with ``tripleo.``

.. code-block:: console

    $ openstack workflow list
    +--------------------------------------+-----------------------------------------+------------------------------+
    | ID                                   | Name                                    | Input                        |
    +--------------------------------------+-----------------------------------------+------------------------------+
    | 1ae040b6-d330-4181-acb9-8638dc486b79 | tripleo.baremetal.v1.set_node_state     | node_uuid, state_action, ... |
    | 2ef20a58-b380-4b6b-a6cd-270352d0f3d2 | tripleo.deployment.v1.deploy_on_servers | server_name, config_name,... |
    +--------------------------------------+-----------------------------------------+------------------------------+

To view the individual workflows in more detail and see the inputs they
accept use the ``openstack workflow show`` command. This command will also
show the default values for input parameters. If no default is given, then it
is required.

.. code-block:: console

    $ openstack workflow show tripleo.plan_management.v1.create_default_deployment_plan
    +------------+-----------------------------------------------------------+
    | Field      | Value                                                     |
    +------------+-----------------------------------------------------------+
    | ID         | fa8256ec-b585-476f-a83e-e800beb26684                      |
    | Name       | tripleo.plan_management.v1.create_default_deployment_plan |
    | Project ID | 65c5259b7a96436f898fd518815c42c1                          |
    | Tags       | <none>                                                    |
    | Input      | container, queue_name=tripleo                             |
    | Created at | 2016-08-19 10:07:10                                       |
    | Updated at | None                                                      |
    +------------+-----------------------------------------------------------+

This workflow can then be executed with the ``openstack workflow execution
create`` command.

.. code-block:: console

    $ openstack workflow execution create tripleo.plan_management.v1.create_default_deployment_plan \
        '{"container": "my_cloud"}'
    +-------------------+-----------------------------------------------------------+
    | Field             | Value                                                     |
    +-------------------+-----------------------------------------------------------+
    | ID                | 824a8cf7-3306-4ef2-8efd-a2715dd0dbce                      |
    | Workflow ID       | fa8256ec-b585-476f-a83e-e800beb26684                      |
    | Workflow name     | tripleo.plan_management.v1.create_default_deployment_plan |
    | Description       |                                                           |
    | Task Execution ID | <none>                                                    |
    | State             | RUNNING                                                   |
    | State info        | None                                                      |
    | Created at        | 2016-08-22 12:33:35.493135                                |
    | Updated at        | 2016-08-22 12:33:35.495764                                |
    +-------------------+-----------------------------------------------------------+

After a Workflow execution is created it will be scheduled up by Mistral and
executed asynchronously. Mistral can either be polled until it is finished or
you can subscribe to the `Zaqar`_ queue for messages from the running
Workflow. By default the TripleO workflows will send messages to a Zaqar queue
with the name ``tripleo``, the workflows all accept a ``queue_name`` parameter
which allows a user defined queue name to be used. It can be useful to use
different queue names if you plan to execute multiple workflows and want the
messages to be handled individually.

Actions can be used in a similar way to workflows, but the CLI commands are
``openstack action definition list``, ``openstack action definition show``
and ``openstack action execution run``.

`API reference documentation`_ is available for all TripleO Workflows.


Creating a Deployment Plan
--------------------------

Deployment plans consist of a Swift container and a Mistral Environment. The
TripleO Heat Templates are stored in Swift and then user defined parameters are
stored in the Mistral environment.

Using the default plan
^^^^^^^^^^^^^^^^^^^^^^

When the undercloud is installed, it will create a default plan with the name
``overcloud``. To create a new plan from the packaged version of
tripleo-heat-templates on the undercloud use the workflow
``tripleo.plan_management.v1.create_default_deployment_plan``. This workflow
accepts a name which will be used for the Swift container and Mistral
environment.

The following command creates a plan called ``my_cloud``.

.. code-block:: console

    $ openstack workflow execution create tripleo.plan_management.v1.create_default_deployment_plan \
        '{"container": "my_cloud"}'
    +-------------------+-----------------------------------------------------------+
    | Field             | Value                                                     |
    +-------------------+-----------------------------------------------------------+
    | ID                | dc4800ef-8d0a-436e-9564-a7ee81ba93d5                      |
    | Workflow ID       | fa8256ec-b585-476f-a83e-e800beb26684                      |
    | Workflow name     | tripleo.plan_management.v1.create_default_deployment_plan |
    | Description       |                                                           |
    | Task Execution ID | <none>                                                    |
    | State             | RUNNING                                                   |
    | State info        | None                                                      |
    | Created at        | 2016-08-23 10:06:45.372767                                |
    | Updated at        | 2016-08-23 10:06:45.376122                                |
    +-------------------+-----------------------------------------------------------+

.. note::

    When updating the packages on the undercloud with yum the TripleO Heat
    Templates will be updated in `/usr/share/..` but any plans that were
    previously created will not be updated automatically. At the moment this
    is a manual process.

Using custom templates
^^^^^^^^^^^^^^^^^^^^^^

Manually creating a plan with custom templates is a three stage process. Each
step must use the same name for the container, we are using ``my_cloud``, but
it can be changed if they are all consistent. This will be the plan name.

1. Create the Swift container.

   .. code-block:: bash

        openstack action execution run tripleo.plan.create_container \
            '{"container":"my_cloud"}'

   .. note::

        Creating a swift container directly isn't sufficient, as this Mistral
        action also sets metadata on the container and may include further
        steps in the future.

2. Upload the files to Swift.

   .. code-block:: bash

        swift upload my_cloud path/to/tripleo/templates

3. Trigger the plan create Workflow, which will create the Mistral environment
   for the uploaded templates, do some initial template processing and generate
   the passwords.

   .. code-block:: bash

        openstack workflow execution create tripleo.plan_management.v1.create_deployment_plan \
            '{"container":"my_cloud"}'


Working with Bare Metal Nodes
-----------------------------

Some functionality for dealing with bare metal nodes is provided by the
``tripleo.baremetal`` workflows.

Register Nodes
^^^^^^^^^^^^^^

Baremetal nodes can be registered with Ironic via Mistral. The input for this
workflow is a bit larger, so this time we will store it in a file and pass it
in, rather than working inline.

.. code-block:: bash

    $ cat nodes.json
    {
        "remove": false,
        "ramdisk_name": "bm-deploy-ramdisk",
        "kernel_name": "bm-deploy-kernel",
        "nodes_json": [
            {
                "pm_password": "$RSA_PRIVATE_KEY",
                "pm_type": "pxe_ssh",
                "pm_addr": "192.168.122.1",
                "mac": [
                    "00:8f:61:0d:6a:e1"
                ],
                "memory": "8192",
                "disk": "40",
                "arch": "x86_64",
                "cpu": "4",
                "pm_user": "root"
            }
        ]
    }


* If ``remove`` is set to true, any nodes that are not passed to the workflow
  will be removed.
* ``ramdisk_name`` and ``kernel_name`` are the Glance names for the kernel and
  ramdisk to use for the nodes.
* If ``instance_boot_option`` is set, it defines whether to set instances for
  booting from the local hard drive (local) or network (netboot).
* The format of the nodes_json is documented in :ref:`instackenv`.

.. code-block:: bash

    $ openstack workflow execution create tripleo.baremetal.v1.register_or_update \
        nodes.json

The result of this workflow can be seen with the following command.

.. code-block:: bash

    $ mistral execution-get-output $EXECUTION_ID
    {
        "status": "SUCCESS",
        "new_nodes": [],
        "message": "Nodes set to managed.",
        "__task_execution": {
            "id": "001892c5-4197-4c04-af74-aff95f6d584f",
            "name": "send_message"
        },
        "registered_nodes": [
            {
                "uuid": "93feecfb-8a4d-418c-9f2c-5ef8db7aff2e",
                ...
            },
        ]
    }

The above information is accessible like this, or via the zaqar queue. The
registered_nodes property will contain each of the nodes registered with all
their properties from Ironic, including the UUID which is useful for
introspection.

Introspect Nodes
^^^^^^^^^^^^^^^^

To introspect the nodes, we need to either use the Ironic UUID's returned by
the register_or_update workflow or retrieve them directly from Ironic. Then
those UUID's can be passed to the introspection workflow. The workflow expects
nodes to be in the "manageable" state.

.. code-block:: bash

    $ openstack workflow execution create tripleo.baremetal.v1.introspect \
        '{"nodes_uuids": ["UUID1", "UUID2"]}'

.. _cleaning_workflow:

Cleaning Nodes
^^^^^^^^^^^^^^

It is recommended to clean previous information from all disks on the bare
metal nodes before new deployments. As TripleO disables automated cleaning, it
has to be done manually via the ``manual_clean`` workflow. A node has to be in
the ``manageable`` state for it to work.

.. note::
    See `Ironic cleaning documentation
    <https://docs.openstack.org/ironic/deploy/cleaning.html>`_ for
    more details.

To remove partitions from all disks on a given node, use the following
command:

.. code-block:: bash

    $ openstack workflow execution create tripleo.baremetal.v1.manual_cleaning \
        '{"node_uuid": "UUID", "clean_steps": [{"step": "erase_devices_metadata", "interface": "deploy"}]}'

To remove all data from all disks (either by ATA secure erase or by shredding
them), use the following command:

.. code-block:: bash

    $ openstack workflow execution create tripleo.baremetal.v1.manual_cleaning \
        '{"node_uuid": "UUID", "clean_steps": [{"step": "erase_devices", "interface": "deploy"}]}'

The node state is set back to ``manageable`` after successful cleaning and to
``clean failed`` after a failure. Inspect node's ``last_error`` field for the
cause of the failure.

.. warning::
    Shredding disks can take really long, up to several hours.

Provide Nodes
^^^^^^^^^^^^^

After the nodes have been introspected they will still be in the manageable
state. To make them available for a deployment, use the provide workflow,
which has the same interface as introspection.

.. code-block:: bash

    $ openstack workflow execution create tripleo.baremetal.v1.provide \
        '{"nodes_uuids": ["UUID1", "UUID2"]}'

Parameters
----------

A number of parameters will need to be provided for a deployment to be
successful. These required parameters will depend on the Heat templates that
are being used. Parameters can be set with the Mistral Action
``tripleo.parameters.update``.

.. note::

    This action will merge the passed parameters with those already set on the
    plan. To set the parameters first use ``tripleo.parameters.reset`` to
    remove any old parameters first.

In the following example we set the ``ComputeCount`` parameter to ``2`` on the
``my_cloud`` plan. This only sets one parameter, but any number can be provided.

.. code-block:: bash

    $ openstack action execution run tripleo.parameters.update \
        '{"container":"my_cloud", "parameters":{"ComputeCount":2}}'


Deployment
----------

After the plan has been configured it should be ready to be deployed.

.. code-block:: bash

    $ openstack workflow execution create tripleo.deployment.v1.deploy_plan \
        '{"container": "my_cloud"}'

Once the deployment is triggered, the templates will be processed and sent to
Heat. This workflow will complete when the Heat action has started, or if there
are any errors.

Deployment progress can be tracked via the Heat API. It is possible to either
follow the Heat events or simply wait for the Heat stack status to change.


.. _Mistral: https://docs.openstack.org/mistral/
.. _python-mistralclient: https://docs.openstack.org/mistral/guides/mistralclient_guide.html
.. _REST API: https://docs.openstack.org/mistral/developer/webapi/index.html
.. _Zaqar: https://docs.openstack.org/zaqar/
.. _API Reference Documentation: https://docs.openstack.org/tripleo-common/reference/index.html
