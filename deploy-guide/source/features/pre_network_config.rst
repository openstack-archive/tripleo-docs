Configure node before Network Config
====================================

In specific deployments, it is required to perform additional configurations
on the overcloud node before network deployment, but after applying kernel
args. For example, OvS-DPDK deployment requires DPDK to be enabled in
OpenvSwitch before network deployment (os-net-config), but after the
hugepages are created (hugepages are created using kernel args). This
requirement is also valid for some 3rd party SDN integration. This kind of
configuration requires additional TripleO service definitions. This document
explains how to achieve such deployments on and after `train` release.

.. note::

        In `queens` release, the resource `PreNetworkConfig` can be overridden to
        achieve the required behavior, which has been deprecated from `train`
        onwards. The implementations based on `PreNetworkConfig` should be
        moved to other available alternates.

The TripleO service `OS::TripleO::BootParams` configures the parameter
`KernelArgs` and reboots the node using the `tripleo-ansible` role
`tripleo_kernel`. Some points to consider on `KernelArgs`:

* `BootParams` service is enabled by default on all the roles.
* The node will be restarted only when kernel args are applied for the first
  time (fresh node configuration).
* In case of adding `KernelArgs` during update/upgrade/scale operations, when
  a particular role does not have `KernelArgs`, it results in node reboot.
  Such scenarios should be treated as role migration instead adding only
  `KernelArgs`.
* `KernelArgs` can be updated from `wallaby` release onwards (where the role
  already has `KernelArgs` but requires modification). In such cases, the
  node reboot has to be planned by the user manually, after the TripleO
  deployment is completed. For example, increasing the hugepages count post
  deployment.


The firstboot_ scripts provide a mechanism to apply the custom node
configuration which is independent of kernel args.

.. _firstboot: https://github.com/openstack/tripleo-heat-templates/tree/master/firstboot

Custom Service
--------------

When a configuration needs to be applied on the node after reboot and before
the network config, then a custom service template should be added that
includes the `BootParams` resource (example below) and any other required
configuration. It is important to allow the default implementation
of `BootParams` service to be included as it is, because any improvements
or fixes will be automatically included in the deployment.

Here is an example OvS-DPDK_ has been configured after `BootParams` but before
network config::

  heat_template_version: wallaby

  description: >
    Open vSwitch Configuration

  parameters:
    ServiceData:
      default: {}
      description: Dictionary packing service data
      type: json
    ServiceNetMap:
      default: {}
      description: Mapping of service_name -> network name. Typically set
                   via parameter_defaults in the resource registry. Use
                   parameter_merge_strategies to merge it with the defaults.
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

  resources
    BootParams:
      type: /usr/share/openstack-tripleo-heat-templates/deployments/kernel/kernel-boot-params-baremetal-ansible.yaml
      properties:
        ServiceData: {get_param: ServiceData}
        ServiceNetMap: {get_param: ServiceNetMap}
        EndpointMap: {get_param: EndpointMap}
        RoleName: {get_param: RoleName}
        RoleParameters: {get_param: RoleParameters}

  outputs:
    role_data:
      description: Role data for the Open vSwitch service.
      value:
        service_name: openvswitch
        deploy_steps_tasks:
          - get_attr: [BootParams, role_data, deploy_steps_tasks]
          - - name: Run ovs-dpdk role
              when: step|int == 0
              include_role:
                name: tripleo_ovs_dpdk

.. _OvS-DPDK: https://github.com/openstack/tripleo-heat-templates/blob/master/deployment/openvswitch/openvswitch-dpdk-baremetal-ansible.yaml

.. note::
   In the above sample service definition, the condition `step|int == 0` in
   the `deploy_steps_tasks` section forces the associated steps to run
   before starting any other node configuration (including network deployment).

Add this service to the roles definition of the required roles so that the
configuration can be applied after reboot but before network deployment.
