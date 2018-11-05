Deploying with OVS DPDK Support
===============================

TripleO can deploy Overcloud nodes with OVS DPDK support. A new role
``ComputeOvsDpdk`` has been added to create a custom ``roles_data.yaml`` with
composable OVS DPDK role.

Execute below command to create the ``roles_data.yaml``::

  openstack overcloud roles generate -o roles_data.yaml Controller ComputeOvsDpdk

Once a roles file is created, the following changes are required:

- Deploy Command
- Parameters
- Network Config

Deploy Command
----------------
Deploy command should include the generated roles data file from the above
command.

Deploy command should also include the OVS DPDK environment file to override the
default neutron-ovs-agent service with neutron-ovs-dpdk-agent service. All the
required parameters are specified in this environment file as commented. The
parameters has to be configured according to the baremetal on which OVS DPDK
is enabled.

Also, OVS-DPDK requires mandatory kernel parameters to be set before
configuring the DPDK driver, like ``intel_iommu=on`` on Intel machines. In
order to enable the configuration of kernel parameters to the host, host-
config-pre-network environment file has to be added for the deploy command.

Adding the following arguments to the ``openstack overcloud deploy`` command
will do the trick::

  openstack overcloud deploy --templates \
    -r roles_data.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/host-config-and-reboot.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/neutron-ovs-dpdk.yaml \
    ...

Parameters
----------
Following are the list of parameters which need to be provided for deploying
with OVS DPDK support.

* OvsPmdCoreList:  List of Logical CPUs to be allocated for Poll Mode Driver
* OvsDpdkCoreList: List of Logical CPUs to be allocated for the openvswitch
  host process (lcore list)
* OvsDpdkMemoryChannels: Number of memory channels
* OvsDpdkSocketMemory: Socket memory list per NUMA node


Example::

  parameter_defaults:
    OvsPmdCoreList: "2,3,18,19"
    OvsDpdkCoreList: "0,1,16,17"
    OvsDpdkMemoryChannels: "4"
    OvsDpdkSocketMemory: "1024,1024"


The parameter ``KernelArgs`` should be provided in the deployment environment
file, with the set of kernel boot parameters to be applied on the
``ComputeOvsDpdk`` role where OVS DPDK is enabled::

  parameter_defaults:
    ComputeOvsDpdkParameters:
      KernelArgs: "default_hugepagesz=1GB hugepagesz=1G hugepages=64 intel_iommu=on iommu=pt"


Network Config
--------------
DPDK supported network interfaces should be specified in the network config
templates to configure OVS DPDK on the node. The following new network config
types have been added to support DPDK.

- ovs_user_bridge
- ovs_dpdk_port
- ovs_dpdk_bond

Example::

          network_config:
            -
              type: ovs_user_bridge
              name: br-link
              use_dhcp: false
              members:
                -
                  type: ovs_dpdk_port
                  name: dpdk0
                  mtu: 2000
                  rx_queu: 2
                  members:
                    -
                      type: interface
                      name: nic3
