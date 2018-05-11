Restoring the Undercloud
========================

The following restore process assumes you are recovering from a failed Undercloud node where you have to reinstall it from scratch.
It assumes that the hardware layout is the same, and the hostname and Undercloud settings of the machine will be the same as well.
Once the machine is installed and is in a clean state, re-enable all the subscriptions/repositories needed to install and run TripleO.

Note that unless specified, all commands are run as root.

Downloading automated Undercloud backups
----------------------------------------

If the user has executed the Undercloud backup from the
TripleO CLI, it will need to download it to a local folder
and from there execute the restore steps.

::

  # From the Undercloud
  source stackrc
  mkdir restore_uc_backup
  cd restore_uc_backup
  openstack container save undercloud-backups

Now, in the `restore_uc_backup` folder there must be a file with the
following naming convention `UC-backup-<timestamp>.tar`.

After getting the backup file and unzipping it in any
selected folder, the user can proceed with the Undercloud restore.

The following is an example of how to download and extract the Undercloud
backup content:

::

  tar -xvf UC-backup-<timestamp>.tar

There, the user will have a tar file with the content of the file system backup
and another gz file with the content of the database backup.

The user can proceed to unzip the database backup by executing::

  gunzip all-databases-<timestamp>.sql.gz

Restoring a backup of your Undercloud on a Fresh Machine
--------------------------------------------------------

The first step is to unzip the Undercloud backup to a temporary folder,
then install the MariaDB server with::

  yum install -y mariadb-server

Restore the MariaDB configuration file,
noticing that depending on the MySQL version, the config file can
be `/etc/my.cnf.d/mariadb-server.cnf` or `/etc/my.cnf.d/server.cnf`.
Also, edit /etc/my.cnf.d/server.cnf and comment out 'bind-address'.

Start the MariaDB server and load the backup following this example::

  systemctl start mariadb
  cat /root/undercloud-all-databases.sql | mysql
  # Now we need to clean out some old permissions to be recreated
  for i in ceilometer glance heat ironic keystone neutron nova;do mysql -e "drop user $i";done
  mysql -e 'flush privileges'

Now create the stack user and restore the stack users home directory::

  sudo useradd stack
  sudo passwd stack  # specify a password

  echo "stack ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/stack
  sudo chmod 0440 /etc/sudoers.d/stack

Next restore the stack users home directory from the Undercloud backup.

We have to now install the swift and glance base packages, and then restore their data::

  yum install -y openstack-glance openstack-swift
  # Restore data from the Backup to: srv/node and var/lib/glance/images
  # Confirm data is owned by correct user
  chown -R swift: /srv/node
  chown -R glance: /var/lib/glance/images

Finally, we rerun the Undercloud installation from the stack user, making sure to run it in the stack user home dir::

  su - stack
  sudo yum install -y python-tripleoclient
  # Double check hostname is correctly set in /etc/hosts
  openstack install undercloud

If you are using Pike and Ceph will be used in the overcloud, install
ceph-ansible on the Undercloud::

  sudo yum install -y ceph-ansible


Reconnect the restored Undercloud to the overcloud
--------------------------------------------------
Having completed the steps above, the Undercloud can be expected to automatically
restore its connection to the overcloud. The nodes will continue to poll
Orchestration (heat) for pending tasks, using a simple HTTP request issued every
few seconds.
