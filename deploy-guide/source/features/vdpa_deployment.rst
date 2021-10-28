Deploying with vDPA Support
===============================

TripleO can deploy Overcloud nodes with vDPA support. A new role ``ComputeVdpa``
has been added to create a custom ``roles_data.yaml`` with composable vDPA role.

vDPA is very similar to SR-IOV and leverages the same Openstack components. It's
important to note that vDPA can't function without OVS Hardware Offload.

Mellanox is the only NIC vendor currently supported with vDPA.

CentOS9/RHEL9 with a kernel of 5.14 or higher is required.

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
      - PciPassthroughFilter
      - NUMATopologyFilter
      - ...
    ComputeVdpaParameters:
      NovaPCIPassthrough:
        - vendor_id: "15b3"
          product_id: "101e"
          address: "06:00.0"
          physical_network: "tenant"
        - vendor_id: "15b3"
          product_id: "101e"
          address: "06:00.1"
          physical_network: "tenant"
      KernelArgs: "[...] iommu=pt intel_iommu=on"
      NeutronBridgeMappings:
        - tenant:br-tenant

.. note::
    It's important to use the ``product_id`` of a VF device and not a PF

      06:00.1 Ethernet controller [0200]: Mellanox Technologies MT2892 Family [ConnectX-6 Dx] [15b3:101d]
      06:00.2 Ethernet controller [0200]: Mellanox Technologies ConnectX Family mlx5Gen Virtual Function [15b3:101e]




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
      --provider-segment 1337 \
      vdpa_net1

  $ openstack subnet create \
      --network vdpa_net1 \
      --subnet-range 192.0.2.0/24 \
      --dhcp \
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

