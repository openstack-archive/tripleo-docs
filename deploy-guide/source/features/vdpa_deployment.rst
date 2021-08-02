Deploying with vDPA Support
===============================

TripleO can deploy Overcloud nodes with vDPA support. A new role ``ComputeVdpa``
has been added to create a custom ``roles_data.yaml`` with composable vDPA role.

vDPA is very similar to SR-IOV and leverages the same Openstack components. It's
important to note that vDPA can't function without OVS Hardware Offload.

Mellanox is the only NIC vendor currently supported with vDPA.

Execute below command to create the ``roles_data.yaml``::

  openstack overcloud roles generate -o roles_data.yaml Controller ComputeVdpa

Once a roles file is created, the following changes are required:

- Deploy Command
- Parameters
- Network Config
- Network and Port creation

Deploy Command
----------------
Deploy command should include the generated roles data file from the above
command.

Deploy command should also include the SR-IOV environment file to include the
``neutron-sriov-agent`` service. All the required parameters are also specified
in this environment file. The parameters has to be configured according to the
baremetal on which vDPA needs to be enabled.

Also, vDPA requires mandatory kernel parameters to be set, like
``intel_iommu=on iommu=pt`` on Intel machines. In order to enable the
configuration of kernel parametres to the host, The ``KernelArgs`` role
parameter has to be defined accordingly.

Adding the following arguments to the ``openstack overcloud deploy`` command
will do the trick::

  openstack overcloud deploy --templates \
    -r roles_data.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/services/neutron-sriov.yaml \
    ...

Parameters
----------

Unlike SR-IOV, vDPA devices shouldn't be added to ``NeutronPhysicalDevMappings`` but to the
``NovaPCIPassthrough``. The vDPA bridge should also be added to the ``NeutronBridgeMappings``
and the ``physical_network`` to the ``NeutronNetworkVLANRanges``.

The parameter ``KernelArgs`` should be provided in the deployment environment
file, with the set of kernel boot parameters to be applied on the
``ComputeVdpa`` role where vDPA is enabled.

The ``PciPassthroughFilter`` is required for vDPA. The ``NUMATopologyFilter`` will become
optional when ``libvirt`` will support the locking of the guest memory. At this time, it
is mandatory to have it::

  parameter_defaults:
    NeutronTunnelTypes: ''
    NeutronNetworkType: 'vlan'
    NeutronNetworkVLANRanges:
      - tenant:1300:1399
    NovaSchedulerDefaultFilters:
      - ...
      - PciPassthroughFilter
      - NUMATopologyFilter
    ComputeVdpaParameters:
      NovaPCIPassthrough:
        - vendor_id: "15b3"
          product_id: "101d"
          address: "06:00.0"
          physical_network: "tenant"
        - vendor_id: "15b3"
          product_id: "101d"
          address: "06:00.1"
          physical_network: "tenant"
      KernelArgs: "[...] iommu=pt intel_iommu=on"
      NeutronBridgeMappings:
        - tenant:br-tenant


Network Config
--------------
vDPA supported network interfaces should be specified in the network config
templates as sriov_pf type. It should also be under an OVS bridge with a ``link_mode``
set to ``switchdev``

Example::

      - type: ovs_bridge
        name: br-tenant
        members:
          - type: sriov_pf
            name: enp6s0f0
            numvfs: 8
            use_dhcp: false
            vdpa: true
            link_mode: switchdev
          - type: sriov_pf
            name: enp6s0f1
            numvfs: 8
            use_dhcp: false
            vdpa: true
            link_mode: switchdev


Network and Port Creation
-------------------------

When creating the network, it has to be mapped to the physical network::

  $ openstack network create \
      --provider-physical-network tenant \
      --provider-network-type vlan \
      --provider-segment 1337
      vdpa_net1

  $ openstack subnet create \
      --network vdpa_net1 \
      --subnet-range 192.0.2.0/24 \
      --dhcp
      vdpa_subnet1

To allocate a port from a vdpa-enabled NIC, create a neutron port and set the
``--vnic-type`` to ``vdpa``::

  $ openstack port create --network vdpa_net1 \
      --vnic-type=vdpa \
      vdpa_direct_port1

