.. _custom_roles:

Deploying with Custom Roles
===========================

TripleO offers the option of deploying with a user-defined list of roles,
each running a user defined list of services (where "role" means group of
nodes, e.g "Controller", and "service" refers to the individual services or
configurations e.g "Nova API").

See :doc:`composable_services` if you only wish to modify the default list of
deployed services, or see below if you wish to modify the deployed roles.


Deploying with custom roles
---------------------------

Each role is defined in the `roles_data.yaml` file (see
`/usr/share/openstack-tripleo-heat-templates`, or the tripleo-heat-templates_ git
repository.)

The data in `roles_data.yaml` is used to perform templating with jinja2_ such
that arbitrary user-defined roles may be added, and the default roles may
be modified or removed.

The steps to define your custom roles configuration are:

1. Copy the default roles_data.yaml::

    cp /usr/share/openstack-tripleo-heat-templates/roles_data.yaml ~/my_roles_data.yaml

2. Edit my_roles_data.yaml to suit your requirements

The roles_data is a simple list of roles, where the only mandatory argument is
the name, and several optional additional keys are supported:


    * name: Name of the role e.g "CustomController", mandatory
    * CountDefault: Default number of nodes, defaults to zero
    * ServicesDefault: List of services, optional, defaults to an empty list
      See the default roles_data.yaml or overcloud-resource-registry-puppet.j2.yaml
      for the list of supported services. Both files can be found in the top
      tripleo-heat-templates folder
    * HostnameFormatDefault: Format string for hostname, optional

For example the following role would deploy a pacemaker managed galera cluster::

  - name: Galera
    HostnameFormatDefault: '%stackname%-galera-%index%'
    ServicesDefault:
      - OS::TripleO::Services::CACerts
      - OS::TripleO::Services::Timezone
      - OS::TripleO::Services::Ntp
      - OS::TripleO::Services::Snmp
      - OS::TripleO::Services::Kernel
      - OS::TripleO::Services::Pacemaker
      - OS::TripleO::Services::MySQL
      - OS::TripleO::Services::TripleoPackages
      - OS::TripleO::Services::TripleoFirewall
      - OS::TripleO::Services::SensuClient
      - OS::TripleO::Services::FluentdClient

.. note::
   In the example above, if you wanted to deploy the Galera role on specific nodes
   you would either use predictable placement :doc:`node_placement` or add a custom
   parameter called OvercloudGaleraFlavor::


     parameter_defaults:
       OvercloudGaleraFlavor: oooq_galera


3. Pass the modified roles_data on deployment as follows::

    openstack overcloud deploy --templates -r ~/my_roles_data.yaml

.. note::
  It is also possible to copy the entire tripleo-heat-templates tree, and modify
  the roles_data.yaml file in place, then deploy via `--templates <copy of tht>`

.. warning::
  Note that in your custom roles you may not use any already predefined name
  So in practice you may not override the following roles: Controller, Compute,
  BlockStorage, SwiftStorage and CephStorage. You need to use different names
  instead.


.. _tripleo-heat-templates: https://git.openstack.org/openstack/tripleo-heat-templates
.. _jinja2: http://jinja.pocoo.org/docs/dev/