The flavor will map to that new aggregate with the ``trait:CUSTOM_VDPA`` property::

  $ openstack --os-compute-api-version 2.86 flavor create \
      --ram 4096 \
      --disk 10 \
      --vcpus 2 \
      --property hw:cpu_policy=dedicated \
      --property hw:cpu_realtime=True \
      --property hw:cpu_realtime_mask=^0 \
      --property trait:CUSTOM_VDPA=required \
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
  pci/0000:06:00.0: mode switchdev inline-mode none encap-mode basic
  [root@computevdpa-0 ~]# devlink dev eswitch show pci/0000:06:00.1
  pci/0000:06:00.1: mode switchdev inline-mode none encap-mode basic

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

  [root@computevdpa-0 ~]# podman exec -u0 nova_virtqemud virsh -c qemu:///system nodedev-list | grep -P "pci_0000_06|enp6|vdpa"
  net_enp6s0f0np0_04_3f_72_ee_ec_84
  net_enp6s0f0np0_0_1a_c1_a5_25_94_ef
  net_enp6s0f0np0_1_3a_dc_1d_36_85_af
  net_enp6s0f0np0_2_6a_95_0c_e9_8f_1a
  net_enp6s0f0np0_3_ba_c8_5b_f5_70_cc
  net_enp6s0f0np0_4_9e_03_86_23_cd_65
  net_enp6s0f0np0_5_0a_5c_8b_c4_00_7a
  net_enp6s0f0np0_6_2e_f6_bc_e6_6f_cd
  net_enp6s0f0np0_7_ce_1e_b2_20_5e_15
  net_enp6s0f1np1_04_3f_72_ee_ec_85
  net_enp6s0f1np1_0_a6_04_9e_5a_cd_3b
  net_enp6s0f1np1_1_56_5d_59_b0_df_17
  net_enp6s0f1np1_2_de_ac_7c_3f_19_b1
  net_enp6s0f1np1_3_16_0c_8c_47_40_5c
  net_enp6s0f1np1_4_0e_a6_15_f5_68_77
  net_enp6s0f1np1_5_e2_73_dc_f9_c2_46
  net_enp6s0f1np1_6_e6_13_57_c9_cf_0f
  net_enp6s0f1np1_7_62_10_4f_2b_1b_ae
  net_vdpa06p00vf2_42_11_c8_97_aa_43
  net_vdpa06p00vf3_2a_59_5e_32_3e_b7
  net_vdpa06p00vf4_9a_5c_3f_c9_cc_42
  net_vdpa06p00vf5_26_73_2a_e3_db_f9
  net_vdpa06p00vf6_9a_bf_a9_e9_6b_06
  net_vdpa06p00vf7_d2_1f_cc_00_a9_95
  net_vdpa06p01vf0_ba_81_cb_7e_01_1d
  net_vdpa06p01vf1_56_95_fa_5e_4a_51
  net_vdpa06p01vf2_72_53_64_8d_12_98
  net_vdpa06p01vf3_9e_ff_1d_6d_c1_4e
  net_vdpa06p01vf4_96_20_f3_b1_69_ef
  net_vdpa06p01vf5_ea_0c_8b_0b_3f_ff
  net_vdpa06p01vf6_0a_53_4e_94_e0_8b
  net_vdpa06p01vf7_16_84_48_e6_74_59
  net_vdpa06p02vf0_b2_cc_fa_16_f0_52
  net_vdpa06p02vf1_0a_12_1b_a2_1a_d3
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
  vdpa_0000_06_00_2
  vdpa_0000_06_00_3
  vdpa_0000_06_00_4
  vdpa_0000_06_00_5
  vdpa_0000_06_00_6
  vdpa_0000_06_00_7
  vdpa_0000_06_01_0
  vdpa_0000_06_01_1
  vdpa_0000_06_01_2
  vdpa_0000_06_01_3
  vdpa_0000_06_01_4
  vdpa_0000_06_01_5
  vdpa_0000_06_01_6
  vdpa_0000_06_01_7
  vdpa_0000_06_02_0
  vdpa_0000_06_02_1


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

  [root@controller-2 neutron]# podman exec -u0 $(podman ps -q -f name=galera) mysql -t -D nova -e "select address,product_id,vendor_id,dev_type,dev_id from pci_devices where address like '0000:06:%' and deleted=0;"
  +--------------+------------+-----------+----------+------------------+
  | address      | product_id | vendor_id | dev_type | dev_id           |
  +--------------+------------+-----------+----------+------------------+
  | 0000:06:01.1 | 101e       | 15b3      | vdpa     | pci_0000_06_01_1 |
  | 0000:06:00.2 | 101e       | 15b3      | vdpa     | pci_0000_06_00_2 |
  | 0000:06:00.3 | 101e       | 15b3      | vdpa     | pci_0000_06_00_3 |
  | 0000:06:00.4 | 101e       | 15b3      | vdpa     | pci_0000_06_00_4 |
  | 0000:06:00.5 | 101e       | 15b3      | vdpa     | pci_0000_06_00_5 |
  | 0000:06:00.6 | 101e       | 15b3      | vdpa     | pci_0000_06_00_6 |
  | 0000:06:00.7 | 101e       | 15b3      | vdpa     | pci_0000_06_00_7 |
  | 0000:06:01.0 | 101e       | 15b3      | vdpa     | pci_0000_06_01_0 |
  | 0000:06:01.2 | 101e       | 15b3      | vdpa     | pci_0000_06_01_2 |
  | 0000:06:01.3 | 101e       | 15b3      | vdpa     | pci_0000_06_01_3 |
  | 0000:06:01.4 | 101e       | 15b3      | vdpa     | pci_0000_06_01_4 |
  | 0000:06:01.5 | 101e       | 15b3      | vdpa     | pci_0000_06_01_5 |
  | 0000:06:01.6 | 101e       | 15b3      | vdpa     | pci_0000_06_01_6 |
  | 0000:06:01.7 | 101e       | 15b3      | vdpa     | pci_0000_06_01_7 |
  | 0000:06:02.0 | 101e       | 15b3      | vdpa     | pci_0000_06_02_0 |
  | 0000:06:02.1 | 101e       | 15b3      | vdpa     | pci_0000_06_02_1 |
  | 0000:06:00.2 | 101e       | 15b3      | vdpa     | pci_0000_06_00_2 |
  | 0000:06:00.3 | 101e       | 15b3      | vdpa     | pci_0000_06_00_3 |
  | 0000:06:00.4 | 101e       | 15b3      | vdpa     | pci_0000_06_00_4 |
  | 0000:06:00.5 | 101e       | 15b3      | vdpa     | pci_0000_06_00_5 |
  | 0000:06:00.6 | 101e       | 15b3      | vdpa     | pci_0000_06_00_6 |
  | 0000:06:00.7 | 101e       | 15b3      | vdpa     | pci_0000_06_00_7 |
  | 0000:06:01.0 | 101e       | 15b3      | vdpa     | pci_0000_06_01_0 |
  | 0000:06:01.1 | 101e       | 15b3      | vdpa     | pci_0000_06_01_1 |
  | 0000:06:01.2 | 101e       | 15b3      | vdpa     | pci_0000_06_01_2 |
  | 0000:06:01.3 | 101e       | 15b3      | vdpa     | pci_0000_06_01_3 |
  | 0000:06:01.4 | 101e       | 15b3      | vdpa     | pci_0000_06_01_4 |
  | 0000:06:01.5 | 101e       | 15b3      | vdpa     | pci_0000_06_01_5 |
  | 0000:06:01.6 | 101e       | 15b3      | vdpa     | pci_0000_06_01_6 |
  | 0000:06:01.7 | 101e       | 15b3      | vdpa     | pci_0000_06_01_7 |
  | 0000:06:02.0 | 101e       | 15b3      | vdpa     | pci_0000_06_02_0 |
  | 0000:06:02.1 | 101e       | 15b3      | vdpa     | pci_0000_06_02_1 |
  +--------------+------------+-----------+----------+------------------+

