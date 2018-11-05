TripleO Containers Architecture
===============================

This document explains the details around TripleO's containers architecture. The
document goes into the details of how the containers are built for TripleO,
how the configuration files are generated and how the containers are eventually
run.

Like other areas of TripleO, the containers based deployment requires a couple
of different projects to play together. The next section will cover each of the
parts that allow for deploying OpenStack in containers using TripleO.


Containers runtime deployment and configuration notes
-----------------------------------------------------

TripleO deploys the containers runtime and image components from the docker
packages. The installed components include the docker daemon system service and
`OCI`_ compliant `Moby`_ and `Containerd`_ - the building blocks for the
container system.

Containers control plane includes `Paunch`_ and `Dockerd`_ for the
stateless services, and Pacemaker `Bundle`_ for the containerized stateful
services, like the messaging system or database.

.. _OCI: https://www.opencontainers.org/
.. _Moby: https://mobyproject.org/
.. _Containerd: https://github.com/containerd/containerd
.. _dockerd: https://docs.docker.com/engine/reference/commandline/dockerd/
.. _Bundle: https://wiki.clusterlabs.org/wiki/Bundle_Walk-Through

There are ``Docker*`` configuration parameters in TripleO Heat Templates
available for operators. Those options may be used to override defaults for the
main docker daemon system service, or help to debug containerized TripleO
deployments. Parameter override example::

  parameter_defaults:
    DockerDebug: true
    DockerOptions: '--log-driver=syslog --live-restore'
    DockerNetworkOptions: '--bip=10.10.0.1/16'
    DockerInsecureRegistryAddress: ['myregistry.local:8787']
    DockerRegistryMirror: 'mirror.regionone.local:8081/myregistry-1.local/'

* ``DockerDebug`` adds more framework-specific details to the deployment logs.

* ``DockerOptions``, ``DockerNetworkOptions``, ``DockerAdditionalSockets`` define
  the docker service startup options, like the default IP address for the
  `docker0` bridge interface (``--bip``) or SELinux mode (``--selinux-enabled``).

  .. note:: Make sure the default CIDR assigned for the `docker0` bridge interface
      does not conflict to other network ranges defined for your deployment.

* ``DockerInsecureRegistryAddress``, ``DockerRegistryMirror`` allow you to
  specify a custom registry mirror which can optionally be accessed insecurely
  by using the ``DockerInsecureRegistryAddress`` parameter.

See the official dockerd `documentation`_ for the reference.

.. _documentation: https://docs.docker.com/engine/reference/commandline/dockerd/


Building Containers
-------------------

The containers used for TripleO are sourced from Kolla.  Kolla is an OpenStack
team that aims to create tools to allow for deploying OpenStack on container
technologies. Kolla (or Kolla Build) is one of the tools produced by this team
and it allows for building and customizing container images for OpenStack
services and their dependencies.

TripleO consumes these images and takes advantage of the customization
capabilities provided by the `Kolla`_ build tool to install some packages that
are required by other parts of TripleO.

TripleO maintains its complete list of kolla customization in the
`tripleo-common`_ project.

.. _Kolla: https://docs.openstack.org/kolla/latest/admin/image-building.html#dockerfile-customisation
.. _tripleo-common: https://github.com/openstack/tripleo-common/blob/master/container-images/tripleo_kolla_template_overrides.j2


Paunch
------

The `paunch`_ hook is used to manage containers. This hook takes json
as input and uses it to create and run containers on demand. The json
describes how the container will be started.  Some example keys are:

* **net**: To specify what network to use. This is commonly set to host.

* **privileged**: Whether to give full access to the host's devices to the
  container, similar to what happens when the service runs directly on the host.

* **volumes**: List of host path volumes, named volumes, or dynamic volumes to
  bind on the container.

* **environment**: List of environment variables to set on the container.

.. note:: The list above is not exhaustive and you should refer to the
   `paunch` docs for the complete list.

The json file passed to this hook is built out of the `docker_config` attribute
defined in the service's yaml file. Refer to the `Docker specific settings`_
section for more info on this.

.. _paunch: https://github.com/openstack/paunch

TripleO Heat Templates
----------------------
.. _containers_arch_tht:

The `TripleO Heat Templates`_ repo is where most of the logic resides in the form
of heat templates. These templates define each service, the containers'
configuration and the initialization or post-execution operations.

.. _TripleO Heat Templates: http://git.openstack.org/cgit/openstack/tripleo-heat-templates

Understanding container related files
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The docker templates can be found under the `docker` sub directory in the
`tripleo-heat-templates` root. The services files are under `docker/service` but
the `docker` directory contains a bit more than just service files and some of
them are worth diving into:

deploy-steps.j2
...............

This file is a jinja template and it's rendered before the deployment is
started. This file defines the resources that are executed before and after the
container initialization.

.. _docker-puppet.py:

docker-puppet.py
................

This script is responsible for generating the config files for each service. The
script is called from the `deploy-steps.j2` file and it takes a `json` file as
configuration. The json files passed to this script are built out of the
`puppet_config` parameter set in every service template (explained in the
`Docker specific settings`_ section).

The `docker-puppet.py` execution results in a oneshot container being executed
(usually named `puppet-$service_name`) to generate the configuration options or
run other service specific initialization tasks. Example: Create Keystone endpoints.

