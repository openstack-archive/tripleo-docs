.. _config_download:

TripleO config-download User's Guide: Deploying with Ansible
=============================================================

Introduction
------------
This documentation details using ``config-download``.

``config-download`` is the feature that enables deploying the Overcloud software
configuration with Ansible in TripleO.

Summary
-------
Starting with the Queens release, it is possible to use Ansible to apply the
overcloud configuration. In the Rocky release, this method is the new default
behavior.

Ansible is used to replace the communication and transport of the software
configuration deployment data between Heat and the Heat agent
(os-collect-config) on the overcloud nodes.

Instead of os-collect-config running on each overcloud node and polling for
deployment data from Heat, the Ansible control node applies the configuration
by running ``ansible-playbook`` with an Ansible inventory file and a set of
playbooks and tasks.

The Ansible control node (the node running ``ansible-playbook``) is the
undercloud by default.

``config-download`` is the feature name that enables using Ansible in this
manner, and will often be used to refer to the method detailed in this
documentation.

Heat is still used to create the stack and all of the OpenStack resources. The
same parameter values and environment files are passed to Heat as they were
previously. During the stack creation, Heat creates any OpenStack service
resources such as Nova servers and Neutron networks and ports, and then creates
the software configuration data necessary to configure the overcloud via
SoftwareDeployment resources.

The difference with ``config-download`` is that although Heat creates all the
deployment data necessary via SoftwareDeployment resources to perform the
overcloud installation and configuration, it does not apply any of the software
deployments. The data is only made available via the Heat API. Once the stack
is created, an additional config-download Mistral workflow is triggered that
downloads all of the deployment data from Heat.

Using the downloaded deployment data, the workflow then generates Ansible
playbooks and tasks that are used by the undercloud to complete the
configuration of the overcloud using ``ansible-playbook``.

This diagram details the overall sequence of how using config-download
completes an overcloud deployment:

.. image:: ../_images/tripleo_ansible_arch.png
    :scale: 40%


Deployment with config-download
-------------------------------
Ansible and ``config-download`` are used by default when ``openstack
overcloud deploy`` (tripleoclient) is run. The command is backwards compatible
in terms of functionality, meaning that running ``openstack overcloud deploy``
will still result in a full overcloud deployment.

The deployment is done through a series of automated workflows and steps in
tripleoclient. All of the workflow steps are automated by tripleoclient and
Mistral workflow(s).  The workflow steps are summarized as:

#. Create deployment plan
#. Create Heat stack along with any OpenStack resources (Neutron networks,
   Nova/Ironic instances, etc)
#. Create software configuration within the Heat stack
#. Create tripleo-admin ssh user
#. Download the software configuration from Heat
#. Applying the downloaded software configuration to the overcloud nodes with
   ``ansible-playbook``.

Creating the ``tripleo-admin`` user on each overcloud node is necessary since
ansible uses ssh to connect to each node to perform configuration.

The following steps are done to create the ``tripleo-admin`` user:

#. Create temporary ssh keys on the undercloud
#. Use a deployer-specified private ssh key (defaults to ``~/.ssh/id_rsa``) to
   connect to each overcloud node as a deployer specified user (defaults to
   ``heat-admin``) and adds the temporary public ssh key to
   ``~/.ssh/authorized_keys`` for that user.
#. Executes a Mistral workflow to create ``tripleo-admin`` on each node,
   passing as input the temporary private ssh key and ssh user to Mistral.
#. The workflow creates the ``tripleo-admin`` user and gives sudo permissions
   to the user, as well as creates and stores a new ssh keypair specific to
   ``tripleo-admin``. This keypair (private and public) are stored in the
   Mistral database.
#. After the completion of the workflow, the temporary ssh public key is
   deleted from ``~/.ssh/authorized_keys`` on each overcloud node, and the
   temporary keypair is then deleted from the undercloud.

With these steps, the deployer-specified ssh key which is used for the initial
connection is never sent or stored by any API service.

To override the deployer specified ssh private key and user, there are cli args
available with ``openstack overcloud deploy``::

    --overcloud-ssh-user # defaults to heat-admin
    --overcloud-ssh-key  # defaults to ~/.ssh/id_rsa

