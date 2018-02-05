Creating a new OpenStack CLI command in python-tripleoclient (openstack undercloud backup)
------------------------------------------------------------------------------------------

The first action needed is to be able to create a new CLI command for
the OpenStack client. In this case, we are going to implement the
**openstack undercloud backup** command.

::

    cd dev-docs
    cd python-tripleoclient

Let’s list the files inside this folder:

::

    [stack@undercloud python-tripleoclient]$ ls
    AUTHORS           doc                            setup.py
    babel.cfg         LICENSE                        test-requirements.txt
    bindep.txt        zuul.d                         tools
    build             README.rst                     tox.ini
    ChangeLog         releasenotes                   tripleoclient
    config-generator  requirements.txt
    CONTRIBUTING.rst  setup.cfg

Once inside the **python-tripleoclient** folder we need to check the
**setup.cfg** file.
This file defines all the CLI commands for the Python
TripleO client.
Specifically, we will need at the end of this file our
new command definition:

::

    undercloud_backup = tripleoclient.v1.undercloud_backup:BackupUndercloud

This means that we have a new command defined as **undercloud backup**
that will instantiate the **BackupUndercloud** class defined in the file
**tripleoclient/v1/undercloud_backup.py**

For further details related to this class definition please go to the
`gerrit review`_.

Now, having our class defined we can call other methods to invoke
Mistral in this way:

::

    clients = self.app.client_manager

    files_to_backup = ','.join(list(set(parsed_args.add_files_to_backup)))

    workflow_input = {
        "sources_path": files_to_backup
    }
    output = undercloud_backup.prepare(clients, workflow_input)

So forth, we will call the **undercloud_backup.prepare** method defined
in the file **tripleoclient/workflows/undercloud_backup.py**
which will call the **tripleo.undercloud_backup.v1.prepare_environment**
Mistral workflow we are about to create:

::

    def prepare(clients, workflow_input):
        workflow_client = clients.workflow_engine
        tripleoclients = clients.tripleoclient
        with tripleoclients.messaging_websocket() as ws:
            execution = base.start_workflow(
                workflow_client,
                'tripleo.undercloud_backup.v1.prepare_environment',
                workflow_input=workflow_input
            )
            for payload in base.wait_for_messages(workflow_client, ws, execution):
                if 'message' in payload:
                    return payload['message']

In this case, we will create a loop within the tripleoclient and wait
until we receive a message from the Mistral workflow
**tripleo.undercloud_backup.v1.prepare_environment** that indicates if
the invoked workflow ended correctly.

.. _gerrit review: https://review.openstack.org/#/c/466213

