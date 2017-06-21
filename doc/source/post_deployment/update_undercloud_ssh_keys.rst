Updating undercloud user's ssh key
==================================

In order to update the ssh key for the user on the undercloud, a few steps must
be done to ensure you do not lock yourself out of the overcloud nodes.  When
the undercloud is installed, an ssh key is created and added to Nova running
on the undercloud for provisioning the overcloud nodes. This key is uploaded
into Nova as the `default` keypair.  To view the keypair run::

    source stackrc
    openstack keypair list

Process to rotate ssh key
^^^^^^^^^^^^^^^^^^^^^^^^^

The process to rotate the user key is as follows:

1. Generate new key and do not replace the existing key. For example::

    ssh-keygen -t rsa -N '' -f ~/new_ssh_key

2. Copy ssh key to all existing hosts for the heat-admin user::

    for HOST in $(openstack server list -f value -c Networks | sed -e 's/ctlplane=//'); do
        ssh-copy-id -i ~/new_ssh_key heat-admin@$HOST
    done

3. Update the Undercloud's Nova default keypair::

    openstack keypair delete default
    openstack keypair create --public-key ~/new_ssh_key.pub default

4. Backup old key and replace it with the new keys::

    mkdir ~/.ssh/old_keys
    mv ~/.ssh/id_rsa ~/.ssh/old_keys/id_rsa.backup-$(date +'%Y-%m-%d')
    mv ~/.ssh/id_rsa.pub ~/.ssh/old_keys/id_rsa.pub.backup-$(date +'%Y-%m-%d')
    mv ~/new_ssh_key ~/.ssh/id_rsa
    mv ~/new_ssh_key.pub ~/.ssh/id_rsa.pub

5. Remove old key from the allowed hosts on the nodes.