The values for these cli arguments must be the same for all nodes in the
overcloud deployment. ``overcloud-ssh-key`` should be the private key that
corresponds with the public key specified by the Heat parameter ``KeyName``
when using Ironic deployed nodes.

config-download related CLI arguments
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
There are some new CLI arguments for ``openstack overcloud deploy`` that can be
used to influence the behavior of the overcloud deployment as it relates to
``config-download``::

    --overcloud-ssh-user   # Initial ssh user used for creating tripleo-admin.
                           # Defaults to heat-admin

    --overcloud-ssh-key    # Initial ssh private key (file path) to be used for
                           # creating tripleo-admin.
                           # Defaults to ~/.ssh/id_rsa

    --override-ansible-cfg # path to an ansible config file, to inject any
                           # arbitrary ansible config to be used when running
                           # ansible-playbook

    --stack-only           # Only update the plan and stack. Skips applying the
                           # software configuration with ansible-playbook.

    --config-download-only # Only apply the software configuration with
                           # ansible-playbook. Skips the stack and plan update.

See ``openstack overcloud deploy --help`` for further help text.

.. include:: deployment_output.rst

.. _deployment_status:

.. include:: deployment_status.rst

.. include:: deployment_log.rst

Ansible configuration
^^^^^^^^^^^^^^^^^^^^^
When ``ansible-playbook`` runs, it will use a configuration file with the
following default values::

    [defaults]
    retry_files_enabled = False
    log_path = <working directory>/ansible.log
    forks = 25

    [ssh_connection]
    ssh_args = -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -o ControlMaster=auto -o ControlPersist=60s
    control_path_dir = <working directory>/ansible-ssh

Any of the above configuration options can be overridden, or any additional
ansible configuration used by passing the path to an ansible configuration file
with ``--override-ansible-cfg`` on the deployment command.

For example the following command will use the configuration options from
``/home/stack/ansible.cfg``. Any options specified in the override file will
take precendence over the defaults::

    openstack overcloud deploy \
      ...
      --override-ansible-cfg /home/stack/ansible.cfg


config-download with deployed-server
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
When using ``config-download`` with :doc:`deployed-server <deployed_server>`
(pre-provisioned nodes), a ``HostnameMap`` parameter **must** be provided.
Create an environment file to define the parameter, and assign the node
hostnames in the parameter value. The following example shows a sample value::

  parameter_defaults:
    HostnameMap:
      overcloud-controller-0: controller-00-rack01
      overcloud-controller-1: controller-01-rack02
      overcloud-controller-2: controller-02-rack03
      overcloud-novacompute-0: compute-00-rack01
      overcloud-novacompute-1: compute-01-rack01
      overcloud-novacompute-2: compute-02-rack01

Write the contents to an environment file such as ``hostnamemap.yaml``, and
pass the environment as part of the deployment command with ``-e``.

Mistral workflow
----------------
The Mistral workflow that will be called by tripleoclient and runs
config-download and ``ansible-playbook`` is
``tripleo.deployment.v1.config_download_deploy``.

Ansible project directory
^^^^^^^^^^^^^^^^^^^^^^^^^
The workflow will create an Ansible project directory with the plan name under
``/var/lib/mistral``. For the default plan name of ``overcloud`` the working
directory will be::

    /var/lib/mistral/overcloud

The project directory is where the downloaded software configuration from
Heat will be saved. It also includes other ansible-related files necessary to
run ``ansible-playbook`` to configure the overcloud.

All of the files in the Ansible project directory at
``/var/lib/mistral/<plan>`` are owned by the mistral user and readable by the
mistral group from the mistral-executor container. The interactive user account
on the undercloud can be granted read-only access to these files by using the
following setacl command::

    sudo setfacl -R -m u:$USER:rwx /var/lib/mistral

Once a member of the ``mistral`` group, the contents of
``/var/lib/mistral/<plan>`` can be browsed, examined, and
``ansible-playbook`` rerun if desired.

The contents of the project directory include the following files:

tripleo-ansible-inventory.yaml
  Ansible inventory file containing hosts and vars for all the overcloud nodes.
ansible.log
  Log file from the last run of ``ansible-playbook``.
