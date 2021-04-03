.. _delete_nodes:

Deleting Overcloud Nodes
========================

There may be situations where it's necessary to remove specific Compute nodes
from the overcloud. In those situations, it is possible to remove specific nodes
by following the process outlined in on this page.

.. note::
   If you're just scaling down nodes and plan to re-use them. Use :ref:`scale_roles`
   instead. For temporary issues with nodes, they can be blacklisted temporarily
   using ``DeploymentServerBlacklist``.
   This guide is specifcally for removing nodes from the environment.

.. note::
   If your Compute node is still hosting VM's, ensure they are migrated to
   another Compute node before continuing.

.. note::
  If you are using :ref:`baremetal_provision` then follow those
  scale-down instructions to call ``openstack overcloud node delete`` with a
  ``--baremetal-deployment`` argument instead of passing a list of nodes to
  delete as arguments.

To delete a specific node from the Overcloud. We use the following command::

    openstack overcloud node delete --stack $STACK_NAME <hostname-of-compute-node>

.. note::
   This command uses the hostnames as it's referring to nodes from the Ansible
   inventory. While there is currently a process to translate Nova UUID's to
   the hostname. This may be removed in future releases. As such, it is 
   recommended to use the hostname instead of Nova UUIDs.

This command updates the heat stack with updated numbers and list of resource
IDs (which represent nodes) to be deleted.

.. admonition:: Train
   :class: train

   In Train, we added a user confirmation to the scale down command to
   prevent accidental node removal.
   To skip it, please use "--yes".

.. admonition:: Train
   :class: train

   Starting in Train and onward, `openstack overcloud node delete` can take
   a list of server hostnames instead of instance ids. However they can't be
   mixed while running the command. Example: if you use hostnames, it would
   have to be for all the nodes to delete.

.. note::
   Before deleting a compute node or a cephstorage node, please make sure that
   the node is quiesced, see :ref:`quiesce_compute` or
   :ref:`quiesce_cephstorage`.

.. note::
   You can generate the list of hostname in the Ansible inventory using::

      . stackrc
      tripleo-ansible-inventory --stack <STACK_NAME> --static-yaml-inventory overcloud-inv.yaml

   This file will contain the Ansible inventory in use and help to identify the
   hostname that needs to be passed to the `node delete` command.

.. note::
   Once the node deletion has completed. Be sure to decrement the node count in your templates.
   For example, if removing a compute node, then the ``ComputeCount:`` value needs to be updated
   to reflect the new correct number of nodes in the environment.
