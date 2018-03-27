Backing up the Undercloud
=========================

In order to backup your Undercloud you need to
make sure a set of files and databases are stored
correctly to be used in case of an issue running
the updates or upgrades workflows.

The following sections will describe how to
execute an Undercloud backup.

CLI driven backups
------------------

There is an automated way of creating an Undercloud backup,
this CLI option allows the operator to run a database and filesystem backup.
By default, all databases are included in the backup, also, the folder `/home/stack`.

The command usage is::

  openstack undercloud backup [--add-path ADD_FILES_TO_BACKUP]

For example, we can run a full MySQL backup with additional paths as::

  openstack undercloud backup --add-path /etc/hosts \
                              --add-path /var/log/ \
                              --add-path /var/lib/glance/images/ \
                              --add-path /srv/node/ \
                              --add-path /etc/

When executing the Undercloud backup via the OpenStack
CLI, the backup is stored in a temporary folder called
`/var/tmp/`.
After this operation, the result of the backup procedure
is stored in the swift container called `undercloud-backups`
and it will expire after 24 hours of its creation.

Manual backups
--------------

If the user needs to run the backup manually,
the following steps must be executed.

Database backups
~~~~~~~~~~~~~~~~

The operator needs to backup all databases in the Undercloud node::

  mysqldump --opt --single-transaction --all-databases > /root/undercloud-all-databases.sql

Filesystem backups
~~~~~~~~~~~~~~~~~~

* MariaDB configuration file on undercloud (so we can restore databases accurately).
* All glance image data in /var/lib/glance/images.
* All swift data in /srv/node.
* All data in stack users home directory.
* Also the DB backup created in the previous step.

The following command can be used to perform a backup of all data from the undercloud node::

  tar -czf undercloud-backup-`date +%F`.tar.gz /root/undercloud-all-databases.sql /etc/my.cnf.d/server.cnf /var/lib/glance/images /srv/node /home/stack /etc/pki /opt/stack


