Troubleshooting Node Management Failures
========================================

Where Are the Logs?
-------------------

Some logs are stored in *journald*, but most are stored as text files in
``/var/log``. They are only accessible by the root user.

ironic-inspector
~~~~~~~~~~~~~~~~

The introspection logs (from ironic-inspector) are located in
``/var/log/ironic-inspector``. If something fails during the introspection
ramdisk run, ironic-inspector stores the ramdisk logs in
``/var/log/ironic-inspector/ramdisk/`` as gz-compressed tar files.
File names contain date, time and IPMI address of the node if it was detected
(only for bare metal).

To collect introspection logs on success as well, set
``always_store_ramdisk_logs = true`` in
``/etc/ironic-inspector/inspector.conf``, restart the
``openstack-ironic-inspector`` service and retry the introspection.

.. _ironic_logs:

ironic
~~~~~~

The deployment logs (from ironic) are located in ``/var/log/ironic``. If
something goes wrong during deployment or cleaning, the ramdisk logs are
stored in ``/var/log/ironic/deploy``. See `ironic logs retrieving documentation
<https://docs.openstack.org/ironic/latest/admin/troubleshooting.html#retrieving-logs-from-the-deploy-ramdisk>`_
for more details.

.. _node_registration_problems:

Node Registration and Management Problems
-----------------------------------------

Nodes in enroll state after registration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you see your nodes staying in the ``enroll`` provision state after the
registration process (which may hang due to this), it means that Ironic is
unable to verify power management credentials, and you need to fix them.
Check the ``pm_addr``, ``pm_user`` and ``pm_password`` fields in your
``instackenv.json``. In some cases (e.g. when using
:doc:`../environments/virtualbmc`) you also need a correct ``pm_port``.
Update the node as explained in `Fixing invalid node information`_.

Fixing invalid node information
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Any problems with node data registered into Ironic can be fixed using the
Ironic CLI.

For example, a wrong MAC can be fixed in two steps:

* Find out the assigned port UUID by running
  ::

    openstack baremetal port list --node <NODE UUID>

* Update the MAC address by running
  ::

    openstack baremetal port set --address <NEW MAC> <PORT UUID>

A Wrong IPMI address can be fixed with the following command::

    openstack baremetal node set <NODE UUID> --driver-info ipmi_address=<NEW IPMI ADDRESS>

Node power state is not enforced by Ironic
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default Ironic will not forcibly sync the power state of the nodes,
because in our HA (high availability) model Pacemaker is the
one responsible for controlling the power state of the nodes
when fencing.  If you are using a non-HA setup and want Ironic
to take care of the power state of the nodes please change the
value of the ``force_power_state_during_sync`` configuration option
in the ``/etc/ironic/ironic.conf`` file to ``True`` and restart the
openstack-ironic-conductor service.

Also, note that if ``openstack undercloud install`` is re-run the value of
the ``force_power_state_during_sync`` configuration option will be set back to
the default, which is ``False``.

How do I repair broken nodes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Usually, the nodes should only be deleted when the hardware is decommissioned.
Before that, you're expected to remove instances from them using scale-down.
However, in some cases, it may be impossible to repair a node with e.g. broken
power management, and it gets stuck in an abnormal state.

.. warning::
    Before proceeding with this section, always try to decommission a node
    normally, by scaling down your cloud. Forcing node deletion may cause
    unpredicable results.

Ironic requires that nodes that cannot be operated normally are put in the
maintenance mode. It is done by the following command::

    openstack baremetal node maintenance set <NODE UUID> --reason "<EXPLANATION>"

Ironic will stop checking power and health state for such nodes, and Nova will
not pick them for deployment. Power command will still work on them, though.

After a node is in the maintenance mode, you can attempt repairing it, e.g. by
`Fixing invalid node information`_. If you manage to make the node operational
again, move it out of the maintenance mode::

    openstack baremetal node maintenance unset <NODE UUID>

If repairing is not possible, you can force deletion of such node::

    openstack baremetal node delete <NODE UUID>

Forcing node removal will leave it powered on, accessing the network with
the old IP address(es) and with all services running. Before proceeding, make
sure to power it off and clean up via any means.

After that, the associated Nova instance is orphaned, and must be deleted.
You can do it normally via the scale down procedure.

.. _introspection_problems:

Hardware Introspection Problems
-------------------------------

Introspection hangs and times out
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

ironic-inspector times out introspection process after some time (defaulting to
1 hour) if it never gets response from the introspection ramdisk.  This can be
a sign of a bug in the introspection ramdisk, but usually it happens due to
environment misconfiguration, particularly BIOS boot settings. Please refer to
`ironic-inspector troubleshooting documentation
<https://docs.openstack.org/ironic-inspector/latest/user/troubleshooting.html>`_
for information on how to detect and fix such problems.

Accessing the ramdisk
~~~~~~~~~~~~~~~~~~~~~

Note that the introspection ramdisk is by default built with the
`dynamic-login element
<http://docs.openstack.org/developer/diskimage-builder/elements/dynamic-login/README.html>`_,
so you can set up an SSH key and log into it for debugging.

First, think of a temporary root password. Generate a hash by feeding it
into ``openssl passwd -1`` command. Edit ``/httpboot/inspector.ipxe``
manually. Find the line starting with "kernel" and append rootpwd="HASH" to it.
Do not append the real password. Alternatively, you can append
sshkey="PUBLIC_SSH_KEY" with your public SSH key.

.. warning::
    In both cases quotation marks are required!

When ramdisk is running, figure out its IP address by checking ``arp`` utility
or DHCP logs from

::

    sudo journalctl -u openstack-ironic-inspector-dnsmasq

SSH as a root user with the temporary password or the SSH key.

.. note::
    Some operating systems, such as RHEL and CentOS, require SELinux to be in permissive or disabled
    mode so that you can log in to the image. This is achieved by building the
    image with the selinux-permissive element for diskimage-builder or by
    passing selinux=0 in the kernel command line.

Refusing to introspect node with provision state "available"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you're running introspection directly using ironic-inspector CLI (or in case
of bugs in our scripts), a node can be in the "AVAILABLE" state, which is meant
for deployment, not for introspection. You should advance node to the
"MANAGEABLE" state before introspection and move it back before deployment.
Please refer to `upstream node states documentation
<https://docs.openstack.org/ironic-inspector/latest/user/usage.html#node-states>`_
for information on how to fix it.

How can introspection be stopped?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Introspection for a node can be stopped with the following command::

    openstack baremetal introspection abort <NODE UUID>

