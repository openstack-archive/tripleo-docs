Ready-state configuration
=========================

.. note:: Ready-state configuration currently works only with Dell DRAC
          machines.

Ready-state configuration can be used to prepare bare-metal resources for
deployment. It includes BIOS configuration based on a predefined profile.


Define the target BIOS configuration
------------------------------------

To define a BIOS setting, list the name of the setting and its target
value for each profile::

    {
        "compute" :{
            "bios_settings": {"ProcVirtualization": "Enabled"}
        }
    }


Trigger the ready-state configuration
-------------------------------------

Make sure the nodes have profiles assigned as described in
:doc:`profile_matching`. Create a JSON file with the target ready-state
configuration for each profile. Then trigger the configuration::

    baremetal configure ready state ready-state.json
