.. _network_v2:

Networking Version 2 (Two)
==========================

Introduction
------------

In the the Wallaby cycle TripleO Networking has been refactored so that no
OS::Neutron heat resources are used. This was a pre-requisite for
:doc:`./ephemeral_heat`. Managing non-ephemeral neutron resources with an
ephemeral heat stack is not feasible, so the management of neutron resources
has been externalized from the overcloud heat stack.

High level overview of the changes
..................................

* NIC config templates was migrated to ansible j2 templates during the
  Victoria release. Replacing the heat templates previously used for NIC
  configuration. Sample ansible j2 templates are available in the
  `tripleo-ansible <https://opendev.org/openstack/tripleo-ansible/src/branch/master/tripleo_ansible/roles/tripleo_network_config/templates>`_
  git repository as well as in
  ``/usr/share/ansible/roles/tripleo_network_config/templates/`` on a deployed
  undercloud.

  Please refer to :ref:`creating_custom_interface_templates` on the
  :ref:`network_isolation` documentation page for further details on writing
  custom Ansible j2 NIC config templates.

* A new schema for the network definitions used for Jinja2 rendering of the
  ``tripleo-heat-templates`` was introduced, in addition to tripleoclient
  commands to provision networks using the new network definitions schema.

* A new schema for network Virtual IPs was introduced in conjunction with
  tripleoclient commands to provision the Virtual IPs.

* Service Virtual IPs (redis and ovsdb) was refactored so that the neutron
  resources are created by the deploy-steps playbook post-stack create/update.

* The baremetal provisioning schema was extended to include advanced network
  layouts. The ``overcloud node provision`` command was extended so that it
  also provision neutron port resources for all networks defined for instances/
  roles in the baremetal provisioning definition.

* The tool (``tripleo-ansible-inventory``) used to generate the ansible
  inventory was extended to use neutron as a source for the inventory in
  addition to the overcloud heat stack outputs.

* With the TripleO ansible inventory's support to use neutron resources as a
  data source, the baremetal provisioning schema and ``overcloud node
  provision`` command was extended to allow arbitrary playbook
  execute against the provisioned nodes, as well as applying node network
  configuration utilizing the ``tripleo_network_config`` ansible role and the
  ansible j2 NIC config templates.

With all of the above in place the ``overcloud deploy`` command was extended so
that it can run all the steps:

#. Create Networks

   Run the ``cli-overcloud-network-provision.yaml`` ansible playbook using the
   network definitions provided via the ``--network-file`` argument. This
   playbook creates/updates the neutron networks on the undercloud and
   generates the ``networks-deployed.yaml`` environment file which is included
   as a user-environment when creating the overcloud heat stack.

#. Create Virtual IPs

   Run the ``cli-overcloud-network-vip-provision.yaml`` ansible playbook using
   the Virtual IP definitions provided via the ``--vip-file`` argument. This
   playbook creates/updates the Virtual IP port resources in neutron on the
   undercloud and generates the ``virtual-ips-deployed.yaml`` environment file
   which is included as a user-environment when creating the overcloud heat
   stack.

#. Provision Baremetal Instances

   Run the ``cli-overcloud-node-provision.yaml`` ansible playbook using the
   baremetal instance definitions provided via the ``--baremetal-deployment``
   argument in combination with the ``--network-config`` argument so that
   baremetal nodes are provisioned and network port resources are created. Also
   run any arbitrary Ansible playbooks provided by the user on the provisioned
   nodes before finally configured overcloud node networking using the
   ``tripleo_network_config`` ansible role.

#. Create the overcloud Ephemeral Heat stack

   The environment files with the parameters and resource registry overrides
   required is automatically included when the ``overcloud deploy`` command is
   run with the arguments: ``--vip-file``, ``--baremetal-deployement`` and
   ``--network-config``.

#. Run Config-Download and the deploy-steps playbook

   As an external deploy step the neutron ports for Service Virtual IPs are
   created, and the properties of the Virtual IPs are included in hieradata.


Using
-----

Pre-Provision networks
......................