Anatomy of a containerized service template
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Containerized services templates inherit almost everything from the puppet based
templates, with some exceptions for some services. New properties have been
added to define container specific configurations, which will be covered in this
section.

Docker specific settings
........................

Each service may define output variable(s) which control config file generation,
initialization, and stepwise deployment of all the containers for this service.
The following sections are available:

* config_settings: This setting is generally inherited from the
  puppet/services templates and may be appended to if required
  to support the docker specific config settings.

* step_config: This setting controls the manifest that is used to
  create docker config files via puppet. The puppet tags below are
  used along with this manifest to generate a config directory for
  this container.

* kolla_config: Contains YAML that represents how to map config files
  into the kolla container. This config file is typically mapped into
  the container itself at the /var/lib/kolla/config_files/config.json
  location and drives how kolla's external config mechanisms work.

* docker_config: Data that is passed to the docker-cmd hook to configure
  a container, or step of containers at each step. See the available steps
  below and the related docker-cmd hook documentation in the heat-agents
  project.

* puppet_config: This section is a nested set of key value pairs
  that drive the creation of config files using puppet.
  Required parameters include:

  * puppet_tags: Puppet resource tag names that are used to generate config
    files with puppet. Only the named config resources are used to generate
    a config file. Any service that specifies tags will have the default
    tags of 'file,concat,file_line,augeas,cron' appended to the setting.
    Example: keystone_config

  * config_volume: The name of the volume (directory) where config files
    will be generated for this service. Use this as the location to
    bind mount into the running Kolla container for configuration.

  * config_image: The name of the docker image that will be used for
    generating configuration files. This is often the same container
    that the runtime service uses. Some services share a common set of
    config files which are generated in a common base container.

  * step_config: This setting controls the manifest that is used to
    create docker config files via puppet. The puppet tags below are
    used along with this manifest to generate a config directory for
    this container.

* docker_puppet_tasks: This section provides data to drive the
  docker-puppet.py tool directly. The task is executed only once
  within the cluster (not on each node) and is useful for several
  puppet snippets we require for initialization of things like
  keystone endpoints, database users, etc. See docker-puppet.py
  for formatting.


Docker steps
............

Similar to baremetal, docker containers are brought up in a stepwise manner. The
current architecture supports bringing up baremetal services alongside of
containers. Therefore, baremetal steps may be required depending on the service
and they are always executed before the corresponding container step.

The list below represents the correlation between the baremetal and the
containers steps. These steps are executed sequentially:

* Containers config files generated per hiera settings.
* Host Prep
* Load Balancer configuration baremetal

   * Step 1 external steps (execute Ansible on Undercloud)
   * Step 1 deployment steps (Ansible)
   * Common Deployment steps

     * Step 1 baremetal (Puppet)
     * Step 1 containers

* Core Services (Database/Rabbit/NTP/etc.)

   * Step 2 external steps (execute Ansible on Undercloud)
   * Step 2 deployment steps (Ansible)
   * Common Deployment steps

     * Step 2 baremetal (Puppet)
     * Step 2 containers

* Early Openstack Service setup (Ringbuilder, etc.)

   * Step 3 external steps (execute Ansible on Undercloud)
   * Step 3 deployment steps (Ansible)
   * Common Deployment steps

     * Step 3 baremetal (Puppet)
     * Step 3 containers

* General OpenStack Services

   * Step 4 external steps (execute Ansible on Undercloud)
   * Step 4 deployment steps (Ansible)
   * Common Deployment steps

     * Step 4 baremetal (Puppet)
     * Step 4 containers (Keystone initialization occurs here)

* Service activation (Pacemaker)

   * Step 5 external steps (execute Ansible on Undercloud)
   * Step 5 deployment steps (Ansible)
   * Common Deployment steps

     * Step 5 baremetal (Puppet)
     * Step 5 containers


Service Bootstrap
~~~~~~~~~~~~~~~~~

Bootstrapping services is a one-shot operation for most services and it's done
by defining a separate container that shares the same structure as the main
service container commonly defined under the `docker_step` number 3 (see `Docker
steps`_ section above).

Unlike normal service containers, the bootstrap container should be run in the
foreground - `detach: false` - so there can be more control on when the
execution is done and whether it succeeded or not.

Example taken from Glance's service file::


      docker_config:
        step_3:
          glance_api_db_sync:
            image: *glance_image
            net: host
            privileged: false
            detach: false
            volumes: &glance_volumes
              - /var/lib/kolla/config_files/glance-api.json:/var/lib/kolla/config_files/config.json
              - /etc/localtime:/etc/localtime:ro
              - /lib/modules:/lib/modules:ro
              - /var/lib/config-data/glance_api/:/var/lib/kolla/config_files/src:ro
              - /run:/run
              - /dev:/dev
              - /etc/hosts:/etc/hosts:ro
            environment:
              - KOLLA_BOOTSTRAP=True
              - KOLLA_CONFIG_STRATEGY=COPY_ALWAYS
        step_4:
          glance_api:
            image: *glance_image
            net: host
            privileged: false
            restart: always
            volumes: *glance_volumes
            environment:
              - KOLLA_CONFIG_STRATEGY=COPY_ALWAYS
