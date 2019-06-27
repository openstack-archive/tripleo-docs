Deploying with SR-IOV Support
===============================

TripleO can deploy Overcloud nodes with SR-IOV support. A new role ``ComputeSriov``
has been added to create a custom ``roles_data.yaml`` with composable SR-IOV role.

Execute below command to create the ``roles_data.yaml``::

  openstack overcloud roles generate -o roles_data.yaml Controller ComputeSriov

Once a roles file is created, the following changes are required:

- Deploy Command
- Parameters
- Network Config

Deploy Command
----------------
Deploy command should include the generated roles data file from the above
command.

Deploy command should also include the SR-IOV environment file to include the
neutron-sriov-agent service. All the required parameters are also specified in
this environment file. The parameters has to be configured according to the
baremetal on which SR-IOV needs to be enabled.

Also, SR-IOV requires mandatory kernel parameters to be set, like
``intel_iommu=on iommu=pt`` on Intel machines. In order to enable the
configuration of kernel parametres to the host, host-config-pre-network
environment file has to be added for the deploy command.

Adding the following arguments to the ``openstack overcloud deploy`` command
will do the trick::

  openstack overcloud deploy --templates \
    -r roles_data.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/services/neutron-sriov.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/host-config-and-reboot.yaml \
    ...

Parameters
----------
Following are the list of parameters which need to be provided for deploying
with SR-IOV support.

* NovaPCIPassthrough: Provide the list of SR-IOV device names, the logical network,
  PCI addresses etc. The PF/VF devices matching the criteria would be available for
  guests.
* NeutronPhysicalDevMappings: The map of logical network name and the physical interface.


Example::

  parameter_defaults:
    NovaPCIPassthrough:
      - devname: "p7p1"
        physical_network: "sriov1_net"
      - devname: "p7p2"
        physical_network: "sriov2_net"
    NeutronPhysicalDevMappings: "sriov1_net:p7p1,sriov2_net:p7p2"


The parameter ``KernelArgs`` should be provided in the deployment environment
file, with the set of kernel boot parameters to be applied on the
``ComputeSriov`` role where SR-IOV is enabled::

  parameter_defaults:
    ComputeSriovParameters:
      KernelArgs: "intel_iommu=on iommu=pt"


Network Config
--------------
SR-IOV supported network interfaces should be specified in the network config
templates as sriov_pf type. This mechanism of configuring numvfs for SR-IOV
device is recommended and NeutronSriovNumVFs shall be avoided.

Example::

          network_config:
            - type: sriov_pf
              name: p7p2
              mtu: 9000
              numvfs: 10
              use_dhcp: false
              defroute: false
              nm_controlled: true
              promisc: false
