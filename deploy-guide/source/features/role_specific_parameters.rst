Role-Specific Parameters
========================

A service can be associated with multiple roles, like ``nova-compute``
service can be associated with **ComputeRole1** and **ComputeRole2**. The
``nova-compute`` service takes multiple parameters like ``NovaVcpuPinSet``,
``NovaReservedHostMemory``, etc. It is possible to provide separate values
specific to a role with the following changes in the user environment file::

    parameter_defaults:
      NovaReservedHostMemory: 512
      ComputeRole1Parameters:
        NovaReservedHostMemory: 2048
      ComputeRole2Parameter:
        NovaReservedHostMemory: 1024

The format to provide role-specific parameters is ``<RoleName>Parameters``,
where the ``RoleName`` is the name of the role as defined in the
``roles_data.yaml`` template.

In the above specified example, the value "512" will be applied all the roles
which has the ``nova-compute`` service, where as the value "2048" will be
applied only on the **ComputeRole1** role and the value "1024" will be applied
only on the **ComputeRole2** role.

With this approach, the service implementation has to merge the role-specific
parameters with the global parameters in their definition template. The role-
specific parameter takes higher precedence than the global parameters.

For any custom service which need to use role-specific parameter, the
parameter merging should be done. Here is a sample parameter merging example
which will be done by the service implementation::

    RoleParametersValue:
      type: OS::Heat::Value
      properties:
        type: json
        value:
          map_replace:
            - map_replace:
              - neutron::agents::ml2::ovs::datapath_type: NeutronDatapathType
                neutron::agents::ml2::ovs::vhostuser_socket_dir: NeutronVhostuserSocketDir
                vswitch::dpdk::driver_type: NeutronDpdkDriverType
                vswitch::dpdk::host_core_list: HostCpusList
                vswitch::dpdk::pmd_core_list: NeutronDpdkCoreList
                vswitch::dpdk::memory_channels: NeutronDpdkMemoryChannels
                vswitch::dpdk::socket_mem: NeutronDpdkSocketMemory
              - values: {get_param: [RoleParameters]}
            - values:
                NeutronDatapathType: {get_param: NeutronDatapathType}
                NeutronVhostuserSocketDir: {get_param: NeutronVhostuserSocketDir}
                NeutronDpdkDriverType: {get_param: NeutronDpdkDriverType}
                HostCpusList: {get_param: HostCpusList}
                NeutronDpdkCoreList: {get_param: NeutronDpdkCoreList}
                NeutronDpdkMemoryChannels: {get_param: NeutronDpdkMemoryChannels}
                NeutronDpdkSocketMemory: {get_param: NeutronDpdkSocketMemory}

A service can have a unique variable name that is different than the role specific one.
The example below shows how to define the service variable ``KeystoneWSGITimeout``, override
it with the role specific variable ``WSGITimeout`` if it is found, and create a new alias variable
named ``wsgi_timeout`` to store the value. Later on, that value can be retrieved by using
``{get_attr: [RoleParametersValue, value, wsgi_timeout]}``.::

    parameters:

      KeystoneWSGITimeout:
        description: The timeout for the Apache virtual host created for the API endpoint.
        type: string
        default: '60'
        tags:
          - role_specific

    resources:

      RoleParametersValue:
        type: OS::Heat::Value
        properties:
          type: json
          value:
            map_replace:
              - map_replace:
                - wsgi_timeout: WSGITimeout
                - values: {get_param: [RoleParameters]}
              - values:
                  WSGITimeout: {get_param: KeystoneWSGITimeout}

    outputs:
      role_data:
        description: Role data for the Keystone API role.
        value:
          config_settings:
            map_merge:
              - keystone::wsgi::apache::vhost_custom_fragment:
                  list_join: [' ', ['Timeout', {get_attr: [RoleParametersValue, value, wsgi_timeout]}]]

Now the variable can optionally have a default set at the composable roles data level.::

    - name: Undercloud
      RoleParametersDefault:
        WSGITimeout: '600'

.. note::
    As of now, not all parameters can be set per role, it is based on the
    service or template implementation. Each service should have the
    implementation to merge the global parameters and role-specific
    parameters, as explained in the above examples. A warning will be shown
    during the deployment, if an invalid parameter (which does not support
    role-specific implementation) is provided as role-specific input.
