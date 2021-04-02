Service template sections description
=====================================

As mentioned in the previous sections of the developer guide, there are several
sections of the template's output that need to be filled out for creating a
service in TripleO.

In this document we will attempt to enumerate all of them and explain the
reasoning behind them.

Note that you can also find useful information in the `tht deployment readme`_.

What's the bare-minimum?
------------------------

Before, digging into details, it's always good to know what the bare-minimum
is. So lets look at a very minimal service template::

  heat_template_version: rocky

  description: Configure Red Hat Subscription Management.

  parameters:
    RoleNetIpMap:
      default: {}
      type: json
    ServiceData:
      default: {}
      description: Dictionary packing service data
      type: json
    ServiceNetMap:
      default: {}
      description: Mapping of service_name -> network name. Typically set
                   via parameter_defaults in the resource registry.  This
                   mapping overrides those in ServiceNetMapDefaults.
      type: json
    RoleName:
      default: ''
      description: Role name on which the service is applied
      type: string
    RoleParameters:
      default: {}
      description: Parameters specific to the role
      type: json
    EndpointMap:
      default: {}
      description: Mapping of service endpoint -> protocol. Typically set
                   via parameter_defaults in the resource registry.
      type: json
    RhsmVars:
      default: {}
      description: Hash of ansible-role-redhat-subscription variables
                   used to configure RHSM.
      # The parameters contains sensible data like activation key or password.
      hidden: true
      tags:
        - role_specific
      type: json

  resources:
    # Merging role-specific parameters (RoleParameters) with the default parameters.
    # RoleParameters will have the precedence over the default parameters.
    RoleParametersValue:
      type: OS::Heat::Value
      properties:
        type: json
        value:
          map_replace:
            - map_replace:
              - vars: RhsmVars
              - values: {get_param: [RoleParameters]}
            - values:
                RhsmVars: {get_param: RhsmVars}

  outputs:
    role_data:
      description: Role data for the RHSM service.
      value:
        service_name: rhsm
        config_settings:
          tripleo::rhsm::firewall_rules: {}
        upgrade_tasks: []
        step_config: ''
        host_prep_tasks:
          - name: Red Hat Subscription Management configuration
            vars: {get_attr: [RoleParametersValue, value, vars]}
            block:
            - include_role:
                name: redhat-subscription

Lets go piece by piece and explain what's going on.

Version and description
^^^^^^^^^^^^^^^^^^^^^^^

As with any other heat template, you do need to specify the
``heat_template_version``, and preferably give a description of what the
stack/template does.

Parameters
^^^^^^^^^^

You'll notice that there are a bunch of heat parameters defined in this
template that are not necessarily used. This is because service templates are
created in the form of a `heat resource chain object`_. This
type of objects can create a "chain" or a set of objects with the same
parameters, and gather the outputs of them. So, eventually we pass the same
mandatory parameters to the chain. This happens in the
`common/services.yaml`_ file. Lets take a look and see how
this is called::

  ServiceChain:
    type: OS::Heat::ResourceChain
    properties:
      resources: {get_param: Services}
      concurrent: true
      resource_properties:
        ServiceData: {get_param: ServiceData}
        ServiceNetMap: {get_param: ServiceNetMap}
        EndpointMap: {get_param: EndpointMap}
        RoleName: {get_param: RoleName}
        RoleParameters: {get_param: RoleParameters}

Here we can see that the mandatory parameters for the services are the
following:

* **ServiceData**: Contains an entry called ``net_cidr_map``, which is a map
  that has the CIDRs for each network in your deployment.

* **ServiceNetMap**: Contains a mapping that tells you what network is each
  service configured at. Typical entries will look like:
  ``BarbicanApiNetwork: internal_api``.

* **EndpointMap**: Contains the keystone endpoints for each service. With this
  you'll be able to get what port, what protocol, and even different entries
  for the public, internal and admin endpoints.

* **RoleName**: This is the name of the role on which the service is applied.
  It could be one of the default roles (e.g. "Controller" or "Compute"), or a
  custom role, depending on how you're deploying.

* **RoleParameters**: A Map containing parameters to be applied to the specific
  role.

So, if you're writing a service template yourself, these are the parameters
you have to copy into your template.

Aside from these parameters, you can define any other parameter yourself for
the service, and in order for your service to consume the parameter, you need
to pass them via ``parameter_defaults``.

The ``role_data`` output
^^^^^^^^^^^^^^^^^^^^^^^^

This is the sole output that will be read and parsed in order to get the
relevant information needed from your service. It's value must be a map, and
from the aforementioned example, it minimally contains the following:

* ``service_name``: This is the name of the service you're configuring. The
  format is lower case letters and underscores. Setting this is quite
  important, since this is how TripleO reports what services are enabled, and
  generates appropriate hieradata, such as a list of all services enabled, and
  flags that say that your service is enabled on a certain node.

* ``config_settings``: This will contain a map of key value pairs; the map will
  be written to the hosts in the form of hieradata, which puppet can then run
  and use to configure your service. Note that the hieradata will only be
  written on hosts that are tagged with a role that enables your service.

* ``upgrade_tasks``: These are ansible tasks that run when TripleO is running
  an upgrade with your service enabled. If you don't have any upgrade tasks to
  do, you still have to specify this output, but it's enough to set it as an
  empty list.

* ``step_config``: This defines what puppet manifest should be run to configure
  your service. It typically is a string with the specific ``include``
  statement that puppet will run. If you're not configuring your service with
  puppet, then you need to set this value as an empty string. There is an
  exception, however: When you're configuring a containerized service. We'll
  dig into that later.

These are the bare-minimum sections of ``role_data`` you need to set up.
However, you might have noticed that the example we linked above has another
section called ``host_prep_data``. This section is not mandatory, but it is one
of the several ways you can execute Ansible tasks on the host in order to
configure your service.

Ansible-related parameters
--------------------------

The following are sections of the service template that allow you to use
Ansible to execute actions or configure your service.

Host prep deployment (or ``host_prep_tasks``)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is seen as ``host_prep_tasks`` in the deployment service templates.
These are Ansible tasks that run before the configuration steps start, and
before any major services are configured (such as pacemaker). Here you would
put actions such as wiping out your disk, or migrating log files.

Lets look at the output section of the example from the previous blog post::

   outputs:
     role_data:
       description: Role data for the RHSM service.
       value:
         service_name: rhsm
         config_settings:
           tripleo::rhsm::firewall_rules: {}
         upgrade_tasks: []
         step_config: ''
         host_prep_tasks:
           - name: Red Hat Subscription Management configuration
             vars: {get_attr: [RoleParametersValue, value, vars]}
             block:
             - include_role:
                 name: redhat-subscription

Here we see that an Ansible role is called directly from the
``host_prep_tasks`` section. In this case, we're setting up the Red Hat
subscription for the node where this is running. We would definitely want this
to happen in the very beginning of the deployment, so ``host_prep_tasks`` is an
appropriate place to put it.

Pre Deploy Step tasks (or ``pre_deploy_step_tasks``)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

These are Ansible tasks that take place in the overcloud nodes. They are run
after the network is completely setup, after the bits to prepare for containers
running are completed (TCIB/Kolla files, container engine installation and configuration).
They are also run before any External deploy tasks.

External deploy tasks
^^^^^^^^^^^^^^^^^^^^^

These are Ansible tasks that take place in the node where you executed the
"overcloud deploy". You'll find these in the service templates in the
``external_deploy_tasks`` section. These actions are also ran as part of the
deployment steps, so you'll have the ``step`` fact available in order to limit
the ansible tasks to only run on a specific step. Note that this runs on each
step before the "deploy steps tasks", the puppet run, and the container
deployment.

Typically you'll see this used when, to configure a service, you need to
execute an Ansible role that has special requirements for the Ansible
inventory.

Such is the case for deploying OpenShift on baremetal via TripleO. The Ansible
role for deploying OpenShift requires several hosts and groups to exist in the
inventory, so we set those up in ``external_deploy_tasks``::

   - name: generate openshift inventory for openshift_master service
     copy:
       dest: "{{playbook_dir}}/openshift/inventory/{{tripleo_role_name}}_openshift_master.yml"
       content: |
         {% if master_nodes | count > 0%}
         masters:
           hosts:
           {% for host in master_nodes %}
           {{host.hostname}}:
               {{host | combine(openshift_master_node_vars) | to_nice_yaml() | indent(6)}}
           {% endfor %}
         {% endif %}

         {% if new_masters | count > 0 %}
         new_masters:
           hosts:
           {% for host in new_masters %}
           {{host.hostname}}:
               {{host | combine(openshift_master_node_vars) | to_nice_yaml() | indent(6)}}
           {% endfor %}

         new_etcd:
           children:
             new_masters: {}
         {% endif %}

         etcd:
           children:
             masters: {}

         OSEv3:
           children:
             masters: {}
             nodes: {}
             new_masters: {}
             new_nodes: {}
             {% if groups['openshift_glusterfs'] | default([]) %}glusterfs: {}{% endif %}

