Advanced Profile Matching
=========================

Profile matching allows a user to specify precisely which nodes will receive
which flavor. Here are additional setup steps to take advantage of the profile
matching. In this document "profile" is a capability that is assigned to both
ironic node and nova flavor to create a link between them.

After profile is assigned to a flavor, nova will only deploy it on ironic
nodes with the same profile. Deployment will fail if not enough ironic nodes
are tagged with a profile.

There are two ways to assign a profile to a node. You can assign it directly
or specify one or many suitable profiles for the deployment command to choose
from. It can be done either manually or using the introspection rules.

.. note::
    Do not miss the "boot_option" part from any commands below,
    otherwise your deployment won't work as expected.

Create flavors to use profile matching
--------------------------------------

Default profile flavors should have been created when the undercloud was
installed, and they will be usable without modification in most environments.
However, if custom profile flavors are needed, they can be created as follows.

In order to use the profiles assigned to the Ironic nodes, Nova needs to have
flavors that have the property ``capabilities:profile`` set to the intended
profile.

For example, with just the compute and control profiles:

* Create the flavors::

    openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 control
    openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 compute

.. note::
    The values for ram, disk, and vcpus should be set to a minimal lower bound,
    as Nova will still check that the Ironic nodes have at least this much.

* Assign the properties::

    openstack flavor set --property "cpu_arch"="x86_64" --property "capabilities:boot_option"="local" --property "capabilities:profile"="compute" compute
    openstack flavor set --property "cpu_arch"="x86_64" --property "capabilities:boot_option"="local" --property "capabilities:profile"="control" control

Manual profile tagging
----------------------

To assign a profile to a node directly, issue the following command::

    ironic node-update <UUID OR NAME> replace properties/capabilities=profile:<PROFILE>,boot_option:local

Alternatively, you can provide a number of profiles as capabilities in form of
``<PROFILE>_profile:1``, which later can be automatically converted to one
assigned profile (see `Use the flavors to deploy`_ for details). For example::

    ironic node-update <UUID OR NAME> replace properties/capabilities=compute_profile:1,control_profile:1,boot_option:local

Finally, to clean all profile information from an available nodes, use::

    ironic node-update <UUID OR NAME> replace properties/capabilities=boot_option:local

.. note::
    We can not update only a single key from the capabilities dictionary, so we
    need to specify both the profile and the boot_option above. Otherwise, the
    boot_option key will get removed.

Also see :ref:`instackenv` for details on how to set profile in the
``instackenv.json`` file.

.. _auto-profile-tagging:

Automated profile tagging
-------------------------

`Introspection rules`_ can be used to conduct automatic profile assignment
based on data received from the introspection ramdisk. A set of introspection
rules should be created before introspection that either set ``profile`` or
``<PROFILE>_profile`` capabilities on a node.

The exact structure of data received from the ramdisk depends on both ramdisk
implementation and enabled plugins, and on enabled *ironic-inspector*
processing hooks. The most basic properties are ``cpus``, ``cpu_arch``,
``local_gb`` and ``memory_mb``, which represent CPU number, architecture,
local hard drive size in GiB and RAM size in MiB. See
:ref:`introspection_data` for more details on what our current ramdisk
provides.

For example, imagine we have the following hardware: with disk sizes > 1 TiB
for object storage and with smaller disks for compute and controller nodes.
We also need to make sure that no hardware with seriously insufficient
properties gets to the fleet at all.

Create a JSON file, for example ``rules.json``, with the following contents::

    [
        {
            "description": "Fail introspection for unexpected nodes",
            "conditions": [
                {"op": "lt", "field": "memory_mb", "value": 4096}
            ],
            "actions": [
                {"action": "fail", "message": "Memory too low, expected at least 4 GiB"}
            ]
        },
        {
            "description": "Assign profile for object storage",
            "conditions": [
                {"op": "ge", "field": "local_gb", "value": 1024}
            ],
            "actions": [
                {"action": "set-capability", "name": "profile", "value": "swift"}
            ]
        },
        {
            "description": "Assign possible profiles for compute and controller",
            "conditions": [
                {"op": "lt", "field": "local_gb", "value": 1024},
                {"op": "ge", "field": "local_gb", "value": 40}
            ],
            "actions": [
                {"action": "set-capability", "name": "compute_profile", "value": "1"},
                {"action": "set-capability", "name": "control_profile", "value": "1"}
            ]
        }
    ]

.. note::
    This example may need to be adjusted to work on a virtual environment.

Before introspection load this file into *ironic-inspector*::

    openstack baremetal introspection rule import /path/to/rules.json

Then (re)start the introspection. Check assigned profiles or possible profiles
using command::

    openstack overcloud profiles list

If you've made a mistake in introspection rules, you can delete them all::

    openstack baremetal introspection rule purge

Then reupload the updated rules file and restart introspection.

Use the flavors to deploy
-------------------------

By default, all nodes are deployed to the **baremetal** flavor.
To use profile matching you have to `Create flavors to use profile matching`_
first, then use specific flavors for deployment. For each node role set
``--ROLE-flavor`` to the name of the flavor and ``--ROLE-scale`` to the number
of nodes you want to end up with for this role.

After profiles and possible profiles are tagged either manually or during
the introspection, we need to turn possible profiles into an appropriate
number of profiles and validate the result. Continuing with the example with
only control and compute profiles::

    openstack overcloud profiles match --control-flavor control --control-scale 1 --compute-flavor compute --compute-scale 1

* This command first tries to find enough nodes with ``profile`` capability.

* If there are not enough such nodes, it then looks at available nodes with
  ``PROFILE_profile`` capabilities. If enough of such nodes is found, then
  their ``profile`` capabilities are updated to make the choice permanent.

This command should exit without errors (and optionally without warnings).

You can see the resulting profiles in the node list provided by

::

    $ openstack overcloud profiles list
    +--------------------------------------+-----------+-----------------+-----------------+-------------------+
    | Node UUID                            | Node Name | Provision State | Current Profile | Possible Profiles |
    +--------------------------------------+-----------+-----------------+-----------------+-------------------+
    | 581c0aca-64f0-48a8-9881-bba3c2882d6a |           | available       | control         | compute, control  |
    | ace8ae8d-d18f-4122-b6cf-e8418c7bb04b |           | available       | compute         | compute, control  |
    +--------------------------------------+-----------+-----------------+-----------------+-------------------+

Make sure to provide the same arguments for deployment later on::

    openstack overcloud deploy --control-flavor control --control-scale 1 --compute-flavor compute --compute-scale 1 --templates


.. _Introspection rules: http://docs.openstack.org/developer/ironic-inspector/usage.html#introspection-rules