ansible.cfg
  Config file used when running ``ansible-playbook``.
ansible-playbook-command.sh
  Executable script that can be used to rerun ``ansible-playbook``.
ssh_private_key
  Private ssh key used to ssh to the overcloud nodes.

Reproducing ansible-playbook
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Once in the project directory created by the Mistral workflow, simply run
``ansible-playbook-command.sh`` to reproduce the deployment::

    ./ansible-playbook-command.sh

Any additional arguments passed to this script will be passed unchanged to the
``ansible-playbook`` command::

    ./ansible-playbook-command.sh --check

Using this method it is possible to take advantage of various Ansible features,
such as check mode (``--check``), limiting hosts (``--limit``), or overriding
variables (``-e``).

Git repository
^^^^^^^^^^^^^^
The ansible project directory is a git repository. Each time config-download
downloads the software configuration data from Heat, the project directory will
be checked for differences. A new commit will be created if there are any
changes from the previous revision.

From within the ansible project directory, standard git commands can be used to
explore each revision. Commands such as ``git log``, ``git show``, and ``git
diff`` are useful ways to describe how each commit to the software
configuration differs from previous commits.

Applying earlier versions of configuration
__________________________________________
Using commands such as ``git revert`` or ``git checkout``, it is possible to
update the ansible project directory to an earlier version of the software
configuration.

It is possible to then apply this earlier version with ``ansible-playbook``.
However, caution should be exercised as this could lead to a broken overcloud
deployment. Only well understood earlier versions should be attempted to be
applied.

.. note::

    Data migration changes will never be undone by applying an earlier version
    of the software configuration with config-download. For example, database
    schema migrations that had already been applied would never be undone by
    only applying an earlier version of the configuration.

    Software changes that were related to hardware changes in the overcloud
    (such as scaling up or down) would also not be completely undone by
    applying earlier versions of the software configuration.

.. note::

    Reverting to earlier revisions of the project directory has no effect on
    the configuration stored in the Heat stack. A corresponding change should
    be made to the deployment templates, and the stack updated to make the
    changes permanent.

.. _manual-config-download:

Manual config-download
----------------------
The Mistral workflow that runs config-download can be skipped when running
``openstack overcloud deploy`` by passing ``--stack-only``. This will cause
tripleoclient to only deploy the Heat stack.

When using ``--stack-only``, the deployment data needs to be pulled from Heat
with a separate command and ``ansible-playbook`` run manually. This enables
more manual interaction and debugging.

This method is described in the following sections.

Run config-download
^^^^^^^^^^^^^^^^^^^
When using the ``--stack-only`` argument, the deployment data needs to be first
downloaded from Heat. To manually download the software configuration data, use
the ``openstack overcloud config download`` command::

    openstack overcloud config download \
      --name overcloud \
      --config-dir config-download

The ansible data will be generated under a directory called ``config-download``
as specified by the ``--config-dir`` CLI argument.

Generate an inventory
^^^^^^^^^^^^^^^^^^^^^
To generate an inventory file to use with ``ansible-playbook`` use the
``tripleo-ansible-inventory`` command::

    tripleo-ansible-inventory \
      --ansible_ssh_user centos \
      --static-yaml-inventory inventory.yaml

The above example shows setting the ansible ssh user as ``centos``. This can be
changed depending on the environment. See ``tripleo-ansible-inventory --help``
for a full list of CLI arguments and options.

Run ansible-playbook
^^^^^^^^^^^^^^^^^^^^
Once the configuration has been downloaded and the inventory generated, run
``ansible-playbook`` to configure the overcloud nodes::

    ansible-playbook \
      -i inventory.yaml \
      --private-key /path/private/ssh/key \
      --become \
      config-download/deploy_steps_playbook.yaml

.. note::

    ``--become`` is required when running ansible-playbook.

All default ansible configuration values will be used when manually running
``ansible-playbook`` in this manner. These values can be customized through
`ansible configuration
<https://docs.ansible.com/ansible/latest/installation_guide/intro_configuration.html>`_.

