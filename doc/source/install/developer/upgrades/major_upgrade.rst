Pike to Queens major upgrade workflow and cli
---------------------------------------------

The purpose of this documentation is to deep-dive into the code which delivers
the Pike to Queens major upgrade workflow in TripleO. For information about
the steps an operator needs to perform when running this upgrade please see the
operator_docs_.

The Pike to Queens (P..Q) major upgrade workflow is significantly different
to its predecesor (Ocata to Pike) or any previously used upgrade workflow.
There are similarities to the Pike_minor_update_ in the general approach of
using an initial Heat stack update to generate but not apply configuration,
followed by *ansible-playbook $ARGS* to actually deliver it via ansible in
a targetted way with respect to the overcloud nodes. The Queens_upgrade_spec_
may also be of interest in describing the design of the P..Q workflow.

The P..Q major upgrade workflow is delivered almost exclusively via ansible
playbook invocations on the overcloud nodes. Heat is still used however and
a Heat stack update is performed against the overcloud stack in the first
and last step of the workflow (the 'prepare_' and 'converge_' respectively).

The 'prepare' Heat stack update does not apply any TripleO configuration and is
exclusively used to generate the ansible playbooks that are subsequently
invoked to deliver the upgrade. The 'converge' stack update *does* apply all
TripleO configuration against all overcloud nodes and thus serves as a sanity
check that the overcloud was successfully upgraded, since the same
configuration will already have been applied.

The 'converge' will also leave the Heat stack in a good state for subsequent
updates, for instance scaling to add nodes. The 'converge' stack update
unsets all upgrade specific tripleo-heat-template Heat stack parameters that
were set during the upgrade 'prepare'. These parameters are set by the TripleO
client (more below) using the prepare_env_file_ and similarly unset using the
converge_env_file_.

The Queens upgrade workflow brings a new cli via python-tripleoclient_, that
allows the operator to invoke the upgrade prepare, run and converge steps,
using mistral workflows in tripleo-common_ to invoke either a Heat stack update
involving the tripleo-heat-templates_ , or ansible-playbook_ (via tripleo-common)
depending on the operation. The following sections will examine each of the
upgrade steps, stepping through and pointing to some of the implementation
details across the various code repos involved:

* `openstack overcloud upgrade prepare $ARGS`_
* `openstack overcloud upgrade run $ARGS`_
* `openstack overcloud upgrade converge $ARGS`_


.. _queens_upgrade_spec: https://github.com/openstack/tripleo-specs/blob/master/specs/queens/tripleo_ansible_upgrades_workflow.rst
.. _operator_docs: https://docs.openstack.org/tripleo-docs/latest/install/post_deployment/upgrade.html
.. _Pike_minor_update: https://docs.openstack.org/tripleo-docs/latest/install/post_deployment/package_update.html
.. _prepare: https://review.openstack.org/#/c/535859/26/doc/source/install/post_deployment/upgrade.rst #TODO UPDATE ME
.. _converge: https://review.openstack.org/#/c/535859/26/doc/source/install/post_deployment/upgrade.rst # TODO UPDATE ME
.. _prepare_env_file: https://github.com/openstack/tripleo-heat-templates/blob/3ab23982a2fd3ffcad09e76f226bd4aab4040d4e/environments/lifecycle/upgrade-prepare.yaml#L4-L12
.. _converge_env_file: https://github.com/openstack/tripleo-heat-templates/blob/3ab23982a2fd3ffcad09e76f226bd4aab4040d4e/environments/lifecycle/upgrade-converge.yaml#L4-L7
.. _python-tripleoclient: https://github.com/openstack/python-tripleoclient/blob/master/tripleoclient/v1/overcloud_upgrade.py
.. _tripleo-common: https://github.com/openstack/tripleo-common/blob/master/workbooks/package_update.yaml
.. _tripleo-heat-templates: https://github.com/openstack/tripleo-heat-templates/blob/8277d675bc9496eb164f429fa265f79252166f2d/common/deploy-steps.j2#L604
.. _ansible-playbook: https://github.com/openstack/tripleo-common/blob/0b682f07502f99e003abafce097ce62fa2d20512/tripleo_common/actions/ansible.py#L243

openstack overcloud upgrade prepare $ARGS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The entry point for the the Q upgrade cli commands, *prepare*, *run* and
*execute*, is given in the python-tripleoclient setup.cfg_. All three are also
defined in the same file, overcloud-upgrade.py_.

