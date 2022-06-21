Node matching with resource classes and profiles
================================================

The `Baremetal Provision Configuration`_ describes all of the instance and
defaults properties which can be used as selection criteria for which node will
be assigned to a provisioned instance. Filtering on the ``resource_class`` property
is recommended for nodes which have special hardware for specific roles. The
``profile`` property is recommended for other matching requirements such as
placing specific roles to groups of nodes, or assigning instances to nodes based
on introspection data.

Resource class matching
-----------------------

As an example of matching on special hardware, this shows how to have a custom
``Compute`` role for PMEM equipped hardware, see :doc:`../features/compute_nvdimm`.

By default all nodes are assigned the ``resource_class`` of ``baremetal``. Each
node which is PMEM enabled needs to have its ``resource_class`` changed to
``baremetal.PMEM``::

    baremetal node set <UUID OR NAME> --resource-class baremetal.PMEM

Assuming there is a custom role called ``ComputePMEM``, the
``~/overcloud_baremetal_deploy.yaml`` file will match on ``baremetal.PMEM``
nodes with:

.. code-block:: yaml

  - name: ComputePMEM
    count: 3
    defaults:
      resource_class: baremetal.PMEM

Advanced profile matching
-------------------------
Profile matching allows a user to specify precisely which nodes provision with each
role (or instance). Here are additional setup steps to take advantage of the
profile matching. In this document ``profile`` is a capability that is assigned to
the ironic node, then matched in the ``openstack overcloud node provision`` yaml.

After profile is specified in ``~/overcloud_baremetal_deploy.yaml``, metalsmith
will only deploy it on ironic nodes with the same profile. Deployment will fail
if not enough ironic nodes are tagged with a profile.

There are two ways to assign a profile to a node. You can assign it directly
or specify one or many suitable profiles for the deployment command to choose
from. It can be done either manually or using the introspection rules.

Manual profile tagging
~~~~~~~~~~~~~~~~~~~~~~

To assign a profile to a node directly, issue the following command::

    baremetal node set <UUID OR NAME> --property capabilities=profile:<PROFILE>

To clean all profile information from a node use::

    baremetal node unset <UUID OR NAME> --property capabilities

.. note::
    We can not update only a single key from the capabilities dictionary, so if
    it contained more then just the profile information then this will need to
    be set for the node.

Also see :ref:`instackenv` for details on how to set profile in the
``instackenv.json`` file.

.. _auto-profile-tagging:

Automated profile tagging
~~~~~~~~~~~~~~~~~~~~~~~~~

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
to apply (see `Example of introspection rules`_). Before the introspection
load this file into *ironic-inspector*::

    baremetal introspection rule import /path/to/rules.json

Then (re)start the introspection. Check assigned profiles using command::

    baremetal node list -c uuid -c name -c properties

If you've made a mistake in introspection rules, you can delete them all::

    baremetal introspection rule purge

Then reupload the updated rules file and restart introspection.

.. note::
    When you use introspection rules to assign the ``profile`` capability, it
    will always override the existing value. On the contrary,
    ``<PROFILE>_profile`` capabilities are ignored for nodes with the existing
    ``profile`` capability.

Example of introspection rules
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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
   and ``control_profile``, so that the ``openstack overcloud node provision``
   command can later make the final choice. For that to work, we remove the
   existing ``profile`` capability, otherwise it will have priority.

#. Other nodes are not changed.

Provision with profile matching
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Assuming nodes have been assigned the profiles ``control_profile`` and
``compute_profile``, the ``~/overcloud_baremetal_deploy.yaml`` can be modified
with the following to match profiles during ``openstack overcloud node
provision``:

.. code-block:: yaml

  - name: Controller
    count: 3
    defaults:
      profile: control_profile
  - name: Compute
    count: 100
    defaults:
      profile: compute_profile

.. _Introspection rules: https://docs.openstack.org/ironic-inspector/usage.html#introspection-rules
.. _Baremetal Provision Configuration: https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/provisioning/baremetal_provision.html#baremetal-provision-configuration
