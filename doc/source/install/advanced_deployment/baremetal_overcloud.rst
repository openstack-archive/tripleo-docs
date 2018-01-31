Bare Metal Instances in Overcloud
=================================

This documentation explains installing Ironic for providing bare metal
instances in the overcloud to end users. This feature is supported starting
with Newton.

Architecture and requirements
-----------------------------

By default, TripleO installs ironic API and conductor services on the
controller nodes. In an HA configuration the 3 conductor services form a hash
ring and balance the nodes across it. For a really big bare metal cloud it's
highly recommended to move ironic-conductor services to separate roles, use
the `IronicConductor role shipped with TripleO`_ as an example.

.. note::
    Ironic services and API in the overcloud and in the undercloud are
    completely independent.

It is recommended to have at least 12 GiB of RAM on the undercloud and
controllers. The controllers (or separate ironic-conductor roles) should have
enough disk space to keep a cache of user instance images, at least 50 GiB
is recommended.

It's also highly recommended that you use at least two networks:

* Undercloud provisioning network (connects undercloud and overcloud nodes)

* Overcloud provisioning network (connects overcloud nodes and tenant bare
  metal instances)

Preparing undercloud
--------------------

If you already have an ``instackenv.json`` file with all nodes prepared, you
might want to leave some of the nodes for overcloud instances. E.g. if you have
three nodes in the ``instackenv.json``, you can split them::

    jq '.nodes[0:2] | {nodes: .}' instackenv.json > undercloud.json

The format of the remaining nodes is TripleO-specific, so we need
to convert it to something Ironic can understand without using
TripleO workflows. E.g. for node using IPMI::

    jq '.nodes[2:3] | {nodes: map({driver: .pm_type, name: .name,
        driver_info: {ipmi_username: .pm_user, ipmi_address: .pm_addr,
                      ipmi_password: .pm_password, ipmi_port: .pm_port},
        properties: {cpus: .cpu, cpu_arch: .arch,
                     local_gb: .disk, memory_mb: .memory},
        ports: .mac | map({address: .})})}' instackenv.json > overcloud-nodes.yaml

.. note::
    This command intentionally omits the capabilities, as they are often
    TripleO-specific, e.g. they force local boot instead of network boot used
    by default in Ironic.

.. note::
    If you use :doc:`../environments/virtualbmc`, make sure to follow
    :ref:`create-vbmc` for the overcloud nodes as well, and correctly populate
    ``ipmi_port``. If needed, change ``ipmi_address`` to the address of the
    virtual host, which is accessible from controllers.

Then enroll only ``undercloud.json`` in your undercloud::

    source stackrc
    openstack overcloud node import --provide undercloud.json

.. admonition:: Virtual
    :class: virtual

    If you used **tripleo-quickstart**, you may have to delete the nodes that
    did not end up in undercloud.json.

Configuring and deploying ironic
--------------------------------

Ironic can be installed by including one of the environment files shipped with
TripleO, however, in most of the cases you'll want to tweak certain parameters.
This section assumes that a custom environment file called
``ironic-config.yaml`` exists. Please pay particular attention to parameters
described in `Essential configuration`_.

Essential configuration
~~~~~~~~~~~~~~~~~~~~~~~

The following parameters should be configured in advance for overcloud Ironic
in an environment file:

* ``IronicEnabledHardwareTypes`` configures which hardware types will be
  supported in Ironic.

  .. note::
    Hardware types are the new generation of Ironic drivers. For example,
    the ``ipmi`` hardware type roughly corresponds to the ``pxe_ipmitool``
    driver. Check `driver configuration guide`_ and `driver-specific
    documentation`_ for more details.

  When enabling hardware types, you usually have to enable more hardware
  interfaces that these types are compatible with. For example, when enabling
  the ``redfish`` hardware type, also enable ``redfish`` power and management
  interfaces. For example::

    parameter_defaults:
        IronicEnabledHardwareTypes:
            - ipmi
            - redfish
        IronicEnabledPowerInterfaces:
            - ipmitool
            - redfish
        IronicEnabledManagementInterfaces:
            - ipmitool
            - redfish

  Some drivers might require additional configuration to work properly. Check
  `driver configuration guide`_ and `driver-specific documentation`_ for more
  details.

  By default, the ``ipmi`` hardware type is enabled.

  .. admonition:: Stable Branches
     :class: stable

     The ``IronicEnabledDrivers`` option can also be used before the Queens
     release. It sets the list of enabled classic drivers. The most often used
     bare metal driver is ``pxe_ipmitool``. Also enabled by default are
     ``pxe_ilo`` and ``pxe_drac`` drivers.

     .. admonition:: Virtual
         :class: virtual

         Starting with the Ocata release, testing on a virtual environment
         requires using :doc:`../environments/virtualbmc`.

         Before the Ocata release, a separate ``pxe_ssh`` driver has to be
         enabled for virtual testing, for example::

              parameter_defaults:
                  IronicEnabledDrivers:
                      - pxe_ssh

         If you used **tripleo-quickstart** to build your environment, the
         resulting configuration is a bit different::

              parameter_defaults:
                  IronicEnabledDrivers:
                      - pxe_ssh
                  ControllerExtraConfig:
                      ironic::drivers::ssh::libvirt_uri: 'qemu:///session'

