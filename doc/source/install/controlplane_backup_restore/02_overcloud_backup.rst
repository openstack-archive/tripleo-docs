Backing up the Overcloud control plane services
===============================================

This backup guide is meant to backup services based on a HA + containers deployment.

Prerequisites
-------------

There is a need to backup the control plane services in the Overcloud, to do so, we need
to apply the same approach from the Undercloud, which is, running a backup of the databases
and create a filesystem backup.

Databases backup
----------------

MySQL backup
~~~~~~~~~~~~

If using HA the operator can run the database backup in any controller node
using the ``--single-transaction`` option when executing the mysqldump.

If the deployment is using containers the hieradata file containing the mysql
root password is located in the folder `/var/lib/config-data/mysql/etc/puppet/hieradata/`.

The file containing the mysql root password is `service_configs.json` and the key is
`mysql::server::root_password`.

Create a temporary folder to store the backups::

  sudo -i
  mkdir -p /var/tmp/mysql_backup/

Store the MySQL root password to be added to further queries::

  MYSQLDBPASS=$(cat /var/lib/config-data/mysql/etc/puppet/hieradata/service_configs.json | grep mysql | grep root_password | awk -F": " '{print $2}' | awk -F"\"" '{print $2}')

Execute from any controller::

  mysql -uroot -p$MYSQLDBPASS -e "select distinct table_schema from information_schema.tables where engine='innodb' and table_schema != 'mysql';" \
        -s -N | xargs mysqldump -uroot -p$MYSQLDBPASS --single-transaction --databases > /var/tmp/mysql_backup/openstack_databases-`date +%F`-`date +%T`.sql

This will dump a database backup called /var/tmp/mysql_backup/openstack_databases-<date>.sql

Then backup all the users and permissions information::

  mysql -uroot -p$MYSQLDBPASS -e "SELECT CONCAT('\"SHOW GRANTS FOR ''',user,'''@''',host,''';\"') FROM mysql.user where (length(user) > 0 and user NOT LIKE 'root')" \
        -s -N | xargs -n1 mysql -uroot -p$MYSQLDBPASS -s -N -e | sed 's/$/;/' > /var/tmp/mysql_backup/openstack_databases_grants-`date +%F`-`date +%T`.sql

This will dump a database backup called `/var/tmp/mysql_backup/openstack_databases_grants-<date>.sql`

MongoDB backup (only needed until Ocata)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Since OpenStack Pike, there is no support for MongoDB, so be sure you backup the data from
your telemetry backend.

If telemetry services are used, then its needed to backup the data stored in the MongoDB instance.
Connect to any controller and get the IP of the MongoDB primary instance::

  MONGOIP=$(cat /etc/mongod.conf | grep bind_ip | awk '{print $3}')

Now, create the backup::

  mkdir -p /var/tmp/mongo_backup/
  mongodump --oplog --host $MONGOIP --out /var/tmp/mongo_backup/

Be sure the files were created successfully.

Redis backup
~~~~~~~~~~~~~~

If telemetry services are used, then it's needed to backup the data stored in the Redis instance.

Let's get the Redis endpoint to get the backup, open `/var/lib/config-data/haproxy/etc/haproxy/haproxy.cfg`
and get the bind IP in the `listen redis` section, should have a string of this form `bind <bind_IP:bind_port> transparent`::

  grep -A1 'listen redis' /var/lib/config-data/haproxy/etc/haproxy/haproxy.cfg
  REDISIP=$(grep -A1 'listen redis' /var/lib/config-data/haproxy/etc/haproxy/haproxy.cfg | grep bind | awk '{print $2}' | awk -F":" '{print $1}')

Let's store the master auth password to connect to the Redis cluster, the config file should be
`/var/lib/config-data/redis/etc/redis.conf` and the password under the `masterauth` parameter.
Let's store it in a variable::

  REDISPASS=$(cat /var/lib/config-data/redis/etc/redis.conf | grep masterauth | grep -v \# | awk '{print $2}')

Let's check connectivity to the Redis cluster::

  redis-cli -a $REDISPASS -h $REDISIP ping

Now, create a database dump by executing::

  redis-cli -a $REDISPASS -h $REDISIP bgsave

Now the database backup should be stored in the
default directory `/var/lib/redis/` directory.

Filesystem backup
-----------------

We need to backup all files that can be used to recover
from a possible failure in the Overcloud controllers when
executing a minor update or a major upgrade.

The option ``--ignore-failed-read`` is added to the `tar`
command because the list of files to backup might be
different on each environment and we make the list of
paths to backup is as much general as possible.

The following folders should be backed up::

  mkdir -p /var/tmp/filesystem_backup/
  tar --xattrs --ignore-failed-read \
      -zcvf /var/tmp/filesystem_backup/fs_backup-`date '+%Y-%m-%d-%H-%M-%S'`.tar.gz \
      /etc/nova \
      /var/log/nova \
      /var/lib/nova \
      --exclude /var/lib/nova/instances \
      /etc/glance \
      /var/log/glance \
      /var/lib/glance \
      /etc/keystone \
      /var/log/keystone \
      /var/lib/keystone \
      /etc/httpd \
      /etc/cinder \
      /var/log/cinder \
      /var/lib/cinder \
      /etc/heat \
      /var/log/heat \
      /var/lib/heat \
      /var/lib/heat-config \
      /var/lib/heat-cfntools \
      /etc/rabbitmq \
      /var/log/rabbitmq \
      /var/lib/rabbitmq \
      /etc/neutron \
      /var/log/neutron \
      /var/lib/neutron \
      /etc/corosync \
      /etc/haproxy \
      /etc/logrotate.d/haproxy \
      /var/lib/haproxy \
      /etc/openvswitch \
      /var/log/openvswitch \
      /var/lib/openvswitch \
      /etc/ceilometer \
      /var/lib/redis \
      /etc/sysconfig/memcached \
      /etc/gnocchi \
      /var/log/gnocchi \
      /etc/aodh \
      /var/log/aodh \
      /etc/panko \
      /var/log/panko \
      /etc/ceilometer \
      /var/log/ceilometer
