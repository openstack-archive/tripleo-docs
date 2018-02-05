Creating Mistral workflows for the new python-tripleoclient CLI command
-----------------------------------------------------------------------

The next step is to define the
**tripleo.undercloud_backup.v1.prepare_environment** Mistral workflow,
all the Mistral workbooks, workflows and actions will be defined in the
**tripleo-common** repository.

Let’s go inside **tripleo-common**

::

    cd dev-docs
    cd tripleo-common

And see it’s content:

::

    [stack@undercloud tripleo-common]$ ls
    AUTHORS           doc                README.rst        test-requirements.txt
    babel.cfg         HACKING.rst        releasenotes      tools
    build             healthcheck        requirements.txt  tox.ini
    ChangeLog         heat_docker_agent  scripts           tripleo_common
    container-images  image-yaml         setup.cfg         undercloud_heat_plugins
    contrib           LICENSE            setup.py          workbooks
    CONTRIBUTING.rst  playbooks          sudoers           zuul.d

Again, we need to check the **setup.cfg** file. This file defines all the
Mistral actions we can call.
Specifically, we will need at the end of this file our new actions:

::

    tripleo.undercloud.get_free_space = tripleo_common.actions.undercloud:GetFreeSpace
    tripleo.undercloud.create_backup_dir = tripleo_common.actions.undercloud:CreateBackupDir
    tripleo.undercloud.create_database_backup = tripleo_common.actions.undercloud:CreateDatabaseBackup
    tripleo.undercloud.create_file_system_backup = tripleo_common.actions.undercloud:CreateFileSystemBackup
    tripleo.undercloud.upload_backup_to_swift = tripleo_common.actions.undercloud:UploadUndercloudBackupToSwift

4.1. Action definition
~~~~~~~~~~~~~~~~~~~~~~

Let’s take the first action to describe it’s definition,
**tripleo.undercloud.get_free_space =
tripleo_common.actions.undercloud:GetFreeSpace**

We have defined the action named as
**tripleo.undercloud.get_free_space** which will instantiate the class
**GetFreeSpace** defined in the file
**tripleo_common/actions/undercloud.py** file.

If we open **tripleo_common/actions/undercloud.py** we can see the class
definition as:

::

    class GetFreeSpace(base.Action):
        """"
        Get the Undercloud free space for the backup.
        The default path to check will be /var/tmp and the
        default minimum size will be 10240 MB (10GB).
        """"
        def __init__(self, min_space=10240):
            self.min_space = min_space
        def run(self, context):
            temp_path = tempfile.gettempdir()
            min_space = self.min_space
            while not os.path.isdir(temp_path):
                head, tail = os.path.split(temp_path)
                temp_path = head
            available_space = (
                (os.statvfs(temp_path).f_frsize * os.statvfs(temp_path).f_bavail) /
                (1024 * 1024))
            if (available_space < min_space):
                msg = "There is no enough space, avail. - %s MB" \
                      % str(available_space)
                return actions.Result(error={'msg': msg})
            else:
                msg = "There is enough space