* ``IronicCleaningDiskErase`` configures erasing hard drives
  before the first and after every deployment. There are two recommended
  values: ``full`` erases all data and ``metadata`` erases only disk metadata.
  The former is more secure, the latter is faster.

  .. admonition:: Virtual
      :class: virtual

      It is highly recommended to set this parameter to ``metadata``
      for virtual environments, as full cleaning can be extremely slow there.

* ``NovaSchedulerDefaultFilters`` configures available scheduler filters.
  For a hybrid deployment it's important to prepend
  ``AggregateInstanceExtraSpecsFilter`` to the default list::

        parameter_defaults:
            NovaSchedulerDefaultFilters:
                - RetryFilter
                - AggregateInstanceExtraSpecsFilter
                - AvailabilityZoneFilter
                - ComputeFilter
                - ComputeCapabilitiesFilter
                - ImagePropertiesFilter

  .. admonition:: Stable Branches
     :class: stable

     Before the Pike release, this list had to also contain ``RamFilter`` and
     ``DiskFilter``::

        parameter_defaults:
            NovaSchedulerDefaultFilters:
                - RetryFilter
                - AggregateInstanceExtraSpecsFilter
                - AvailabilityZoneFilter
                - RamFilter
                - DiskFilter
                - ComputeFilter
                - ComputeCapabilitiesFilter
                - ImagePropertiesFilter

     For a deployment with **only** bare metal hosts you might want
     to replace the filters with their ``Exact`` counterparts. In such case
     the scheduler will require a strict match between bare metal nodes
     and flavors. Otherwise, any bare metal node with higher or equal specs
     would match.

     ::

           parameter_defaults:
               NovaSchedulerDefaultFilters:
                   - RetryFilter
                   - AvailabilityZoneFilter
                   - ExactRamFilter
                   - ExactDiskFilter
                   - ExactCoreFilter
                   - ComputeFilter
                   - ComputeCapabilitiesFilter
                   - ImagePropertiesFilter

     The ``Exact`` filters are deprecated as of the Pike release and will be
     removed in Queens.

Additional configuration
~~~~~~~~~~~~~~~~~~~~~~~~

* ``IronicCleaningNetwork`` sets the name or UUID of the **overcloud** network
  to use for node cleaning. Initially is set to ``provisioning`` and should be
  set to an actual UUID later when `Configuring cleaning`_.

  .. admonition:: Newton
      :class: newton

      In the Newton release this parameter was not available, and no default
      value was set for the cleaning network.

* ``IronicDefaultBootOption`` specifies whether the instances will boot from
  local disk (``local``) or from PXE or iPXE (``netboot``). This parameter was
  introduced in the Pike release with the default value of ``local``. Before
  that ``netboot`` was used by default.

  .. note::
    This value can be overriden per node by setting the ``boot_option``
    capability on both the node and a flavor.

* ``IronicDefaultNetworkInterface`` specifies the network management
  implementation for bare metal nodes. The default value of ``flat`` means
  that the provisioning network is shared between all nodes, and will also be
  available to tenants.

  If you configure an ML2 mechanism driver that supports bare metal port
  binding (networking-fujitsu, networking-cisco and some others), then you can
  use the ``neutron`` implementation. In that case, Ironic and Neutron will
  fully manage networking for nodes, including plugging and unplugging
  the provision and cleaning network. The ``IronicProvisioningNetwork``
  parameter has to be configured in a similar way to ``IronicCleaningNetwork``
  (and in most cases to the same value). See the `multi-tenant networking
  documentation`_ for more details.

  .. note::
    Please check with your switch vendor to learn if your switch and its
    ML2 driver support bare metal port binding.

    Alternatively, you can use the networking-generic-switch_ ML2 plugin. It
    supports a large variety of switch vendors and models, but it's not
    currently supported by TripleO out-of-box.

  This parameter was introduced in the Pike release, and only the ``flat``
  networking is covered in this guide.

