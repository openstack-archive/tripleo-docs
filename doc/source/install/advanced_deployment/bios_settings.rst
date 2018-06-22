BIOS Settings
=============

Tripleo can support BIOS configuration for bare metal nodes via node manual
:doc:`cleaning`. Several commands are added to allow administrator to apply
and reset BIOS settings.

Apply BIOS settings
-------------------

#. To apply given BIOS configuration to all manageable nodes::

    openstack overcloud node bios configure --configuration <..> --all-manageable

#. To apply given BIOS configuration to specified nodes::

    openstack overcloud node bios configure --configuration <..> node_uuid1 node_uuid2 ..

The configuration parameter passed to above commands must be YAML/JSON string
or a file name which contains YAML/JSON string of BIOS settings, for example::

    {
      "settings": [
        {
          "name": "setting name",
          "value": "setting value"
        },
        {
          "name": "setting name",
          "value": "setting value"
        },
        ..
      ]
    }

With the parameter ``--all-manageable``, the command applies given BIOS
settings to all manageable nodes.

With the parameter ``node_uuid1 node_uuid2``, the command applies given BIOS
settings to nodes which uuid equal to ``node_uuid1`` and ``node_uuid2``.

Reset BIOS settings
-------------------

#. To reset the BIOS configuration to factory default on specified nodes::

    openstack overcloud node bios reset --all-manageable

#. To reset the BIOS configuration on specified nodes::

    openstack overcloud node bios reset node_uuid1 node_uuid2 ..
