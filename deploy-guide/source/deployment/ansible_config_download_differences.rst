.. _config_download_differences:

Ansible config-download differences
===================================
Starting with the Queens release, it is possible to use Ansible to apply the
overcloud configuration. In the Rocky release, this method is the new default
behavior.

The feature is fully documented at
:doc:`ansible_config_download`, while this page details
the differences to the deployer experience with config-download.

Ansible vs. os-collect-config
-----------------------------
Previously, TripleO used an agent running on each overcloud node called
``os-collect-config``. This agent periodically polled the undercloud Heat API for
software configuration changes that needed to be applied to the node.

``os-collect-config`` ran ``os-refresh-config`` and ``os-apply-config`` as
needed whenever new software configuration changes were detected. This model
is a **"pull"** style model given each node polled the Heat API and pulled changes,
then applied them locally.

With config-download, TripleO has switched to a **"push"** style model. Ansible
is run from a central control node which is the undercloud.
``ansible-playbook`` is run from the undercloud and software configuration
changes are pushed out to each overcloud node via ssh.

With the new model, ``os-collect-config``, ``os-refresh-config``, and
``os-apply-config`` are no longer used in a TripleO deployment. The
``os-collect-config`` service is now disabled by default and won't start on
boot.

.. note::

    Heat standalone software deployments still rely on ``os-collect-config``.
    They are a type of deployment that can be applied to overcloud nodes
    directly via Heat outside of the overcloud stack, and without having to do
    a full stack update of the overcloud stack.

    These types of deployments are **NOT** typically used when doing TripleO.

    However, if these deployments are being used in an environment to manage
    overcloud nodes, then the ``os-collect-config`` service must be started and
    enabled on the overcloud nodes where these types of deployments are
    applied.

    For reference, the Heat CLI commands that are used to create these types of
    deployments are::

        openstack software config create ...
        openstack software deployment create ...

    If these commands are not being used in the environment, then
    ``os-collect-config`` can be left disabled.

Deployment workflow
-------------------
The default workflow executed by ``openstack overcloud deploy`` takes care of
all the necessary changes when using config-download. In both the previous and
new workflows, ``openstack overcloud deploy`` (tripleoclient) takes care of
automating all the steps through Mistral workflow(s). Therefore, existing CLI
scripts that called ``openstack overcloud deploy`` will continue to work with
no changes.

It's important to recognize the differences in the workflow to aid in
understanding the deployment and operator experience. Previously, Heat was
responsible for:

#. (Heat) Creating OpenStack resources (Neutron networks, Nova/Ironic instances, etc)
#. (Heat) Creating software configuration
#. (Heat) Applying the created software configuration to the Nova/Ironic instances

With config-download, Heat is no longer responsible for the last item of
applying the created software configuration as ``ansible-playbook`` is used
instead.

Therefore, only creating the Heat stack for an overcloud is no longer all that
is required to fully deploy the overcloud. Ansible also must be run from the
undercloud to apply the software configuration, and do all the required tasks
to fully deploy an overcloud such as configuring services, bootstrap tasks, and
starting containers.

The new steps are summarized as:

#. (Heat) Creating OpenStack resources (Neutron networks, Nova/Ironic instances, etc)
#. (Heat) Creating software configuration
#. (tripleoclient) Enable tripleo-admin ssh user
#. (ansible) Applying the created software configuration to the Nova/Ironic instances

See :doc:`ansible_config_download` for details on the
tripleo-admin ssh user step.

Deployment CLI output
---------------------
During a deployment, the expected output from ``openstack overcloud deploy``
has changed. Output up to and including the stack create/update is similar to
previous releases. Stack events will be shown until the stack operation is
complete.

After the stack goes to ``CREATE_COMPLETE`` (or ``UPDATE_COMPLETE``), output
from the steps to enable the tripleo-admin user via ssh are shown.

.. include:: deployment_output.rst

.. include:: deployment_status.rst

.. include:: deployment_log.rst
