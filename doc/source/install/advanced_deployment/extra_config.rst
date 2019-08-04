Node customization and Third-Party Integration
==============================================

It is possible to enable additional configuration during one of the
following deployment phases:

* firstboot - run once config (performed on each node by cloud-init)
* per-node - run after the node is initially created but before services are deployed and configured (e.g by puppet)
* post-deploy - run after the services have been deployed and configured (e.g by puppet)

.. note::

    This documentation assumes some knowledge of heat HOT_ template
    syntax, and makes use of heat environment_ files.  See the upstream
    heat documentation_ for further information.

.. _HOT: https://docs.openstack.org/heat/template_guide/hot_guide.html
.. _environment: https://docs.openstack.org/heat/template_guide/environment.html
.. _documentation: https://docs.openstack.org/heat/template_guide/index.html

Firstboot extra configuration
-----------------------------

Firstboot configuration is optional, and is performed on *all* nodes on initial
deployment.

Any configuration possible via cloud-init may be performed at this point,
either by applying cloud-config yaml or running arbitrary additional
scripts.

The heat templates used for deployment provide the `OS::TripleO::NodeUserData`
resource as the interface to enable this configuration. A basic example of its
usage is provided below, followed by some notes related to real world
usage.

The script snippet below shows how to create a simple example containing two
scripts, combined via the MultipartMime_ resource::

    mkdir firstboot
    cat > firstboot/one_two.yaml << EOF
    heat_template_version: 2014-10-16

    resources:
      userdata:
        type: OS::Heat::MultipartMime
        properties:
          parts:
          - config: {get_resource: one_config}
          - config: {get_resource: two_config}

      one_config:
        type: OS::Heat::SoftwareConfig
        properties:
          config: |
            #!/bin/bash
            echo "one" > /tmp/one

      two_config:
        type: OS::Heat::SoftwareConfig
        properties:
          config: |
            #!/bin/bash
            echo "two" > /tmp/two

    outputs:
      OS::stack_id:
        value: {get_resource: userdata}
    EOF

.. _MultipartMime: https://docs.openstack.org/heat/template_guide/openstack.html#OS::Heat::MultipartMime

.. note::

    The stack must expose an `OS::stack_id` output which references an
    OS::Heat::MultipartMime resource.

This template is then mapped to the `OS::TripleO::NodeUserData` resource type
via a heat environment file::

    cat > userdata_env.yaml << EOF
    resource_registry:
        OS::TripleO::NodeUserData: firstboot/one_two.yaml
    EOF

You may then deploy your overcloud referencing the additional environment file::

    openstack overcloud deploy --templates \
      -e <full environment> -e userdata_env.yaml

.. note::

    Make sure you pass the same environment parameters that were used at
    deployment time in addition to your customization environments at the
    end (`userdata_env.yaml`).

.. note::

    The userdata is applied to *all* nodes in the deployment. If you need role
    specific logic, the userdata scripts can contain conditionals which use
    e.g the node hostname to determine the role.

.. note::

    OS::TripleO::NodeUserData is only applied on initial node deployment,
    not on any subsequent stack update, because cloud-init only processes the
    nova user-data once, on first boot. If you need to add custom configuration
    that runs on all stack creates and updates, see the
    `Post-Deploy extra configuration`_ section below.

For a more complete example, which creates an additional user and configures
SSH keys by accessing the nova metadata server, see
`/usr/share/openstack-tripleo-heat-templates/firstboot/userdata_example.yaml`
on the undercloud node or the tripleo-heat-templates_ repo.

.. _tripleo-heat-templates: https://opendev.org/openstack/tripleo-heat-templates

Per-node extra configuration
----------------------------

This configuration happens after any "firstboot" configuration is applied,
but before any Post-Deploy configuration takes place.

Typically these interfaces are suitable for preparing each node for service
deployment, such as registering nodes with a content repository, or creating
additional data to be consumed by the post-deploy phase.  They may also be suitable
integration points for additional third-party services, drivers or plugins.


.. note::
   If you only need to provide some additional data to the existing service
   configuration, see :ref:`node_config` as this may provide a simpler solution.

.. note::
    The per-node interface only enable *individual* nodes to be configured,
    if cluster-wide configuration is required, the Post-Deploy interfaces should be
    used instead.

The following interfaces are available:

* `OS::TripleO::ControllerExtraConfigPre`: Controller node additional configuration
* `OS::TripleO::ComputeExtraConfigPre`: Compute node additional configuration
* `OS::TripleO::CephStorageExtraConfigPre` : CephStorage node additional configuration
* `OS::TripleO::NodeExtraConfig`: additional configuration applied to all nodes (all roles).

Below is an example of a per-node configuration template that shows additional node configuration
via standard heat SoftwareConfig_ resources::

    mkdir -p extraconfig/per-node
    cat > extraconfig/per-node/example.yaml << EOF

    heat_template_version: 2014-10-16

    parameters:
      server:
        description: ID of the controller node to apply this config to
        type: string

    resources:
      NodeConfig:
        type: OS::Heat::SoftwareConfig
        properties:
          group: script
          config: |
            #!/bin/sh
            echo "Node configured" > /root/per-node

      NodeDeployment:
        type: OS::Heat::SoftwareDeployment
        properties:
          config: {get_resource: NodeConfig}
          server: {get_param: server}
    outputs:
      deploy_stdout:
        description: Deployment reference, used to trigger post-deploy on changes
        value: {get_attr: [NodeDeployment, deploy_stdout]}

    EOF

The "server" parameter must be specified in all per-node ExtraConfig templates,
this is the server to apply the configuration to, and is provided by the parent
template.  Optionally additional implementation specific parameters may also be
provided by parameter_defaults, see below for more details.

Any resources may be defined in the template, but the outputs must define a "deploy_stdout"
value, which is an identifier used to detect if the configuration applied has changed,
hence when any post-deploy actions (such as re-applying puppet manifests on update)
may need to be performed.

For a more complete example showing how to apply a personalized map of per-node configuration
to each node, see `/usr/share/openstack-tripleo-heat-templates/puppet/extraconfig/pre_deploy/per_node.yaml`
or the tripleo-heat-templates_ repo.

.. _SoftwareConfig: https://docs.openstack.org/heat/template_guide/software_deployment.html


Post-Deploy extra configuration
-------------------------------

Post-deploy additional configuration is possible via the
`OS::TripleO::NodeExtraConfigPost` interface, which is applied after any
per-node configuration has completed.

.. note::

  The `OS::TripleO::NodeExtraConfigPost` applies configuration to *all* nodes,
  there is currently no per-role NodeExtraConfigPost interface.

Below is an example of a post-deployment configuration template::

    mkdir -p extraconfig/post-deploy/
    cat > extraconfig/post-deploy/example.yaml << EOF
    heat_template_version: 2014-10-16

    parameters:
      servers:
        type: json

      # Optional implementation specific parameters
      some_extraparam:
        type: string

    resources:

      ExtraConfig:
        type: OS::Heat::SoftwareConfig
        properties:
          group: script
          config:
            str_replace:
              template: |
                #!/bin/sh
                echo "extra _APARAM_" > /root/extra
              params:
                _APARAM_: {get_param: some_extraparam}

      ExtraDeployments:
        type: OS::Heat::SoftwareDeploymentGroup
        properties:
          servers:  {get_param: servers}
          config: {get_resource: ExtraConfig}
          actions: ['CREATE'] # Only do this on CREATE
    EOF

The "servers" parameter must be specified in all NodeExtraConfigPost
templates, this is the server list to apply the configuration to,
and is provided by the parent template.

Optionally, you may define additional parameters which are consumed by the
implementation.  These may then be provided via parameter_defaults in the
environment which enables the configuration.

.. note::

    If the parameter_defaults approach is used, care must be used to avoid
    unintended reuse of parameter names between multiple templates, because
    parameter_defaults is applied globally.

The "actions" property of the `OS::Heat::SoftwareDeploymentGroup` resource may be
used to specify when the configuration should be applied, e.g only on CREATE,
only on DELETE etc.  If this is omitted, the heat default is to apply the
config on CREATE and UPDATE, e.g on initial deployment and every subsequent
update.

The extra config may be enabled via an environment file::

    cat > post_config_env.yaml << EOF
    resource_registry:
        OS::TripleO::NodeExtraConfigPost: extraconfig/post-deploy/example.yaml
    parameter_defaults:
        some_extraparam: avalue123
    EOF

You may then deploy your overcloud referencing the additional environment file::

    openstack overcloud deploy --templates \
      -e <full environment> -e post_config_env.yaml
