.. _introspection_data:

Accessing Introspection Data
----------------------------

Every introspection run (as described in
:doc:`../basic_deployment/basic_deployment_cli`) collects a lot of facts about
the hardware and puts them as JSON in Swift. Starting with
``python-ironic-inspector-client`` version 1.4.0 there is a command to retrieve
this data::

    openstack baremetal introspection data save <UUID>

You can provide a ``--file`` argument to save the data in a file instead of
displaying it.

If you don't have a new enough version of ``python-ironic-inspector-client``,
you can use cURL to access the API::

    token=$(openstack token issue -f value -c id)
    curl -H "X-Auth-Token: $token" http://127.0.0.1:5050/v1/introspection/<UUID>/data

Accessing raw additional data
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Extra hardware data can be collected using the python-hardware_ library. If
you have enabled this, by setting ``inspection_extras`` to ``True`` in your
``undercloud.conf``, then even more data is available.

The command above will display it in a structured format under the ``extra``
key in the resulting JSON object. This format is suitable for using in
the **ironic-inspector** introspection rules (see e.g.
:ref:`auto-profile-tagging`). However, if you want to access it in its
original format (list of lists instead of nested objects), you can query
Swift for it directly.

The Swift container name is ``ironic-inspector``, which can be modified in
**/etc/ironic-inspector/inspector.conf**. The Swift object is called
``extra_hardware-<UUID>`` where ``<UUID>`` is a node UUID. In the default
configuration you have to use the ``service`` tenant to access this object.

As an example, to download the Swift data for all nodes to a local directory
and use that to collect a list of node mac addresses::

    # You will need the ironic-inspector user password
    # from the [swift] section of /etc/ironic-inspector/inspector.conf:
    export IRONIC_INSPECTOR_PASSWORD=xxxxxx

    # Download the extra introspection data from swift:
    for node in $(ironic node-list | grep -v UUID | awk '{print $2}');
      do swift -U service:ironic -K $IRONIC_INSPECTOR_PASSWORD download ironic-inspector extra_hardware-$node;
    done

    # Use jq to access the local data - for example gather macs:
    for f in extra_hardware-*;
      do cat $f | jq -r 'map(select(.[0]=="network" and .[2]=="serial"))';
    done


.. _python-hardware: https://github.com/redhat-cip/hardware