Scheduling instances
--------------------

Normally, the ``PciPassthroughFilter`` is sufficient to ensure that a vDPA instance will
land on a vDPA host. If we want to prevent other instances from using a vDPA host, we need
to setup the `isolate-aggreate feature
<https://docs.openstack.org/nova/latest/reference/isolate-aggregates.html>`_.

Example::

  $ openstack --os-placement-api-version 1.6 trait create CUSTOM_VDPA
  $ openstack aggregate create \
      --zone vdpa-az1 \
      vdpa_ag1
  $ openstack hypervisor list -c ID -c "Hypervisor Hostname" -f value | grep vdpa | \
    while read l
      do UUID=$(echo $l | cut -f 1 -d " ")
        H_NAME=$(echo $l | cut -f 2 -d " ")
        echo $H_NAME $UUID
        openstack aggregate add host vdpa_ag1 $H_NAME
        traits=$(openstack --os-placement-api-version 1.6 resource provider trait list \
                   -f value $UUID | sed 's/^/--trait /')
        openstack --os-placement-api-version 1.6 resource provider trait set \
          $traits --trait CUSTOM_VDPA $UUID
     done
  $ openstack --os-compute-api-version 2.53 aggregate set \
      --property trait:CUSTOM_VDPA=required \
      vdpa_ag1

The flavor will map to that new aggregate with the ``traits:CUSTOM_VDPA`` property::

  $ openstack --os-compute-api-version 2.86 flavor create \
      --ram 4096 \
      --disk 10 \
      --vcpus 2 \
      --property hw:cpu_policy=dedicated \
      --property hw:cpu_realtime=True \
      --property hw:cpu_realtime_mask=^0 \
      --property traits:CUSTOM_VDPA=required \
      vdpa_pinned

.. note::
    It's also important to have the ``hw:cpu_realtime*`` properties here since
    ``libvirt`` doesn't currently support the locking of guest memory.


This should launch an instance on one of the vDPA hosts::

  $ openstack server create \
      --image cirros \
      --flavor vdpa_pinned \
      --nic port-id=vdpa_direct_port1 \
      vdpa_test_1

Validations
-----------

Confirm that a PCI device is in switchdev mode::

  [root@computevdpa-0 ~]# devlink dev eswitch show pci/0000:06:00.0
  pci/0000:06:00.0: mode switchdev inline-mode none encap enable
  [root@computevdpa-0 ~]# devlink dev eswitch show pci/0000:06:00.1
  pci/0000:06:00.1: mode switchdev inline-mode none encap enable

Verify if offload is enabled in OVS::

  [root@computevdpa-0 ~]# ovs-vsctl get Open_vSwitch . other_config:hw-offload
  "true"

Validate the interfaces are added to the tenant bridge::

  [root@computevdpa-0 ~]# ovs-vsctl show
  be82eb5b-94c3-449d-98c8-0961b6b6b4c4
      Manager "ptcp:6640:127.0.0.1"
          is_connected: true
  [...]
    Bridge br-tenant
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port br-tenant
            Interface br-tenant
                type: internal
        Port enp6s0f0
            Interface enp6s0f0
        Port phy-br-tenant
            Interface phy-br-tenant
                type: patch
                options: {peer=int-br-tenant}
        Port enp6s0f1
            Interface enp6s0f1
  [...]


Verify if the NICs have ``hw-tc-offload`` enabled::

  [root@computevdpa-0 ~]# for i in {0..1};do ethtool -k enp6s0f$i | grep tc-offload;done
  hw-tc-offload: on
  hw-tc-offload: on

