.. _root_device:

Setting the Root Device for Deployment
--------------------------------------

If your hardware has several hard drives, it's highly recommended that you
specify the exact device to be used during introspection and deployment
as a root device. This is done by setting a ``root_device`` property on the
node in Ironic. Please refer to the `Ironic root device hints documentation`_
for more details.

For example::

    openstack baremetal node set <UUID> --property root_device='{"wwn": "0x4000cca77fc4dba1"}'

To remove a hint and fallback to the default behavior::

    openstack baremetal node unset <UUID> --property root_device

Note that the root device hints should be assigned *before* both introspection
and deployment. After changing the root device hints you should either re-run
introspection or manually fix the ``local_gb`` property for a node::

    openstack baremetal node set <UUID> --property local_gb=<NEW VALUE>

Where the new value is calculated as a real disk size in GiB minus 1 GiB to
account for partitioning (the introspection process does this calculation
automatically).

Setting root device hints automatically
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Starting with the Newton release it is possible to autogenerate root
device hints for all nodes instead of setting them one by one. Pass the
``--root-device`` argument to the ``openstack overcloud node
configure`` **after a successful introspection**. This argument can
accept a device list in the order of preference, for example::

    openstack overcloud node configure --all-manageable --root-device=sdb,sdc,vda

It can also accept one of two strategies: ``smallest`` will pick the smallest
device, ``largest`` will pick the largest one. By default only disk devices
larger than 4 GiB are considered at all, set the ``--root-device-minimum-size``
argument to change.

.. note::
   Subsequent runs of this command on the same set of nodes does nothing,
   as root device hints are already recorded on nodes and are not overwritten.
   If you want to change existing root device hints, first remove them manually
   as described above.

.. note::
   This command relies on introspection data, so if you change disk devices on
   the machines, introspection must be rerun before rerunning this command.

Using introspection data to find the root device
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you don't know the information required to make a choice, you can use
introspection to figure it out. First start with :ref:`introspection` as usual
without setting any root device hints. Then use the stored introspection data
to list all disk devices::

    openstack baremetal introspection data save fdf975ae-6bd7-493f-a0b9-a0a4667b8ef3 | jq '.inventory.disks'

For **python-ironic-inspector-client** versions older than 1.4.0 you can use
the ``curl`` command instead, see :ref:`introspection_data` for details.

This command will yield output similar to the following (some fields are empty
for a virtual node)::

    [
        {
            "size": 11811160064,
            "rotational": true,
            "vendor": "0x1af4",
            "name": "/dev/vda",
            "wwn_vendor_extension": null,
            "wwn_with_extension": null,
            "model": "",
            "wwn": null,
            "serial": null
        },
        {
            "size": 11811160064,
            "rotational": true,
            "vendor": "0x1af4",
            "name": "/dev/vdb",
            "wwn_vendor_extension": null,
            "wwn_with_extension": null,
            "model": "",
            "wwn": null,
            "serial": null
        }
    ]

You can use all these fields, except for ``rotational``, for the root device
hints. Note that ``size`` should be converted to GiB and that ``name``,
``wwn_with_extension`` and ``wwn_vendor_extension`` can only be used starting
with the Mitaka release. Also note that the ``name`` field, while convenient,
`may be unreliable and change between boots
<https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Storage_Administration_Guide/persistent_naming.html>`_.

Do not forget to re-run the introspection after setting the root device hints.

.. _Ironic root device hints documentation: https://docs.openstack.org/ironic/deploy/install-guide.html#specifying-the-disk-for-deployment
