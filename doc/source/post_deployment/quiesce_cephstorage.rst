.. _quiesce_cephstorage:

Quiescing a CephStorage Node
============================

The process of quiescing a cephstorage node means to inform the Ceph
cluster that one or multiple OSDs will be permanently removed so that
the node can be shut down without affecting the data availability.

Take the OSDs out of the cluster
--------------------------------

Before you remove an OSD, you need to take it out of the cluster so that Ceph
can begin rebalancing and copying its data to other OSDs. Running the following
commands on a given cephstorage node will take all data out of the OSDs hosted
on it::

    OSD_IDS=$(ls /var/lib/ceph/osd | awk 'BEGIN { FS = "-" } ; { print $2 }')
    for OSD_ID in $OSD_IDS; do ceph crush reweight osd.$OSD_ID 0.0; done

Ceph will begin rebalancing the cluster by migrating placement groups out of
the OSDs. You can observe this process with the ceph tool::

    ceph -w

You should see the placement group states change from active+clean to active,
some degraded objects, and finally active+clean when migration completes.

Removing the OSDs
-----------------

After the rebalancing, the OSDs will still be running. Running the following on
that same cephstorage node will stop all OSDs hosted on it, remove them from the
CRUSH map, from the OSDs map and delete the authentication keys::

    OSD_IDS=$(ls /var/lib/ceph/osd | awk 'BEGIN { FS = "-" } ; { print $2 }')
    for OSD_ID in $OSD_IDS; do
      ceph osd out $OSD_ID
      systemctl stop ceph-osd@$OSD_ID
      ceph osd crush remove osd.$OSD_ID
      ceph auth del osd.$OSD_ID
      ceph osd rm $OSD_ID
    done

.. admonition:: Mitaka
   :class: mitaka

   TripleO/Mitaka uses and supports Ceph/Hammer, not Jewel, which does not
   use systemd but sysv init scripts. For Mitaka the systemctl command above
   which stops the OSD should be replaced by::

       service ceph stop osd.$OSD_ID

You are now free to reboot or shut down the node (using the Ironic API), or
even remove it from the overcloud altogether by scaling down the overcloud
deployment, see :ref:`delete_nodes`.
