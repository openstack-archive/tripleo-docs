.. _deploy-octavia:

Deploying Octavia in the Overcloud
==================================

This guide assumes that your undercloud is already installed and ready to
deploy an overcloud with Octavia enabled. Please note that only container
deployments are supported.

Preparing to deploy
-------------------

TripleO can upload an Octavia Amphora image to the overcloud if one is
available when deploying.

Configuring the amphora image
-----------------------------

If the Octavia Amphora image is available when deploying it should be placed
in a readable path with the default location being a good choice. On CentOS,
the default location is::

   /usr/share/openstack-octavia-amphora-images/amphora-x64-haproxy.qcow2

If deploying on Red Hat Enterprise Linux, the default location is::

   /usr/share/openstack-octavia-amphora-images/octavia-amphora.qcow2

On Red Hat Enterprise Linux, downloading an image may be unnecessary as the
amphora image may already be installed.

If using a non-default location, make sure to specify the location through the
``OctaviaAmphoraImageFilename`` variable in an environment file. For example::

    parameter_defaults:
        OctaviaAmphoraImageFilename: /usr/share/openstack-images/amphora-image.qcow2

.. warning:: Home directories are typically not readable by the workflow
             tasks that upload the file image to Glance. Please use a generally
             accessible path.

Deploying the overcloud with the octavia services
-------------------------------------------------

To deploy Octavia services in the overcloud, include the sample environment
file provided. For example::

    openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/services/octavia.yaml \
    -e ~/containers-default-parameters.yaml

.. note:: Don't forget to include any additional environment files containing
          parameters such as those for the amphora image file.

Uploading/Updating the amphora image after deployment
-----------------------------------------------------

Uploading a new amphora image to Glance in the overcloud can be done after
deployment. This may be required if the amphora image was not available at the
time of deployment or the image needs to be updated.

There are two Octavia specific requirements::

 -  The image must be tagged in Glance (default value 'amphora-image')

 -  The image must belong the 'service' project

To upload an amphora image into glance::

    openstack image create --disk-format qcow2 --container-format bare \
        --tag 'amphora-image' --file [amphora image filename] \
        --project service new-amphora-image

.. note:: The amphora image tag name can be customized by setting the
          ``OctaviaAmphoraImageTag`` variable. Note that if this is changed
          after deployment, Octavia will not be able to use any previously
          uploaded images until they are retagged.
