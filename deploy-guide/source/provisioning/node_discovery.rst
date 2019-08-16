Node Discovery
==============

As an alternative to creating an inventory file (``instackenv.json``) and
enrolling nodes from it, you can discover and enroll the nodes automatically.

TripleO supports two approaches to the discovery process:

* `Automatic enrollment of new nodes`_
* `Scanning BMC range`_

Automatic enrollment of new nodes
---------------------------------

You can enable **ironic-inspector** to automatically enroll all unknown nodes
that boot the introspection ramdisk. See `ironic-inspector discovery
documentation`_ for more details on the process.

Configuration
~~~~~~~~~~~~~

Set the following in your ``undercloud.conf`` before installing the undercloud:

.. code-block:: ini

    enable_node_discovery = True

Make sure to get (or build) and upload the introspection image, as described
in :doc:`../deployment/install_overcloud`.

Basic usage
~~~~~~~~~~~

After the discovery is enabled, any node that boots the introspection ramdisk
and posts back to **ironic-inspector** will be enrolled in **ironic**. Make
sure the nodes are connected to the provisioning network, and default to
booting from PXE. Power them on using any available means (e.g. by pushing the
power button on them).

New nodes appear in the ``enroll`` state by default and use the
``pxe_ipmitool`` driver (configurable via the ``discovery_default_driver``
option in ``undercloud.conf``). You have to set the power credentials
for these nodes and make them available. See :doc:`node_states` for details.

Using introspection rules
~~~~~~~~~~~~~~~~~~~~~~~~~

Alternatively, you can use **ironic-inspector** introspection rules to
automatically set the power credentials based on certain properties.

For example, to set the same credentials for all new nodes, you can use
the following rules:

.. code-block:: json

    [
        {
            "description": "Set default IPMI credentials",
            "conditions": [
                {"op": "eq", "field": "data://auto_discovered", "value": true}
            ],
            "actions": [
                {"action": "set-attribute", "path": "driver_info/ipmi_username",
                 "value": "admin"},
                {"action": "set-attribute", "path": "driver_info/ipmi_password",
                 "value": "paSSw0rd"}
            ]
        }
    ]

To set specific credentials for a certain vendor, use something like:

.. code-block:: json

    [
        {
            "description": "Set default IPMI credentials",
            "conditions": [
                {"op": "eq", "field": "data://auto_discovered", "value": true},
                {"op": "ne", "field": "data://inventory.system_vendor.manufacturer",
                 "value": "Dell Inc."}
            ],
            "actions": [
                {"action": "set-attribute", "path": "driver_info/ipmi_username",
                 "value": "admin"},
                {"action": "set-attribute", "path": "driver_info/ipmi_password",
                 "value": "paSSw0rd"}
            ]
        },
        {
            "description": "Set the vendor driver for Dell hardware",
            "conditions": [
                {"op": "eq", "field": "data://auto_discovered", "value": true},
                {"op": "eq", "field": "data://inventory.system_vendor.manufacturer",
                 "value": "Dell Inc."}
            ],
            "actions": [
                {"action": "set-attribute", "path": "driver", "value": "pxe_drac"},
                {"action": "set-attribute", "path": "driver_info/drac_username",
                 "value": "admin"},
                {"action": "set-attribute", "path": "driver_info/drac_password",
                 "value": "paSSw0rd"},
                {"action": "set-attribute", "path": "driver_info/drac_address",
                 "value": "{data[inventory][bmc_address]}"}
            ]
        }
    ]

The rules should be put to a file and uploaded to **ironic-inspector** before
the discovery process:

.. code-block:: console

    openstack baremetal introspection rule import /path/to/rules.json

See :doc:`profile_matching` for more examples on introspection rules.

.. _ironic-inspector discovery documentation: https://docs.openstack.org/ironic-inspector/usage.html#discovery

Scanning BMC range
------------------

You can discover new nodes by scanning an IP range for accessible BMCs.
You need to provide a set of credentials to try, and optionally a list of
ports. Use the following command to run the scan:

.. code-block:: console

    openstack overcloud node discover --range <RANGE> \
        --credentials <USER1:PASSWORD1> --credentials <USER2:PASSWORD2>

Here, ``<RANGE>`` is an IP range, e.g. ``10.0.0.0/24``. Credentials are
provided separated by a colon, e.g. ``root:calvin``.

With this approach, new nodes end up in ``manageable`` state, and will already
have the deploy properties, such as deploy kernel/ramdisk, assigned.

You can use the same command to introspect the nodes and make them available
for deployment:

.. code-block:: console

    openstack overcloud node discover --range <RANGE> \
        --credentials <USER1:PASSWORD1> --credentials <USER2:PASSWORD2> \
        --introspect --provide

The resulting node UUIDs will be printed on the screen.
