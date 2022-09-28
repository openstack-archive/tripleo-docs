Integrating 3rd Party Containers in TripleO
===========================================

.. _build_container_images:

One of the following methods can be used to extend or build from scratch
custom 3rd party containers.

Extend TripleO Containers
-------------------------

Any extra RPMs required by 3rd party drivers may need to be post-installed into
our stock TripleO containers.  In this case the 3rd party vendor may opt to add
a layer to an existing container in order to deploy their software.

Adding layers to existing containers using TripleO tooling
..........................................................

The example below demonstrates how to extend a container image, where the goal
is to create a layer on top of the cinder-volume image that will be named
"cinder-cooldriver".

* Make sure python-tripleoclient and the dependencies are installed:

  .. code-block:: shell

    sudo dnf install -y python-tripleoclient


* Create a vendor directory (which later can be pushed into a git
  repository):

  .. code-block:: shell

    mkdir ~/vendor

* Create a tcib directory under the vendor folder. All container build
  yaml needs to live in a tcib folder as a root directory.

  .. code-block:: shell

    mkdir ~/vendor/tcib

* Create the `~/vendor/containers.yaml` which contains the list
  of images that we want to build:

  .. code-block:: yaml

    container_images:
      - image_source: tripleo
        imagename: localhost/tripleomaster/openstack-cinder-cooldriver:latest

* Create `~/vendor/tcib/cinder-cooldriver` to hold our container image
  configuration.

  .. code-block:: shell

    mkdir ~/vendor/tcib/cinder-cooldriver

* Optionally, add custom files into the build environment.

  .. code-block:: shell

    mkdir ~/vendor/tcib/cinder-cooldriver/files
    cp custom-package.rpm ~/vendor/tcib/cinder-cooldriver/files


