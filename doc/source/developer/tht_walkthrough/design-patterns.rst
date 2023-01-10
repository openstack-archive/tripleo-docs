THT design patterns
-------------------

.. _duplicated-parameters:

Duplicated parameters
~~~~~~~~~~~~~~~~~~~~~

Problem: When defining multiple related services, it can be necessary
to define the same parameters (such as rabbit or DB credentials) in
multiple service templates.  To avoid this, it is possible to define a
"base" heat template that contains the common parameters and config_settings
mapping for those services that require it.

This pattern will describe how to avoid duplicated parameters in the THT yaml
files.

``mongodb-base.yaml``: This file should have all the common parameters between
the different environments (With pacemaker and without pacemaker).
::

  heat_template_version: rocky
  description: >
    Configuration details for MongoDB service using composable roles
  parameters:
    MongoDbNoJournal:
      default: false
      description: Should MongoDb journaling be disabled
      type: boolean
    MongoDbIPv6:
      default: false
      description: Enable IPv6 if MongoDB VIP is IPv6
      type: boolean
    MongoDbReplset:
      type: string
      default: "tripleo"
  outputs:
    role_data:
      description: Role data for the MongoDB base service.
      value:
        config_settings:
          mongodb::server::nojournal: {get_param: MongoDbNoJournal}
          mongodb::server::ipv6: {get_param: MongoDbIPv6}
          mongodb::server::replset: {get_param: MongoDbReplset}

In this way we will be able to reuse the common parameter among all the
template files requiring it.

Referencing the common parameter:

``mongodb.yaml``: Will have specific parameters to deploy mongodb without
pacemaker.
::

  heat_template_version: rocky
  description: >
    MongoDb service deployment using puppet
  parameters:
    #Parameters not used EndpointMap
    EndpointMap:
      default: {}
      description: Mapping of service endpoint -> protocol. Typically set
                   via parameter_defaults in the resource registry.
      type: json
  resources:
    MongoDbBase:
      type: ./mongodb-base.yaml
  outputs:
    role_data:
      description: Service mongodb using composable services.
      value:
        config_settings:
          map_merge:
            - get_attr: [MongoDbBase, role_data, config_settings]
            - mongodb::server::service_manage: True
        step_config: |
          include ::tripleo::profile::base::database::mongodb

In this case mongodb.yaml is using all the common parameter added in the
MongoDbBase resource.

If using the parameter 'EndpointMap' in the base template, you must the pass it from the service file,
and even if it is not used in the service template, it must still be defined.

In the service file:
::

  parameters:
    EndpointMap:
      default: {}
      description: Mapping of service endpoint -> protocol. Typically set
                   via parameter_defaults in the resource registry.
      type: json
  resources:
    <ServiceName>ServiceBase:
      type: ./<ServiceName>-base.yaml
      properties:
        EndpointMap: {get_param: EndpointMap}

This will pass the endpoint information to the base config file.

.. note::

  Even if the EndpointMap parameter is optional in the base template,
  for consistency is advised always using it in all service templates.