The command to pre-provision networks for one or more overcloud stack(s) is
``openstack overcloud network provision``. The command takes a network-v2
version networks definitions YAML file as input, and writes a heat environment
file to the file specified using the ``--output`` argument.

Please refer to the :ref:`network_definition_opts` reference section on the
:ref:`custom_networks` document page for a reference on available options in
the network data YAML schema.

Sample network definition YAML files can be located in the
`tripleo-heat-templates git repository
<https://opendev.org/openstack/tripleo-heat-templates/src/branch/master/network-data-samples/>`_,
or in the ``/usr/share/openstack-tripleo-heat-templates/network-data-samples``
directory on the undercloud.


**Example**: Networks definition YAML file defining the external network.

.. code-block:: yaml

  - name: External
    name_lower: external
    vip: true
    mtu: 1500
    subnets:
      external_subnet:
        ip_subnet: 10.0.0.0/24
        allocation_pools:
          - start: 10.0.0.4
            end: 10.0.0.250
        gateway_ip: 10.0.0.1
        vlan: 10

**Example**: Create or update networks

.. code-block:: bash

  $ openstack overcloud network provision \
      --output ~/overcloud-networks-deployed.yaml \
      ~/network_data_v2.yaml

When deploying the overcloud include the environment file generated by the
``overcloud network provision`` command.

.. code-block:: bash

  $ openstack overcloud deploy --templates \
      -e ~/overcloud-networks-deployed.yaml

Pre-Provision network Virtual IPs
.................................

The command to pre-provision Virtual IPs for an overcloud stack is:
``openstack overcloud network vip provision``. The command takes a Virtual IPs
definitions YAML file as input, and writes a heat environment file to the file
specified using the ``--output`` argument. The ``--stack`` argument defines the
name of the overcloud stack for which Virtual IPs will be provisioned.

Please refer to the :ref:`virtual_ips_definition_opts` reference section on the
:ref:`custom_networks` document page for a reference on available options in
the Virtual IPs YAML schema.

Sample network definition YAML files can be located in the
`tripleo-heat-templates git repository
<https://opendev.org/openstack/tripleo-heat-templates/src/branch/master/network-data-samples/>`_,
or in the ``/usr/share/openstack-tripleo-heat-templates/network-data-samples``
directory on the undercloud.

**Example**: Virtual IPs definition YAML file defining the ctlplane and the
external network Virtual IPs.

.. code-block:: yaml

  - network: ctlplane
    dns_name: overcloud
  - network: external
    dns_name: overcloud

**Example**: Create or update Virtual IPs

.. code-block:: bash

  $ openstack overcloud network vip provision \
      --stack overcloud \
      --output ~/overcloud-vip-deployed.yaml \
      ~/vip_data.yaml

When deploying the overcloud include the environment file generated by the
``overcloud network provision`` command. For example:

.. code-block:: bash

  $ openstack overcloud deploy --templates \
      -e ~/overcloud-vip-deployed.yaml


Service Virtual IPs
...................

Service Virtual IPs are created as needed when the service is enabled. To
configure the subnet to use the existing ``ServiceVipMap`` heat parameter.
For a fixed IP allocation the existing heat parameters ``RedisVirtualFixedIPs``
and/or ``OVNDBsVirtualFixedIPs`` can be used.

**Example**: Setting fixed ips:

.. code-block:: yaml

  parameter_defaults:
    RedisVirtualFixedIPs: [{'ip_address': '172.20.0.11'}]
    OVNDBsVirtualFixedIPs: [{'ip_address': '172.20.0.12'}]

**Example**: Setting fixed IP address and not creating a neutron resource:

.. code-block:: yaml

  parameter_defaults:
    RedisVirtualFixedIPs: [{'ip_address': '172.20.0.11', 'use_neutron': false}]
    OVNDBsVirtualFixedIPs: [{'ip_address': '172.20.0.12', 'use_neutron': false}]

.. note:: Overriding the Service Virtual IPs using the resource registry
          entries ``OS::TripleO::Network::Ports::RedisVipPort`` and
          ``OS::TripleO::Network::Ports::OVNDBsVipPort`` is no longer
          supported.