* ``IronicIPXEEnabled`` parameter turns on iPXE (HTTP-based) for deployment
  instead of PXE (TFTP-based). iPXE is more reliable and scales better, so
  it's on by default. Also iPXE is required for UEFI boot support.

Deployment
~~~~~~~~~~

Add the ironic environment file when deploying::

    openstack overcloud deploy --templates \
        -e /usr/share/openstack-tripleo-heat-templates/environments/services/ironic.yaml \
        -e ironic-config.yaml

To deploy Ironic in containers (starting with the Pike release), use
``/usr/share/openstack-tripleo-heat-templates/environments/services-docker/ironic.yaml``
instead.

.. note::
    We don't require any virtual compute nodes for the bare metal only case,
    so feel free to set ``ComputeCount: 0`` in your environment file, if you
    don't need them.

Validation
~~~~~~~~~~

Check that Ironic works by connecting to the overcloud and trying to list the
nodes (you should see an empty response, but not an error)::

    source overcloudrc
    openstack baremetal node list

You can also check the enabled driver list::

    $ openstack baremetal driver list
    +---------------------+-------------------------+
    | Supported driver(s) | Active host(s)          |
    +---------------------+-------------------------+
    | ipmi                | overcloud-controller-0. |
    | pxe_drac            | overcloud-controller-0. |
    | pxe_ilo             | overcloud-controller-0. |
    | pxe_ipmitool        | overcloud-controller-0. |
    | redfish             | overcloud-controller-0. |
    +---------------------+-------------------------+

.. note::
    This commands shows both hardware types and classic drivers combined.

For HA configuration you should see all three controllers::

    $ openstack baremetal driver list
    +---------------------+------------------------------------------------------------------------------------------------------------+
    | Supported driver(s) | Active host(s)                                                                                             |
    +---------------------+------------------------------------------------------------------------------------------------------------+
    | ipmi                | overcloud-controller-0.localdomain, overcloud-controller-1.localdomain, overcloud-controller-2.localdomain |
    | pxe_drac            | overcloud-controller-0.localdomain, overcloud-controller-1.localdomain, overcloud-controller-2.localdomain |
    | pxe_ilo             | overcloud-controller-0.localdomain, overcloud-controller-1.localdomain, overcloud-controller-2.localdomain |
    | pxe_ipmitool        | overcloud-controller-0.localdomain, overcloud-controller-1.localdomain, overcloud-controller-2.localdomain |
    | redfish             | overcloud-controller-0.localdomain, overcloud-controller-1.localdomain, overcloud-controller-2.localdomain |
    +---------------------+------------------------------------------------------------------------------------------------------------+

If this list is empty or does not show any of the controllers, then the
``openstack-ironic-conductor`` service on this controller failed to start.
The likely cause is missing dependencies for vendor drivers.

Finally, check that Nova recognizes both virtual and bare metal compute
services. In HA case there should be at least 4 services in total::

    $ openstack compute service list --service nova-compute
    +----+--------------+-------------------------------------+------+---------+-------+----------------------------+
    | ID | Binary       | Host                                | Zone | Status  | State | Updated At                 |
    +----+--------------+-------------------------------------+------+---------+-------+----------------------------+
    | 21 | nova-compute | overcloud-novacompute-0.localdomain | nova | enabled | up    | 2017-10-11T13:57:21.000000 |
    | 30 | nova-compute | overcloud-controller-2.localdomain  | nova | enabled | up    | 2017-10-11T13:57:16.000000 |
    | 33 | nova-compute | overcloud-controller-1.localdomain  | nova | enabled | up    | 2017-10-11T13:57:16.000000 |
    | 54 | nova-compute | overcloud-controller-0.localdomain  | nova | enabled | up    | 2017-10-11T13:57:14.000000 |
    +----+--------------+-------------------------------------+------+---------+-------+----------------------------+

Post-deployment configuration
-----------------------------

In this section we configure OpenStack for both bare metal and virtual
machines provisioning.

You need at least 3 nodes to use bare metal provisioning: one for the
undercloud, one for the controller and one for the actual instance.
This guide assumes using both virtual and bare metal computes, so to follow it
you need at least one more node, 4 in total for a non-HA configuration or 6
for HA.

This guide uses one network for simplicity. If you encounter weird DHCP, PXE
or networking issues with such a single-network configuration, try shutting
down the introspection DHCP server on the undercloud after the initial
introspection is finished::

        sudo systemctl stop openstack-ironic-inspector-dnsmasq

