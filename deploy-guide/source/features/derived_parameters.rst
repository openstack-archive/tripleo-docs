Derived Parameters
==================

TripleO can populate environment files with parameters which are
derived from formulas. Such formulas can take introspected hardware
data, workload type, or deployment type as input and return as output
a system tuning to be applied via parameter overrides. TripleO
supports this feature for both NFV (Network Function Virtualization)
and HCI (Hyper-converged Infrastructure; nodes with collocated Ceph
OSD and Nova Compute services) deployments.

Using derived parameters during a deployment
--------------------------------------------

To have TripleO derive parameters during deployment, specify an
alternative *deployment plan* containing directives which trigger
either a Mistral workflow (prior to Victoria) or an Ansible playbook
(in Victoria and newer) which derives the parameters.

A default *deployment plan* is created during deployment. This
deployment plan may be overridden by passing the ``-p`` or
``--plan-environment-file`` option to the ``openstack overcloud
deploy`` command. If the ``plan-environment-derived-params.yaml``
file, located in
``/usr/share/openstack-tripleo-heat-templates/plan-samples/``,
is specified this way, then the process to derive parameters,
via a playbook or workflow, will be executed.

The process is able to determine which roles have which features.
E.g. ComputeHCI roles need parameters derived from HCI formulas and
ComputeOvsDpdk roles need parameters derived from NFV formulas. The
deployment definition will indicate which hardware in the Ironic
database may be used and the process will extract the relevant
introspection data to use as input in the derivation. The output of
the process will be saved in the deployment plan and applied as
specific settings for different roles. If a role uses both NFV and HCI
features, then the process will apply NFV tuning first and then limit
the available HCI resources accordingly. E.g. if an NFV process is
pinning an entire CPU, then that CPU shouldn't be considered available
for use by a Ceph OSD.

Parameters which are derived for HCI deployments
------------------------------------------------

The derived parameters for HCI sets the NovaReservedHostMemory and
NovaCPUAllocationRatio per role based on the amount and type of Ceph
OSDs requested during deployment, the available hardware in Ironic,
and the average Nova guest workload.

Deriving the parameters is useful because in an HCI deployment the Nova
scheduler does not, by default, take into account the requirements of
the Ceph OSD services which are collocated with the Nova Compute
services. Thus, it's possible for Compute resources needed by an OSD
to be given instead to a Nova guest. To prevent this from happening,
the NovaReservedHostMemory and NovaCPUAllocationRatio may be adjusted
so that Nova reserves the resources that the Ceph OSDs need.

To perform well, each Ceph OSD requires 5 GB of memory and a certain
amount of vCPU (if hyperthreading is enabled). The faster the storage
medium the more vCPUs an OSD should use in order for the CPU resources
to not become a performance bottle-neck. All of this is taken into
account by the derived parameters for HCI.

The workload of the Nova guests may also be taken into account.
The ``plan-environment-derived-params.yaml`` file contains the
following::

    hci_profile: default
    hci_profile_config:
      default:
        average_guest_memory_size_in_mb: 0
        average_guest_cpu_utilization_percentage: 0
      many_small_vms:
        average_guest_memory_size_in_mb: 1024
        average_guest_cpu_utilization_percentage: 20
      few_large_vms:
        average_guest_memory_size_in_mb: 4096
        average_guest_cpu_utilization_percentage: 80
      nfv_default:
        average_guest_memory_size_in_mb: 8192
        average_guest_cpu_utilization_percentage: 90

The ``hci_profile_config`` describes the requirements for the average
Nova guest. The provided profiles (many_small_vms, few_large_vms, and
nfv_default) are not necessarily prescriptive but are examples that
may be adjusted based on your average workload. For example the
many_small_vms profile indicates that the average Nova guest uses a
flavor with 1 GB of RAM and that it is anticipated that the guest will
use on average 20% of it's available CPU. The ``hci_profile``
parameter should be set to a profile describing the expected workload.

In Victoria and newer the default profile sets the average memory and
CPU utilization to 0 because by default the workload is unknown. When
the ``tripleo_derive_hci_parameters`` Ansible module is passed these
values it sets the NovaReservedHostMemory to the number of OSDs
requested multiplied by 5 (for 5 GBs of RAM per OSD) but is not able
to take into account the memory overhead per guest for the hypervisor.
It also does not set the NovaCPUAllocationRatio. Thus, passing an
expected average workload will produce a more accurate set of derived
HCI parameters. However, this default does allow for a simpler
deployment where derived parameters may be used without having to
specify a workload but the OSDs are protected from having their memory
allocated to Nova guests.

