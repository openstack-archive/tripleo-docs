Undercloud backups
------------------

Most of the Undercloud backup procedure is available in the `TripleO
official documentation site`_.

We will focus on the automation of backing up the resources required to
restore the Undercloud in case of a failed upgrade.

-  All MariaDB databases on the undercloud node
-  MariaDB configuration file on undercloud (so we can restore databases
   accurately)
-  All glance image data in /var/lib/glance/images
-  All swift data in /srv/node
-  All data in stack user home directory

For doing this we need to be able to:

-  Connect to the database server as root.
-  Dump all databases to file.
-  Create a filesystem backup of several folders (and be able to access
   folders with restricted access).
-  Upload this backup to a swift container to be able to get it from the
   TripleO web UI.

.. _TripleO official documentation site: https://docs.openstack.org/tripleo-docs/latest/install/post_deployment/backup_restore_undercloud.html