As you can see the UpgradePrepare_ class inherits from DeployOvercloud_. The
reason for this is to prevent duplication of the logic concerned with validating
the configuration passed into the prepare command (all the -e env.yaml files),
as well as updating_the_swift_stored_plan_ with the overcloud configuration.

As a result the UpgradePrepare class inherits all the Deploy_parser_arguments_,
including --stack and -e for the additional environment files. We explicitly
set the update_plan_only_ argument so that the Heat stack update does not get
executed by the parent class and returns after completing all the template
processing.

Instead, the Heat stack update is performed by a mistral workflow. On the
client side the hook is in the update method defined in package_update.py_.
This invokes the package_update_plan_ mistral workflow in tripleo-common.
The package_update_plan_ workflow has a number of tasks, one of which invokes
the heat stack update using the update_stack_action_.

Back on the tripleoclient side, we use base_wait_for_messages_  to listen
for messages on the Zaqar_queue_ that is used by the mistral workflow.

It is worth noting here that before invoking the Heat stack update,
prepare_command_prepends_ the prepare_env_file_ to the list of environment
files passed to Heat. Also worth highlighting is that the operator must
include all environment files used in deploying the overcloud that is being
upgraded. It is especially important that the operator includes the environment
file containing the references for the target version container images. See
the operator_docs_ for pointers to how that file is generated and for
reference it will look something like

    .. code-block:: bash

       parameter_defaults:
         DockerAodhApiImage: 192.168.24.1:8787/queens/centos-binary-aodh-api:current-tripleo-rdo
         DockerAodhConfigImage: 192.168.24.1:8787/queens/centos-binary-aodh-api:current-tripleo-rdo
         DockerAodhEvaluatorImage: 192.168.24.1:8787/queens/centos-binary-aodh-evaluator:current-tripleo-rdo
         DockerAodhListenerImage: 192.168.24.1:8787/queens/centos-binary-aodh-listener:current-tripleo-rdo

Once the Heat stack update has been completed successfully and the stack is
in UPDATE_COMPLETE state, you can download the configuration ansible playbooks
using the config download cli


    .. code-block:: bash

       [stack@521-m--undercloud ~]$ source stackrc
       (undercloud) [stack@521-m--undercloud ~]$ openstack overcloud config download --config-dir MYCONFIGDIR
       The TripleO configuration has been successfully generated into: MYCONFIGDIR/tripleo-gep7gh-config

and you can inspect the ansible playbooks which are used by the *upgrade run*
before executing them.


.. _setup.cfg: https://github.com/openstack/python-tripleoclient/blob/e9a68430400a6b99005c6aa675bf9bd27ed810a1/setup.cfg#L88-L90
.. _overcloud-upgrade.py: https://github.com/openstack/python-tripleoclient/blob/f0110cdff0edcf40d8e94d4848c543310ea5c54e/tripleoclient/v1/overcloud_upgrade.py#L14
.. _UpgradePrepare: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_upgrade.py#L29
.. _DeployOvercloud: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_deploy.py#L44
.. _updating_the_swift_stored_plan: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_deploy.py#L301
.. _update_plan_only: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_upgrade.py#L72
.. _deploy_parser_arguments: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_deploy.py#L689-L890
.. _package_update.py: https://github.com/openstack/python-tripleoclient/blob/3d9183fc03aa96bce093e774ab4bf51655579a9c/tripleoclient/workflows/package_update.py#L34
.. _package_update_plan: https://github.com/openstack/tripleo-common/blob/1d3aefbe2f0aac2828eba69ee9efc57a7b7bf385/workbooks/package_update.yaml#L9
.. _update_stack_action: https://github.com/openstack/tripleo-common/blob/4d7258c2d8a521818146368568da07fd429e5a23/tripleo_common/actions/package_update.py#L100
.. _base_wait_for_messages: https://github.com/openstack/python-tripleoclient/blob/3d9183fc03aa96bce093e774ab4bf51655579a9c/tripleoclient/workflows/package_update.py#L38
.. _zaqar_queue: https://github.com/openstack/tripleo-common/blob/1d3aefbe2f0aac2828eba69ee9efc57a7b7bf385/workbooks/package_update.yaml#L17
.. _prepare_command_prepends: https://github.com/openstack/python-tripleoclient/blob/3d9183fc03aa96bce093e774ab4bf51655579a9c/tripleoclient/v1/overcloud_upgrade.py#L76-L79

