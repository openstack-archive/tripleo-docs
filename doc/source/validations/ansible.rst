
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
undercloud (referred to ``$UNDERCLOUD_HOST`` here)::

  $ sudo yum install ansible
  $ git clone https://git.openstack.org/openstack/tripleo-validations
  $ cd tripleo-validations
  $ printf "[undercloud]\n$UNDERCLOUD_HOST" > hosts

Then get the ``prep`` validations::

  $ grep -l '^\s\+-\s\+prep' -r playbooks

And run them one by one::

  $ ansible-playbook -i hosts playbooks/validation-name.yaml
