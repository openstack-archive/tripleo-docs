Deploy and Scale Swift in the Overcloud
=======================================

This guide assumes that you are ready to deploy a new overcloud. To ensure
that Swift nodes are all using the same Ring, some manual steps are required.

Initial Deploy
--------------

To correctly deploy Swift, we need to manually manage the Swift Rings. This
can be achieved by disabling the Ring building process in TripleO by setting
the ``SwiftRingBuild`` and ``RingBuild`` parameters both to ``false``. For
example::

    parameter_defaults:
      SwiftRingBuild: false
      RingBuild: false

.. note::

    If this is saved in a file named ``deploy-parameters.yaml`` then it can
    be deployed with ``openstack overcloud deploy --templates -e
    deploy-parameters.yaml``.

After the deploy is completed, you will need to ssh onto the overcloud node as
the ``heat-admin`` user and switch to the root user with ``sudo -i``. The IP
addresses is available in the output of ``openstack server list``. Once
connected, in the ``/etc/swift/`` directory follow the instructions in the
`Swift documentation <http://docs.openstack.org/mitaka/install-guide-rdo
/swift-initial-rings.html>`_ to create the Rings.

After this is completed you will need to copy the ``/etc/swift/*.ring.gz`` and
``/etc/swift/*.builder`` files from the controller to all other controllers and
Swift storage nodes. These files will also be used when adding additional Swift
nodes. You should have six files::

    /etc/swift/account.builder
    /etc/swift/account.ring.gz
    /etc/swift/container.builder
    /etc/swift/container.ring.gz
    /etc/swift/object.builder
    /etc/swift/object.ring.gz

.. note::

    These files will be updated each time a new node is added with
    swift-ring-builder.


Scaling Swift
-------------

TripleO doesn't currently automatically update and scale Swift Rings. This
needs to be done manually, with similar steps to the above initial
deployment. First we need to define how many dedicated Swift nodes we want to
deploy with the ``ObjectStorageCount`` parameter. In this example we are
adding two Swift nodes::

    parameter_defaults:
      SwiftRingBuild: false
      RingBuild: false
      ObjectStorageCount: 2

After we have deployed again with this new environment we will have two Swift
nodes that need to be added to the ring we created during the initial
deployment. Follow the instructions on `Managing the Rings
<https://docs.openstack.org/swift/admin_guide.html#managing-the-rings>`_
to add the new devices to the rings and copy the new rings to *all* nodes in
the Swift cluster.

.. note::

    Also read the section on `Scripting ring creation
    <https://docs.openstack.org/swift/admin_guide.html#scripting-ring-creation>`_
    to automate this process of scaling the Swift cluster.


Viewing the Ring
----------------

The swift ring can be viewed on each node with the ``swift-ring-builder``
command. It can be executed against all of the ``*.builder`` files. Its
output will display all the nodes in the Ring like this::

    $ swift-ring-builder /etc/swift/object.builder
    /etc/swift/object.builder, build version 4
    1024 partitions, 3.000000 replicas, 1 regions, 1 zones, 3 devices, 0.00 balance, 0.00 dispersion
    The minimum number of hours before a partition can be reassigned is 1
    The overload factor is 0.00% (0.000000)
    Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
                 0       1     1      192.168.24.22  6000      192.168.24.22              6000        d1 100.00       1024    0.00
                 1       1     1      192.168.24.24  6000      192.168.24.24              6000        d1 100.00       1024    0.00
                 2       1     1       192.168.24.6  6000       192.168.24.6              6000        d1 100.00       1024    0.00

Ring configuration be verified by checking the hash of the ``*.ring.gz``
files. It should be the same on all nodes in the ring.::

    $ sha1sum /etc/swift/*.ring.gz
    d41c1b4f93a98a693a6ede074a1b78585af2dc89  /etc/swift/account.ring.gz
    1d10d8cb826308a058c7089fdedfeca122426da9  /etc/swift/container.ring.gz
    f26639938660ee0111e4e7bc1b45f28a0b9f6079  /etc/swift/object.ring.gz

You can also check this by using the ``swift-recon`` command on one of the
overcloud nodes. It will query all other servers and compare all checksums and
a summary like this::

    [root@overcloud-controller-0 ~]# swift-recon --md5
    ===============================================================================
    --> Starting reconnaissance on 3 hosts (object)
    ===============================================================================
    [2016-10-14 12:37:11] Checking ring md5sums
    3/3 hosts matched, 0 error[s] while checking hosts.
    ===============================================================================
    [2016-10-14 12:37:11] Checking swift.conf md5sum
    3/3 hosts matched, 0 error[s] while checking hosts.
    ===============================================================================
