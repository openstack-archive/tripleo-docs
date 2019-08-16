Building a Single Image
=======================

The ``openstack overcloud image build --all`` command builds all the images
needed for an overcloud deploy.  However, you may need to rebuild a single
one of them. Use the following commands if you want to do it::

   openstack overcloud image build --type {agent-ramdisk|deploy-ramdisk|fedora-user|overcloud-full}

If the target image exist, this commands ends silently. Make sure to delete a
previous version of the image to run the command as you expect.

Moreover, you can build the image with an extra element of your choice using the
``--builder-extra-args`` argument::

   openstack overcloud image build --type overcloud-full \
       --builder-extra-args overcloud-network-midonet

.. note::
    Make sure the element is available in the ``$ELEMENTS_PATH`` environment
    variable

Uploading the New Single Image
------------------------------

After the new image is built, it can be uploaded using the same command as
before, with the ``--update-existing`` flag added::

    openstack overcloud image upload --update-existing

Note that if the new image is a ramdisk, the Ironic nodes need to be
re-configured to use it.  This can be done by re-running::

    openstack overcloud node configure --all-manageable

.. note::
    If you want to use custom images for boot configuration, specify their names in
    ``--deploy-kernel`` and ``--deploy-ramdisk`` options.

Now the new image should be fully ready for use by new deployments.