The following minimum configuration is recommended and matches the default
values used by the mistral workflow that runs ``config-download``::

    [defaults]
    log_path = ansible.log
    forks = 25
    timeout = 30

    [ssh_connection]
    ssh_args = -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -o ControlMaster=auto -o ControlPersist=30m
    retries = 8
    pipelining = True

.. note::

   When running ``ansible-playbook`` manually, the overcloud status as returned
   by ``openstack overcloud status`` won't be automatically updated due to the
   configuration being applied outside of the API.

   See :ref:`deployment_status` for setting the status manually.

Ansible project directory contents
----------------------------------
This section details the structure of the ``config-download`` generated
Ansible project directory.

Playbooks
^^^^^^^^^
deploy_steps_playbook.yaml
  Initial deployment or template update (not minor update)

  Further detailed in :ref:`deploy_steps_playbook.yaml`
fast_forward_upgrade_playbook.yaml
  Fast forward upgrades
post_upgrade_steps_playbook.yaml
  Post upgrade steps for major upgrade
pre_upgrade_rolling_steps_playbook.yaml
  Pre upgrade steps for major upgrade
update_steps_playbook.yaml
  Minor update steps
upgrade_steps_playbook.yaml
  Major upgrade steps

.. _deploy_steps_playbook.yaml:

deploy_steps_playbook.yaml
__________________________
``deploy_steps_playbook.yaml`` is the playbook used for deployment and template
update. It applies all the software configuration necessary to deploy a full
overcluod based on the templates provided as input to the deployment command.

This section will summarize at high level the different ansible plays used
within this playbook. The play names shown here are the same names used within
the playbook and are what will be shown in the output when ``ansible-playbook`` is
run.

The ansible tags set on each play are also shown below.

Gather facts from undercloud
  Fact gathering for the undercloud node

  tags: facts
Gather facts from overcloud
  Fact gathering for the overcloud nodes

  tags: facts
Load global variables
  Loads all varaibles from `l`global_vars.yaml``

  tags: always
Common roles for TripleO servers
  Applies common ansible roles to all overcloud nodes. Includes
  ``tripleo-bootstrap`` for installing bootstrap packages and
  ``tripleo-ssh-known-hosts`` for configuring ssh known hosts.

  tags: common_roles
Overcloud deploy step tasks for step 0
  Applies tasks from the ``deploy_steps_tasks`` template interface

  tags: overcloud, deploy_steps
Server deployments
  Applies server specific Heat deployments for configuration such as networking
  and hieradata. Includes ``NetworkDeployment``, ``<Role>Deployment``,
  ``<Role>AllNodesDeployment``, etc.

  tags: overcloud, pre_deploy_steps
Host prep steps
  Applies tasks from the ``host_prep_steps`` template interface

  tags: overcloud, host_prep_steps
External deployment step [1,2,3,4,5]
  Applies tasks from the ``external_deploy_steps_tasks`` template interface.
  These tasks are run against the undercloud node only.

  tags: external, external_deploy_steps
Overcloud deploy step tasks for [1,2,3,4,5]
  Applies tasks from the ``deploy_steps_tasks`` template interface

  tags: overcloud, deploy_setps
Overcloud common deploy step tasks [1,2,3,4,5]
  Applies the common tasks done at each step to include puppet host
  configuration, ``docker-puppet.py``, and ``paunch`` (container configuration).

  tags: overcloud, deploy_setps
Server Post Deployments
  Applies server specific Heat deployments for configuration done after the 5
  step deployment process.

  tags: overcloud, post_deploy_steps
External deployment Post Deploy tasks
  Applies tasks from the ``external_post_deploy_steps_tasks`` template interface.
  These tasks are run against the undercloud node only.

  tags: external, external_deploy_steps


Task files
^^^^^^^^^^
These task files include tasks specific to their intended function. The task
files are automatically used by specific playbooks from the previous section.

**boot_param_tasks.yaml**

**common_deploy_steps_tasks.yaml**

**docker_puppet_script.yaml**

**external_deploy_steps_tasks.yaml**

**external_post_deploy_steps_tasks.yaml**

**fast_forward_upgrade_bootstrap_role_tasks.yaml**

**fast_forward_upgrade_bootstrap_tasks.yaml**

**fast_forward_upgrade_post_role_tasks.yaml**

