TripleO Containers Architecture
===============================

.. Warning::

   The TripleO containers support is still under heavy development. Things
   documented here may change during the Pike cycle.

This document explains the details around TripleO's containers architecture. The
document goes into the details of how the containers are created from TripleO,
how the configuration files are generated and how the containers are eventually
run.

Like other areas of TripleO, the containers based deployment requires a couple
of different projects to play together. The next section will cover each of the
parts that allow for deploying OpenStack on containers using TripleO.

Kolla Build
-----------

Kolla is an OpenStack team that aims to create tools to allow for deploying
OpenStack on container technologies. Kolla (or Kolla Build) is one of the tools
produced by this team and it allows for building and customizing container
images for OpenStack services and their dependencies.

TripleO consumes these images and takes advantage of the customization
capabilities provided by the `Kolla`_ build tool to install some packages that
are required by other parts of TripleO.

The following template is an example of the template used for building the base
images that are consumed by TripleO. Further customization is required for some
of the services, like mariadb::


    {% extends parent_template %}
    {% set base_centos_binary_packages_append = ['puppet'] %}
    {% set nova_scheduler_packages_append = ['openstack-tripleo-common'] %}


.. note:: `parent_template` is the literal string to include. No need to replace
   it.

Use the following command to build an image using kolla-build and the template
above (`template-overrides.j2`)::

  $ kolla-build --base centos \
    --template-override /usr/share/tripleo-common/container-images/tripleo_kolla_template_overrides.j2 \
    --template-override template-overrides.j2

TripleO maintains its complete list of kolla customization in the
`tripleo-common`_ project.

.. _Kolla: https://docs.openstack.org/developer/kolla/image-building.html#dockerfile-customisation
.. _tripleo-common: https://github.com/openstack/tripleo-common/blob/master/container-images/tripleo_kolla_template_overrides.j2

heat-config-docker-cmd
----------------------

The `heat-config-docker-cmd`_ hook is used to manage containers. This hook takes json as
input and uses it to create and run containers on demand. The `docker-cmd`
accepts different keys that allow for configuring the container process. Some of
the keys are:

* **net**: To specify what network to use. This is commonly set to host.

* **privileged**: Whether to give full access to the host's devices to the
  container, similar to what happens when the service runs directly on the host.

* **volumes**: List of host path volumes, named volumes, or dynamic volumes to
  bind on the container.

* **environment**: List of environment variables to set on the container.

.. note:: The list above is not exhaustive and you should refer to the
   `heat-config-docker-cmd` docs for the complete list.

The json file passed to this hook is built out of the `docker_config` attribute
defined in the service's yaml file. Refer to the `Docker specific settings`_
section for more info on this.

.. _heat-config-docker-cmd: https://github.com/openstack/heat-agents/tree/master/heat-config-docker-cmd

heat-json-config-file
---------------------

The `heat-json-config-file`_ takes a json config as input and dumps it onto disk
in the specified directory. This is used to write on disk the json required to
run the kolla images and the docker-puppet-tasks.

.. _heat-json-config-file: https://github.com/openstack/heat-agents/blob/master/heat-config-json-file/README.rst

TripleO Heat Templates
----------------------

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

post.yaml.j2
............

This file is a jinja template and it's rendered before the deployment is
started. This file defines the resources that are executed before and after the
container initialization.

.. _docker-puppet.py:

docker-puppet.py
................

This script is responsible for generating the config files for each service. The
script is called from the `post.yaml` file and it takes a `json` file as
configuration. The json files passed to this script are built out of the
`puppet_config` parameter set in every service template (explained in the
`Docker specific settings`_ section).

The `docker-puppet.py` execution results in a oneshot container being executed
(usually named `puppet-$service_name`) to generate the configuration options or
run other service specific operations. Example: Create Keystone endpoints.

Anatomy of a containerized service template
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Containerized services templates inherit almost everything from the puppet based
templates, with some exceptions for some services. New properties have been
added to define container specific configurations, which will be covered in this
section.

Docker specific settings
........................

Each service may define an output variable which returns a puppet manifest
snippet that will run at each of the following steps. Earlier manifests
are re-asserted when applying latter ones.

* **service_name**: This setting is inherited from the puppet service templates
  and it shouldn't be modified as some of the hiera tasks depend on this.

* **config_settings**: This setting is generally inherited from the
  puppet/services templates and only need to be appended to on occasion if
  docker specific config settings are required. For example::

    config_settings:
      map_merge:
        - get_attr: [MongodbPuppetBase, role_data, config_settings]
        - mongodb::server::fork: false

* **step_config**: This setting controls the manifest that is used to create
  docker config files via puppet. The puppet tags below are used along with
  this manifest to generate a config directory for this container.