In the case of OpenShift, Ansible itself is also called as a command from here,
using variables and the inventory that's generated in this section. This way we
don't need to mix the inventory that the overcloud deployment itself is using
with the inventory that the OpenShift deployment uses.

Deploy steps tasks
^^^^^^^^^^^^^^^^^^

These are Ansible tasks that take place in the overcloud nodes. Note that like
any other service, these tasks will only execute on the nodes whose role has
this service enabled. You'll find this as the ``deploy_steps_tasks`` section in
the service templates. These actions are also ran as part of the deployment
steps, so you'll have the ``step`` fact available in order to limit the
ansible tasks to only run on a specific step. Note that on each step, this runs
after the "external deploy tasks", but before the puppet run and the container
deployment.

Typically you'll run quite simple tasks in this section, such as setting the
boot parameters for the nodes. Although, you can also run more complex roles,
such as the IPSec service deployment for TripleO::

   - name: IPSEC configuration on step 1
     when: step == '1'
     block:
     - include_role:
         name: tripleo-ipsec
       vars:
         map_merge:
         - ipsec_configure_vips: false
           ipsec_skip_firewall_rules: false
         - {get_param: IpsecVars}

This type of deployment applies for services that are better tied to TripleO's
Ansible inventory or that don't require a specific inventory to run.

Container-related parameters
----------------------------

This covers the sections that allow you to write a containerized service for
TripleO.

Containerized services brought a big change to TripleO. From packaging puppet
manifests and relying on them for configuration, we now have to package
containers, make sure the configuration ends up in the container somehow, then
run the containers. Here I won't describe the whole workflow of how we
containerized OpenStack services, but instead I'll describe what you need to
know to deploy a containerized service with TripleO.

``puppet_config`` section
^^^^^^^^^^^^^^^^^^^^^^^^^

Before getting into the deployment steps where TripleO starts running services
and containers, there is a step where puppet is ran in containers and all the
needed configurations are created. The ``puppet_config`` section controls this
step.

There are several options we can pass here:

* ``puppet_tags``: This describes the puppet resources that will be allowed to
  run in puppet when generating the configuration files. Note that deeper
  knowledge of your manifests and what runs in puppet is required for this.
  Else, it might be better to generate the configuration files with Ansible
  with the mechanisms described in previous sections of this document.
  Any service that specifies tags will have the default tags of
  ``'file,concat,file_line,augeas,cron'`` appended to the setting.
  To know what settings to set here, as mentioned, you need to know your puppet
  manifests. But, for instance, for keystone, an appropriate setting would be:
  ``keystone_config``. For our etcd example, no tags are needed, since the
  default tags we set here are enough.

* ``config_volume``: The name of the directory where configuration files
  will be generated for this service. You'll eventually use this to know what
  location to bind-mount into the container to get the configuration. So, the
  configuration will be persisted in:
  ``/var/lib/config-data/puppet-generated/<config_volume>``

* ``config_image``: The name of the container image that will be used for
  generating configuration files. This is often the same container
  that the runtime service uses. Some services share a common set of
  config files which are generated in a common base container. Typically
  you'll get this from a parameter you pass to the template, e.g.
  ``<Service name>Image`` or ``<Service name>ConfigImage``. Dealing with these
  images requires dealing with the `container image prepare workflow`_.
  The parameter should point to the specific image to be used, and it'll be
  pulled from the registry as part of the
  deployment.

* ``step_config``: Similarly to the ``step_config`` that's described earlier in
  this document, this setting controls the puppet manifest that is ran for this
  service. The aforementioned puppet tags are used along with this manifest to
  generate a config directory for this container.

One important thing to note is that, if you're creating a containerized
service, you don't need to output a ``step_config`` section from the
``roles_data`` output. TripleO figured out if you're creating a containerized
service by checking for the existence of the ``docker_config`` section in the
``roles_data`` output.

``kolla_config`` section
^^^^^^^^^^^^^^^^^^^^^^^^

As you might know, TripleO uses kolla to build the container images. Kolla,
however, not only provides the container definitions, but provides a rich
framework to extend and configure your containers. Part of this is the fact
that it provides an entry point that receives a configuration file, with which
you can modify several things from the container on start-up. We take advantage
of this in TripleO, and it's exactly what the ``kolla_config`` represents.

For each container we create, we have a relevant ``kolla_config`` entry, with a
mapping key that has the following format::

    /var/lib/kolla/config_files/<container name>.json

This, contains YAML that represents how to map config files into the container.
In the container, this typically ends up mapped as
``/var/lib/kolla/config_files/config.json`` which kolla will end up reading.

The typical configuration settings we use with this setting are the following:

