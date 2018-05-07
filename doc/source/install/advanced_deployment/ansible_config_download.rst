.. _config_download:

Deploying with Ansible
======================

Summary
-------

Starting in the Queens release, it is possible to use Ansible to apply the
overcloud configuration. In the Rocky release, this method using Ansible is the
new default behavior.

Ansible is used to replace the communication and transport of the software
configuration deployment data between Heat and the Heat agent
(os-collect-config) on the overcloud nodes.

Instead of os-collect-config running on each overcloud node and polling for
deployment data from Heat, the Ansible control node applies the configuration
by running ``ansible-playbook`` with an Ansible inventory file and a set of
playbooks and tasks.

The Ansible control node (the node running ``ansible-playbook``) is the
undercloud. The terms "control node" and undercloud refer to the same node
where the undercloud installation has been performed.

``config-download`` is the feature name that enables using Ansible in this
manner, and will often be used to refer to the method detailed in this
documentation.

Heat is still used in the typical fashion to create the stack and all of the
OpenStack resources. The same parameter values and environment files are
passed to Heat as usual. Heat then creates any OpenStack service resources as
it usually does, such as Nova servers and Neutron networks and ports.

The difference is that although Heat creates all the deployment data necessary
via SoftwareDeployment resources to perform the overcloud installation and
configuration, it does not apply any of the software deployments. The data is
only made available via the Heat API, and an additional config-download
Mistral workflow is triggered after the Heat stack is completed that downloads
all of the deployment data from Heat.

Using the downloaded data, the workflow generates Ansible playbooks and tasks
that are then used by the undercloud to complete the configuration of the
overcloud.

This diagram details the overall sequence of how using config-download
completes an overcloud deployment:

.. image:: ../_images/tripleo_ansible_arch.png
    :scale: 40%


Deployment workflow
-------------------
Ansible and config-download will be used by default when ``openstack overcloud
deploy`` (tripleoclient) is run. All of the workflow steps are automated by
tripleoclient and Mistral workflow(s).

The workflow steps are summarized as:

#. (Heat) Creating OpenStack resources (Neutron networks, Nova/Ironic instances, etc)
#. (Heat) Creating software configuration
#. (tripleoclient) Enable tripleo-admin ssh user
#. (ansible) Applying the created software configuration to the Nova/Ironic instances

The deployment procedure starts by creating the Heat stack, which creates the
OpenStack resources and software configuration. Once the stack operation is
completed, tripleoclient creates a ``tripleo-admin`` user on each overcloud
node.

The following steps are done to create the ``tripleo-admin`` user:

#. Create temporary ssh keys on the undercloud
#. Use a deployer specified private ssh key (defaults to ``~/.ssh/id_rsa``) to
   connect to each overcloud node as a deployer specified user (defaults to
   ``heat-admin``) and adds the temporary public ssh key to
   ``~/.ssh/authorized_keys`` for that user.
#. Executes a Mistral workflow to create ``tripleo-admin`` on each node,
   passing as input the temporary private ssh key and ssh user to Mistral.
#. The workflow creates ``tripleo-admin`` and gives sudo permissions to the
   user, as well as creates and stores a new ssh keypair specific to
   ``tripleo-admin``. This keypair (private and public) are stored in the
   Mistral database.
#. After the completion of the workflow, the temporary ssh public key is
   deleted from ``~/.ssh/authorized_keys`` on each overcloud node, and the
   temporary keypair is then deleted from the undercloud.

To override the deployer specified ssh private key and user, there are cli args
available with ``openstack overcloud deploy``::

    --overcloud-ssh-user
    --overcloud-ssh-key

After ``tripleo-admin`` is created, ``ansible-playbook`` will be used to apply
the software configuration to the overcloud nodes.

.. include:: deployment_output.rst

.. include:: deployment_status.rst

.. include:: deployment_log.rst

config-download with deployed-server
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
When using ``config-download`` with :doc:`deployed-server <deployed_server>`
(pre-provisioned nodes), a ``HostnameMap`` parameter must be provided. Create
an environment file to define the parameter, and assign the node hostnames in
the parameter value. The following example shows a sample value::

  parameter_defaults:
    HostnameMap:
      overcloud-controller-0: controller-00-rack01
      overcloud-controller-1: controller-01-rack02
      overcloud-controller-2: controller-02-rack03
      overcloud-novacompute-0: compute-00-rack01
      overcloud-novacompute-1: compute-01-rack01
      overcloud-novacompute-2: compute-02-rack01

