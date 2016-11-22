Use whole disk images for overcloud
-----------------------------------

By default, TripleO **overcloud-full** image is a *partition* image. Such images carry only the
root partition contents and no partition table. Alternatively, *whole disk* images can be used,
which carry all partitions, a partition table and a boot loader.

Whole disk images can be built with **diskimage-builder** - see
`Ironic images documentation <http://docs.openstack.org/project-install-guide/baremetal/draft/configure-integration.html#create-and-add-images-to-the-image-service>`_
for details. Note that this does not affect **ironic-python-agent** images.

Use the following command to treat **overcloud-full** as a whole disk image when uploading images::

    openstack overcloud image upload --whole-disk

In this case only ``overcloud-full.qcow2`` file is required, ``overcloud-full.initrd`` and
``overcloud-full.vmlinuz`` are not used.