openstack overcloud upgrade run $ARGS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Unlike the first step in the workflow, the *upgrade prepare*, the UpgradeRun_
class does not inherit from DeployOvercloud. There is no need for the operator
to pass all the environment files and configuration here. The template processing
and update of the stack and swift stored plan have already taken place.
The ansible playbooks are ready to be retrieved by config download as demonstrated
above. The upgrade run operation thus will simply execute those ansible playbooks
generated by the upgrade prepare command, against the nodes specified in the
parameters.

The `--nodes` and `--roles` parameters are used to limit the ansible playbook
execution to specific nodes. These are defined in a mutually
exclusive nodes_or_roles_ group and it is declared as required so that the
operator must pass either one of those and never both. Both `--roles` and
`--nodes` are used by ansible with the tripleo-ansible-inventory_. This creates
the ansible inventory based on the Heat stack outputs, so that for example
`Controller` and `overcloud-controller-0` are both valid values for the
ansible-playbook `--limit`_ parameter.

As documented in the run_user_docs_ and the nodes_or_roles_helptext_, the operator
*must* use `--roles` for the controllers. Upgrading the controlplane, one node
at a time is not supported, mainly due to limitations in the pacemaker cluster
upgrade which needs to occur across all nodes in the same operation. The
operator may use `--roles` for non controlplane nodes or may prefer to specify
one or more specific nodes by name with `--nodes`. In either case the value
specified by the operator is simply passed through to ansible as the
limit_hosts_ parameter.

The `--ssh-user` and all other parameters are similarly
collected and passed to the ansible invocation which starts on the client side
in the run_update_ansible_action_ method call. Before diving into more detail
about the ansible playbook run it is also worth highlighting the `--skip-tags`_
parameter that is used to skip certain ansible tasks with the ansible-skip-tags_
ansible-playbook parameter. For the Queens upgrade workflow we only support
skipping the step0 validation tasks that check services are running and this is
enforced by checking the value passed by the operator against the
MAJOR_UPGRADE_SKIP_TAGS_. Finally, the `--playbook`_ parameter as the name
suggests is used to specify the ansible playbook(s) to run. By default and
as you can see in the definition, this defaults to a special value 'all'
which causes all-upgrade-playbooks-to-run_. The value of all_playbooks
in that previous reference, is stored in the MAJOR_UPGRADE_PLAYBOOKS_ constant.

As with the *upgrade prepare*, for *upgrade run* a mistral workflow is used
to perform the 'main' operation, which in this case is execution of the
ansible playbooks. On the client side the update_nodes_workflow_invocation_
is where mistral is invoked and takes as workflow input the various collected
parameters described above. You can see that the update_nodes_workflow_ which
lives in tripleo-common has parameters defined under the 'input:' section which
correspond to the *openstack overcloud upgrade run* parameters.

There are two main tasks in the update_nodes_workflow_, the download-config_action_
which is invoked in a first 'download_config' task, and the ansible-playbook_action_
action which is invoked in the 'node_update' task. This is ultimately where
ansible-playbook-is-executed_ with processutils.execute.

Finally back on the client side we listen for messages on the run_zaqar_queue_
before declaring the upgrade-run-success_!


