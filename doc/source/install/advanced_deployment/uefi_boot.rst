Booting in UEFI mode
====================

TripleO supports booting overcloud nodes in UEFI_ mode instead of the default
BIOS mode. This is required to use advanced features like *secure boot* (not
covered by this guide), and some hardware may only feature UEFI support.

Configuring nodes
-----------------

Depending on the driver, nodes have to be put in the UEFI mode manually or the
driver can put them in it. For example, manual configuration is required for
``ipmi`` (including ``pxe_ipmitool``) and ``idrac`` (including ``pxe_drac``)
drivers, while ``ilo`` (including ``pxe_ilo``) and ``irmc`` (starting with
the Queens release) drivers can set boot mode automatically.

Independent of the driver, you have to configure the UEFI mode manually, if
you want introspection to run in it.

Manual configuration is usually done by entering node's *system setup* and
changing boot setting there.

Introspection
-------------

The introspection process is flexible enough to automatically detect the boot
mode of the node. The only requirement is iPXE: TripleO currently does not
support using PXE with UEFI. Make sure the following options are enabled
in your ``undercloud.conf`` (they are on by default):

.. code-block:: ini

    ipxe_enabled = True
    inspection_enable_uefi = True

Then you can run introspection as usual.

Deployment
----------

Starting with the Pike release, the introspection process configures bare
metal nodes to run in the same boot mode as it was run in. For example, if
introspection was run on nodes in UEFI mode, **ironic-inspector** will
configure introspected nodes to deploy in UEFI mode as well.

Here is how the ``properties`` field looks for nodes configured in BIOS mode::

    $ openstack baremetal node show <NODE> -f value -c properties
    {u'capabilities': u'profile:compute,boot_option:local,boot_mode:bios', u'memory_mb': u'6144', u'cpu_arch': u'x86_64', u'local_gb': u'49', u'cpus': u'1'}

Note that ``boot_mode:bios`` capability is set. For a node in UEFI mode, it
will look like this::

    $ openstack baremetal node show <NODE> -f value -c properties
    {u'capabilities': u'profile:compute,boot_option:local,boot_mode:uefi', u'memory_mb': u'6144', u'cpu_arch': u'x86_64', u'local_gb': u'49', u'cpus': u'1'}

You can change the boot mode with the following command (required for UEFI
before the Pike release)::

    $ openstack baremetal node set <NODE> --property capabilities=profile:compute,boot_option:local,boot_mode:uefi

.. warning::
    Do not forget to copy all other capabilities, particularly ``profile`` and
    ``boot_option``, literally.

Finally, you may configure your flavors to explicitly request nodes that boot
in UEFI mode, for example::

    $ openstack flavor set --property capabilities:boot_mode='uefi' compute

Then proceed with the deployment as usual.

.. _UEFI: https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface
