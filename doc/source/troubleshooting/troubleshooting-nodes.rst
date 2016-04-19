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

Starting with the Mitaka release, introspection for a node can be stopped with
the following command::

    openstack baremetal introspection abort <NODE UUID>

For older versions the recommended path is to wait until it times out.
Changing ``timeout`` setting in ``/etc/ironic-inspector/inspector.conf``
may be used to reduce this timeout from 1 hour (which usually too much,
especially on virtual environment).

If you do need to stop introspection **for all nodes** right now, do the
following for each node::

    ironic node-set-power-state UUID off

then remove ironic-inspector cache, rebuild it and restart ironic-inspector::

    rm /var/lib/ironic-inspector/inspector.sqlite
    sudo ironic-inspector-dbsync --config-file /etc/ironic-inspector/inspector.conf upgrade
    sudo systemctl restart openstack-ironic-inspector


.. _ironic-inspector troubleshooting documentation: http://docs.openstack.org/developer/ironic-inspector/troubleshooting.html