* ``command``: This defines the command we'll be running on the container.
  Typically it'll be the command that runs the "server". So, in the example you
  see ``/usr/bin/etcd ...``, which will be the main process running.

* ``config_files``: This tells kolla where to read the configuration files
  from, and where to persist them to. Typically what this is used for is that
  the configuration generated by puppet is read from the host as "read-only",
  and mounted on ``/var/lib/kolla/config_files/src``. Subsequently, it is
  copied on to the right location by the kolla mechanisms. This way we make
  sure that the container has the right permissions for the right user, given
  we'll typically be in another user namespace in the container.

* ``permissions``: As you would expect, this sets up the appropriate
  permissions for a file or set of files in the container.

``docker_config`` section
^^^^^^^^^^^^^^^^^^^^^^^^^

This is the section where we tell TripleO what containers to start. Here, we
explicitly write on which step to start which container. Steps are set as keys
with the ``step_<step number>`` format. Inside these, we should set up keys
with the specific container names. In our example, we're running only the etcd
container, so we use a key called ``etcd`` to give it such a name.
`Paunch`_ or tripleo_container_manage_ Ansible role will read these parameters,
and start the containers with those settings.

Here's an example of the container definition::

   step_2:
     etcd:
       image: {get_param: ContainerEtcdImage}
       net: host
       privileged: false
       restart: always
       healthcheck:
         test: /openstack/healthcheck
       volumes:
         - /var/lib/etcd:/var/lib/etcd
         - /etc/localtime:/etc/localtime:ro
         - /var/lib/kolla/config_files/etcd.json:/var/lib/kolla/config_files/config.json:ro
         - /var/lib/config-data/puppet-generated/etcd/:/var/lib/kolla/config_files/src:ro
       environment:
         - KOLLA_CONFIG_STRATEGY=COPY_ALWAYS

This is what we're telling TripleO to do:

* Start the container on step 2

* Use the container image coming from the ``ContainerEtcdImage`` heat parameter.

* For the container, use the host's network.

* The container is not `privileged`_.

* The container will use the ``/openstack/healthcheck`` endpoint for healthchecking

* We tell it what volumes to mount

    - Aside from the necessary mounts, note that we're bind-mounting the
      file ``/var/lib/kolla/config_files/etcd.json`` on to
      ``/var/lib/kolla/config_files/config.json``. This will be read by kolla
      in order for the container to execute the actions we configured in the
      ``kolla_config`` section.

    - We also bind-mount ``/var/lib/config-data/puppet-generated/etcd/``, which
      is where the puppet ran (which was ran inside a container) persisted the
      needed configuration files. We bind-mounted this to
      ``/var/lib/kolla/config_files/src`` since we told kolla to copy this to
      the correct location inside the container on the ``config_files`` section
      that's part of ``kolla_config``.

* Environment tells the container engine which environment variables to set

    - We set ``KOLLA_CONFIG_STRATEGY=COPY_ALWAYS`` in the example, since this
      tells kolla to always execute the ``config_files`` and ``permissions``
      directives as part of the kolla entry point. If we don't set this, it
      will only be executed the first time we run the container.

``container_puppet_tasks`` section
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

These are containerized puppet executions that are meant as bootstrapping
tasks. They typically run on a "bootstrap node", meaning, they only run on one
relevant node in the cluster. And are meant for actions that you should only
execute once. Examples of this are: creating keystone endpoints, creating
keystone domains, creating the database users, etc.

The format for this is quite similar to the one described in ``puppet_config``
section, except for the fact that you can set several of these, and they also
run as part of the steps (you can specify several of these, divided by the
``step_<step number>`` keys).

.. note:: This was docker_puppet_tasks prior to the Train cycle.


.. References

.. _tht deployment readme: https://opendev.org/openstack/tripleo-heat-templates/src/branch/master/deployment/README.rst
.. _heat resource chain object: https://docs.openstack.org/heat/pike/template_guide/openstack.html#OS::Heat::ResourceChain
.. _common/services.yaml: https://github.com/openstack/tripleo-heat-templates/blob/stable/queens/common/services.yaml#L44
.. _container image prepare workflow: https://docs.openstack.org/tripleo-docs/latest/install/containers_deployment/overcloud.html#preparing-overcloud-images
.. _Paunch: https://docs.openstack.org/paunch/readme.html
.. _tripleo_container_manage: https://docs.openstack.org/tripleo-ansible/latest/roles/role-tripleo_container_manage.html
.. _privileged: https://www.linux.com/blog/learn/sysadmin/2017/5/lazy-privileged-docker-containers
