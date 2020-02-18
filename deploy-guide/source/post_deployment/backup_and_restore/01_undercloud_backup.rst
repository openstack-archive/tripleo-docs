Backing up the Undercloud
=========================

In order to backup your Undercloud you need to
make sure a set of files and databases are stored
correctly to be used in case of an issue running
the updates or upgrades workflows.

The following sections will describe how to
execute an Undercloud backup.

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
you can continue with the backup tasks.

CLI driven backups
------------------

There is an automated way of creating an Undercloud backup,
this CLI option allows the operator to run a database and filesystem backup.
By default, all databases are included in the backup, also, the folder `/home/stack`.

The command usage is::

  openstack undercloud backup [--add-path ADD_FILES_TO_BACKUP] [--exclude-path EXCLUDE_FILES_TO_BACKUP]

For example, we can run a full MySQL backup with additional paths as::

  openstack undercloud backup --add-path /etc/ \
                              --add-path /var/log/ \
                              --add-path /root/ \
                              --add-path /var/lib/glance/ \
                              --add-path /var/lib/docker/ \
                              --add-path /var/lib/certmonger/ \
                              --add-path /var/lib/registry/ \
                              --add-path /srv/node/ \
                              --exclude-path /home/stack/

Note that we are excluding the folder `/home/stack/`
from the backup, but this folder is not included using the ``--add-path``,
CLI option, this is due to the fact that the `/home/stack/` folder is
added by default in any backup as it contains necessary files
to restore correctly the Undercloud.
You can exclude that folder and add specific files if you are required to
do so.

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

The operator needs to backup all databases in the Undercloud node

.. admonition:: Stein and Train
   :class: stable

   ::

    /bin/hiera -c /etc/puppet/hiera.yaml mysql::server::root_password
    podman exec mysql bash -c "mysqldump -uroot -pPASSWORD --opt --all-databases" > /root/undercloud-all-databases.sql

.. admonition:: Rocky
   :class: stable

   ::

    /bin/hiera -c /etc/puppet/hiera.yaml mysql::server::root_password
    docker exec mysql bash -c "mysqldump -uroot -pPASSWORD --opt --all-databases" > /root/undercloud-all-databases.sql

.. admonition:: Queens
   :class: stable

   ::

    mysqldump --opt --single-transaction --all-databases > /root/undercloud-all-databases.sql

Filesystem backups
~~~~~~~~~~~~~~~~~~

* MariaDB configuration file on undercloud (so we can restore databases accurately).
* All glance image data in /var/lib/glance/images.
* All swift data in /srv/node.
* All data in stack users home directory.
* Also the DB backup created in the previous step.

The following command can be used to perform a backup of all data from the undercloud node::

  sudo tar --xattrs --ignore-failed-read -cf \
      UC-backup-`date +%F`.tar \
      /root/undercloud-all-databases.sql \
      /etc \
      /var/log \
      /root \
      /var/lib/glance \
      /var/lib/docker \
      /var/lib/certmonger \
      /var/lib/registry \
      /srv/node \
      /home/stack