Resource classes
~~~~~~~~~~~~~~~~

Starting with the Pike release, bare metal instances are scheduled based on
*custom resource classes*. In case of Ironic, a resource class will correspond
to a flavor. When planning your bare metal cloud, think of a way to split all
nodes into classes, and create flavors accordingly. See `bare metal flavor
documentation`_ for more details.

Preparing networking
~~~~~~~~~~~~~~~~~~~~

Next, we need to create at least one network for nodes to use. By default
Ironic uses the tenant network for the provisioning process, and the same
network is often configured for cleaning. Using separate networks is beyond
the scope of this guide.

As already mentioned, this guide assumes only one physical network shared
between undercloud and overcloud. In this case the subnet address must match
the one on the undercloud, but the allocation pools must not overlap (including
the pool used by undercloud introspection).

For example, the following commands will work with the default undercloud
parameters::

    source overcloudrc
    openstack network create --share --provider-network-type flat \
        --provider-physical-network datacentre --external provisioning
    openstack subnet create --network provisioning \
        --subnet-range 192.168.24.0/24 --gateway 192.168.24.40 \
        --allocation-pool start=192.168.24.41,end=192.168.24.100 provisioning-subnet
    openstack router create default-router
    openstack router add subnet default-router provisioning-subnet

.. warning::
    Network types other than "flat" are not supported when using ``flat``
    bare metal networking implementation (see `Additional configuration`_
    above for more details).

We will use this network for bare metal instances (both for provisioning and
as a tenant network), as well as an external network for virtual instances.
In a real situation you will only use it as provisioning, and create a separate
physical network as external.

Now you can create a regular tenant network to use for virtual instances
and use the ``default-router`` to link the provisioning and tenant networks::

    openstack network create tenant-net
    openstack subnet create --network tenant-net --subnet-range 192.0.3.0/24 \
        --allocation-pool start=192.0.3.10,end=192.0.3.20 tenant-subnet
    openstack router add subnet default-router tenant-subnet

Configuring cleaning
~~~~~~~~~~~~~~~~~~~~

Starting with the Ocata release, Ironic is configured to use network called
``provisioning`` for node cleaning. However, network names are not unique.
A user creating another network with the same name will break bare metal
provisioning. Thus, it's highly recommended to update the deployment,
providing the provider network UUID.

Use the following command to get the UUID::

    openstack network show provisioning -f value -c id

Update the environment file you've created, setting ``IronicCleaningNetwork``
to the this UUID, for example::

    parameter_defaults:
        IronicCleaningNetwork: c71f4bfe-409b-4292-818f-21cdf910ee06

.. admonition:: Newton
   :class: newton

   In the Newton release this parameter was not available, use
   ``cleaning_network_uuid`` hieradata value instead, for example::

        parameter_defaults:
            ControllerExtraConfig:
                ironic::conductor::cleaning_network_uuid: c71f4bfe-409b-4292-818f-21cdf910ee06

   This variable does not support node names and does not have a default value
   in this release.

Finally, run the deploy command with exactly the same arguments as before
(don't forget to include the environment file if it was not included
previously).

Adding deployment images
~~~~~~~~~~~~~~~~~~~~~~~~

Ironic requires the ironic-python-agent image stored in Glance.
You can use the same images you already have on the undercloud::

    source overcloudrc
    openstack image create --public --container-format aki \
        --disk-format aki --file ~/ironic-python-agent.kernel deploy-kernel
    openstack image create --public --container-format ari \
        --disk-format ari --file ~/ironic-python-agent.initramfs deploy-ramdisk

.. note::
    These commands assume that the images are in the home directory, which is
    often the case for TripleO.

Creating flavors
~~~~~~~~~~~~~~~~

As usual with OpenStack, you need to create at least one flavor to be used
during deployment. As bare metal resources are inherently not divisible,
the flavor will set minimum requirements (CPU count, RAM and disk sizes) that
a node must fulfil, see `bare metal flavor documentation`_ for details.

Creating a single flavor is sufficient for the simplest case::

    source overcloudrc
    openstack flavor create --ram 1024 --disk 20 --vcpus 1 baremetal

.. note::
    The ``disk`` argument will be used to determine the size of the root
    partition. The ``ram`` and ``vcpus`` arguments are ignored for bare metal,
    starting with the Pike release, if the flavor is configured as explained
    below.