.. _UpgradeRun: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_upgrade.py#L94
.. _nodes_or_roles: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_upgrade.py#L110
.. _tripleo-ansible-inventory: https://github.com/openstack/tripleo-common/blob/cef9c406514fd0b01b7984b89334d8e8abd7a244/tripleo_common/inventory.py#L1
.. _`--limit`: https://docs.ansible.com/ansible/2.4/ansible-playbook.html#cmdoption-ansible-playbook-l
.. _run_user_docs: https://docs.openstack.org/tripleo-docs/latest/install/post_deployment/upgrade.html#openstack-overcloud-upgrade-run
.. _nodes_or_roles_helptext: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_upgrade.py#L111-L131
.. _limit_hosts: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_upgrade.py#L207-L212
.. _run_update_ansible_action: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_upgrade.py#L212-L217
.. _`--skip-tags`: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_upgrade.py#L211
.. _ansible-skip-tags: https://docs.ansible.com/ansible/2.4/ansible-playbook.html#cmdoption-ansible-playbook-skip-tags
.. _MAJOR_UPGRADE_SKIP_TAGS: https://github.com/openstack/python-tripleoclient/blob/3931606423a17c40a4458eb4df3c47cc6a829dbb/tripleoclient/constants.py#L56
.. _`--playbook`: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_upgrade.py#L133-L150
.. _all-upgrade-playbooks-to-run: https://github.com/openstack/python-tripleoclient/blob/3931606423a17c40a4458eb4df3c47cc6a829dbb/tripleoclient/utils.py#L946
.. _MAJOR_UPGRADE_PLAYBOOKS: https://github.com/openstack/python-tripleoclient/blob/3931606423a17c40a4458eb4df3c47cc6a829dbb/tripleoclient/constants.py#L53
.. _update_nodes_workflow_invocation: https://github.com/openstack/python-tripleoclient/blob/3931606423a17c40a4458eb4df3c47cc6a829dbb/tripleoclient/workflows/package_update.py#L85
.. _update_nodes_workflow: https://github.com/openstack/tripleo-common/blob/cef9c406514fd0b01b7984b89334d8e8abd7a244/workbooks/package_update.yaml#L99-L114
.. _download-config_action: https://github.com/openstack/tripleo-common/blob/cef9c406514fd0b01b7984b89334d8e8abd7a244/tripleo_common/actions/config.py#L65
.. _ansible-playbook_action: https://github.com/openstack/tripleo-common/blob/cef9c406514fd0b01b7984b89334d8e8abd7a244/tripleo_common/actions/ansible.py#L243
.. _ansible-playbook-is-executed: https://github.com/openstack/tripleo-common/blob/cef9c406514fd0b01b7984b89334d8e8abd7a244/tripleo_common/actions/ansible.py#L533-L535
.. _run_zaqar_queue: https://github.com/openstack/python-tripleoclient/blob/3931606423a17c40a4458eb4df3c47cc6a829dbb/tripleoclient/workflows/package_update.py#L89
.. _upgrade-run-success: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_upgrade.py#L219-L222


openstack overcloud upgrade converge $ARGS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The UpgradeConverge_ class like the UpgradePrepare class also inherits from
the DeployOvercloud_ class thus getting all of its parameters and template
processing. The operator needs to pass in all Heat environment files
used as part of the upgrade prepare including the container images file.

The main objective of the upgrade converge operation is to unset the
upgrade specific parameters that have been set on the overcloud Heat stack
as part of prepare. These can be found in the prepare_env_file_
environment file that is used by the upgrade prepare operation. These are
thus unset by the converge operation using the converge_env_file_ which is
included in the list of client_converge_env_files_ passed to the Heat stack
update.