* Create `~/vendor/tcib/cinder-cooldriver/cinder-cooldriver.yaml` file which
  contains the container image configuration:

  .. code-block:: yaml

    ---
    # that's the parent layer, here cinder-volume
    tcib_from: localhost/tripleomaster/openstack-cinder-volume:latest
    tcib_actions:
      - user: root
      - run: mkdir /tmp/cooldriver/example.py
      - run: mkdir -p /rpms
      - run: dnf install -y cooldriver_package
    tcib_copies:
      - '{{lookup(''env'',''HOME'')}}/vendor/tcib/cinder-cooldriver/files/custom-package.rpm /rpms'
    tcib_gather_files: >
      {{ lookup('fileglob', '~/vendor/tcib/cinder-cooldriver/files/*', wantlist=True) }}
    tcib_runs:
      - dnf install -y /rpms/*.rpm
    tcib_user: cinder

.. note:: Here `tcib_runs` provides a shortcut to `tcib_actions:run`. See more tcib parameters documented in the `tcib`_ role.

.. _tcib: https://docs.openstack.org/tripleo-ansible/latest/roles/role-tripleo_container_image_build.html#r-o-l-e-d-e-f-a-u-l-t-s


* The result file structure should look something like:

  .. code-block:: shell

    $ tree vendor
    vendor
    ├── containers.yaml
    └── tcib
        └── cinder-cooldriver
                └── cinder-cooldriver.yaml
                └── files
                    └── custom-package.rpm

* Build the vendor container image:

  .. code-block:: shell

    openstack tripleo container image build \
      --config-file ~/vendor/containers.yaml \
      --config-path ~/vendor

* Use `sudo buildah images` command to check if the image was built:

  .. code-block:: shell

      localhost/tripleomaster/openstack-cinder-cooldriver latest  257592a90133   1 minute ago    1.22 GB

.. note:: If you want to push the image into a Docker Registry, you can use
          `--push` with `--registry`. Use
          `openstack tripleo container image build --help` for more details.

* Push the image into the TripleO Container registry:

  .. code-block:: shell

    sudo openstack tripleo container image push \
        --local --registry-url 192.168.24.1:8787 \
        localhost/tripleomaster/openstack-cinder-cooldriver:latest

* Use `openstack tripleo container image list` to check if the image was pushed:

  .. code-block:: shell

    +--------------------------------------------------------------------------------------------------+
    | Image Name                                                                                       |
    +--------------------------------------------------------------------------------------------------+
    | docker://undercloud.ctlplane.localdomain:8787/tripleomaster/openstack-cinder-vendor:latest       |
    +--------------------------------------------------------------------------------------------------+

Adding layers to existing containers using Docker
.................................................

.. note:: Note that this method has been simplified in Victoria and backported
          down to train, with the new `openstack tripleo container image build`
          command.

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

Building new containers with tripleo container image build
----------------------------------------------------------

Usage
.....

Use the following command to build all of the container images used in TripleO:

  .. code-block:: shell

    openstack tripleo container image build

Different options are provided for advanced usage. They can be discovered
by using `--help` argument.
Here are some of them:

* `--config-file` to use a custom YAML config file specifying the images to build.
* `--config-path` to use a custom base configuration path.
  This is the base path for all container-image files. If this option is set,
  the default path for <config-file> will be modified.
* `--extra-config` to apply additional options from a given configuration YAML
  file. This will apply to all containers built.
* `--exclude` to skip some containers during the build.
* `--registry` to specify a Container Registry where the images will be pushed.
* `--authfile` to specify an authentication file if the Container Registry
  requires authentication.
* `--skip-build` if we don't want to build and push images. It will only
  generate the configuration files.
* `--push` to push the container images into the Container Registry.
* `--volume` to overrides the default bind mounts needed when the container
  images are built. If you use this argument, don't forget that you might need
  to include the default ones.
* `--work-dir` to specify the place where the configuration files will be generated.

Tips and Tricks with tripleo_container_image_build
..................................................

Here's a non-exhaustive list of tips and tricks that might make things faster,
especially on a dev env where you need to build multiple times the containers.

Inject a caching proxy
______________________

Using a caching proxy can make things faster when it comes to package fetching.

One of the way is to either expose the dnf.conf/yum.conf using `--volume`.
Since `dnf.conf is edited during the container build`_, you want to expose a
copy of your host config::

  sudo cp -r /etc/dnf /srv/container-dnf
  openstack tripleo container image build --volume /srv/container-dnf:/etc/dnf:z

Another way is to expose the `http_proxy` and `https_proxy` environment
variable.

In order to do so, create a simple yaml file, for instance ~/proxy.yaml::

  ---
  tcib_envs:
    LANG: en_US.UTF-8
    container: oci
    http_proxy: http://PROXY_HOST:PORT
    https_proxy: http://PROXY_HOST:PORT

Then, pass that file using the `--extra-config` parameter::

  openstack tripleo container image build --extra-config proxy.yaml

And you're set.

.. note:: Please ensure you also pass the `default values`_, since ansible
          isn't configured to `merge dicts/lists`_ by default.

.. _dnf.conf is edited during the container build: https://opendev.org/openstack/tripleo-common/src/commit/156b565bdf74c19d3513f9586fa5fcf1181db3a7/container-images/tcib/base/base.yaml#L3-L14
.. _default values: https://opendev.org/openstack/tripleo-common/src/commit/156b565bdf74c19d3513f9586fa5fcf1181db3a7/container-images/tcib/base/base.yaml#L35-L37
.. _merge dicts/lists: https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-hash-behaviour


Get a minimal environment to build containers
_____________________________________________

As a dev, you might want to get a daily build of your container images. While
you can, of course, run this on an Undercloud, you actually don't need an
undercloud: you can use `this playbook`_ from `tripleo-operator-ansible`_
project

With this, you can set a nightly cron that will ensure you're always getting
latest build on your registry.

.. _this playbook: https://opendev.org/openstack/tripleo-operator-ansible/src/branch/master/playbooks/container-build.yaml
.. _tripleo-operator-ansible: https://docs.openstack.org/tripleo-operator-ansible/latest/


Building new containers with kolla-build
........................................

.. note:: Note that this method will be deprecated during the Victoria cycle
          and replaced by the new `openstack tripleo container image build`
          command.

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

.. note:: Add --use-buildah argument to use Buildah instead of Docker.
          It'll be the default once CentOS8 becomes the testing platform during the Train cycle
          and onward.

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
Deploy the module by using the ``tripleo-mount-image`` tool and create a
``chroot``.

First you need to create a repository file where the module will be downloaded from, and copy the repo file into the image::

    temp_dir=$(mktemp -d)
    sudo tripleo-mount-image -a /path/to/overcloud-full.qcow2 -m $temp_dir
    sudo cp my-repo.repo $temp_dir/etc/yum.repos.d/

You can now start a chroot and install the rpm that contains the kernel module::

    sudo mount -o bind /dev $temp_dir/dev/
    sudo cp /etc/resolv.conf $temp_dir/etc/resolv.conf
    sudo chroot $temp_dir /bin/bash
    dnf install my-rpm
    exit

Then unmount the image::

    sudo rm $temp_dir/etc/resolv.conf
    sudo umount $temp_dir/dev
    sudo tripleo-unmount-image -m $temp_dir

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

If privilege mode isn't required, it is suggested to set it to false for security reasons.

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