Starting with the Pike release, switch to scheduling based on resource
classes, as explained in the `bare metal flavor documentation`_::

    openstack flavor set baremetal --property resources:CUSTOM_BAREMETAL=1
    openstack flavor set baremetal --property resources:VCPU=0
    openstack flavor set baremetal --property resources:MEMORY_MB=0
    openstack flavor set baremetal --property resources:DISK_GB=0

Creating host aggregates
~~~~~~~~~~~~~~~~~~~~~~~~

.. note::
    If you don't plan on using virtual instances, you can skip this step.
    It also won't be required in the Queens release, once bare metal nodes
    stop report CPU, memory and disk properties.

For a hybrid bare metal and virtual environment, you have to set up *host
aggregates* for virtual and bare metal hosts. We will use a property
called ``baremetal`` to link flavors to host aggregates::

    openstack aggregate create --property baremetal=true baremetal-hosts
    openstack aggregate create --property baremetal=false virtual-hosts
    openstack flavor set baremetal --property baremetal=true

.. warning::
    This association won't work without ``AggregateInstanceExtraSpecsFilter``
    enabled as described in `Essential configuration`_.

.. warning::
    Any property you set on flavors has to be duplicated on aggregates,
    otherwise scheduling will fail.

Then for all flavors you've created for virtual instances set the same
``baremetal`` property to ``false``, for example::

    openstack flavor create --ram 1024 --disk 20 --vcpus 1 virtual
    openstack flavor set virtual --property baremetal=false

Creating instance images
~~~~~~~~~~~~~~~~~~~~~~~~

You can build your images using ``diskimage-builder`` tool already available
on the undercloud, for example::

    disk-image-create centos7 baremetal dhcp-all-interfaces grub2 -o centos-image

.. note::
    The following elements are actually optional:

    * ``dhcp-all-interfaces`` makes the resulting instance get IP addresses for
      all NICs via DHCP.

    * ``grub2`` installs the grub bootloader on the image, so that local boot
      can be used in additional to PXE booting.

This command creates a so called *partition image*, i.e. an image containing
only root partition. Ironic also supports *whole disk images*, i.e. images
with the whole partition table embedded. This may be the only option when
running non-Linux images. Please check the `images documentation`_
for more details on building and using images.

Three components are created for every partition image: the main image with
``qcow2`` extension, the kernel with ``vmlinuz`` extension and the initrd
image with ``initrd`` extension.

Upload them with the following command::

    source overcloudrc
    KERNEL_ID=$(openstack image create --file centos-image.vmlinuz --public \
        --container-format aki --disk-format aki -f value -c id \
        centos-image.vmlinuz)
    RAMDISK_ID=$(openstack image create --file centos-image.initrd --public \
        --container-format ari --disk-format ari -f value -c id \
        centos-image.initrd)
    openstack image create --file centos-image.qcow2 --public \
        --container-format bare --disk-format qcow2 \
        --property kernel_id=$KERNEL_ID --property ramdisk_id=$RAMDISK_ID \
        centos-image

.. note::
    A whole disk image will only have one component - the image itself with
    ``qcow2`` extension. Do not set ``kernel_id`` and ``ramdisk_id``
    properties for such images.

Enrolling nodes
---------------

For all nodes you're enrolling you need to know:

* BMC (IPMI, iDRAC, iLO, etc) address and credentials,

* MAC address of the PXE booting NIC,

* CPU count and architecture, memory size in MiB and root disk size in GiB,

* Serial number or WWN of the root device, if the node has several hard drives.

In the future some of this data will be provided by the introspection process,
which is not currently available in the overcloud.

This guide uses inventory files to enroll nodes. Alternatively, you can enroll
nodes directly from CLI, see the `enrollment documentation`_ for details.

Preparing inventory
~~~~~~~~~~~~~~~~~~~

Your inventory file (e.g. ``overcloud-nodes.yaml`` from `Preparing
undercloud`_) should be in the following format::

    nodes:
        - name: node-0
          driver: ipmi
          driver_info:
            ipmi_address: <BMC HOST>
            ipmi_username: <BMC USER>
            ipmi_password: <BMC PASSWORD>
            ipmi_port: <BMC PORT>
          resource_class: baremetal
          properties:
            cpus: <CPU COUNT>
            cpu_arch: <CPU ARCHITECTURE>
            memory_mb: <RAM SIZE IN MIB>
            local_gb: <ROOT DISK IN GIB>
            root_device:
                serial: <ROOT DISK SERIAL>
          ports:
            - address: <PXE NIC MAC>
              pxe_enabled: true

