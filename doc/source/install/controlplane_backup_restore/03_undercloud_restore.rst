Restoring the Undercloud
========================

The following restore process assumes you are recovering from a failed Undercloud node where you have to reinstall it from scratch.
It assumes that the hardware layout is the same, and the hostname and Undercloud settings of the machine will be the same as well.
Once the machine is installed and is in a clean state, re-enable all the subscriptions/repositories needed to install and run TripleO.

Note that unless specified, all commands should run as the stack user.

NTP service
-----------

OpenStack services are time sensitive, users need to
be sure their environment have the time synchronized
correctly before proceeding with any backup task.

By default, both Undercloud and Overcloud should have
configured correctly the NTP service as there are
parameters specifically defined to manage this service.

The user is responsible to ensure that the Undercloud
restore is consistent in time. For example, a user
installs the Undercloud at the time 'm', then they deploy
the Undercloud and the Overcloud at the time 'n', and
they create an Undercloud backup at the time 'o'. When the user
restore the Undercloud it needs to be sure is restored
at a time later than 'o'. So, before and after restoring the Undercloud
node is important to have all the deployment with the time
updated and synchronized correctly.

In case this is done manually, execute:

::

  sudo yum install -y ntp
  sudo chkconfig ntpd on
  sudo service ntpd stop
  sudo ntpdate pool.ntp.org
  sudo service ntpd restart

After ensuring the environment have the time synchronized correctly
you can continue with the restore tasks.

Downloading automated Undercloud backups
----------------------------------------

If the user has executed the Undercloud backup from the
TripleO CLI, it will need to download it to a local folder
and from there execute the restore steps.

::

  # From the Undercloud
  source stackrc
  mkdir /var/tmp/test_bk_down
  cd /var/tmp/test_bk_down
  openstack container save undercloud-backups

Now, in the `restore_uc_backup` folder there must be a file with the
following naming convention `UC-backup-<timestamp>.tar`.

After getting the backup file and unzipping it in any
selected folder, the user can proceed with the Undercloud restore.

The following is an example of how to extract the Undercloud
backup content:

::

  sudo tar -xvf /var/tmp/test_bk_down/UC-backup-*.tar -C /var/tmp/test_bk_down || true

There, the user will have a tar file with the content of the file system backup
and another gz file with the content of the database backup.

The user can proceed to unzip the database
and filesystem backup by executing:

::

  sudo gunzip /var/tmp/test_bk_down/*.gz -c > /var/tmp/test_bk_down/all-databases.sql
  sudo tar -xvf /var/tmp/test_bk_down/filesystem-*.tar -C /var/tmp/test_bk_down

Restoring a backup of your Undercloud on a Fresh Machine
--------------------------------------------------------

Assuming that the user has a fresh installed Undercloud
node, the user is able to log in as the stack user, and
have the Backup restored in the folder
`/var/tmp/test_bk_down`, follow the next steps.

Syncronize the stack home directory, haproxy configuration,
certificates and hieradata with the backup content:

::

  sudo rsync -a /var/tmp/test_bk_down/home/stack/ /home/stack
  sudo rsync -a /var/tmp/test_bk_down/etc/haproxy/ /etc/haproxy/
  sudo rsync -a /var/tmp/test_bk_down/etc/pki/instack-certs/ /etc/pki/instack-certs/
  sudo mkdir -p /etc/puppet/hieradata/
  sudo rsync -a /var/tmp/test_bk_down/etc/puppet/hieradata/ /etc/puppet/hieradata/
  sudo rsync -a /var/tmp/test_bk_down/srv/node/ /srv/node/
  sudo rsync -a /var/tmp/test_bk_down/var/lib/glance/ /var/lib/glance/

If the user is using SSL, you need to refresh the CA certificate:

::

  sudo mkdir -p /etc/pki/instack-certs || true
  sudo cp /home/stack/undercloud.pem /etc/pki/instack-certs
  sudo cp /home/stack/cacert.pem /etc/pki/ca-trust/source/anchors/
  sudo cp /home/stack/overcloud-cacert.pem /etc/pki/ca-trust/source/anchors/
  sudo semanage fcontext -a -t etc_t "/etc/pki/instack-certs(/.*)?"
  sudo restorecon -R /etc/pki/instack-certs
  sudo update-ca-trust extract

Install the required packages with:

::

  sudo yum install -y mariadb mariadb-server python-tripleoclient

If you are using Pike and Ceph will be used in the Overcloud, install
ceph-ansible on the Undercloud:

::

  sudo yum install -y ceph-ansible

Restart MySQL:

::

  sudo systemctl restart mariadb

Allow restore big dump DB files:

::

  mysql -uroot -e"set global max_allowed_packet = 1073741824;"


Restore the DB backup:

::

  mysql -u root < /var/tmp/test_bk_down/all-databases.sql

Restart Mariadb to refresh the permissions from the backup file:

::

  sudo systemctl restart mariadb

Register the root password from the configuration file and clean
the DB password to be able to reinstall the Undercloud:

::

  oldpassword=$(sudo cat /var/tmp/test_bk_down/root/.my.cnf | grep -m1 password | cut -d'=' -f2 | tr -d "'")
  mysqladmin -u root -p$oldpassword password ''

Remove old user permisology if it exists, replace <node> with the host related to each user.

::

  mysql -e 'select host, user, password from mysql.user;'
  for i in ceilometer glance heat ironic keystone neutron nova mistral zaqar;do mysql -e "drop user $i@<node>" || true ;done
  mysql -e 'flush privileges'

We have to now install the swift and glance base packages, and then restore their data:

::

  sudo yum install -y openstack-glance openstack-swift
  # Restore data from the Backup to: srv/node and var/lib/glance/images
  # Confirm data is owned by correct user
  sudo chown -R swift: /srv/node
  sudo chown -R glance: /var/lib/glance/images

Finally, we rerun the Undercloud installation from the stack user, making sure to run it in the stack user home dir:

::

  # Double check hostname is correctly set in /etc/hosts
  openstack undercloud install

Reconnect the restored Undercloud to the Overcloud
--------------------------------------------------
Having completed the steps above, the Undercloud can be expected to automatically
restore its connection to the Overcloud. The nodes will continue to poll
Orchestration (heat) for pending tasks, using a simple HTTP request issued every
few seconds.
