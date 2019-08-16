Restoring the Overcloud control plane services
==============================================

Restoring the Overcloud control plane from a failed state
depends on the specific issue the operator is facing.

This section provides a restore method for
the backups created in previous steps.

The general strategy of restoring an Overcloud control plane
will be to have the services working back again to
re-run the update/upgrade tasks.

YUM update rollback
-------------------

Depending on the updated packages, running a yum rollback
based on the `yum history` command might not be a good idea.
In the specific case of an OpenStack minor update or a major upgrade
will be harder as there will be several dependencies and packages
to downgrade based on the number of transactions yum had to run to upgrade
all the node packages.
Also, using `yum history` to rollback transactions
can lead to target to remove packages needed for the
system to work correctly.


Database restore
----------------

In the case we have updated the packages correctly, and the user has an
issue with updating the database schemas, we might need to restore the
database cluster.

With all the services stoped in the Overcloud controllers (except MySQL), go through
the following procedure:

On all the controller nodes, drop connections to the database port via the VIP by running::

  MYSQLIP=$(grep -A1 'listen mysql' /var/lib/config-data/haproxy/etc/haproxy/haproxy.cfg | grep bind | awk '{print $2}' | awk -F":" '{print $1}')
  sudo /sbin/iptables -I INPUT -d $MYSQLIP -p tcp --dport 3306 -j DROP

This will isolate all the MySQL traffic to the nodes.

On only one controller node, unmanage galera so that it is out of pacemaker's control::

  pcs resource unmanage galera

Remove the wsrep_cluster_address option from `/var/lib/config-data/mysql/etc/my.cnf.d/galera.cnf`.
This needs to be executed on all nodes::

  grep wsrep_cluster_address /var/lib/config-data/mysql/etc/my.cnf.d/galera.cnf
  vi /var/lib/config-data/mysql/etc/my.cnf.d/galera.cnf

On all the controller nodes, stop the MariaDB database::

  mysqladmin -u root shutdown

On all the controller nodes, move existing MariaDB data directories and prepare new data directories::

  sudo -i
  mv /var/lib/mysql/ /var/lib/mysql.old
  mkdir /var/lib/mysql
  chown mysql:mysql /var/lib/mysql
  chmod 0755 /var/lib/mysql
  mysql_install_db --datadir=/var/lib/mysql --user=mysql
  chown -R mysql:mysql /var/lib/mysql/
  restorecon -R /var/lib/mysql

On all the controller nodes, move the root configuration to a backup file::

  sudo mv /root/.my.cnf /root/.my.cnf.old
  sudo mv /etc/sysconfig/clustercheck /etc/sysconfig/clustercheck.old

On the controller node we previously set to `unmanaged`, bring the galera cluster up with pacemaker::

  pcs resource manage galera
  pcs resource cleanup galera

Wait for the galera cluster to come up properly and run the following
command to wait and see all nodes set as masters as follows::

  pcs status | grep -C3 galera
  # Master/Slave Set: galera-master [galera]
  # Masters: [ overcloud-controller-0 overcloud-controller-1 overcloud-controller-2 ]

NOTE: If the cleanup does not show all controller nodes as masters, re-run the following command::

  pcs resource cleanup galera

On the controller node we previously set to `unmanaged` which is managed back
by pacemaker, restore the OpenStack database that was backed up in a previous section.
This will be replicated to the other controllers by Galera::

  mysql -u root < openstack_database.sql

On the same controller node, restore the users and permissions::

  mysql -u root < grants.sql

Pcs status will show the galera resource in error because it's now using the wrong user/password to connect to poll the database status.
On all the controller nodes, restore the root/clustercheck configuration to a backup file::

  sudo mv /root/.my.cnf.old /root/.my.cnf
  sudo mv /etc/sysconfig/clustercheck.old /etc/sysconfig/clustercheck

Test the clustercheck locally for each controller node::

  /bin/clustercheck

Perform a cleanup in pacemaker to reprobe the state of the galera nodes::

  pcs resource cleanup galera

Test clustercheck on each controller node via xinetd.d::

  curl overcloud-controller-0:9200
  # curl overcloud-controller-1:9200
  # curl overcloud-controller-2:9200

Remove the iptables rule from each node for the services to restore access to the database::

  sudo /sbin/iptables -D INPUT -d $MYSQLIP -p tcp --dport 3306 -j DROP

Filesystem restore
------------------

On all overcloud nodes, copy the backup tar file to a temporary
directory and uncompress all the data::

  mkdir /var/tmp/filesystem_backup/data/
  cd /var/tmp/filesystem_backup/data/
  mv <path_to_the_backup_file> .
  tar --xattrs -xvzf <backup_file>.tar.gz

NOTE: Untarring directly on the / directory will
override your current files. Its recommended to
untar the file in a different directory.

Cleanup the redis resource
--------------------------

Run::

  pcs resource cleanup redis

Start up the services on all the controller nodes
-------------------------------------------------

The operator must check that all services are starting correctly,
the services installed in the controllers depend on the operator
needs so the following commands might not apply completely.
The goal of this section is to show that all services must be
started correctly before proceeding to retry an update, upgrade or
use the Overcloud on a regular basis.

Non containerized environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Command to start services::

  sudo -i ;systemctl start openstack-ceilometer-central; systemctl start memcached; pcs resource enable rabbitmq; systemctl start openstack-nova-scheduler; systemctl start openstack-heat-api; systemctl start mongod; systemctl start redis; systemctl start httpd; systemctl start neutron-ovs-cleanup

Once all the controller nodes are up, start the compute node services on all the compute nodes::

  sudo -i; systemctl start openstack-ceilometer-compute.service; systemctl start openstack-nova-compute.service

Containerized environment
~~~~~~~~~~~~~~~~~~~~~~~~~

The operator must check all containerized services are running correctly, please identify those stopped services by running::

  sudo docker ps

Once the operator finds a stopped service, proceed to start it by running::

  sudo docker start <service name>