* The ``driver`` field must be one of ``IronicEnabledDrivers`` or
  ``IronicEnabledHardwareTypes``, which we set when `Configuring and deploying
  ironic`_.

  .. admonition:: Stable Branch
     :class: stable

     Hardware types are only available since the Pike release. In the example
     above use ``pxe_ipmitool`` instead of ``ipmi`` for older releases.

* The ``resource_class`` field corresponds to a custom resource
  class, as explained in `Resource classes`_.

* The ``root_device`` property is optional, but it's highly recommended
  to set it if the bare metal node has more than one hard drive.
  There are several properties that can be used instead of the serial number
  to designate the root device, see the `root device hints documentation`_
  for details.

* The ``local_gb`` field specifies the size (in GiB) of the root device. Its
  value must match the size of the device specified by the ``root_device``
  property. However, to allow for partitioning, it's highly recommended to
  subtract 1 GiB from it.

* Exactly one port with ``pxe_enabled`` set to ``true`` must be specified in
  the ``ports`` list. It has to match the NIC used for provisioning.

  .. note::
    More ports with ``pxe_enabled=false`` can be specified safely here. They
    won't be used for provisioning, but they are used with advanced networking
    (not covered in this guide).

* The ``memory_mb`` and ``local_gb`` properties will not be mandatory any more
  in the Queens release.

Enrolling nodes
~~~~~~~~~~~~~~~

The ``overcloud-nodes.yaml`` file prepared in the previous steps can now be
imported in Ironic::

    source overcloudrc
    openstack baremetal create overcloud-nodes.yaml

.. warning::
    This command is provided by Ironic, not TripleO. It also does not feature
    support for updates, so if you need to change something, you have to use
    ``openstack baremetal node set`` and similar commands.

The nodes appear in the ``enroll`` provision state, you need to check their BMC
credentials and make them available::

    DEPLOY_KERNEL=$(openstack image show deploy-kernel -f value -c id)
    DEPLOY_RAMDISK=$(openstack image show deploy-ramdisk -f value -c id)

    for uuid in $(openstack baremetal node list --provision-state enroll -f value -c UUID);
    do
        openstack baremetal node set $uuid \
            --driver-info deploy_kernel=$DEPLOY_KERNEL \
            --driver-info deploy_ramdisk=$DEPLOY_RAMDISK
        openstack baremetal node manage $uuid --wait &&
            openstack baremetal node provide $uuid
    done

The deploy kernel and ramdisk were created as part of `Adding deployment
images`_.

The ``baremetal node provide`` command makes a node go through cleaning
procedure, so it might take some time depending on the configuration. Check
your nodes status with::

    openstack baremetal node list --fields uuid name provision_state last_error

Wait for all nodes to reach the ``available`` state. Any failures during
cleaning has to be corrected before proceeding with deployment.

Populating host aggregates
~~~~~~~~~~~~~~~~~~~~~~~~~~

For hybrid bare metal and virtual case you need to specify which host
belongs to which host aggregates (``virtual`` or ``baremetal`` as created in
`Creating host aggregates`_).

When the default host names are used, we can take advantage of the fact
that every virtual host will have ``compute`` in its name. All bare metal
hypervisors will be assigned to one (non-HA) or three (HA) controller hosts.
So we can do the assignment with the following commands::

    source overcloudrc
    for vm_host in $(openstack hypervisor list -f value -c "Hypervisor Hostname" | grep compute);
    do
        openstack aggregate add host virtual-hosts $vm_host
    done

    openstack aggregate add host baremetal-hosts overcloud-controller-0.localdomain
    # Ignore the following two for a non-HA environment
    openstack aggregate add host baremetal-hosts overcloud-controller-1.localdomain
    openstack aggregate add host baremetal-hosts overcloud-controller-2.localdomain

.. note::
    Every time you scale out compute nodes, you need to add newly added
    hosts to the ``virtual-hosts`` aggregate.

Checking available resources
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Check that nodes are really enrolled and the power state is reflected correctly
(it may take some time)::

    $ source overcloudrc
    $ openstack baremetal node list
    +--------------------------------------+------------+---------------+-------------+--------------------+-------------+
    | UUID                                 | Name       | Instance UUID | Power State | Provisioning State | Maintenance |
    +--------------------------------------+------------+---------------+-------------+--------------------+-------------+
    | a970c5db-67dd-4676-95ba-af1edc74b2ee | instance-0 | None          | power off   | available          | False       |
    | bd99ec64-4bfc-491b-99e6-49bd384b526d | instance-1 | None          | power off   | available          | False       |
    +--------------------------------------+------------+---------------+-------------+--------------------+-------------+