As these values are set in parameter_defaults a Heat stack update is required
against the overcloud Heat stack to explicitly unset them. In particular and
as pointed out in the operator_converge_docs_ until converge has completed,
any operations that require a Heat stack update will likely fail, as the
'noop' of the DeploymentSteps in the prepare_env_file_ in particular means
none of the usual docker/puppet/* config is applied. Setting something with
parameter_defaults means it is used until explicitly unset via parameter_defaults
as that value will override any other default value specified via the
tripleo-heat-templates.

Unlike the prepare command there is no mistral workflow here and instead
we rely on the parent DeployOvercloud_ class to invoke the
converge_heat_stack_update_ and so the implementation is also simpler.

.. _UpgradeConverge: https://github.com/openstack/python-tripleoclient/blob/3931606423a17c40a4458eb4df3c47cc6a829dbb/tripleoclient/v1/overcloud_upgrade.py#L225
.. _client_converge_env_files: https://github.com/openstack/python-tripleoclient/blob/c7b7b4e3dcd34f9e51686065e328e73556967bab/tripleoclient/v1/overcloud_upgrade.py#L253
.. _operator_converge_docs: https://docs.openstack.org/tripleo-docs/latest/install/post_deployment/upgrade.html#openstack-overcloud-upgrade-converge
.. _converge_heat_stack_update: https://github.com/openstack/python-tripleoclient/blob/3931606423a17c40a4458eb4df3c47cc6a829dbb/tripleoclient/v1/overcloud_deploy.py#L223

Queens upgrade cli developer workflow
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This section will give some examples of a potential developer workflow for
testing fixes or in-progress gerrit reviews against python-tripleoclient,
tripleo-common or tripleo-heat-templates for the Queens upgrade workflow. This
may be useful if you are working on an upgrades related bug for example.

Making changes to the ansible playbooks
.......................................

If there is a failure running one of the upgrades related ansible playbooks,
you might need to examine and if necessary fix the related ansible task.
The tasks themselves live in each of the tripleo-heat-templates service
manifests, under the upgrade_tasks section of the template outputs. For example
see the containerized rabbitmq_upgrade_tasks_.

If you make a change in service upgrade_tasks, then to test it you will need to

1. Patch the tripleo-heat-templates in your environment with the fix
2. Rerun `openstack overcloud upgrade prepare $ARGS`_, so that the resulting
   ansible playbooks include your fix.
3. Finally run the playbooks with `openstack overcloud upgrade run $ARGS`_.

Assuming you are using the default /usr/share/openstack-tripleo-heat-templates
directory for the deployment templates you can use the following as just one
example:

    .. code-block:: bash

       # backup tht in case you want to revert - or just yum re-install ;)
       sudo cp -r /usr/share/openstack-tripleo-heat-templates \
           /usr/share/openstack-tripleo-heat-templates.ORIG
       # Apply patch from gerrit e.g. https://review.openstack.org/#/c/563073/
       curl -4sSL 'https://review.openstack.org/changes/563073/revisions/current/patch?download' | \
           base64 -d | \
           sudo patch -d /usr/share/openstack-tripleo-heat-templates/ -p1

Making changes to the upgrades workflow
.......................................

If instead you need to add or fix something in the upgrades workflow itself,
for example to handle a new parameter needed passed through to ansible, or any
other change, you will need to patch python-tripleoclient and tripleo-common,
depending on whether your fixes extend to the mistral workflow too.

There are many ways to patch your environment and the following is a different
approach to the one used in the tripleo-heat-templates above where we patched
the installed templates in place. In the following examples instead we clone
tripleo-common and tripleoclient, patch them using gerrit reviews and then
re-install from source.

    .. note::

       The following example commands include complete removal and replacement
       of the installed tripleoclient and tripleo-common!

Patching python-tripleoclient:

    .. code-block:: bash

       # python-tripleoclient - clone source, patch from gerrit and install
       git clone https://github.com/openstack/python-tripleoclient.git -b stable/queens ~/python-tripleoclient
       pushd ~/python-tripleoclient

       # Apply patches from gerrit e.g. https://review.openstack.org/#/c/564267
       curl "https://review.openstack.org/changes/564267/revisions/current/patch" | \
             base64 --decode > /home/stack/"564267.patch"
       patch -N -p1 -b -z .first < /home/stack/564267.patch
       # Remove current version and re-install
       sudo rm -rf /usr/lib/python2.7/site-packages/python_tripleoclient*
       sudo rm -rf /usr/lib/python2.7/site-packages/tripleoclient
       sudo python setup.py clean --all install
       popd

Patching tripleo-common:

    .. code-block:: bash

       # tripleo-common - clone from source, patch from gerrit and install
       git clone https://github.com/openstack/tripleo-common -b stable/queens
       pushd ~/tripleo-common
       # Apply patches from gerrit e.g. https://review.openstack.org/#/c/562995
       curl "https://review.openstack.org/changes/562995/revisions/current/patch" | \
            base64 --decode > /home/stack/"562995.patch"
       patch -N -p1 -b -z .first < /home/stack/562995.patch
       # Remove current version and re install
       sudo rm -rf /usr/lib/python2.7/site-packages/tripleo_common*
       sudo python setup.py clean --all install
       popd
       sudo cp /usr/share/tripleo-common/sudoers /etc/sudoers.d/tripleo-common

Finally you need to update the mistral workbooks with the newly installed
versions. In code block above, the tripleo-common change at 562995_ has changed
package_update.yaml and so that is what we need to update here:

    .. code-block:: bash

       mistral workbook-update /usr/share/tripleo-common/workbooks/package_update.yaml
       # Since entry_points.txt is affected next steps are required:
       # Re populate mistral db and restart services
       sudo mistral-db-manage  populate
       sudo systemctl restart openstack-mistral-api.service
       sudo systemctl restart openstack-mistral-engine.service
       sudo systemctl restart openstack-mistral-executor.service

.. _rabbitmq_upgrade_tasks: https://github.com/openstack/tripleo-heat-templates/blob/9a2923210c2773e59175571f5ecbfe2b8682737a/docker/services/pacemaker/rabbitmq.yaml#L261
.. _562995: https://review.openstack.org/#/c/562995
