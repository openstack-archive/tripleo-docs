.. _introspection_data:

Accessing Introspection Data
----------------------------

Every introspection run (as described in
:doc:`../deployment/install_overcloud`) collects a lot of facts about
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
``undercloud.conf`` (enabled by default starting with the Mitaka release),
then even more data is available.

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
    for node in $(openstack baremetal node list -f value -c UUID);
      do swift -U service:ironic -K $IRONIC_INSPECTOR_PASSWORD download ironic-inspector extra_hardware-$node;
    done

    # Use jq to access the local data - for example gather macs:
    for f in extra_hardware-*;
      do cat $f | jq -r 'map(select(.[0]=="network" and .[2]=="serial"))';
    done

Running benchmarks
~~~~~~~~~~~~~~~~~~

Benchmarks for CPU, memory and hard drive can be run during the introspection
process. However, they are time consuming, and thus are disabled by default.
To enable benchmarks set ``inspection_runbench`` to ``true`` in the
``undercloud.conf`` (also requires ``inspection_extras`` set to ``true``),
then (re)run ``openstack undercloud install``.

Extra data examples
~~~~~~~~~~~~~~~~~~~

Here is an example of CPU extra data, including benchmark results::

    $ openstack baremetal introspection data save <UUID> | jq '.extra.cpu'
    {
        "physical": {
            "number": 1
        },
        "logical": {
            "number": 1,
            "loops_per_sec": 636
        },
        "logical_0": {
            "bandwidth_4K": 3657,
            "bandwidth_1G": 6775,
            "bandwidth_128M": 8353,
            "bandwidth_2G": 7221,
            "loops_per_sec": 612,
            "bogomips": "6983.57",
            "bandwidth_1M": 10781,
            "bandwidth_16M": 9808,
            "bandwidth_1K": 1204,
            "cache_size": "4096KB"
        },
        "physical_0":
        {
            "physid": 400,
            "product": "QEMU Virtual CPU version 2.3.0",
            "enabled_cores": 1,
            "vendor": "Intel Corp.",
            "threads": 1,
            "flags": "fpu fpu_exception wp de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pse36 clflush mmx fxsr sse sse2 syscall nx x86-64 rep_good nopl pni cx16 x2apic hypervisor lahf_lm abm",
            "version": "RHEL 7.2.0 PC (i440FX + PIIX, 1996)",
            "frequency": 2e+09,
            "cores": 1
        }
    }

Here is an example of disk extra data, including benchmark results::

    $ openstack baremetal introspection data save <UUID> | jq '.extra.disk'
    {
        "logical": {
            "count": 1
        },
        "sda": {
            "SMART/Raw_Read_Error_Rate(1)/value": 100,
            "SMART/Spin_Up_Time(3)/thresh": 0,
            "model": "QEMU HARDDISK",
            "SMART/Power_Cycle_Count(12)/when_failed": "NEVER",
            "SMART/Reallocated_Sector_Ct(5)/worst": 100,
            "SMART/Power_Cycle_Count(12)/raw": 0,
            "standalone_read_1M_KBps": 1222758,
            "SMART/Power_On_Hours(9)/worst": 100,
            "Read Cache Disable": 0,
            "SMART/Power_On_Hours(9)/raw": 1,
            "rotational": 1,
            "SMART/Start_Stop_Count(4)/thresh": 20,
            "SMART/Start_Stop_Count(4)/raw": 100,
            "SMART/Power_Cycle_Count(12)/thresh": 0,
            "standalone_randread_4k_KBps": 52491,
            "physical_block_size": 512,
            "SMART/Reallocated_Sector_Ct(5)/value": 100,
            "SMART/Reallocated_Sector_Ct(5)/when_failed": "NEVER",
            "SMART/Power_Cycle_Count(12)/value": 100,
            "SMART/Spin_Up_Time(3)/when_failed": "NEVER",
            "size": 44,
            "SMART/Power_On_Hours(9)/thresh": 0,
            "id": "ata-QEMU_HARDDISK_QM00005",
            "SMART/Reallocated_Sector_Ct(5)/raw": 0,
            "SMART/Raw_Read_Error_Rate(1)/when_failed": "NEVER",
            "SMART/Airflow_Temperature_Cel(190)/worst": 69,
            "SMART/Airflow_Temperature_Cel(190)/when_failed": "NEVER",
            "SMART/Spin_Up_Time(3)/value": 100,
            "standalone_read_1M_IOps": 1191,
            "SMART/Airflow_Temperature_Cel(190)/thresh": 50,
            "SMART/Power_On_Hours(9)/when_failed": "NEVER",
            "SMART/firmware_version": "2.3.0",
            "optimal_io_size": 0,
            "SMART/Raw_Read_Error_Rate(1)/thresh": 6,
            "SMART/Raw_Read_Error_Rate(1)/raw": 0,
            "SMART/Raw_Read_Error_Rate(1)/worst": 100,
            "SMART/Power_Cycle_Count(12)/worst": 100,
            "standalone_randread_4k_IOps": 13119,
            "rev": 0,
            "SMART/Start_Stop_Count(4)/worst": 100,
            "SMART/Start_Stop_Count(4)/when_failed": "NEVER",
            "SMART/Spin_Up_Time(3)/worst": 100,
            "SMART/Reallocated_Sector_Ct(5)/thresh": 36,
            "SMART/device_model": "QEMU HARDDISK",
            "SMART/Airflow_Temperature_Cel(190)/raw": " 31 (Min/Max 31/31)",
            "SMART/Start_Stop_Count(4)/value": 100,
            "SMART/Spin_Up_Time(3)/raw": 16,
            "Write Cache Enable": 1,
            "vendor": "ATA",
            "SMART/serial_number": "QM00005",
            "SMART/Power_On_Hours(9)/value": 100,
            "SMART/Airflow_Temperature_Cel(190)/value": 69
        }
    }

.. _python-hardware: https://github.com/redhat-cip/hardware
