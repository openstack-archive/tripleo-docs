Enable LVM2 filtering on overcloud nodes
========================================

While by default the overcloud image will not use LVM2 volumes, it is
possible that with some Cinder backends, for example remote iSCSI or FC,
the remote LUNs hosting OpenStack volumes will be visible on the nodes
hosting cinder-volume or nova-compute containers.

In that case, should the OpenStack guest create LVM2 volumes inside its
additional disks, those volumes will be scanned by the LVM2 tools
installed on the hosting node.

To prevent that, it is possible to configure an LVM2 global_filter when
deploying or updating the overcloud. The feature is, by default, disabled
and can be enabled passing `LVMFilterEnabled: true` in a Heat environment
file.

When enabled, a global_filter will be computed from the list of physical
devices hosting active LVM2 volumes. This list can be extended further,
manually, listing any additional block device via `LVMFilterAllowlist`
parameter, which supports regexp. A deny list can be configured as well,
via `LVMFilterDenylist` parameter; it defaults to ['.*'] so that any
block device which isn't in the allow list will be ignored by the LVM2
tools by default.

Any of the template parameters can be set per-role; for example, to enable
the feature only on Compute nodes and add `/dev/sdb` to the deny list use::

  $ cat ~/environment.yaml
  parameter_defaults:
    ComputeParameters:
      LVMFilterEnabled: true
      LVMFilterDenylist:
        - /dev/sdb

Then add the following argument to your `openstack overcloud deploy` command::

  -e environment.yaml
