Give elevated privileges to specific Mistral actions that need to run with elevated privileges.
-----------------------------------------------------------------------------------------------

Sometimes it is not possible to execute some restricted actions from
the mistral user, for example, when creating the Undercloud backup we
won’t be able to access the **/home/stack/** folder to create a tarball
of it. For this cases it’s possible to execute elevates actions from the
mistral user:

This is the content of the **sudoers** file in the root of the
**tripleo-common** `repository`_
at the time of the creation of this guide.

::

    Defaults!/usr/bin/run-validation !requiretty
    Defaults:validations !requiretty
    Defaults:mistral !requiretty
    mistral ALL = (validations) NOPASSWD:SETENV: /usr/bin/run-validation
    mistral ALL = NOPASSWD: /usr/bin/chown -h validations\: /tmp/validations_identity_[A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_], \
            /usr/bin/chown validations\: /tmp/validations_identity_[A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_], \
            !/usr/bin/chown /tmp/validations_identity_* *, !/usr/bin/chown /tmp/validations_identity_*..*
    mistral ALL = NOPASSWD: /usr/bin/rm -f /tmp/validations_identity_[A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_][A-Za-z0-9_], \
            !/usr/bin/rm /tmp/validations_identity_* *, !/usr/bin/rm /tmp/validations_identity_*..*
    mistral ALL = NOPASSWD: /bin/nova-manage cell_v2 discover_hosts *
    mistral ALL = NOPASSWD: /usr/bin/tar --ignore-failed-read -C / -cf /tmp/undercloud-backup-*.tar *
    mistral ALL = NOPASSWD: /usr/bin/chown mistral. /tmp/undercloud-backup-*/filesystem-*.tar
    validations ALL = NOPASSWD: ALL

Here you can grant permissions for specific tasks in when executing
Mistral workflows from **tripleo-common**

.. _repository: https://github.com/openstack/tripleo-common/blob/63ab54411e56ad0e70e5e145fcb0ce60a55eb3f8/sudoers
