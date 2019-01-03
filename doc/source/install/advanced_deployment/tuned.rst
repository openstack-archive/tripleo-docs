Deploying custom tuned profiles
===============================

TripleO can be used to deploy Overcloud nodes with different tuned
profiles in addition to custom tuned profiles.

Deploying with existing tuned profiles
--------------------------------------

Create an environment file, e.g. `~/tuned.yaml`, with the following
content:

.. code-block:: yaml

    parameter_defaults:
      TunedProfileName: throughput-performance

Deploy the Overcloud as usual using the :doc:`CLI
<../basic_deployment/basic_deployment_cli>` and pass the environment
file using the `-e` option:

.. code-block:: bash

  openstack overcloud deploy --templates -e ~/tuned.yaml

In the above example, the `throughput-performance` tuned profile will
be applied to the overcloud nodes. The TunedProfileName parameter may
be set to any tuned profile already on the node.

Deploying with custom tuned profiles
------------------------------------

If the tuned profile you wish to apply is not already on the overcloud
node being deployed, then TripleO can create the tuned profile for
you and will set the name of the new profile to whatever
TunedProfileName parameter you supply.

The following example creates a custom tuned profile called
`my_profile` which inherits from the existing throughput-performance
tuned profile and then adds a few extra tunings:

.. code-block:: yaml

    parameter_defaults:
      TunedCustomProfile: |
        [main]
        summary=my profile
        include=throughput-performance
        [sysctl]
        vm.dirty_ratio = 10
        vm.dirty_background_ratio = 3
        [sysfs]
        /sys/kernel/mm/ksm/run=0
      TunedProfileName: my_profile

The above will create the file `/etc/tuned/my_profile/tuned.conf`
on the overcloud nodes and tuned.conf will contain the tuned
directives defined by the TunedCustomProfile parameter. The
TunedCustomProfile parameter should be set to a multiline string using
YAML's literal block scalar (i.e. the pipe '|') and that string should
contain valid tuned directives in INI format.
