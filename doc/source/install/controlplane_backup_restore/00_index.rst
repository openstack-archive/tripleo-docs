TripleO backup and restore (Undercloud and Overcloud control plane)
===================================================================

This documentation section will describe a method to backup and restore both Undercloud and Overcloud
control plane.

The use case involved in the creation and restore of these procedures are related to the
possible failures of a minor update or major upgrade for both Undercloud and Overcloud.

The general approach to recover from failures during the minor update or major upgrade workflow
is to fix the environment and restart services before re-running the last executed step.

There are specific cases in which rolling back to previous steps in the upgrades
workflow can lead to general  failures in the system, i.e.
when executing `yum history` to rollback the upgrade of certain packages,
the dependencies resolution might select to remove critical packages like `systemd`.

.. toctree::
  :maxdepth: 2
  :includehidden:

  01_undercloud_backup
  02_overcloud_backup
  03_undercloud_restore
  04_overcloud_restore
