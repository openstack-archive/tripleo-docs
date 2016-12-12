Uploading a Single Image
========================

After a new image is built, it can be uploaded using the same command as
before, with the ``--update-existing`` flag added::

    openstack overcloud image upload --update-existing

Note that if the new image is a ramdisk, the Ironic nodes need to be
re-configured to use it.  This can be done by re-running::

    openstack overcloud configure boot

Now the new image should be fully ready for use by new deployments.
