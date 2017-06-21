.. _custom_roles:

Deploying with Custom Roles
===========================

TripleO offers the option of deploying with a user-defined list of roles,
each running a user defined list of services (where "role" means group of
nodes, e.g "Controller", and "service" refers to the individual services or
configurations e.g "Nova API").

See :doc:`composable_services` if you only wish to modify the default list of
deployed services, or see below if you wish to modify the deployed roles.


Provided example roles
----------------------

TripleO offers examples roles provided in `openstack-tripleo-heat-templates`.
These roles can be listed using the `tripleoclient` by running::

    openstack overcloud role list

With these provided roles, the user deploying the overcloud can generate a
`roles_data.yaml` file that contains the roles they would like to use for the
overcloud nodes.  Additionally the user can manage their personal custom roles
in a similar manor by storing the individual files in a directory and using
the `tripleoclient` to generate their `roles_data.yaml`. For example, a user
can execute the following to create a `roles_data.yaml` containing only the
`Controller` and `Compute` roles::

    openstack overcloud roles generate -o ~/roles_data.yaml Controller Compute

Deploying with custom roles
---------------------------

Each role is defined in the `roles_data.yaml` file. There is a sample file in
`/usr/share/openstack-tripleo-heat-templates`, or the tripleo-heat-templates_ git
repository.

The data in `roles_data.yaml` is used to perform templating with jinja2_ such
that arbitrary user-defined roles may be added, and the default roles may
be modified or removed.

The steps to define your custom roles configuration are:

1. Copy the default roles provided by `tripleo-heat-templates`:

    mkdir ~/roles
    cp /usr/share/openstack-tripleo-heat-templates/roles/* ~/roles

2. Create a new role file with your custom role.

Additional details about the format for the roles file can be found in the
`README.rst <http://git.openstack.org/cgit/openstack/tripleo-heat-templates/tree/roles/README.rst>`_
in the roles/ directory from `tripleo-heat-templates`. The filename should
match the name of the role. For example if adding a new role named `Galera`,
the role file name should be `Galera.yaml`. The file should at least contain
the following items:

    * name: Name of the role e.g "CustomController", mandatory
    * ServicesDefault: List of services, optional, defaults to an empty list
      See the default roles_data.yaml or overcloud-resource-registry-puppet.j2.yaml
      for the list of supported services. Both files can be found in the top
      tripleo-heat-templates folder

Additional items like the ones below should be included as well:

    * CountDefault: Default number of nodes, defaults to zero
    * HostnameFormatDefault: Format string for hostname, optional
    * Description: A few sentences describing the role and information
      pertaining to the usage of the role.

The role file format is a basic yaml structure. The expectation is that there
is a single role per file. See the roles `README.rst` for additional details. For
example the following role might be used to deploy a pacemaker managed galera
cluster::

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

.. warning::
   When scaling your deployment out, you need as well set the role counts in the
   "parameter_defaults" section. The ``--control-scale`` and ``--compute-scale``
   CLI args are hardcoded to the "Control" and "Compute" role names, so they're in
   fact ignored when using custom roles.

3. Create a `roles_data.yaml` file that contains the custom role in addition
   to the other roles that will be deployed. For example::

    openstack overcloud roles generate --roles-path ~/roles -o ~/my_roles_data.yaml Controller Compute Galera

4. Pass the modified roles_data on deployment as follows::

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