After a few minutes, new hypervisors should appear in Nova and the stats
should display the sum of bare metal and virtual resources::

    $ openstack hypervisor list
    +----+--------------------------------------+
    | ID | Hypervisor Hostname                  |
    +----+--------------------------------------+
    |  2 | overcloud-novacompute-0.localdomain  |
    | 17 | bd99ec64-4bfc-491b-99e6-49bd384b526d |
    | 20 | a970c5db-67dd-4676-95ba-af1edc74b2ee |
    +----+--------------------------------------+

.. note::
    Each bare metal node becomes a separate hypervisor in Nova. The hypervisor
    host name always matches the associated node UUID.

Next you can use the Placement API (available only via cURL for the time being)
to check that bare metal resources are properly exposed. Start with checking
that all nodes are recorded::

    $ token=$(openstack token issue -f value -c id)
    $ endpoint=$(openstack endpoint show placement -f value -c publicurl)
    $ curl -sH "X-Auth-Token: $token" $endpoint/resource_providers | jq -r '.resource_providers | map({node: .name, uuid})'
    [
      {
        "uuid": "9dff98a8-6fc9-4a05-8d78-c1d5888d9fde",
        "node": "overcloud-novacompute-0.localdomain"
      },
      {
        "uuid": "61d741b5-33d6-40a1-bcbe-b38ea34ca6c8",
        "node": "bd99ec64-4bfc-491b-99e6-49bd384b526d"
      },
      {
        "uuid": "e22bc261-53be-43b3-848f-e29c728142d3",
        "node": "a970c5db-67dd-4676-95ba-af1edc74b2ee"
      }
    ]

Then for each of the bare metal resource providers (having node UUIDs as
names) check their inventory::

    $ curl -sH "X-Auth-Token: $token" $endpoint/resource_providers/e22bc261-53be-43b3-848f-e29c728142d3/inventories | jq .inventories
    {
      "DISK_GB": {
        "max_unit": 50,
        "min_unit": 1,
        "step_size": 1,
        "reserved": 0,
        "total": 50,
        "allocation_ratio": 1
      },
      "CUSTOM_BAREMETAL": {
        "max_unit": 1,
        "min_unit": 1,
        "step_size": 1,
        "reserved": 0,
        "total": 1,
        "allocation_ratio": 1
      }
    }

You see the custom ``baremetal`` resource class reported, as well as available
disk space (only before the Queens release). If you see an empty inventory,
nova probably consider the node unavailable. Check :ref:`no-valid-host` for
tips on a potential cause.

Booting a bare metal instance
-----------------------------

You will probably want to create a keypair to use for logging into instances.
For example, using SSH public key from undercloud::

    source overcloudrc
    openstack keypair create --public-key ~/.ssh/id_rsa.pub undercloud-key

Now you're ready to boot your first bare metal instance::

    openstack server create --image centos-image --flavor baremetal \
        --nic net-id=$(openstack network show provisioning -f value -c id) \
        --key-name undercloud-key instance-0

After some time (depending on the image), you will see the prepared instance::

    $ openstack server list
    +--------------------------------------+------------+--------+-----------------------------+
    | ID                                   | Name       | Status | Networks                    |
    +--------------------------------------+------------+--------+-----------------------------+
    | 2022d237-e249-44bd-b864-e7f536a8e439 | instance-0 | ACTIVE | provisioning=192.168.24.50  |
    +--------------------------------------+------------+--------+-----------------------------+

.. note::
    If you encounter *"No valid host found"* error from Nova, make sure to read
    the undercloud troubleshooting guide on this topic: :ref:`no-valid-host`.

Let's check that it actually got scheduled on a bare metal machine::

    $ openstack server show instance-0 -c "OS-EXT-SRV-ATTR:host" -c "OS-EXT-SRV-ATTR:hypervisor_hostname"
    +-------------------------------------+--------------------------------------+
    | Field                               | Value                                |
    +-------------------------------------+--------------------------------------+
    | OS-EXT-SRV-ATTR:host                | overcloud-controller-0.localdomain   |
    | OS-EXT-SRV-ATTR:hypervisor_hostname | bd99ec64-4bfc-491b-99e6-49bd384b526d |
    +-------------------------------------+--------------------------------------+

