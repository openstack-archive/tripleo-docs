Deploying with OVS DPDK Support
===============================

TripleO can deploy Overcloud nodes with OVS DPDK support. The following
changes are required:

 - Environment File
 - Parameters
 - Network Config

Environment File
----------------

Deploy command should include the OVS DPDK environment file to override the
default neutron-ovs-agent service with neutron-ovs-dpdk-agent service. All the
required parameters are specified in this environment file as commented. The
parameters has to be configured according to the baremetal on which OVS DPDK
is enabled.

Also, OVS-DPDK requires mandatory kernel parameters to be set before
configuring the DPDK driver, like ``intel_iommu=on`` on Intel machines. In
order to enable the configuration of kernel parametres to the host, host-
config-pre-network environment file has to be added for the deploy command.

Adding the following arguments to the ``openstack overcloud deploy`` command
will do the trick::

  -e /usr/share/openstack-tripleo-heat-templates/environments/neutron-ovs-dpdk.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/host-config-pre-network.yaml \


Parameters
----------
The parameters ``NeutronDpdkCoreList`` and ``NeutronDpdkMemoryChannels`` are
mandatory for OVS-DPDK deployment. And other optional parameter to be
considered is ``NeutronDpdkSocketMemory``.::

  NeutronDpdkCoreList: "'1,2,18,19'"
  NeutronDpdkMemoryChannels: "4"
  NeutronDpdkSocketMemory: "'1024,1024'"


The parameter ``ComputeKernelArgs`` should be provided in the deployment
environment file, with the set of kernel boot parameters to be applied on the
``Compute`` role where OVS DPDK is enabled::

 ComputeKernelArgs: "default_hugepagesz=1GB hugepagesz=1G hugepages=64 intel_iommu=on"

.. note::
    The parameter ``ComputeKernelArgs`` is specific to a role. In case of
    introducing a new role like ``ComputeOvsDpdk``, the kernel args should be
    given as ``ComputeOvsDpdkKernelArgs`` parameter.

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
                  members:
                    -
                      type: interface
                      name: nic3

By default, the interface will be bound to ``vfio-pci`` DPDK driver. In case
of binding to a diffrenet driver, network config types ``ovs_dpdk_port`` and
``ovs_dpdk_bond`` each take an additional parameter ``driver`` to specify the
driver name.