Verify that the udev rules have been created::

  [root@computevdpa-0 ~]# cat /etc/udev/rules.d/80-persistent-os-net-config.rules
  # This file is autogenerated by os-net-config
  SUBSYSTEM=="net", ACTION=="add", ATTR{phys_switch_id}!="", ATTR{phys_port_name}=="pf*vf*", ENV{NM_UNMANAGED}="1"
  SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", KERNELS=="0000:06:00.0", NAME="enp6s0f0"
  SUBSYSTEM=="net", ACTION=="add", ATTR{phys_switch_id}=="80ecee0003723f04", ATTR{phys_port_name}=="pf0vf*", IMPORT{program}="/etc/udev/rep-link-name.sh $attr{phys_port_name}", NAME="enp6s0f0_$env{NUMBER}"
  SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", KERNELS=="0000:06:00.1", NAME="enp6s0f1"
  SUBSYSTEM=="net", ACTION=="add", ATTR{phys_switch_id}=="80ecee0003723f04", ATTR{phys_port_name}=="pf1vf*", IMPORT{program}="/etc/udev/rep-link-name.sh $attr{phys_port_name}", NAME="enp6s0f1_$env{NUMBER}"


Validate that the ``numvfs`` are correctly defined::

  [root@computevdpa-0 ~]# cat /sys/class/net/enp6s0f0/device/sriov_numvfs
  8
  [root@computevdpa-0 ~]# cat /sys/class/net/enp6s0f1/device/sriov_numvfs
  8

Validate that the ``pci/passthrough_whitelist`` contains all the PFs::

  [root@computevdpa-0 ~]# grep ^passthrough_whitelist /var/lib/config-data/puppet-generated/nova_libvirt/etc/nova/nova.conf
  passthrough_whitelist={"address":"06:00.0","physical_network":"tenant","product_id":"101d","vendor_id":"15b3"}
  passthrough_whitelist={"address":"06:00.1","physical_network":"tenant","product_id":"101d","vendor_id":"15b3"}

Verify the ``nodedev-list`` from ``libvirt``::

  [root@computevdpa-0 ~]# podman exec -u0 nova_libvirt virsh nodedev-list | grep -P "pci_0000_06|enp6|vdpa"
  net_enp6s0f0_04_3f_72_ee_ec_80
  net_enp6s0f0_0_5a_86_bd_4b_06_d9
  net_enp6s0f0_1_72_b9_6b_12_33_57
  net_enp6s0f0_2_f6_f2_db_7c_52_90
  net_enp6s0f0_3_66_e5_9e_b8_79_7f
  net_enp6s0f0_4_32_04_6f_ef_ef_c3
  net_enp6s0f0_5_a2_fe_8d_4a_95_64
  net_enp6s0f0_6_8e_23_fa_bb_95_41
  net_enp6s0f0_7_8a_9f_0f_53_f6_19
  net_enp6s0f0v0_ee_a1_e2_4e_80_8d
  net_enp6s0f0v1_ce_b7_e1_33_33_56
  net_enp6s0f0v2_fe_91_a8_ee_2e_79
  net_enp6s0f0v3_2a_34_e0_a0_e6_ff
  net_enp6s0f0v4_26_59_82_da_65_4e
  net_enp6s0f0v5_a6_fd_db_97_c6_8a
  net_enp6s0f0v6_36_5d_5c_ff_e8_00
  net_enp6s0f0v7_4e_23_6c_95_b6_a4
  net_enp6s0f1_04_3f_72_ee_ec_81
  net_enp6s0f1_0_0e_0c_86_b5_43_c1
  net_enp6s0f1_1_be_f5_75_f4_da_b1
  net_enp6s0f1_2_ea_6a_21_37_91_24
  net_enp6s0f1_3_06_95_51_55_de_80
  net_enp6s0f1_4_86_a4_d5_83_bd_56
  net_enp6s0f1_5_86_d1_a9_ba_b7_f0
  net_enp6s0f1_6_82_ae_32_56_07_84
  net_enp6s0f1_7_62_b7_93_7e_5c_30
  net_enp6s0f1v0_b2_b3_0d_bd_6f_5d
  net_enp6s0f1v1_4a_24_a1_24_ae_39
  net_enp6s0f1v2_8e_19_b2_aa_ae_d7
  net_enp6s0f1v3_b6_e2_4b_fa_d8_f0
  net_enp6s0f1v4_5e_31_7f_17_ee_4d
  net_enp6s0f1v5_5e_77_99_09_1a_89
  net_enp6s0f1v6_96_68_4b_70_c5_1b
  net_enp6s0f1v7_c2_bb_14_95_81_29
  pci_0000_06_00_0
  pci_0000_06_00_1
  pci_0000_06_00_2
  pci_0000_06_00_3
  pci_0000_06_00_4
  pci_0000_06_00_5
  pci_0000_06_00_6
  pci_0000_06_00_7
  pci_0000_06_01_0
  pci_0000_06_01_1
  pci_0000_06_01_2
  pci_0000_06_01_3
  pci_0000_06_01_4
  pci_0000_06_01_5
  pci_0000_06_01_6
  pci_0000_06_01_7
  pci_0000_06_02_0
  pci_0000_06_02_1
  vdpa_vdpa0
  vdpa_vdpa1
  vdpa_vdpa10
  vdpa_vdpa11
  vdpa_vdpa12
  vdpa_vdpa13
  vdpa_vdpa14
  vdpa_vdpa15
  vdpa_vdpa2
  vdpa_vdpa3
  vdpa_vdpa4
  vdpa_vdpa5
  vdpa_vdpa6
  vdpa_vdpa7
  vdpa_vdpa8
  vdpa_vdpa9