The ``vdpa`` command::

  [root@computevdpa-0 ~]# vdpa dev
  0000:06:01.0: type network mgmtdev pci/0000:06:01.0 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:00.6: type network mgmtdev pci/0000:06:00.6 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:00.4: type network mgmtdev pci/0000:06:00.4 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:00.2: type network mgmtdev pci/0000:06:00.2 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:01.1: type network mgmtdev pci/0000:06:01.1 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:00.7: type network mgmtdev pci/0000:06:00.7 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:00.5: type network mgmtdev pci/0000:06:00.5 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:00.3: type network mgmtdev pci/0000:06:00.3 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:02.0: type network mgmtdev pci/0000:06:02.0 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:01.6: type network mgmtdev pci/0000:06:01.6 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:01.4: type network mgmtdev pci/0000:06:01.4 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:01.2: type network mgmtdev pci/0000:06:01.2 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:02.1: type network mgmtdev pci/0000:06:02.1 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:01.7: type network mgmtdev pci/0000:06:01.7 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:01.5: type network mgmtdev pci/0000:06:01.5 vendor_id 5555 max_vqs 16 max_vq_size 256
  0000:06:01.3: type network mgmtdev pci/0000:06:01.3 vendor_id 5555 max_vqs 16 max_vq_size 256

Validating the OVN agents::

  (overcloud) [stack@undercloud-0 ~]$ openstack network agent list --host computevdpa-0.home.arpa
  +--------------------------------------+----------------------+-------------------------+-------------------+-------+-------+----------------------------+
  | ID                                   | Agent Type           | Host                    | Availability Zone | Alive | State | Binary                     |
  +--------------------------------------+----------------------+-------------------------+-------------------+-------+-------+----------------------------+
  | ef2e6ced-e723-449c-bbf8-7513709f33ea | OVN Controller agent | computevdpa-0.home.arpa |                   | :-)   | UP    | ovn-controller             |
  | 7be39049-db5b-54fc-add1-4a0687160542 | OVN Metadata agent   | computevdpa-0.home.arpa |                   | :-)   | UP    | neutron-ovn-metadata-agent |
  +--------------------------------------+----------------------+-------------------------+-------------------+-------+-------+----------------------------+


Other usefull commands for troubleshooting::

  [root@computevdpa-0 ~]# ovs-appctl dpctl/dump-flows -m type=offloaded
  [root@computevdpa-0 ~]# ovs-appctl dpctl/dump-flows -m
  [root@computevdpa-0 ~]# tc filter show dev enp6s0f1_1 ingress
  [root@computevdpa-0 ~]# tc -s filter show dev enp6s0f1_1 ingress
  [root@computevdpa-0 ~]# tc monitor