Deriving HCI parameters before a deployment
-------------------------------------------

The ``tripleo_derive_hci_parameters`` Ansible module may be run
independently on the undercloud before deployment to generate a YAML
file to pass to the ``openstack overcloud deploy`` command with the
``-e`` option. If this option is used it's not necessary to derive HCI
parameters during deployment. Using this option also allows the
deployer to quickly see the values of the derived parameters.

.. warning::
   This playbook computes HCI parameters without running OVS-DPDK and
   SRIOV roles and may result in incorrect values in case OVS-DPDK and
   SRIOV are enabled.

On the undercloud within ``/usr/share/ansible/tripleo-playbooks/`` a
simple playbook ``derive-local-hci-parameters.yml`` is available
which calls the ``tripleo_derive_hci_parameters`` Ansible module. To
use the playbook before deployment determine the Ironic node UUID
which will correspond to the role being deployed. In the example below
a server with the name `ceph-2` has already been introspected and will
be used later during deployment for servers in the ComputeHCI role. We
will use this server's Ironic UUID so that the playbook gets its
introspection data::

  [stack@undercloud ~]$ baremetal node list | grep ceph-2
  | ef4cbd49-3773-4db2-80da-4210a7c24047 | ceph-2       | None          | power off   | available | False       |
  [stack@undercloud ~]$

Make a copy of the playbook in the stack users home directory and then
modify it to set the four playbook variables as below::

  [stack@undercloud ~]$ head derive-local-hci-parameters.yml
  ---
  - name: Derive HCI parameters before deployment
    hosts: localhost
    gather_facts: false
    vars:
      # Set the following variables for your environment
      ironic_node_id: ef4cbd49-3773-4db2-80da-4210a7c24047
      role: ComputeHCI
      average_guest_cpu_utilization_percentage: 50
      average_guest_memory_size_in_mb: 8192
      heat_environment_input_file: /home/stack/ceph_overrides.yaml
  [stack@undercloud ~]$

In the above example it is assumed the ``role`` `ComputeHCI` will use
nodes with the same type of hardware which is set to the
``ironic_node_id`` and that the average guest will use 50% of its CPU
and will use 8 GB of RAM. If the workload is unknown, remove these
variables. The system tuning will not be as accurate but the Ansible
module will at least set the NovaReservedHostMemory as a function of
the number of OSDs.

The ``heat_environment_input_file`` must be set to the path of the
Heat environment file which defines the OSDs.

.. admonition:: Victoria or earlier

  When ceph-ansible is used, in place of cephadm, this should be the
  file where the ``CephAnsibleDisksConfig`` parameter is set. This
  parameter is used to define which disks are used as Ceph OSDs and
  might look like the following if bluestore was being deployed on 4
  NVMe SSDs::

    parameter_defaults:
      CephAnsibleDisksConfig:
        osd_scenario: lvm
        osd_objectstore: bluestore
        osds_per_device: 4
        devices:
          - /dev/nvme0n1
          - /dev/nvme0n2
          - /dev/nvme0n3
          - /dev/nvme0n4

  The derived parameters workflow would use the values above to
  determine the number of OSDs requested (e.g. 4 devices * 4 OSDs per
  device = 16) and the type of device based on the Ironic data
  (e.g. during introspection, ironic can determine if a storage device
  is rotational).

If cephadm is used, in place of ceph-ansible (for Wallaby and newer),
then the ``heat_environment_input_file`` must be set to the path of
the file where the ``CephHciOsdCount`` and ``CephHciOsdType``
parameters are set.

The ``CephHciOsdCount`` and ``CephHciOsdType`` exist because
``CephOsdSpec``, as used by cephadm, might only specify a description
of devices to be used as OSDs (e.g. "all devices"), and not a list of
devices like ``CephAnsibleDisksConfig``, setting the count directly is
necessary in order to know how much CPU/RAM to reserve. Similarly,
because a device path is not hard coded, we cannot look up that device
in Ironic to determine its type. For information on the
``CephOsdSpec`` parameter see the :doc:`deployed_ceph` documentation.

``CephHciOsdType`` is the type of data_device (not db_device) used for
each OSD and must be one of hdd, ssd, or nvme. These are used by
the Ansible module tripleo_derive_hci_parameters.