**fast_forward_upgrade_prep_role_tasks.yaml**

**fast_forward_upgrade_prep_tasks.yaml**

**fast_forward_upgrade_release_tasks.yaml**

**upgrade_steps_tasks.yaml**

**update_steps_tasks.yaml**

**pre_upgrade_rolling_steps_tasks.yaml**

**post_upgrade_steps_tasks.yaml**

**post_update_steps_tasks.yaml**

Heat Role directories
^^^^^^^^^^^^^^^^^^^^^
Each Heat role from the roles data file used in the deployment (specified with
``-r`` from the ``openstack overcloud deploy`` command), will have a
correspondingly named directory.

When using the default roles, these directories would be:

**Controller**

**Compute**

**ObjectStorage**

**BlockStorage**

**CephStorage**

A given role directory contains role specific task files and a subdirectory for
each host for that role. For example, when using the default hostnames, the
**Controller** role directory would contain the following host subdirectories:

**overcloud-controller-0**

**overcloud-controller-1**

**overcloud-controller-2**

Variable and template related files
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
group_vars
  Directory which contains variables specific to different ansible inventory
  groups.
global_vars.yaml
  Global ansible variables applied to all overcloud nodes
templates
  Directory containing any templates used during the deployment

Other files
^^^^^^^^^^^
Files in this section are only present in the project directory if the mistral
workflow was used to generate the project directory under
``/var/lib/mistral/<plan>``

ansible.cfg
  Ansible configuration file
ansible-errors.json
  JSON structured file containing any deployment errors
ansible.log
  Ansilbe log file
ansible-playbook-command.sh
  Script to reproduce ansible-playbook command
ssh_private_key
  SSH private key used by ansible to access overcloud nodes
tripleo-ansible-inventory.yaml
  Ansible inventory file
overcloud-config.tar.gz
  Tarball of Ansible project directory

Running specific tasks
----------------------
Running only specific tasks (or skipping certain tasks) can be done from within
the ansible project directory.

.. note::

    Running specific tasks is an advanced use case and only recommended for
    specific scenarios where the deployer is aware of the impact of skipping or
    only running certain tasks.

    This can be useful during troubleshooting and debugging scenarios, but
    should be used with caution as it can result in an overcloud that is not
    fully configured.

.. warning::

   All tasks that are part of the deployment need to be run, and in the order
   specified.  When skipping tasks with ``--tags``, ``-skip-tags``,
   ``--start-at-task``, the deployment could be left in an inoperable state.

   The functionality to skip tasks or only run certain tasks is meant to aid in
   troubleshooting and iterating more quickly on failing deployments and
   updates.

   All changes to the deployed cloud must still be applied through the Heat
   templates and environment files passed to the ``openstack overcloud deploy``
   command. Doing so ensures that the deployed cloud is kept in sync with the
   state of the templates and the state of the Heat stack.

.. warning::

   When skipping tasks, the overcloud must be in the state expected by the task
   starting task. Meaning, the state of the overcloud should be the same as if
   all the skipped tasks had been applied. Otherwise, the result of the tasks
   that get executed will be undefined and could leave the cloud in an
   inoperable state.

   Likewise, the deployed cloud may not be left in its fully configured state
   if tasks are skipped at the end of the deployment.

Complete the :ref:`manual-config-download` steps to create the ansible project
directory, or use the existing project directory at
``/var/lib/mistral/<plan>``.

.. note::

    The project directory under ``/var/lib/mistral/<plan>`` is only updated
    by ``openstack overcloud deploy`` if the mistral workflow is used for
    ``config-download`` (e.g., ``--stack-only`` is **not** used).

Tags
^^^^
The playbooks use tagged tasks for finer-grained control of what to apply if
desired. Tags can be used with the ``ansible-playbook`` CLI arguments ``--tags`` or
``--skip-tags`` to control what tasks are executed. The enabled tags are:

facts
  fact gathering
common_roles
  ansible roles common to all nodes
overcloud
  all plays for overcloud deployment
pre_deploy_steps
  deployments that happen pre deploy_steps
host_prep_steps
  Host preparation steps
deploy_steps
  deployment steps
post_deploy_steps
  deployments that happen post deploy_steps
