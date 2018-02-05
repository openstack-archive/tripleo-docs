Show how to test locally the changes in python-tripleoclient and tripleo-common
-------------------------------------------------------------------------------

The following procedures will allow you to test a
change in python-tripleoclient or tripleo-common locally.

For a change in **python-tripleoclient**, assuming you already have
downloaded the change you want to test, execute:

::

    cd python-tripleoclient
    sudo rm -Rf /usr/lib/python2.7/site-packages/tripleoclient*
    sudo rm -Rf /usr/lib/python2.7/site-packages/python_tripleoclient*
    sudo python setup.py clean --all install

For a change in **tripleo-common**, assuming you already have downloaded
the change you want to test, execute:

::

    cd tripleo-common
    sudo rm -Rf /usr/lib/python2.7/site-packages/tripleo_common*
    sudo python setup.py clean --all install
    sudo cp /usr/share/tripleo-common/sudoers /etc/sudoers.d/tripleo-common
    # this loads the actions via entrypoints
    sudo mistral-db-manage --config-file /etc/mistral/mistral.conf populate
    # make sure the new actions got loaded
    mistral action-list | grep tripleo
    for workbook in workbooks/*.yaml; do
        mistral workbook-create $workbook
    done

    for workbook in workbooks/*.yaml; do
        mistral workbook-update $workbook
    done
    sudo systemctl restart openstack-mistral-executor
    sudo systemctl restart openstack-mistral-engine

If we want to execute a Mistral action or a Mistral workflow you can
execute:

Examples about how to test Mistral actions independently:

::

    mistral run-action tripleo.undercloud.get_free_space #Without parameters
    mistral run-action tripleo.undercloud.get_free_space '{"path": "/etc/"}' # With parameters
    mistral run-action tripleo.undercloud.create_file_system_backup '{"sources_path": "/tmp/asdf.txt,/tmp/asdf", "destination_path": "/tmp/"}'

Examples about how to test a Mistral workflow independently:

::

    mistral execution-create tripleo.undercloud_backup.v1.prepare_environment # No parameters
    mistral execution-create tripleo.undercloud_backup.v1.filesystem_backup '{"sources_path": "/tmp/asdf.txt,/tmp/asdf", "destination_path": "/tmp/"}' # With parameters
