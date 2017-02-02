Updating tripleo-heat-templates
-------------------------------

This section will describe the changes needed for tripleo-heat-templates.

Folder structure convention for tripleo-heat-templates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Services should be defined in the services folder, depending on the service
purpose.
::

  puppet
    services          ---> To host all services.
      <service type>             ---> Folder to store a specific type services (If time, will store time based services like: NTP, timezone, Chrony among others).
        <service name>.yaml      ---> Heat template defining per-service configuration.
        <service name>-base.yaml ---> Heat template defining common service configuration.

.. note::

  No puppet manifests may be defined in the `THT repository`_, they
  should go to the `puppet-tripleo repository`_ instead.

.. note::

  The use of a base heat template (<service>-base.yaml) is necessary in cases where
  a given 'service' (e.g. "heat") is comprised of a number of individual
  component services (e.g. heat-api, heat-engine) which need to share some
  of the base configuration (such as rabbit credentials).
  Using a base template in those cases means we don't need to
  duplicate that configuration.
  Refer to: https://review.openstack.org/#/c/313577/ for further details.
  Also, refer to :ref:`duplicated-parameters` for an use-case description.

Changes list
~~~~~~~~~~~~

The list of changes in THT are:

- If there is any configuration of the given feature/service
  in any of the ``tripleo-heat-templates/puppet/manifests/*.pp``
  files, then this will need to be removed and migrated to the
  puppet-tripleo repository.

- Create a service type specific folder in the root services folder
  (``puppet/services/time``).

- Create a heat template for the service inside the puppet/services folder
  (``puppet/services/time/ntp.yaml``).

- Optionally, create a common heat template to reuse common configuration
  data, which is referenced from each per-service heat template
  (``puppet/services/time/ntp-base.yaml``).

Step 1 - Updating puppet references
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Remove all puppet references for the composable service from the current
manifests (\*.pp). All the puppet logic will live in the puppet-tripleo
repository based on a configuration step, so it is mandatory to remove all the
puppet references from tripleo-heat-templates.

The updated .pp files for the NTP example were:

- ``puppet/manifests/overcloud_cephstorage.pp``

- ``puppet/manifests/overcloud_compute.pp``

- ``puppet/manifests/overcloud_controller.pp``

- ``puppet/manifests/overcloud_controller_pacemaker.pp``

- ``puppet/manifests/overcloud_object.pp``

- ``puppet/manifests/overcloud_volume.pp``



Step 2 - overcloud-resource-registry-puppet.j2.yaml resource registry changes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The resource ``OS::TripleO::Services::Ntp`` must be defined in the resource
registry (``overcloud-resource-registry-puppet.j2.yaml``)

Create a new resource type alias which references the per-service
heat template file, as described above.

By updating the resource registry we are forcing to use a nested template to
configure our resources. In the example case the created resource
(OS::TripleO::Services::Ntp), will point to the corresponding service yaml file
(puppet/services/time/ntp.yaml).


Step 3 - overcloud.yaml initial changes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Mandatory services should be added to the ControllerServices parameter default value,
which defines all the services enabled by default in the controller(s).

From ``overcloud.yaml`` in the parameters section find:
::

  ControllerServices:
    default:
      - OS::TripleO::Services::Keystone
      - OS::TripleO::Services::GlanceApi
      - OS::TripleO::Services::GlanceRegistry
      - OS::TripleO::Services::Ntp              ---> New service deployed in the controller overcloud
    description: A list of service resources (configured in the Heat
                 resource_registry) which represent nested stacks
                 for each service that should get installed on the Controllers.
    type: comma_delimited_list


Update this section with your new service to be deployed to the controllers in
the overcloud.

The parameter ControllerServices will be used by the ControllerServiceChain
resource as follows:
::

  ControllerServiceChain:
    type: OS::TripleO::Services
    properties:
      Services: {get_param: ControllerServices}
      EndpointMap: {get_attr: [EndpointMap, endpoint_map]}

THT changes for all the different roles are covered in:

- https://review.openstack.org/#/c/310421/ (tripleo-heat-templates controller)

- https://review.openstack.org/#/c/330916/ (tripleo-heat-templates compute)

- https://review.openstack.org/#/c/330921/ (tripleo-heat-templates cephstorage)

- https://review.openstack.org/#/c/330923/ (tripleo-heat-templates objectstorage)

.. note::

  In the case of the controller services, they are defined as part of the ControllerServiceChain
  resource. If it is needed to add optional services, they need to be appended to the current
  services list defined by the default value of the ControllerServices parameter.


Step 4 - Create the services yaml files
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create: ``puppet/services/time/ntp.yaml``

This file will have all the configuration details for the service to be
configured.
::

  heat_template_version: 2016-04-08
  description: >
    NTP service deployment using puppet, this YAML file
    creates the interface between the HOT template
    and the puppet manifest that actually installs
    and configure NTP.
  parameters:
    EndpointMap:
      default: {}
      description: Mapping of service endpoint -> protocol. Typically set
                   via parameter_defaults in the resource registry.
      type: json
    NtpServers:
      default: ['0.pool.ntp.org', '1.pool.ntp.org']
      description: NTP servers
      type: comma_delimited_list
    NtpInterfaces:
      default: ['0.0.0.0']
      description: Listening interfaces
      type: comma_delimited_list
  outputs:
    role_data:
      description: Role ntp using composable services.
      value:
        config_settings:
          ntp::ntpservers: {get_param: NtpServers}
          ntp::ntpinterfaces: {get_param: NtpInterfaces}
        step_config: |
          include ::tripleo::profile::base::time::ntp

.. note::

  It is required for all service templates to accept the EndpointMap parameter,
  all other parameters are optional and may be defined per-service. Care should
  be taken to avoid naming collisions between service parameters, e.g via using
  the service name as a prefix, "Ntp" in this example.

  Service templates should output a role_data value, which is a mapping containing
  "config_settings" which is a mapping of hiera key/value pairs required to configure
  the service, and "step_config", which is a puppet manifest fragment that references
  the puppet-tripleo profile that configures the service.

  If it is needed, the templates can be decomposed to remove
  duplicated parameters among different deployment environments
  (i.e. using pacemaker). To do this see
  section :ref:`duplicated-parameters`.
