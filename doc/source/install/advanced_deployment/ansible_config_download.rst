Deploying with Ansible
======================

Summary
-------

Starting in the Queens release, it is possible to use Ansible to apply the
Overcloud configuration. Ansible can be used to replace the communication and
transport of the configuration deployment data between Heat and the Heat agent
(os-collect-config) on the Overcloud nodes.

Instead of os-collect-config running on each Overcloud node and polling for
deployment data from Heat, the Ansible control node applys the configuration by
running ansible-playbook with an Ansible inventory file and a set of playbooks
and tasks.

The Ansible control node (the node running ``ansible-playbook``) is the
Undercloud. The terms "control node" and Undercloud refer to the same node
where the Undercloud installation has been performed.

``config-download`` is the feature name that enables using Ansible in this
manner, and will often be used to refer to the method detailed in this
documentation.

At a high level, an additonal cli arg (``--config-download``) is passed to
``openstack overcloud deploy``, which enables the various pieces to use
Ansible in the manner detailed here.

Heat is still used in the typical fashion to create the stack and all of the
descendant resources. The same parameter values and environment files are
passed to Heat as usual. Heat also will still create any OpenStack service
resources as it usually does, such as Nova servers and Neutron networks and
ports.

The difference is that although Heat creates all the deployment data necessary
via SoftwareDeployment resources to perform the Overcloud installation and
configuration, it does not apply any of the deployments. The data is only made
available via the Heat API, and an additonal config-download workflow is
triggered after the Heat stack is completed that downloads all of the
deployment data from Heat.

Using the downloaded data, the workflow generates Ansible playbooks and tasks
that are then used by the Undercloud to complete the configuration of the
Overcloud.

This diagram details the overall sequence of how using config-download
completes an Overcloud deployment:

.. image:: ../_images/tripleo_ansible_arch.png
    :scale: 40%


Using config-download
---------------------

CLI
^^^
To enable deployment with Ansible and config-download pass the additional arg
to the deployment command::

    openstack overcloud deploy \
        <other cli args> \
        --config-download

After the normal deployment procedure of creating the Heat stack completes,
there will be additional output from the ``ansible-playbook`` command that
shows the progress and status of the Overcloud configuration.

If not successful, the error message along with the ``ansible-playbook`` play
recap should be shown.

When successful, the end of the output will look like the following example::

    PLAY RECAP *********************************************************************
    192.168.24.3               : ok=150  changed=43   unreachable=0    failed=0
    localhost                  : ok=1    changed=0    unreachable=0    failed=0

    Overcloud configuration completed.
    Overcloud Endpoint: http://192.168.24.8:5000/
    Overcloud rc file: /home/zuul/overcloudrc
    Overcloud Deployed

Review the ``PLAY RECAP`` which will show each host that is part of the
Overcloud and the grouped count of each task status.

Mistral workflow
^^^^^^^^^^^^^^^^
The Mistral workflow triggered by config-download is
``tripleo.deployment.v1.config_download_deploy``.

Ansible working directory
^^^^^^^^^^^^^^^^^^^^^^^^^
The workflow uses a working directory under ``/var/lib/mistral`` to store the generated
files needed to run ``ansible-playbook``.

The directory for a given execution of the workflow is
``/var/lib/mistral/<execution-id>``.

All of the files under ``/var/lib/mistral`` are owned by the mistral user and
readble by the mistral group. The interactive user account on the Undercloud
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

tripleo-ansible-inventory
  Ansible inventory file containing hosts and vars for all the Overcloud nodes.

ansible.log
  Log file from the last run of ``ansible-playbook``.

ansible.cfg
  Config file used when running ``ansible-playbook``.

ansible-playbook-command.sh
  Executable script that can be used to rerun ``ansible-playbook``.

ssh_private_key
  Private ssh key used to ssh to the Overcloud nodes.

The rest of the files are the actual Ansible playbooks, tasks, templates, and
vars to complete the deployment.


Reproducing ansible-playbook
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Once in the ``mistral`` working directory, simply run
``ansible-playbook-command.sh`` to reproduce the deployment::

    ./ansible-playbook-command.sh

Any additional arguments passed to this script will be passed unchanged to the
``ansible-playbook`` command::

    ./ansible-playbook-command.sh --check

Using this method it is possible to take advantage of Ansible features, such as
check mode (``--check``), limiting hosts (``--limit``), overriding variables
(``-e``), etc.

Tags
^^^^
The playbooks use tagged tasks for finer grained control of what to apply if
desired. The enabled tags are:

facts
  Run fact gathering
overcloud
  Run all plays for overcloud deployment
pre_deploy_steps
  Run deployments that happen pre deploy_steps
host_prep_steps
  Run host_prep_tasks
deploy_steps
  Run deploy_steps
post_deploy_steps
  Run deployments that happen post deploy_steps
external
  Run all external deployments
external_deploy_steps
  Run all external deployments
