Integrating 3rd Party Containers in TripleO
===========================================

.. _build_container_images:

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
undercloud host where the registry IP address is 192.168.24.1.

Configure Kolla to build images for TripleO, in `/etc/kolla/kolla-build.conf`::

  [DEFAULT]
  base=centos
  type=binary
  namespace=master
  registry=192.168.24.1:8787
  tag=latest
  template_override=/usr/share/tripleo-common/container-images/tripleo_kolla_template_overrides.j2
  rpm_setup_config=http://trunk.rdoproject.org/centos7/current-tripleo/delorean.repo,http://trunk.rdoproject.org/centos7/delorean-deps.repo
  push=True

Use the following command to build all of the container images used in TripleO::

  openstack overcloud container image build \
        --config-file /usr/share/tripleo-common/container-images/overcloud_containers.yaml \
        --kolla-config-file /etc/kolla/kolla-build.conf

Or use `kolla-build` to build the images yourself, which provides more
flexibility and allows you to rebuild selectively just the images matching
a given name, for example to build only the heat images with the TripleO
customization::

  kolla-build heat

Notice that TripleO already uses the
``/usr/share/tripleo-common/container-images/tripleo_kolla_template_overrides.j2``
to add or change specific aspects of the containers using the `kolla template
override mechanism`_.  This file can be copied and modified to create custom
containers.  The original copy of this file can be found in the
`tripleo-common`_ repository.

The following template is an example of the template used for building the base
images that are consumed by TripleO. In this case we are adding the `puppet`
RPM to the base image::

    {% extends parent_template %}
    {% set base_centos_binary_packages_append = ['puppet'] %}

.. _Kolla: https://github.com/openstack/kolla
.. _kolla template override mechanism: https://docs.openstack.org/kolla/latest/admin/image-building.html#dockerfile-customisation
.. _tripleo-common: https://github.com/openstack/tripleo-common/blob/master/container-images/tripleo_kolla_template_overrides.j2


Integrating 3rd party containers with tripleo-heat-templates
------------------------------------------------------------

The `TripleO Heat Templates`_ repo is where most of the logic resides in the form
of heat templates. These templates define each service, the containers'
configuration and the initialization or post-execution operations.

.. _TripleO Heat Templates: https://opendev.org/openstack/tripleo-heat-templates

The docker templates can be found under the `docker` sub directory in the
`tripleo-heat-templates` root. The services files are under the
`docker/service` directory.

For more information on how to integrate containers into the TripleO Heat templates,
see the :ref:`Containerized TripleO architecture<containers_arch_tht>` document.

If all you need to do is change out a container for a specific service, you can
create a custom heat environment file that contains your override.  To swap out
the cinder container from our previous example we would add::

    parameter_defaults:
        ContainerCinderVolumeImage: centos-binary-cinder-volume-vendorx:rev1

.. note:: Image parameters were named Docker*Image prior to the Train cycle.


3rd party kernel modules
------------------------

Some applications (like Neutron or Cinder plugins) require specific kernel modules to be installed
and loaded on the system.

We recommend two different methods to deploy and load these modules.

kernel module is deployed on the host
.....................................

The kernel module is deployed on the base Operating System via RPM or DKMS.
It is suggested to deploy the module via virt-customize.
The libguestfs-tools package contains the virt-customize tool. Install the libguestfs-tools::

    sudo yum install libguestfs-tools

Then you need to create a repository file where the module will be downloaded from, and uplaod the repo into the image::

    virt-customize --selinux-relabel -a overcloud-full.qcow2 --upload my-repo.repo:/etc/yum.repos.d/

Once the repository is deployed, you can now install the rpm that contains the kernel module::

    virt-customize --selinux-relabel -a overcloud-full.qcow2 --install my-rpm

Now that the rpm is deployed with the kernel module, we need to configure TripleO to load it.
To configure an extra kernel module named "dpdk_module" for a specific role, we would add::

    parameter_defaults:
      ControllerExtraKernelModules:
        dpdk_module: {}

Since our containers don't get their own kernels, we load modules on the host.
Therefore, ExtraKernelModules parameter is used to configure which modules we want to configure.
This parameter will be applied to the Puppet manifest (in the kernel.yaml service).
The container needs the modules mounted from the host, so make sure the plugin template has the
following configuration (at minimum)::

    volumes:
      - /lib/modules:/lib/modules:ro

However, this method might be problematic if RPMs dependencies are too complex to deploy the kernel
module on the host.


kernel module is containerized
..............................

Kernel modules can be loaded from the container.
The module can be deployed in the same container as the application that will use it, or in a separated
container.

Either way, if you need to run a privileged container, make sure to set this parameter::

    privileged: true

If privilege mode isn't required, it is suggested to set it to false for security reaons.

Kernel modules will need to be loaded when the container will be started by Docker. To do so, it is
suggested to configure the composable service which deploys the module in the container this way::

          kolla_config:
            /var/lib/kolla/config_files/neutron_ovs_agent.json:
            command: /dpdk_module_launcher.sh
          docker_config_scripts:
            dpdk_module_launcher.sh:
              mode: "0755"
              content: |
                #!/bin/bash
                set -xe
                modprobe dpdk_module
          docker_config:
            step_3:
              neutron_ovs_bridge:
                volumes:
                  list_concat:
                    - {get_attr: [ContainersCommon, volumes]}
                    -
                      - /var/lib/docker-config-scripts/dpdk_module_launcher.sh:/dpdk_module_launcher.sh:ro

That way, the container will be configured to load the module at start, so the operator can restart containers without caring about loading the module manually.
