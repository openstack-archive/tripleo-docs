Debugging actions
-----------------

Let’s assume the action is written, added to setup.cfg but
is still not being invoked as expected in the CLI.
Firstly, check if the action was added by
``sudo mistral-db-manage populate``. Run

::

    mistral action-list -f value -c Name | grep -e '^tripleo.undercloud'

If you don’t see your actions check output of
``sudo mistral-db-manage populate`` as

::

    sudo mistral-db-manage populate 2>&1| grep ERROR | less

The following output may indicate issues in code. Simply fix code.

::

    2018-01-01:00:59.730 7218 ERROR stevedore.extension [-] Could not load 'tripleo.undercloud.get_free_space': unexpected indent (undercloud.py, line 40):   File "/usr/lib/python2.7/site-packages/tripleo_common/actions/undercloud.py", line 40

Execute a single action or execute a workflow from an existing
workbook to make sure it works as expected, for example:


Run a single Mistral action.

::

    mistral run-action tripleo.undercloud.get_free_space '{"path": "/etc/"}'

    mistral run-action tripleo.undercloud.create_file_system_backup '{"sources_path": "/tmp/asdf.txt,/tmp/asdf", "destination_path": "/tmp/"}'

Run a single Mistral workflow.

::

    mistral execution-create tripleo.undercloud_backup.v1.filesystem_backup '{"sources_path": "/tmp/asdf.txt,/tmp/asdf", "destination_path": "/tmp/"}'