Write the contents to an environment file such as ``hostnamemap.yaml``, and
pass it the environment as part of the deployment command.

Mistral workflow
^^^^^^^^^^^^^^^^
The Mistral workflow that runs config-download and ``ansible-playbook`` is
``tripleo.deployment.v1.config_download_deploy``.

The workflow will create a working directory under ``/var/lib/mistral`` where
all of the ansible related files are stored.

Ansible working directory
_________________________
The workflow uses a working directory under ``/var/lib/mistral`` to store the generated
files needed to run ``ansible-playbook``.

The directory for a given execution of the workflow is
``/var/lib/mistral/<execution-id>``.

All of the files under ``/var/lib/mistral`` are owned by the mistral user and
readable by the mistral group. The interactive user account on the undercloud
can be granted read-only access to these files by adding the user to the
``mistral`` group::

    sudo usermod -a -G mistral $USER

Once a member of the ``mistral`` group, the contents of ``/var/lib/mistral``
can be browsed, examined, and ``ansible-playbook`` rerun if needed for
debugging purposes.

For example, to see the latest execution of
``tripleo.deployment.v1.config_download_deploy``, run::

    ls -ltr /var/lib/mistral

Change to the latest directory shown (example)::

    cd /var/lib/mistral/de35fb93-aa73-4535-9b71-c50011952969

Within this directory, all the files are present to rerun
``ansible-playbook``:

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

The rest of the files are the actual Ansible playbooks, tasks, templates, and
vars to complete the deployment.

Reproducing ansible-playbook
____________________________
Once in the ``mistral`` working directory, simply run
``ansible-playbook-command.sh`` to reproduce the deployment::

    ./ansible-playbook-command.sh

Any additional arguments passed to this script will be passed unchanged to the
``ansible-playbook`` command::

    ./ansible-playbook-command.sh --check

Using this method it is possible to take advantage of Ansible features, such as
check mode (``--check``), limiting hosts (``--limit``), overriding variables
(``-e``), etc.

ansible-playbook manual execution
---------------------------------
The Mistral workflow that runs config-download can be skipped when running
``openstack overcloud deploy`` by passing ``--stack-only``. This will cause
tripleoclient to only deploy the Heat stack.

If not using the Mistral workflow, the deployment data needs to be pulled from
Heat with a separate command and ``ansible-playbook`` run manually. This
enables more manual interaction and debugging.

These methods are described in the following sections.

Manual ansible-playbook
^^^^^^^^^^^^^^^^^^^^^^^
When using the ``--stack-only`` argument, the deployment data needs to be
downloaded from Heat and ``ansible-playbook`` run manually.

To manually download and generate all of the ansible playbook and deployment
data, use the ``openstack overcloud config download`` command::

    openstack overcloud config download \
      --name overcloud \
      --config-dir config-download

The ansible data will be generated under a directory called ``config-download``
based on the ``--config-dir`` CLI argument.

To generate an inventory file to use with ``ansible-playbook`` use
``tripleo-ansible-inventory``::

    tripleo-ansible-inventory \
      --ansible_ssh_user centos \
      --static-yaml-inventory inventory.yaml

The above example shows setting the ansible ssh user as ``centos``. This can be
changed depending on the environment.

The following illustrates an example execution of ``ansible-playbook``::

    ansible-playbook \
      -i inventory.yaml \
      --private-key /path/private/ssh/key \
      --become \
      config-download/deploy_steps_playbook.yaml

Adjust the command as needed for a given environment.

Ansible playbook structure
--------------------------
This section details the structure of the ``config-download`` generated
playbooks and tasks.

Playbooks
^^^^^^^^^
The following playbooks are generated with ``config-download``:

deploy_steps_playbook.yaml
  Used for deployment and stack updates. Not for minor updates or major
  upgrades.
update_steps_playbook.yaml
  Used for minor updates.

Tags
^^^^
The playbooks use tagged tasks for finer grained control of what to apply if
desired. The enabled tags are:

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