You can now log into it::

    $ ssh centos@192.168.24.50
    The authenticity of host '192.168.24.50 (192.168.24.50)' can't be established.
    ECDSA key fingerprint is eb:35:45:c5:ed:d9:8a:e8:4b:20:db:06:10:6f:05:74.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '192.168.24.50' (ECDSA) to the list of known hosts.
    [centos@instance-0 ~]$

Now let's try the same with a virtual instance::

    openstack server create --image centos-image --flavor virtual \
        --nic net-id=$(openstack network show tenant-net -f value -c id) \
        --key-name undercloud-key instance-1

This instance gets scheduled on a virtual host::

    $ openstack server show instance-1 -c "OS-EXT-SRV-ATTR:host" -c "OS-EXT-SRV-ATTR:hypervisor_hostname"
    +-------------------------------------+-------------------------------------+
    | Field                               | Value                               |
    +-------------------------------------+-------------------------------------+
    | OS-EXT-SRV-ATTR:host                | overcloud-novacompute-0.localdomain |
    | OS-EXT-SRV-ATTR:hypervisor_hostname | overcloud-novacompute-0.localdomain |
    +-------------------------------------+-------------------------------------+

Booting a bare metal instance from a cinder volume
--------------------------------------------------

Cinder volumes can be used to back a baremetal node over iSCSI, in order to
do this each baremetal node must first be configured to boot from a volume.
The connector ID for each node should be unique, below we achieve this by
incrementing the value of <NUM>::

    $ openstack baremetal node set --property capabilities=iscsi_boot:true --storage-interface cinder <NODEID>
    $ openstack baremetal volume connector create --node <NODEID> --type iqn --connector-id iqn.2010-10.org.openstack.node<NUM>

The image used should be configured to boot from a iSCSI root disk, on Centos
7 this is achieved by ensuring that the `iscsi` module is added to the ramdisk
and passing `rd.iscsi.firmware=1` to the kernel in the grub config::

    $ mkdir /tmp/mountpoint
    $ guestmount -i -a /tmp/CentOS-7-x86_64-GenericCloud.qcow2 /tmp/mountpoint
    $ chroot /tmp/mountpoint /bin/bash
    chroot> mv /etc/resolv.conf /etc/resolv.conf_
    chroot> echo "nameserver 8.8.8.8" > /etc/resolv.conf
    chroot> yum install -y iscsi-initiator-utils
    chroot> mv /etc/resolv.conf_ /etc/resolv.conf
    # Be careful here to update the correct ramdisk (check/boot/grub2/grub.cfg)
    chroot> dracut --force --add "network iscsi" /boot/initramfs-3.10.0-693.5.2.el7.x86_64.img 3.10.0-693.5.2.el7.x86_64
    # Edit the file /etc/default/grub and add rd.iscsi.firmware=1 to GRUB_CMDLINE_LINUX=...
    chroot> vi /etc/default/grub
    chroot> exit
    $ guestunmount /tmp/mountpoint
    $ guestfish -a /tmp/CentOS-7-x86_64-GenericCloud.qcow2 -m /dev/sda1 sh "/sbin/grub2-mkconfig -o /boot/grub2/grub.cfg"

.. note::
    This image can no longer be used to do regular local boot, a situation
    that should be fixed in future versions.

This image can then be added to glance and a volume created from it::

    $ openstack image create --disk-format qcow2 --container-format bare --file /tmp/CentOS-7-x86_64-GenericCloud.qcow2 centos-bfv
    $ openstack volume create --size 10 --image centos-bfv --bootable centos-test-volume

Finally this volume can be used to back a baremetal instance::

    $ openstack server create --flavor baremetal --volume centos-test-volume --key default centos-test

.. _IronicConductor role shipped with TripleO: https://git.openstack.org/cgit/openstack/tripleo-heat-templates/plain/roles/IronicConductor.yaml
.. _driver configuration guide: https://docs.openstack.org/ironic/latest/install/enabling-drivers.html
.. _driver-specific documentation: https://docs.openstack.org/ironic/latest/admin/drivers.html
.. _bare metal flavor documentation: https://docs.openstack.org/ironic/latest/install/configure-nova-flavors.html
.. _enrollment documentation: https://docs.openstack.org/ironic/latest/install/enrollment.html
.. _root device hints documentation: https://docs.openstack.org/ironic/latest/install/advanced.html#specifying-the-disk-for-deployment-root-device-hints
.. _images documentation: https://docs.openstack.org/ironic/latest/install/configure-glance-images.html
.. _multi-tenant networking documentation: https://docs.openstack.org/ironic/latest/admin/multitenancy.html
.. _networking-generic-switch: https://github.com/openstack/networking-generic-switch
