====================
Minor version update
====================

To get developer understanding of minor updates, first read the
:doc:`operator docs for minor updates <../../post_deployment/package_update>`
and perhaps try to go through the update as an operator would, to get
the basic idea.

Assuming operator-level familiarity with the minor updates, let's look
at individual pieces in more detail.

How update commands work
========================

The following subsections describe the individual update commands:

* `openstack overcloud update prepare`_
* `openstack overcloud update run`_
* `openstack overcloud ceph-upgrade run`_
* `openstack overcloud update converge`_

`openstack overcloud update prepare`
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `update prepare` command performs a Heat stack update, mapping
some resources to ``OS::Heat::None`` in order to prevent the usual
deployment config management tasks being performed (running Puppet,
starting containers, running external installers like
ceph-ansible). See the `update prepare environment file`_.

.. _`update prepare environment file`: https://github.com/openstack/tripleo-heat-templates/blob/4286727ae70b1fa4ca6656c3f035afeac6eb2a95/environments/lifecycle/update-prepare.yaml

The purpose of this stack update is to regenerate fresh outputs of the
Heat stack. These outputs contain Ansible playbooks and task lists
which are then used in the later in the `update run` phase.

`openstack overcloud update run`
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `update run` command utilizes the previously generated Heat stack
outputs. It downloads the playbook yamls and their included task list
yaml via the config-download mechanisms, and executes the
`update steps playbook`_.

.. _`update steps playbook`: https://github.com/openstack/tripleo-heat-templates/blob/4286727ae70b1fa4ca6656c3f035afeac6eb2a95/common/deploy-steps.j2#L558-L592

The command accepts ``--nodes`` or ``--roles`` argument to limit which
nodes will be targeted during a particular `update run`
execution. Even if the limit matches multiple nodes (e.g. all nodes
within one role), the play is executed with ``serial: 1``, meaning
that all actions are finished on one node before starting the update
on another.

The play first executes `update_steps_tasks.yaml` which are tasks
collected from the ``update_tasks`` entry in composable
services.

After the update tasks are finished, deployment workflow is is
performed on the node being updated. That means reusing
`host_prep_tasks.yaml` and `common_deploy_steps_tasks.yaml`, which are
executed like on a fresh deployment, except during minor update
they're within a play with the aforementioned ``serial: 1`` limiting.

Finally, ``post_update_tasks`` are executed. They are utilized by
services which need to perform something *after* deployment workflow
during the minor update. The update of the node is complete and the
Ansible play continues to update another node.

`openstack overcloud ceph-upgrade run`
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `ceph-upgrade run` command performs a Heat stack update. It
reverts the previous ``OS::Heat::None`` resource mappings back to the
values used for regular deployments and configuration updates, and it
sets ``CephAnsiblePlaybook`` parameter to perform a rolling update of
Ceph.

See the `ceph upgrade environment file`_.

.. _`ceph upgrade environment file`: https://github.com/openstack/tripleo-heat-templates/blob/4286727ae70b1fa4ca6656c3f035afeac6eb2a95/environments/lifecycle/ceph-upgrade-prepare.yaml

Currently there's no difference between update and upgrade workflow of
Ceph in TripleO, so the `ceph-upgrade run` command serves both
purposes.

It is likely that with the switch to config-download, we won't be
doing a stack update when updating Ceph.

`openstack overcloud update converge`
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `update converge` command performs a Heat stack update, reverting
the previous ``OS::Heat::None`` resource mappings back to the values
used for regular deployments and configuration updates, and
potentially also resets some parameter values. For environments with
Ceph, majority of this already happened on `ceph-upgrade run`, so the
final `update converge` effectively just resets the
CephAnsiblePlaybook parameter.

See the `update converge environment file`_.

.. _`update converge environment file`: https://github.com/openstack/tripleo-heat-templates/blob/4286727ae70b1fa4ca6656c3f035afeac6eb2a95/environments/lifecycle/update-converge.yaml

The purpose of this stack update is to re-run config management
mechanisms and assert that the overcloud state matches what is
provided by the templates and environment files.

Writing update logic for a service
==================================

Simple config/image replacement
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If the service is managed by Paunch_, it may be that there's no need to
write any update tasks. Paunch can automatically handle simple
updates: change in configuration or change of container image URL
triggers automatic removal of the old container and creation of new
one with latest config and latest image. If that's all the service
needs for updates, you don't need to create any ``update_tasks``.

Custom tasks during updates
~~~~~~~~~~~~~~~~~~~~~~~~~~~

If the service is not managed by Paunch_, or if the simple container
replacement done by Paunch is not sufficient for the service update,
you will need to include custom update logic. This is done via
providing these outputs in your composable service template:

* ``update_tasks`` -- these are executed before deployment tasks on the
  node being updated.

* ``post_update_tasks`` -- these are executed after deployment tasks on
  the node being updated.

.. _Paunch: https://git.openstack.org/cgit/openstack/paunch/tree/README.rst

Update tasks are generally meant to bring the service into a stopped
state (sometimes with pre-fetched new images, this is necessary for
services managed by Pacemaker). Then the same workflow as during
deployment is used to bring the node back up into a running state, and
the post-update tasks can then perform any actions needed after the
deployment workflow.

Similarly as deployment tasks, the update tasks and post-update tasks
are executed in steps_.

.. _steps: https://github.com/openstack/tripleo-heat-templates/blob/4286727ae70b1fa4ca6656c3f035afeac6eb2a95/common/deploy-steps.j2#L17-L18
