.. _package_update:

Updating Packages on Overcloud Nodes
====================================

Updating packages on all overcloud nodes involves two steps. The first one
makes sure that the overcloud plan is updated (a new tripleo-heat-templates rpm
might have brought fixes/changes to the templates)::

    openstack overcloud deploy --update-plan-only \
    --templates \
    -e <full environment>

By using the parameter ``--update-plan-only`` we make sure we update only the
stored overcloud plan and not the overcloud itself. Make sure you pass the
exact same environment parameters that were used at deployment time.

The second step consists in updating the packages themselves on all overcloud
nodes with a command similar to the following::

    openstack overcloud update stack -i overcloud

This command updates the ``UpdateIdentifier`` parameter and triggers stack update
operation. If this parameter is set, ``yum update`` command is executed on each
node. Because running update on all nodes in parallel might be unsafe (an
update of a package might involve restarting a service), the command above
sets breakpoints on each overcloud node so nodes are updated one by one. When
the update is finished on a node the command will prompt for removing
breakpoint on next one.

.. note::
   Make sure you use the ``-i`` parameter, otherwise update runs on background
   and does not prompt for removing of breakpoints.

.. note::
   Multiple breakpoints can be removed by specifying list of nodes with a
   regular expression.

.. note::
   If the update command is aborted for some reason you can always continue
   in the process by re-running same command.

.. note::
   The --templates and --environment-file (-e) are now deprecated. They can still
   be passed to the command, but they will be silently ignored. This is due to
   the plan now used for deployment should only be modified via plan modification
   commands.