Provision Baremetal Instances
.............................

Pre provisioning baremetal instances using Metalsmith has been supported for a
while. The TripleO Network v2 work extended the workflow that provision
baremetal instances to also provision the neutron network port resources and
added the interface to run arbitrary Ansible playbooks after node provisioning.

Please refer to the :ref:`baremetal_provision` document page for a reference on
available options in the Baremetal Deployment YAML schema.

**Example**: Baremetal Deployment YAML set up for default the default
network-isolation scenario, including one pre-network config Ansible playbook
that will be run against the nodes in each role.

.. code-block:: yaml

  - name: Controller
    count: 1
    hostname_format: controller-%index%
    ansible_playbooks:
      - playbook: bm-deploy-playbook.yaml
    defaults:
      profile: control
      networks:
        - network: external
          subnet: external_subnet
        - network: internal_api
          subnet: internal_api_subnet01
        - network: storage
          subnet: storage_subnet01
        - network: storage_mgmt
          subnet: storage_mgmt_subnet01
        - network: tenant
          subnet: tenant_subnet01
      network_config:
        template: templates/multiple_nics/multiple_nics_dvr.j2
        default_route_network:
          - external
  - name: Compute
    count: 1
    hostname_format: compute-%index%
    ansible_playbooks:
      - playbook: bm-deploy-playbook.yaml
    defaults:
      profile: compute-leaf2
      networks:
        - network: internal_api
          subnet: internal_api_subnet02
        - network: tenant
          subnet: tenant_subnet02
        - network: storage
          subnet: storage_subnet02
      network_config:
        template: templates/multiple_nics/multiple_nics_dvr.j2

**Example**: Arbitrary Ansible playbook example bm-deploy-playbook.yaml

.. code-block:: yaml

  - name: Overcloud Node Network Config
    hosts: allovercloud
    any_errors_fatal: true
    gather_facts: false
    tasks:
    - name: A task
      debug:
        msg: "A message"

To provision baremetal nodes, create neuron port resource and apply network
configuration as defined in the above definition run the ``openstack overcloud
node provision`` command including the ``--network-config`` argument as shown
in the below example:

.. code-block:: bash

  $ openstack overcloud node provision \
      --stack overcloud \
      --network-config \
      --output ~/overcloud-baremetal-deployed.yaml \
      ~/baremetal_deployment.yaml

When deploying the overcloud include the environment file generated by the
``overcloud node provision`` command and enable the ``--deployed-server``
argument.

.. code-block:: bash

  $ openstack overcloud deploy --templates \
      --deployed-server \
      -e ~/overcloud-baremetal-deployed.yaml

The *All-in-One* alternative using overcloud deploy command
.............................................................

It is possible to instruct the ``openstack overcloud deploy`` command to do all
of the above steps in one go. The same YAML definitions can be used and the
environment files will be automatically included.

**Example**: Use the **All-in-One** deploy command:

.. code-block:: bash

  $ openstack overcloud deploy \
      --templates \
      --stack overcloud \
      --network-config \
      --deployed-server \
      --roles-file ~/my_roles_data.yaml \
      --networks-file ~/network_data_v2.yaml \
      --vip-file ~/vip_data.yaml \
      --baremetal-deployment ~/baremetal_deployment.yaml


Managing Multiple Overclouds
............................

When managing multiple overclouds using a single undercloud one would have to
use a different ``--stack`` name and ``--output`` as well as per-overcloud
YAML definitions for provisioning Virtual IPs and baremetal nodes.

Networks can be shared, or separate for each overcloud stack. If they are
shared, use the same network definition YAML and deployed network environment
for all stacks. In the case where networks are not shared, a separate network
definitions YAML and a separate deployed network environment file must be used
by each stack.

.. note:: The ``ctlplane`` provisioning network will always be shared.


Migrating existing deployments
------------------------------

To facilitate the migration for deployed overclouds tripleoclient commands to
extract information from deployed overcloud stacks has been added. During the
upgrade to Wallaby these tools will be executed as part of the underlcoud
upgrade, placing the generated YAML definition files in the working directory
(Defaults to: ``~/overcloud-deploy/$STACK_NAME/``). Below each export command
is described, and examples to use them manually with the intent for developers
and operators to be able to better understand what happens "under the hood"
during the undercloud upgrade.

