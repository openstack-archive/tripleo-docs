Service template sections description
=====================================

As mentioned in the previous sections of the developer guide, there are several
sections of the template's output that need to be filled out for creating a
service in TripleO.

In this document we will attempt to enumerate all of them and explain the
reasoning behind them.

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
    DefaultPasswords:
      default: {}
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
        DefaultPasswords: {get_param: DefaultPasswords}
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

* **DefaultPasswords**: Defines the default passwords for only some of the
  services... Namely, the following parameters are available through here:
  DefaultMysqlRootPassword, DefaultRabbitCookie, DefaultHeatAuthEncryptionKey,
  DefaultPcsdPassword, DefaultHorizonSecret. Note that TripleO usually will
  autogenerate the passwords with secure, randomly generated defaults, so this
  is barely used.

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

.. References

.. _heat resource chain object: https://docs.openstack.org/heat/pike/template_guide/openstack.html#OS::Heat::ResourceChain
.. _common/services.yaml: https://github.com/openstack/tripleo-heat-templates/blob/stable/queens/common/services.yaml#L44
