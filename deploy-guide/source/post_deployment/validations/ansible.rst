
Running validations using Ansible
---------------------------------

Validations
^^^^^^^^^^^

You can run the ``prep`` validations to verify the hardware. Later in
the process, the validations will be run by the undercloud processes.

However, the undercloud is not set up yet. You can install Ansible on
your local machine (that has SSH connectivity to the undercloud) and
validate the undercloud from there.

You need Ansible version 2 and the hostname/IP address of the
undercloud (referred to ``$UNDERCLOUD_HOST`` here):

.. code-block:: bash

  $ sudo yum install ansible
  $ git clone https://git.openstack.org/openstack/tripleo-validations
  $ cd tripleo-validations
  $ printf "[undercloud]\n$UNDERCLOUD_HOST" > hosts
  $ export ANSIBLE_STDOUT_CALLBACK=validation_output
  $ export ANSIBLE_CALLBACK_PLUGINS="${PWD}/callback_plugins"
  $ export ANSIBLE_ROLES_PATH="${PWD}/roles"
  $ export ANSIBLE_LOOKUP_PLUGINS="${PWD}/lookup_plugins"
  $ export ANSIBLE_LIBRARY="${PWD}/library"

Then get the ``prep`` validations:

.. code-block:: bash

  $ grep -l '^\s\+-\s\+prep' -r playbooks

And run them one by one:

.. code-block:: bash

  $ ansible-playbook -i hosts playbooks/validation-name.yaml

Or run them all in one shot:

.. code-block:: bash

  $ for PREP_VAL in `grep -l '^\s\+-\s\+prep' -r playbooks`; do echo "=== $PREP_VAL ==="; ansible-playbook -i hosts $PREP_VAL; done
