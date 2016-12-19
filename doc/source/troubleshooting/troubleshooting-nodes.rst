Troubleshooting Node Management Failures
----------------------------------------

Where Are the Logs?
^^^^^^^^^^^^^^^^^^^

Some logs are stored in *journald*, but most are stored as text files in
``/var/log``.  Ironic and ironic-inspector logs are stored in journald. Note
that Ironic has 2 units: ``openstack-ironic-api`` and
``openstack-ironic-conductor``. Similarly, ironic-inspector has
``openstack-ironic-inspector`` and ``openstack-ironic-inspector-dnsmasq``.  So
for example to get all ironic-inspector logs use::

    sudo journalctl -u openstack-ironic-inspector -u openstack-ironic-inspector-dnsmasq

If something fails during the introspection ramdisk run, ironic-inspector
stores the ramdisk logs in ``/var/log/ironic-inspector/ramdisk/`` as
gz-compressed tar files. File names contain date, time and IPMI address of the
node if it was detected (only for bare metal).

.. _node_registration_problems:

Node Registration Problems
^^^^^^^^^^^^^^^^^^^^^^^^^^

Any problems with node data registered into Ironic can be fixed using the
Ironic CLI.

For example, a wrong MAC can be fixed in two steps:

* Find out the assigned port UUID by running
  ::

    ironic node-port-list <NODE UUID>

* Update the MAC address by running
  ::

    ironic port-update <PORT UUID> replace address=<NEW MAC>

A Wrong IPMI address can be fixed with the following command::

    ironic node-update <NODE UUID> replace driver_info/ipmi_address=<NEW IPMI ADDRESS>

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


.. _introspection_problems:

Hardware Introspection Problems
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Introspection hangs and times out
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ironic-inspector times out introspection process after some time (defaulting to
1 hour) if it never gets response from the introspection ramdisk.  This can be
a sign of a bug in the introspection ramdisk, but usually it happens due to
environment misconfiguration, particularly BIOS boot settings. Please refer to
`ironic-inspector troubleshooting documentation`_ for information on how to
detect and fix such problems.

Accessing the ramdisk
~~~~~~~~~~~~~~~~~~~~~

Note that the introspection ramdisk is by default built with `the
dynamic-login element`_, so you can set up an SSH key and log into it for
debugging.

First, think of a temporary root password. Generate a hash by feeding it
into ``openssl passwd -1`` command. Edit ``/httpboot/inspector.ipxe``
manually. Find the line starting with "kernel" and append rootpwd="HASH" to it.
Do not append the real password. Alternatively, you can append
sshkey="PUBLIC_SSH_KEY" with your public SSH key.

.. note::
    In both cases quotation marks are required!

When ramdisk is running, figure out its IP address by checking ``arp`` utility
or DHCP logs from

::

    sudo journalctl -u openstack-ironic-inspector-dnsmasq

SSH as a root user with the temporary password or the SSH key.

Accessing logs from the ramdisk
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Introspection logs are saved on ramdisk failures. Starting with the Newton
release, they are actually stored on all introspection failures. The standard
location is ``/var/log/ironic-inspector/ramdisk``, and the files there are
actually ``tar.gz`` without an extension.

To collect introspection logs in other cases, set
``always_store_ramdisk_logs = true`` in
``/etc/ironic-inspector/inspector.conf``, restart the
``openstack-ironic-inspector`` service and retry the introspection.

Refusing to introspect node with provision state "available"
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you're running introspection directly using ironic-inspector CLI (or in case
of bugs in our scripts), a node can be in the "AVAILABLE" state, which is meant
for deployment, not for introspection. You should advance node to the
"MANAGEABLE" state before introspection and move it back before deployment.
Please refer to `upstream node states documentation
<http://docs.openstack.org/developer/ironic-inspector/usage.html#node-states>`_
for information on how to fix it.

How can introspection be stopped?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Introspection for a node can be stopped with the following command::

    openstack baremetal introspection abort <NODE UUID>

