Node cleaning
=============

In Ironic *cleaning* is a process of preparing a bare metal node for
provisioning. There are two types of cleaning: *automated* and *manual*.
See `cleaning documentation
<https://docs.openstack.org/ironic/latest/admin/cleaning.html>`_ for more
details.

.. warning::
   It is highly recommended to at least wipe metadata (partitions and
   partition table(s)) from all disks before deployment.

Automated cleaning
------------------

*Automated cleaning* runs before a node gets to the ``available`` state (see
:doc:`node_states` for more information on provisioning states). It happens
after the first enrollment and after every unprovisioning.

In the TripleO undercloud automated cleaning is **disabled** by default.
Starting with the Ocata release, it can be enabled by setting the following
option in your ``undercloud.conf``:

.. code-block:: ini

  [DEFAULT]
  clean_nodes = True

Alternatively, you can use `Manual cleaning`_ as described below.

Manual cleaning
---------------

*Manual cleaning* is run on request for nodes in the ``manageable`` state.

If you have *automated cleaning* disabled, you can use the following procedure
to wipe the node's metadata starting with the Rocky release:

#. If the node is not in the ``manageable`` state, move it there::

    baremetal node manage <UUID or name>

#. Run manual cleaning on a specific node::

    openstack overcloud node clean <UUID or name>

   or all manageable nodes::

    openstack overcloud node clean --all-manageable

#. Make the node available again::

    openstack overcloud node provide <UUID or name>

   or provide all manageable nodes::

    openstack overcloud node provide --all-manageable
