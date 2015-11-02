Modifying default node configuration
====================================

Many service configurarion options are already exposed via parameters in the
top-level `overcloud-without-mergepy.yaml` template, and these options should
be used wherever available to influence overcloud configuration.

However in the event the service configuration required is not exposed
as a top-level parameter, there are flexible interfaces which enable passing
arbitrary additional configuration to the nodes on deployment.

Making configuration changes
----------------------------

If you want to make a configuration change, either prior to initial deployment,
or subsequently via an update, you can pass additional data to puppet via hiera
data, using either the global "ExtraConfig" parameter, or one of the role-specific
parameters, e.g using `NovaComputeExtraConfig` to set the reserved_host_memory
value for compute nodes::


    cat > compute_params.yaml << EOF
    parameters:
        NovaComputeExtraConfig:
          nova::compute::reserved_host_memory: some_value
    EOF

   openstack overcloud deploy -e compute_params.yaml

The parameters available are:

  * `ExtraConfig`: Apply the data to all nodes, e.g all roles
  * `NovaComputeExtraConfig`: Apply the data only to Compute nodes
  * `controllerExtraConfig`: Apply the data only to Controller nodes *(note the inconsistent capitalization...)*
  * `BlockStorageExtraConfig`: Apply the data only to BlockStorage nodes
  * `ObjectStorageExtraConfig`: Apply the data only to ObjectStorage nodes
  * `CephStorageExtraConfig`: Apply the data only to CephStorage nodes

.. note::

    Passing data via the ExtraConfig parameters will override any statically
    defined values in the Hiera data files included as part of tripleo-heat-templates,
    e.g those located in `puppet/hieradata` directory.

.. note::

   If you set a configuration of a puppet class which is not being included
   yet, make sure you include it in the ExtraConfig definition, for example
   if you want to change CPU allocation ratio::

       parameters:
         NovaComputeExtraConfig:
           'nova::scheduler::filter::cpu_allocation_ratio': '11.0'
           compute_classes:
           - '::nova::scheduler::filter'

    The compute_classes data is included via the hiera_include in the
    overcloud_compute.pp puppet manifest.
