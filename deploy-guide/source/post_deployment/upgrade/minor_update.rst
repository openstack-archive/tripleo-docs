.. _package_update:

Updating Content on Overcloud Nodes
===================================

The update of overcloud packages and containers to the latest version
of the current release is referred to as the 'minor update' in TripleO
(distinguishing it from the 'major upgrade' to the next release). In
the Queens cycle the minor update workflow has changed compared to
previous cycles. There are thus version specific sections below.

Updating your Overcloud - Queens and beyond
-------------------------------------------

The Queens release brings common CLI and workflow conventions to the
main deployment lifecycle operations (minor updates, major upgrades,
and fast forward upgrades). This means that the minor update workflow
has changed compared to previous releases, and it should now be easier
to learn and reason about the lifecycle operations in general.

To update your overcloud to the latest packages / container images of
the OpenStack release that you currently operate, perform these steps:

#. **Software sources setup**

   In case you use pinned repositories (e.g. to some DLRN hash), make
   sure to update your repository files on overcloud to get the latest
   RPMs. If you use stable RDO repositories, you don't need to change
   anything.

   Update container image parameter files:

   .. admonition:: Queens
      :class: queens

      Fetch latest container images to your undercloud registry and
      generate a Heat environment file pointing to new container
      images. This is done via workflow described in
      :doc:`containerized deployment documentation<../../deployment/overcloud>`.

#. **Update preparation**

   To prepare the overcloud for the update, run:

   .. code-block:: bash

      openstack overcloud update prepare \
          <OPTIONS> \
          -e containers-prepare-parameter.yaml

   In place of the `<OPTIONS>` token should go all parameters that you
   used with previous `openstack overcloud deploy` command.

   The last argument `containers-prepare-parameter.yaml` differs in
   content depending on release. In Queens and before, it has a list
   of individual container image parameters, pointing to images you've
   already uploaded to local registry in previous step. In Rocky and
   beyond, this file contains the ``ContainerImagePrepare`` parameter.
   The upload of images to local registry is yet to happen, in a
   separate step after `update prepare`.

   .. note::

      The `update prepare` command performs a Heat stack update, and
      as such it should be passed all parameters currently used by the
      Heat stack (most notably environment files, role counts, roles
      data, and network data). This is crucial in order to keep
      correct state of the stack.

   .. note::

      The `containers-prepare-parameter.yaml` file is intended to
      replace any previous container parameters file. You should drop
      the previous container parameter file and pass the new one for
      any subsequent stack update operations.

   The `update prepare` command updates the Heat stack outputs with
   Ansible snippets used in the next steps of the update.

#. **Container image upload**

      Since Rocky, we will need to upload the container images
      to the local registry at this point. Run:

      .. code-block:: bash

         openstack overcloud external-update run --tags container_image_prepare

#. **Update run**

   Run the update procedure on a subset of nodes selected via the
   ``--limit`` parameter:

   .. code-block:: bash

      openstack overcloud update run --limit overcloud-controller-0

   You can specify a role name, e.g. 'Compute', to execute the minor
   update on all nodes of that role in a rolling fashion (`serial: 1`
   is used on the playbooks).

   There is no required node ordering for performing the minor update
   on the overcloud, but it's a good practice to keep some consistency
   in the process. E.g. all controllers first, then all computes, etc.

   Do this for all the overcloud nodes before proceeding to next step.

#. **Ceph update (optional)**

   If your environment includes Ceph managed by TripleO (i.e. *not*
   what TripleO calls "external Ceph"), you'll want to update Ceph at
   this point too. The procedure differs between Queens and newer
   releases:

   .. admonition:: Queens
      :class: queens

      Run:

      .. code-block:: bash

         openstack overcloud ceph-upgrade run <OPTIONS>

      In place of the `<OPTIONS>` token should go all parameters that you
      used with previous `openstack overcloud update prepare` command
      (including the new `-e container-params.yaml`).

      .. note::

         The `ceph-upgrade run` command performs a Heat stack update, and
         as such it should be passed all parameters currently used by the
         Heat stack (most notably environment files, role counts, roles
         data, and network data). This is crucial in order to keep
         correct state of the stack.

      The `ceph-upgrade run` command re-enables config management
      operations previously disabled by `update prepare`, and triggers
      the rolling update playbook of the Ceph installer (`ceph-ansible`).

   .. admonition:: Rocky
      :class: rocky

      Run:

      .. code-block:: bash

         openstack overcloud external-update run --tags ceph

      This will update Ceph by running ceph-ansible installer with
      update playbook.

