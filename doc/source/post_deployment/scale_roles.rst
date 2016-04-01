Scaling overcloud roles
=======================
If you want to increase or decrease resource capacity of a running overcloud,
you can start more servers of a selected role or delete some servers if
capacity should be decreased. To set the capacity for the compute role,
the following command can be used::

    openstack overcloud deploy --templates [templates dir] --compute-scale 5

.. note::
   Scaling out assumes that newly added nodes has already been
   registered in Ironic.

.. note::
   When scaling down random servers of specified role will be deleted, how to
   delete specific nodes is described in :ref:`delete_nodes`.

.. note::
   The different scale parameters can be seen in the output of::

       openstack help overcloud deploy