Validate that the vDPA devices have been created, this should match the vdpa
devices from ``virsh nodedev-list``::

  [root@computevdpa-0 ~]# ls -tlra /dev/vhost-vdpa-*
  crw-------. 1 root root 241,  0 Jun 30 12:52 /dev/vhost-vdpa-0
  crw-------. 1 root root 241,  1 Jun 30 12:52 /dev/vhost-vdpa-1
  crw-------. 1 root root 241,  2 Jun 30 12:52 /dev/vhost-vdpa-2
  crw-------. 1 root root 241,  3 Jun 30 12:52 /dev/vhost-vdpa-3
  crw-------. 1 root root 241,  4 Jun 30 12:52 /dev/vhost-vdpa-4
  crw-------. 1 root root 241,  5 Jun 30 12:53 /dev/vhost-vdpa-5
  crw-------. 1 root root 241,  6 Jun 30 12:53 /dev/vhost-vdpa-6
  crw-------. 1 root root 241,  7 Jun 30 12:53 /dev/vhost-vdpa-7
  crw-------. 1 root root 241,  8 Jun 30 12:53 /dev/vhost-vdpa-8
  crw-------. 1 root root 241,  9 Jun 30 12:53 /dev/vhost-vdpa-9
  crw-------. 1 root root 241, 10 Jun 30 12:53 /dev/vhost-vdpa-10
  crw-------. 1 root root 241, 11 Jun 30 12:53 /dev/vhost-vdpa-11
  crw-------. 1 root root 241, 12 Jun 30 12:53 /dev/vhost-vdpa-12
  crw-------. 1 root root 241, 13 Jun 30 12:53 /dev/vhost-vdpa-13
  crw-------. 1 root root 241, 14 Jun 30 12:53 /dev/vhost-vdpa-14
  crw-------. 1 root root 241, 15 Jun 30 12:53 /dev/vhost-vdpa-15

Validate the ``pci_devices`` table in the database from one of the controllers::

  [root@controller-0 ~]# podman exec -u0 $(podman ps -q -f name=galera) mysql -t -D nova -e "select address,product_id,vendor_id,dev_type,dev_id from pci_devices where address like '0000:06:%';"
  +--------------+------------+-----------+----------+------------------+
  | address      | product_id | vendor_id | dev_type | dev_id           |
  +--------------+------------+-----------+----------+------------------+
  | 0000:06:00.0 | 101d       | 15b3      | vdpa     | pci_0000_06_00_0 |
  | 0000:06:00.1 | 101d       | 15b3      | vdpa     | pci_0000_06_00_1 |
  +--------------+------------+-----------+----------+------------------+


Other usefull commands for troubleshooting::

  [root@computevdpa-0 ~]# ovs-appctl dpctl/dump-flows -m type=offloaded
  [root@computevdpa-0 ~]# ovs-appctl dpctl/dump-flows -m
  [root@computevdpa-0 ~]# tc filter show dev enp6s0f1_1 ingress
  [root@computevdpa-0 ~]# tc -s filter show dev enp6s0f1_1 ingress
  [root@computevdpa-0 ~]# tc monitor
