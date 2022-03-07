(DEPRECATED)Advanced Profile Matching
=====================================

.. note:: Flavor based scheduling is not supported since Wallaby as
   compute service is not used/available in the undercloud. However,
   one can still assign a profile to a node and use that for node
   filtering in `Baremetal Provision Configuration`_.


Profile matching allows a user to specify precisely which nodes will receive
which flavor. Here are additional setup steps to take advantage of the profile
matching. In this document "profile" is a capability that is assigned to both
ironic node and nova flavor to create a link between them.

Default profile flavors ``compute``, ``control``, ``swift-storage``,
``ceph-storage`` and ``block-storage`` are created when the undercloud is
installed, and they are usable without modification in most environments.

After profile is assigned to a flavor, nova will only deploy it on ironic
nodes with the same profile. Deployment will fail if not enough ironic nodes
are tagged with a profile.

There are two ways to assign a profile to a node. You can assign it directly
or specify one or many suitable profiles for the deployment command to choose
from. It can be done either manually or using the introspection rules.

Manual profile tagging
----------------------

To assign a profile to a node directly, issue the following command::

    openstack baremetal node set <UUID OR NAME> --property capabilities=profile:<PROFILE>

Alternatively, you can provide a number of profiles as capabilities in form of
``<PROFILE>_profile:1``, which later can be automatically converted to one
assigned profile (see `Use the flavors to deploy`_ for details). For example::

    openstack baremetal node set <UUID OR NAME> --property capabilities=compute_profile:1,control_profile:1

Finally, to clean all profile information from a node use::

    openstack baremetal node unset <UUID OR NAME> --property capabilities

.. note::
    We can not update only a single key from the capabilities dictionary, so if
    it contained more then just the profile information then this will need to
    be set for the node.

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

Create a JSON file, for example ``rules.json``, with the introspection rules
to apply (see `Examples of introspection rules`_). Before the introspection
load this file into *ironic-inspector*::

    openstack baremetal introspection rule import /path/to/rules.json

Then (re)start the introspection. Check assigned profiles or possible profiles
using command::

    openstack overcloud profiles list

If you've made a mistake in introspection rules, you can delete them all::

    openstack baremetal introspection rule purge

Then reupload the updated rules file and restart introspection.

.. note::
    When you use introspection rules to assign the ``profile`` capability, it
    will always override the existing value. On the contrary,
    ``<PROFILE>_profile`` capabilities are ignored for nodes with the existing
    ``profile`` capability.

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

Examples of introspection rules
-------------------------------

Example 1
~~~~~~~~~

Imagine we have the following hardware: with disk sizes > 1 TiB
for object storage and with smaller disks for compute and controller nodes.
We also need to make sure that no hardware with seriously insufficient
properties gets to the fleet at all.

::

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
                {"action": "set-capability", "name": "profile", "value": "swift-storage"}
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
                {"action": "set-capability", "name": "control_profile", "value": "1"},
                {"action": "set-capability", "name": "profile", "value": null}
            ]
        }
    ]

This example consists of 3 rules:

#. Fail introspection if memory is lower is 4096 MiB. Such rules can be
   applied to exclude nodes that should not become part of your cloud.

#. Nodes with hard drive size 1 TiB and bigger are assigned the
   ``swift-storage`` profile unconditionally.

#. Nodes with hard drive less than 1 TiB but more than 40 GiB can be either
   compute or control nodes. So we assign two capabilities ``compute_profile``
   and ``control_profile``, so that the ``openstack overcloud profiles match``
   command can later make the final choice. For that to work, we remove the
   existing ``profile`` capability, otherwise it will have priority.

#. Other nodes are not changed.

Create flavors to use profile matching
--------------------------------------

In most environment the pre-created profile flavors should be enough for use
with profile matching. However, if custom profile flavors are needed,
they can be created as follows.

* Create a flavor::

    openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 my-flavor

  .. note::
    The values for ram, disk, and vcpus should be set to a minimal lower bound,
    as Nova will still check that the Ironic nodes have at least this much.

* In order to use the profile assigned to the Ironic nodes, the Nova flavor
  needs to have the property ``capabilities:profile`` set to the intended
  profile::

    openstack flavor set --property "cpu_arch"="x86_64" --property "capabilities:profile"="my-profile" my-flavor

  .. note::
    The flavor name does not have to match the profile name, although it's
    highly recommended.


.. _Introspection rules: https://docs.openstack.org/ironic-inspector/usage.html#introspection-rules
.. _Baremetal Provision Configuration: https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/provisioning/baremetal_provision.html#baremetal-provision-configuration
