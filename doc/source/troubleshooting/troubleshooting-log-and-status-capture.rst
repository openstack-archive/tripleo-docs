Performing Log and Status Capture
---------------------------------

The tripleoclient provides commands to allow operators to run sosreport on the
overcloud nodes and download the log and status log bundles with tripleoclient.
This can aide with troubleshooting problems as the results can be sent to an
external support for analysis. The `openstack overcloud support report
collect` command can be used to execute sosreport on select (or all) overcloud
nodes, upload the logs to swift running on the undercloud, and download the
logs to the host that the command is executed from.


Example: Download logs from all controllers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The required `server_name` option for the command can be a paritial name
match for the overcloud nodes. This means `openstack overcloud support report
colect controller` will match all the overcloud nodes that contain the word
`controller`.  To download the run the command and download them to a local
directory, run the following command::

    openstack overcloud support report collect controller

.. note:: By default if -o is not specified, the logs will be downloaded to a folder
          in the current working directory called `support_logs`


Example: Download logs from a single host
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To download logs from a specific host, you must specify the complete name as
reported by `openstack service list` from the undercloud::

    openstack overcloud support report collect -o /home/stack/logs overcloud-novacompute-0


Example: Leave logs in a swift container
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you want to perform a sosreport but do not currently wish to download the
logs, you can leave them in a swift container for later retrieval. The
`--collect-only` and `-c` options can be leveraged to store the
logs in a swift container. For example::

    openstack overcloud support report collect -c logs_20170601 --collect-only controller

This will run sosreport on the nodes and upload the logs to a container named
`logs_20170601` on the undercloud. From which standard swift tooling can be
used to download the logs. Alternatively, you can then fetch the logs using
the `openstack overcloud support report collect` command by running::

    openstack overcloud support report collect -c logs_20170601 --download-only -o /tmp/mylogs controller

.. note:: There is a `--skip-container-delete` option that can be used if you
          want to leave the logs in swift but still download them. This option
          is ignored if `--collect-only` or `--download-only` options are
          provided.


Additional Options
^^^^^^^^^^^^^^^^^^

The `openstack overcloud support report collect` command has additional
that can be passed to work with the log bundles. Run the command with
`--help` to see additional options::

    openstack overcloud support report collect --help