* **kolla_config**: Contains YAML that represents how to map config files into
  the kolla container. This config file is typically mapped into the container
  itself at the /var/lib/kolla/config_files/config.json location and drives how
  kolla's external config mechanisms work. Example::

      kolla_config:
        /var/lib/kolla/config_files/mongodb.json:
          command: /usr/bin/mongod --unixSocketPrefix=/var/run/mongodb --config /etc/mongod.conf run
          config_files:
          - dest: /etc/mongod.conf
            source: /var/lib/kolla/config_files/src/etc/mongod.conf
            owner: mongodb
            perm: '0600'
          - dest: /etc/mongos.conf
            source: /var/lib/kolla/config_files/src/etc/mongos.conf
            owner: mongodb
            perm: '0600'

* **docker_config**: Data that is passed to the `heat-config-docker-cmd`_ hook to
  configure a container, or step of containers at each step. See the available
  steps below and the related docker-cmd hook documentation in the heat-agents
  project.

* **puppet_config**:

  * **step_config**: Usually a reference to the one defined outside this section.

  * **puppet_tags**: Puppet resource tag names that are used to generate config
    files with puppet. Only the named config resources are used to generate a
    config file. Any service that specifies tags will have the default tags of
    'file,concat,file_line' appended to the setting. For example::

      puppet_tags: keystone_config

    Some puppet modules do a bit more than just generating config files. Some have
    custom resources with providers that execute commands. It's possible to
    overwrite these providers by changing the `step_config` property. For example::

      puppet_tags: keystone_config
      step_config:
        list_join:
          - "\n"
          - - "['Keystone_user', 'Keystone_endpoint', 'Keystone_domain', 'Keystone_tenant', 'Keystone_user_role', 'Keystone_role', 'Keystone_service'].each |String $val| { noop_resource($val) }"
            - {get_attr: [KeystoneBase, role_data, step_config]}


    The example above will overwrite the provider for all the `Keystone_*` puppet
    tags (except `keystone_config`) using the `noop_resource` function that comes
    with `puppet-tripleo`. This function dynamically configures the default
    provider for each of the `puppet_tags` in the array.

  * **config_volume**: The name of the docker volume where config files will be
    generated for this service. Use this as the location to bind mount into the
    running Kolla container for configuration.

  * **config_image**: The name of the docker image that will be used for
    generating configuration files. This is often the same container that the
    runtime service uses. Some services share a common set of config files which
    are generated in a common base container.

* **docker_puppet_tasks**: This section provides data to drive the
  docker-puppet.py tool directly. The task is executed only once within the
  cluster (not on each node) and is useful for several puppet snippets we
  require for initialization of things like keystone endpoints, database users,
  etc. See docker-puppet.py for formatting. Here's an example of Keystone's
  `docker_puppet_tasks`::

      docker_puppet_tasks:
        # Keystone endpoint creation occurs only on single node
        step_4:
          - 'keystone_init_tasks'
          - 'keystone_config,keystone_domain_config,keystone_endpoint,keystone_identity_provider,keystone_paste_ini,keystone_role,keystone_service,keystone_tenant,keystone_user,keystone_user_role,keystone_domain'
          - 'include ::tripleo::profile::base::keystone'
          - list_join:
            - '/'
            - [ {get_param: DockerNamespace}, {get_param: DockerKeystoneImage} ]

* **host_prep_tasks**: Ansible tasks to execute on the host before any
  containers are started. Useful e.g. for ensuring existence of
  directories that we want bind mounted into the containers.

* **upgrade_tasks**: Ansible tasks to execute during upgrade. First
  these tasks are run on all nodes, and then the normal puppet/docker
  operations happen the same way as during a fresh deployment.

Docker steps
............

Similar to baremetal, docker containers are brought up in a stepwise manner. The
current architecture supports bringing up baremetal services alongside of
containers. Therefore, baremetal steps may be required depending on the service
and they are always executed before the corresponding container step.

The list below represents the correlation between the baremetal and the
containers steps. These steps are executed sequentially:

#. Containers config files generated per hiera settings.
#. Load Balancer configuration baremetal

   #. Step 1 baremetal
   #. Step 1 containers

#. Core Services (Database/Rabbit/NTP/etc.)

   #. Step 2 baremetal
   #. Step 2 containers

#. Early Openstack Service setup (Ringbuilder, etc.)

   #. Step 3 baremetal
   #. Step 3 containers

#. General OpenStack Services

   #. Step 4 baremetal
   #. Step 4 containers
   #. Keystone containers post initialization (tenant, service, endpoint creation)

#. Service activation (Pacemaker)

   #. Step 5 baremetal
   #. Step 5 containers


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