#. **Update convergence**

   To finish the update procedure, run:

   .. code-block:: bash

      openstack overcloud update converge <OPTIONS>

   In place of the `<OPTIONS>` token should go all parameters that you
   used with previous `openstack overcloud update prepare` command
   (including the new `-e container-params.yaml`).

   .. note::

      The `update converge` command performs a Heat stack update, and
      as such it should be passed all parameters currently used by the
      Heat stack (most notably environment files, role counts, roles
      data, and network data). This is crucial in order to keep
      correct state of the stack.

   The `update converge` command updates Heat stack outputs with
   Ansible snippets the same way as `overcloud deploy` would, and it
   runs the config management operations to assert that the overcloud
   state matches the used overcloud templates.

Updating your Overcloud - Pike
------------------------------

.. note::
   The minor update workflow described below is generally not well tested for
   *non* containerized Pike environments. The main focus for the TripleO
   upgrades engineering and QE teams has been on testing the minor update
   within a containerized Pike environment.

   In particular there are currently no pacemaker update_tasks for the non
   containerized cluster services (i.e., `puppet/services/pacemaker`_) and
   those will need to be considered and added. You should reach out to the
   TripleO community if this is an important feature for you and you'd like
   to contribute to it.

For the Pike cycle the minor update workflow is significantly different to
previous cycles. In particular, rather than using a static yum_update.sh_
we now use service specific ansible update_tasks_ (similar to the upgrade_tasks
used for the major upgrade workflow since Ocata). Furthermore, these are not
executed directly via a Heat stack update, but rather, together with the
docker/puppet config, collected and written to ansible playbooks. The operator
then invokes these to deliver the minor update to particular nodes.

There are essentially two steps: first perform a (relatively short) Heat stack
update against the overcloud to generate the "config" ansible playbooks, and
then execute these. See bug 1715557_ for more information about this mechanism
and its implementation.


1. Confirm that your `$HOME/containers-prepare-parameter.yaml`
`ContainerImagePrepare` parameter includes a `tag_from_label` value, so that
the latest images are discovered on update, otherwise edit the `tag` value
to specify what image versions to update to.


2. Perform a heat stack update to generate the ansible playbooks, specifying
the registry file generated from the first step above::

    openstack overcloud update --init-minor-update --container-registry-file latest-images.yaml

3. Invoke the minor update on the nodes specified with the ``--limit``
parameter::

    openstack overcloud update --limit controller-0

.. admonition:: Stable Branch
   :class: stable

   The `--limit` was introduced in the Stein release. In previous versions,
   use `--nodes` or `--roles` parameters.

You can specify a role name, e.g. 'Compute', to execute the minor update on
all nodes of that role in a rolling fashion (serial:1 is used on the playbooks).

.. _yum_update.sh: https://github.com/openstack/tripleo-heat-templates/blob/53db241cfbfc1b6a237b7f33486a051aa6934579/extraconfig/tasks/yum_update.sh
.. _update_tasks: https://github.com/openstack/tripleo-heat-templates/blob/e1a9638732290c247e5dac10392bc8702b531981/puppet/services/tripleo-packages.yaml#L59
.. _1715557: https://bugs.launchpad.net/tripleo/+bug/1715557
.. _puppet/services/pacemaker: https://github.com/openstack/tripleo-heat-templates/tree/2e182bffeeb099cb5e0b1747086fb0e0f57b7b5d/puppet/services/pacemaker

Updating your Overcloud - Ocata and earlier
-------------------------------------------

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
   The ``--templates`` and ``--environment-file`` (``-e``) are now deprecated.
   They can still be passed to the command, but they will be silently ignored.
   This is due to the plan now used for deployment should only be modified via
   plan modification commands.
