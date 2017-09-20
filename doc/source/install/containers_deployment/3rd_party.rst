Integrating 3rd Party Containers in TripleO
===========================================

Building Containers
-------------------

One of the following methods can be used to extend or build from scratch
custom 3rd party containers.

Adding layers to existing containers
....................................

Any extra RPMs required by 3rd party drivers may need to be post-installed into
our stock TripleO containers.  In this case the 3rd party vendor may opt to add
a layer to an existing container in order to deploy their software.

The example below demonstrates how to extend a container on the Undercloud host
machine. It assumes you are running a local docker registry on the undercloud.
We recommend that you create a Dockerfile to extend the existing container.
Here is an example extending the cinder-volume container::

    FROM 127.0.0.1:8787/tripleo/centos-binary-cinder-volume
    MAINTAINER Vendor X
    LABEL name="tripleo/centos-binary-cinder-volume-vendorx" vendor="Vendor X" version="2.1" release="1"

    # switch to root and install a custom RPM, etc.
    USER root
    COPY vendor_x.rpm /tmp
    RUN rpm -ivh /tmp/vendor_x.rpm

    # switch the container back to the default user
    USER cinder

Docker build the container above using `docker build` on the command line. This
will output a container image <ID> (used below to tag it). Create a docker tag
and push it into the local registry::

    docker tag <ID> 127.0.0.1:8787/tripleo/centos-binary-cinder-volume-vendorx:rev1
    docker push 127.0.0.1:8787/tripleo/centos-binary-cinder-volume-vendorx:rev1

Start an overcloud deployment as normal with the extra custom Heat environment
above to obtain the new container.

.. warning:: Note that the new container will have the complete software stack
             built into it as is normal for containers.  When other containers
             are updated and include security fixes in these lower layers, this
             container will NOT be updated as a result and will require rebuilding.

Building new containers with kolla-build
........................................

To create new containers, or modify existing ones, you can use ``kolla-build``
from the `Kolla`_ project to build and push the images yourself.  The command
to build a new containers is below.  Note that this assumes you are on an
undercloud host where the registry IP address is 192.168.24.1::

    kolla-build --base centos --type binary --namespace master --registry 192.168.24.1:8787 --tag latest --template-override /usr/share/tripleo-common/container-images/tripleo_kolla_template_overrides.j2 --push

Notice that TripleO already uses the
``/usr/share/tripleo-common/container-images/tripleo_kolla_template_overrides.j2``
to add or change specific aspects of the containers.  This file can be copied
and modified to create custom containers.  The original copy of this file can
be found in the `tripleo-common`_ repository.

To build specific containers you can add a `custom` section to the kolla
configuration.  For example, to build the cron containers you would add::

    [profiles]
    custom=cron

The following template is an example of the template used for building the base
images that are consumed by TripleO. In this case we are adding the `puppet`
RPM to the base image::

    {% extends parent_template %}
    {% set base_centos_binary_packages_append = ['puppet'] %}
    {% set nova_scheduler_packages_append = ['openstack-tripleo-common'] %}

.. _Kolla: https://docs.openstack.org/developer/kolla/image-building.html#dockerfile-customisation
.. _tripleo-common: https://github.com/openstack/tripleo-common/blob/master/container-images/tripleo_kolla_template_overrides.j2


Integrating 3rd party containers with tripleo-heat-templates
------------------------------------------------------------

The `TripleO Heat Templates`_ repo is where most of the logic resides in the form
of heat templates. These templates define each service, the containers'
configuration and the initialization or post-execution operations.

.. _TripleO Heat Templates: http://git.openstack.org/cgit/openstack/tripleo-heat-templates

The docker templates can be found under the `docker` sub directory in the
`tripleo-heat-templates` root. The services files are under the
`docker/service` directory.

For more information on how to integrate containers into the TripleO Heat templates,
see the ``install/containers_deployment/architecture.rst`` document. (FIXME: proper link)

If all you need to do is change out a container for a specific service, you can
create a custom heat environment file that contains your override.  To swap out
the cinder container from our previous example we would add::

    parameter_defaults:
        DockerCinderVolumeImage: centos-binary-cinder-volume-vendorx:rev1
