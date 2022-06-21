Extending overcloud nodes provisioning
======================================

Starting with the Queens release, the *ansible* deploy interface became
available in Ironic. Unlike the default `iSCSI deploy interface`_, it is
highly customizable through operator-provided Ansible playbooks. These
playbooks will run on the target image when Ironic boots the deploy ramdisk.

.. TODO(dtantsur): link to ansible interface docs when they merge

.. note::
    This feature is not related to the ongoing work of switching overcloud
    configuration to Ansible.

Enabling Ansible deploy
-----------------------

The *ansible* deploy interface is enabled by default starting with Queens.
However, additional configuration is required when installing an undercloud.

Custom ansible playbooks
~~~~~~~~~~~~~~~~~~~~~~~~

To avoid modifying playbooks, provided by the distribution, you must copy
them to a new location that is accessible by Ironic. In this guide it is
``/var/lib/ironic``.

.. note::
    Use of the ``/var/lib`` directory is not fully compliant to FHS. We do it
    because for containerized undercloud this directory is shared between
    the host and the ironic-conductor container.

#. Set up repositories and install the Ironic common package, if it is not
   installed yet::

    sudo yum install -y openstack-ironic-common

#. Copy the files to the new location (``/var/lib/ironic/playbooks``)::

    sudo cp -R /usr/lib/python2.7/site-packages/ironic/drivers/modules/ansible/playbooks/ \
        /var/lib/ironic

Installing undercloud
~~~~~~~~~~~~~~~~~~~~~

#. Generate an SSH key pair, for example::

    ssh-keygen -t rsa -b 2048 -f ~/ipa-ssh -N ''

   .. warning:: The private part should not be password-protected or Ironic
                will not be able to use it.

#. Create a custom hieradata override. Pass the **public** SSH key for the
   deploy ramdisk to the common PXE parameters, and set the new playbooks path.

   For example, create a file called ``ansible-deploy.yaml`` with the
   following content:

   .. code-block:: yaml

    ironic::drivers::ansible::default_username: 'root'
    ironic::drivers::ansible::default_key_file: '/var/lib/ironic/ipa-ssh'
    ironic::drivers::ansible::playbooks_path: '/var/lib/ironic/playbooks'
    ironic::drivers::pxe::pxe_append_params: 'nofb nomodeset vga=normal selinux=0 sshkey="<INSERT PUBLIC KEY HERE>"'

#. Link to this file in your ``undercloud.conf``:

   .. code-block:: ini

    hieradata_override=/home/stack/ansible-deploy.yaml

#. Deploy or update your undercloud as usual.

#. Move the private key to ``/var/lib/ironic`` and ensure correct ACLs::

    sudo mv ~/ipa-ssh /var/lib/ironic
    sudo chown ironic:ironic /var/lib/ironic/ipa-ssh
    sudo chmod 0600 /var/lib/ironic/ipa-ssh

Enabling temporary URLs
~~~~~~~~~~~~~~~~~~~~~~~

#. First, enable the ``admin`` user access to other Swift accounts::

    $ openstack role add --user admin --project service ResellerAdmin

#. Check if the ``service`` account has a temporary URL key generated in the
   Object Store service. Look for ``Temp-Url-Key`` properties in the output
   of the following command::

    $ openstack --os-project-name service object store account show
    +------------+---------------------------------------+
    | Field      | Value                                 |
    +------------+---------------------------------------+
    | Account    | AUTH_97ae97383424400d8ee1a54c3a2c41a0 |
    | Bytes      | 2209530996                            |
    | Containers | 5                                     |
    | Objects    | 42                                    |
    +------------+---------------------------------------+

#. If the property is not present, generate a value and add it::

    $ openstack --os-project-name service object store account set \
        --property Temp-URL-Key=$(uuidgen | sha1sum | awk '{print $1}')

Configuring nodes
-----------------

Nodes have to be explicitly configured to use the Ansible deploy. For example,
to configure all nodes, use::

    for node in $(baremetal node list -f value -c UUID); do
        baremetal node set $node --deploy-interface ansible
    done

Editing playbooks
-----------------

.. TODO(dtantsur): link to ansible interface docs when they merge

Example: kernel arguments
~~~~~~~~~~~~~~~~~~~~~~~~~

Let's modify the playbooks to include additional kernel parameters for some
nodes.

#. Update ``/var/lib/ironic/playbooks/roles/configure/tasks/grub.yaml`` from

   .. code-block:: yaml

      - name: create grub config
        become: yes
        command: chroot {{ tmp_rootfs_mount }} /bin/sh -c '{{ grub_config_cmd }} -o {{ grub_config_file }}'

   to

   .. code-block:: yaml

      - name: append kernel params
        become: yes
        lineinfile:
          dest: "{{ tmp_rootfs_mount }}/etc/default/grub"
          state: present
          line: 'GRUB_CMDLINE_LINUX+=" {{ ironic_extra.kernel_params | default("") }}"'
      - name: create grub config
        become: yes
        command: chroot {{ tmp_rootfs_mount }} /bin/sh -c '{{ grub_config_cmd }} -o {{ grub_config_file }}'

#. Set the newly introduced ``kernel_params`` extra variable to the desired
   kernel parameters. For example, to update only compute nodes use::

    for node in $(baremetal node list -c Name -f value | grep compute); do
        baremetal node set $node \
            --extra kernel_params='param1=value1 param2=value2'
    done

.. _iSCSI deploy interface: https://docs.openstack.org/ironic/latest/admin/interfaces/deploy.html#iscsi-deploy
