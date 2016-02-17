.. _introspection_data:

Accessing additional introspection data
---------------------------------------

Every introspection run (as described in
:doc:`../basic_deployment/basic_deployment_cli`) collects a lot of additional
facts about the hardware and puts them as JSON in Swift. Swift container name
is ``ironic-inspector`` and can be modified in
**/etc/ironic-inspector/inspector.conf**. Swift object name is stored under
``hardware_swift_object`` key in Ironic node extra field.

As an example, to download the swift data for all nodes to a local directory
and use that to collect a list of node mac addresses::

    # You will need the ironic-inspector user password
    # from /etc/ironic-inspector/inspector.conf:
    export IRONIC_INSPECTOR_PASSWORD=

    # Download the extra introspection data from swift:
    for node in $(ironic node-list | grep -v UUID| awk '{print $2}');
      do swift -U service:ironic -K $IRONIC_INSPECTOR_PASSWORD download ironic-inspector extra_hardware-$node;
    done

    # Use jq to access the local data - for example gather macs:
    for f in extra_hardware-*;
      do cat $f | jq -r 'map(select(.[0]=="network" and .[2]=="serial"))';
    done