``CephHciOsdCount`` is the number of expected Ceph OSDs per HCI
node. If a server has eight HDD drives, then the parameters should be
set like this::

  parameter_defaults:
    CephHciOsdType: hdd
    CephHciOsdCount: 8

To fully utilize nvme devices for data (not metadata), multiple
OSDs are required. If the ``CephOsdSpec`` parameter is used to set
`osds_per_device` to 4, and there are four NVMe drives on a host (and
no HDD drives), then the parameters should be set like this::

  parameter_defaults:
    CephHciOsdType: nvme
    CephHciOsdCount: 16

After these values are set run the playbook::

  [stack@undercloud ~]$ ansible-playbook derive-local-hci-parameters.yml
  [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit
  localhost does not match 'all'

  PLAY [Derive HCI parameters before deployment] ***********************************************

  TASK [Get baremetal inspection data] *********************************************************
  ok: [localhost]

  TASK [Get tripleo CephDisks environment parameters] *******************************************
  ok: [localhost]

  TASK [Derive HCI parameters] *****************************************************************
  changed: [localhost]

  TASK [Display steps on what to do next] ******************************************************
  ok: [localhost] => {
      "msg": "You may deploy your overcloud using -e /home/stack/hci_result.yaml so that the role ComputeHCI has its Nova configuration tuned to reserve CPU and Memory for its collocated Ceph OSDs. For an explanation see /home/stack/hci_report.txt."
  }

  PLAY RECAP ***********************************************************************************
  localhost                  : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

  [stack@undercloud ~]$

The playbook will generate two files in the stack user's home
directory unless the ``new_heat_environment_output_file`` and
``report_path`` variables are modified. The file denoted by the first
variable generated will be the derived parameters for the ``role``
specified. For example::

  [stack@undercloud ~]$ cat /home/stack/hci_result.yaml
  parameter_defaults:
    ComputeHCIParameters:
      NovaCPUAllocationRatio: 8.2
      NovaReservedHostMemory: 75000
  [stack@undercloud ~]$

The above could be used during a deployment by running a command like
``openstack overcloud deploy ... -e /home/stack/hci_result.yaml``.
The ``hci_result.yaml`` should be appended near the end of the
``openstack overcloud deploy`` command so that the derived values take
precedence.

The second file, defined by the ``report_path`` variable, will contain
an explanation of how the parameters were derived and what relevant
information was provided as input including the disks types as found
in Ironic. It might look like the following::

  [stack@undercloud ~]$ cat /home/stack/hci_report.txt
  Derived Parameters results
   Inputs:
   - Total host RAM in GB: 256
   - Total host vCPUs: 56
   - Ceph OSDs per host: 10
   - Average guest memory size in GB: 2
   - Average guest CPU utilization: 10%

   Outputs:
   - number of guests allowed based on memory = 90
   - number of guest vCPUs allowed = 460
   - nova.conf reserved_host_memory = 75000 MB
   - nova.conf cpu_allocation_ratio = 8.214286

  Compare "guest vCPUs allowed" to "guests allowed based on memory"
  for actual guest count.

  OSD type distribution:
    HDDs 10 | Non-NVMe SSDs 0 | NVMe SSDs 0
    vCPU to OSD ratio: 1
  [stack@undercloud ~]$


Verifying that HCI derived parameters have been applied
-------------------------------------------------------

If derived parameters were computed during deployment, then their
parameter override outputs may be found in the deployment plan.
Download the deployment plan for the stack, e.g. overcloud with a
command like the following::

  openstack overcloud plan export overcloud
  tar xf overcloud.tar.gz

Locate the ``plan-environment.yaml`` file and check if it contains the
the derived ``NovaCPUAllocationRatio`` and ``NovaReservedHostMemory``,
for example::

  $ head -5 plan-environment.yaml
  derived_parameters:
    ComputeHCIParameters:
      NovaCPUAllocationRatio: 8.2
      NovaReservedHostMemory: 75000
  description: 'Default Deployment plan'
  $

Regardless of if the parameters were derived before or during the
deployment, they should be applied to the overcloud. The following
example shows commands being executed on a node from the ComputeHCI
role and where expected Nova settings were applied::

  $ sudo podman exec -ti nova_compute /bin/bash
  # egrep 'reserved_host_memory_mb|cpu_allocation_ratio' /etc/nova/nova.conf
  reserved_host_memory_mb=75000
  cpu_allocation_ratio=8.2
  #