external
  all external deployments
external_deploy_steps
  external deployments that run on the undercloud

See :ref:`deploy_steps_playbook.yaml` for a description of which tags apply to
specific plays in the deployment playbook.

Server specific pre and post deployments
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The list of server specific pre and post deployments run during the `Server
deployments` and `Server Post Deployments` plays (see
:ref:`deploy_steps_playbook.yaml`) are dependent upon what custom roles and
templates are used with the deployment.

The list of these tasks are defined in an ansible group variable that applies
to each server in the inventory group named after the Heat role. From the
ansible project directory, the value can be seen within the group variable file
named after the Heat role::

    $ cat group_vars/Compute
    Compute_pre_deployments:
      - UpgradeInitDeployment
      - HostsEntryDeployment
      - DeployedServerBootstrapDeployment
      - InstanceIdDeployment
      - NetworkDeployment
      - ComputeUpgradeInitDeployment
      - ComputeDeployment
      - ComputeHostsDeployment
      - ComputeAllNodesDeployment
      - ComputeAllNodesValidationDeployment
      - ComputeHostPrepDeployment
      - ComputeArtifactsDeploy

    Compute_post_deployments:  []

``<Role>_pre_deployments`` is the list of pre deployments, and
``<Role>_post_deployments`` is the list of post deployments.

To specify the specific task to run for each deployment, the value of the
variable can be defined on the command line when running ``ansible-playbook``,
which will overwrite the value from the group variable file for that role.

For example::

    ansible-playbook \
      -e Compute_pre_deployments=NetworkDeployment \
      --tags pre_deploy_steps
      # other CLI arguments

Using the above example, only the task for the ``NetworkDeployment`` resource
would get applied since it would be the only value defined in
``Compute_pre_deployments``, and ``--tags pre_deploy_steps`` is also specified,
causing all other plays to get skipped.

Starting at a specific task
^^^^^^^^^^^^^^^^^^^^^^^^^^^
To start the deployment at a specific task, use the ``ansible-playbook`` CLI
argument ``--start-at-task``. To see a list of task names for a given playbook,
``--list-tasks`` can be used to list the task names.

.. note::

   Tasks that include the ``step`` variable or other ansible variables in the
   task name do not work with ``--start-at-task`` due to a limitation in
   ansible. For example the task with the name::

         Start containers for step 1

   won't work with ``--start-at-task`` since the step number is in the name
   (1).

Previewing changes
------------------
Changes can be previewed to see what will be changed before any changes are
applied to the overcloud. To preview changes, the stack update must be run with
the ``--stack-only`` cli argument::

    openstack overcloud deploy \
      --stack-only
      # other CLI arguments

Once the update is complete, complete the :ref:`manual-config-download` steps
to create the ansible project directory.

When ansible-playbook is run, use the ``--check`` CLI argument with
ansible-playbook to preview any changes. The extent to which changes can be
previewed is dependent on many factors such as the underlying tools in use
(puppet, docker, etc) and the support for ansible check mode in the given
ansible module.

The ``--diff`` option can aloso be used with ``--check`` to show the
differences that would result from changes.

See `Ansible Check Mode ("Dry Run")
<https://docs.ansible.com/ansible/2.5/user_guide/playbooks_checkmode.html>`_
for more details.

Generating overcloudrc
----------------------
In some cases, it may be required to manually generate the ``overcloudrc.v3``
file if ``ansible-playbook`` was used manually outside of the workflow.

The following command can be used to generate the ``overcloudrc.v3`` file. Save
the output of the command to the file where you want the contents saved::

    openstack action execution run \
      --save-result \
      --run-sync \
      tripleo.deployment.overcloudrc \
      '{"container":"overcloud"}' \
      | jq -r '.["result"]["overcloudrc.v3"]' \
      > overcloudrc.v3

If needed, substitute the name of the deployment for overcloud.

config-download with Heat SoftwareDeployment outputs
----------------------------------------------------
``config-download`` does not support outputs on Heat
SoftwareDeployment/SoftwareConfig resources. Often, ``deploy_steps_tasks`` can
be used to reproduce the same behavior that would be handled by an output, by
using Ansible tasks and the ``register`` keyword.
