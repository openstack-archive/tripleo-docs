.. _delete_nodes:

Deleting Overcloud Nodes
========================

You can delete specific nodes from an overcloud with command::

    openstack overcloud node delete --stack $STACK_NAME --templates [templates dir] <list of nova instance IDs>

This command updates the heat stack with updated numbers and list of resource
IDs (which represent nodes) to be deleted.

.. note::
   If you passed any extra environment files when you created the overcloud (for
   instance, in order to configure :doc:`network isolation
   <../advanced_deployment/network_isolation>`), you must pass them again here
   using the ``-e`` or ``--environment-file`` option to avoid making undesired
   changes to the overcloud.

.. note::
   Before deleting a compute node or a cephstorage node, please make sure that
   the node is quiesced, see :ref:`quiesce_compute` or
   :ref:`quiesce_cephstorage`.

.. note::
   A list of nova instance IDs can be listed with command::

       nova list
