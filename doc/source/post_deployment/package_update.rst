.. _package_update:

Updating Packages on Overcloud Nodes
====================================

You can update packages on all overcloud nodes with a command similar to the
following::

    openstack overcloud update stack --templates [templates dir] -i overcloud -e /usr/share/openstack-tripleo-heat-templates/overcloud-resource-registry-puppet.yaml -e <all extra files>

This command updates the ``UpdateIdentifier`` parameter and triggers stack update
operation. If this parameter is set, ``yum update`` command is executed on each
node. Because running update on all nodes in parallel might be unsafe (an
update of a package might involve restarting a service), the command above
sets breakpoints on each overcloud node so nodes are updated one by one. When
the update is finished on a node the command will prompt for removing
breakpoint on next one.

.. note::
   When passing any extra environment files while creating the overcloud (for
   instance, in order to configure :doc:`network isolation
   <../advanced_deployment/network_isolation>`), pass them again here using the
   ``-e`` or ``--environment-file`` option to avoid making undesired changes to
   the overcloud.

   If you use Heat templates from the default location
   (`/usr/share/openstack-tripleo-heat-templates`), it is possible that
   these templates have changed when updating the undercloud machine. When
   updating overcloud nodes, make sure you pass both the default registry
   file and any extra environment files.

   The reason is that CLI commands, which are changing an existing overcloud,
   will pass new templates to Heat reusing the existing environment of the
   overcloud. This leads to the condition in which a new `resource type`
   referenced in updated templates might be missing in the existing environment.

.. note::
   Make sure you use the ``-i`` parameter, otherwise update runs on background
   and does not prompt for removing of breakpoints.

.. note::
   Multiple breakpoints can be removed by specifying list of nodes with a
   regular expression.

.. note::
   If the update command is aborted for some reason you can always continue
   in the process by re-running same command.