There is also a tool ``convert_heat_nic_config_to_ansible_j2.py`` that can be
used to convert heat template NIC config to Ansible j2 templates.

.. warning:: If migrating to use Networking v2 while using the non-Epemeral
             heat i.e ``--heat-type installed``, the existing overcloud stack
             must **first** be updated to set the ``deletion_policy`` for
             ``OS::Nova`` and ``OS::Neutron`` resources. This can be done
             using a ``--stack-only`` update, including an environment file
             setting the following tripleo-heat-templates parameters
             ``NetworkDeletionPolicy``, ``PortDeletionPolicy`` and
             ``ServerDeletionPolicy`` to ``retain``.

             If the deletion policy is not set to ``retain`` the
             orchestration service will **delete** the existing resources
             when an update using the Networking v2 environments is
             performed.

Conflicting legacy environment files
....................................

The heat environment files created by the Networking v2 commands uses resource
registry overrides to replace the existing resources with *pre-deployed*
resource types. These resource registry entries was also used by legacy
environment files, such as ``network-isolation.yaml``. The legacy files should
no longer be used, as they will nullify the new overrides.

It is recommended to compare the generated environment files with existing
environment files used with the overcloud deployment prior to the migration and
remove all settings that overlap with the settings in the generated environment
files.

Convert NIC configs
...................

In the tripleo-heat-templates ``tools`` directory there is a script
``convert_heat_nic_config_to_ansible_j2.py`` that can be used to convert heat
NIC config templates to Ansible j2 NIC config templates.

**Example**: Convert the compute.yaml heat NIC config template to Ansible j2.

.. code-block:: bash

  $ /usr/share/openstack-tripleo-heat-templates/convert_heat_nic_config_to_ansible_j2.py \
      --stack overcloud \
      --networks-file network_data.yaml \
      ~/nic-configs/compute.yaml

.. warning:: The tool does a best-effort to fully automate the conversion. The
             new Ansible j2 template files should be inspected, there may be
             a need to manually edit the new Ansible j2 template. The tool will
             try to highlight any issues that need manual intervention by
             adding comments in the Ansible j2 file.

The :ref:`migrating_existing_network_interface_templates` section on the
:ref:`network_isolation` documentation page provides a guide for manual
migration.

Generate Network YAML
.....................

The command ``openstack overcloud network extract`` can be used to generate
a Network definition YAML file from a deployed overcloud stack. The YAML
definition file can then be used with ``openstack overcloud network provision``
and the ``openstack overcloud deploy`` command.

**Example**: Generate a Network definition YAML for the ``overcloud`` stack:

.. code-block:: bash

  $ openstack overcloud network extract \
      --stack overcloud \
      --output ~/network_data_v2.yaml

Generate Virtual IPs YAML
.........................

The command ``openstack overcloud network vip extract`` can be used to generate
a Virtual IPs definition YAML file from a deployed overcloud stack. The YAML
definition file can then be used with ``openstack overcloud network vip
provision`` command and/or the ``openstack overcloud deploy`` command.

**Example**: Generate a Virtual IPs  definition YAML for the ``overcloud``
stack:

.. code-block:: bash

  $ openstack overcloud network vip extract \
      --stack overcloud \
      --output /home/centos/overcloud/network_vips_data.yaml

Generate Baremetal Provision YAML
.................................

The command ``openstack overcloud node extract provisioned`` can be used to
generate a Baremetal Provision definition YAML file from a deployed overcloud
stack. The YAML definition file can then be used with ``openstack overcloud
node provision`` command and/or the ``openstack overcloud deploy`` command.

**Example**: Export deployed overcloud nodes to Baremtal Deployment YAML
definition

.. code-block:: bash

  $ openstack overcloud node extract provisioned \
      --stack overcloud \
      --roles-file ~/tht_roles_data.yaml \
      --output ~/baremetal_deployment.yaml
