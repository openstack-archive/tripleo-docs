.. _node_config:

Modifying default node configuration
====================================

Many service configuration options are already exposed via parameters in the
top-level `overcloud.yaml` template, and these options should
be used wherever available to influence overcloud configuration.

However in the event the service configuration required is not exposed
as a top-level parameter, there are flexible interfaces which enable passing
arbitrary additional configuration to the nodes on deployment.

Making configuration changes
----------------------------

If you want to make a configuration change, either prior to initial deployment,
or subsequently via an update, you can pass additional data to puppet via hiera
data, using either the global "ExtraConfig" parameter, or one of the role-specific
parameters, e.g using `ComputeExtraConfig` to set the reserved_host_memory
value for compute nodes::


    cat > compute_params.yaml << EOF
    parameter_defaults:
        ComputeExtraConfig:
          nova::compute::reserved_host_memory: some_value
    EOF

    openstack overcloud deploy -e compute_params.yaml

The parameters available are:

* `ExtraConfig`: Apply the data to all nodes, e.g all roles
* `ComputeExtraConfig`: Apply the data only to Compute nodes
* `ControllerExtraConfig`: Apply the data only to Controller nodes
* `BlockStorageExtraConfig`: Apply the data only to BlockStorage nodes
* `ObjectStorageExtraConfig`: Apply the data only to ObjectStorage nodes
* `CephStorageExtraConfig`: Apply the data only to CephStorage nodes

For any custom roles (defined via roles_data.yaml) the parameter name will
be RoleNameExtraConfig where RoleName is the name specified in roles_data.yaml.

.. note::

    Previously the parameter for Controller nodes was named
    `controllerExtraConfig` (note the inconsistent capitalization). If
    you are updating a deployment which used the old parameter, all
    values previously passed to `controllerExtraConfig` should be
    passed to `ControllerExtraConfig` instead, and
    `controllerExtraConfig: {}` should be explicitly set in
    `parameter_defaults`, to ensure that values from the old parameter
    will not be used anymore.  Also ComputeExtraConfig was previously
    named NovaComputeExtraConfig, so a similar update should be performed
    where the old naming is used.

.. note::

    Passing data via the ExtraConfig parameters will override any statically
    defined values in the Hiera data files included as part of tripleo-heat-templates,
    e.g those located in `puppet/hieradata` directory.

.. note::

    If you set a configuration of a puppet class which is not being included
    yet, make sure you include it in the ExtraConfig definition, for example
    if you want to change the Max IOPS per host setting::

       parameter_defaults:
         ComputeExtraConfig:
           'nova::scheduler::filter::max_io_ops_per_host': '4.0'
           Compute_classes:
           - '::nova::scheduler::filter'

    The Compute_classes data is included via the hiera_include in the
    overcloud_common.pp puppet manifest.
