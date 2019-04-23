Baremetal Environment
---------------------

|project| can be used in an all baremetal environment. One machine will be
used for Undercloud, the others will be used for your Overcloud.

Minimum System Requirements
^^^^^^^^^^^^^^^^^^^^^^^^^^^

To deploy a minimal TripleO cloud with |project| you need the following baremetal
machines:

* 1 Undercloud
* 1 Overcloud Controller
* 1 Overcloud Compute

For each additional Overcloud role, such as Block Storage or Object Storage,
you need an additional baremetal machine.

..
    <REMOVE WHEN HA IS AVAILABLE>

    For minimal **HA (high availability)** deployment you need at least 3 Overcloud
    Controller machines and 2 Overcloud Compute machines.

The baremetal machines must meet the following minimum specifications:

* 8 core CPU
* 12 GB memory
* 60 GB free disk space

Larger systems are recommended for production deployments, however.

For instance, the undercloud needs a bit more capacity, especially regarding RAM (minimum of 16G is advised)
and is pretty intense for the I/O - fast disks (SSD, SAS) are strongly advised.

Please also note the undercloud needs space in order to store twice the "overcloud-full" image (one time
in its glance, one time in /var/lib subdirectories for PXE/TFTP).

TripleO is supporting only the following operating systems:

* RHEL 7 x86_64
* CentOS 7 x86_64

Please also ensure your node clock is set to UTC in order to prevent any issue
when the OS hwclock syncs to the BIOS clock before applying timezone offset,
causing files to have a future-dated timestamp.


Preparing the Baremetal Environment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Networking
^^^^^^^^^^

The overcloud nodes will be deployed from the undercloud machine and therefore the machines need to have have their network settings modified to allow for the overcloud nodes to be PXE boot'ed using the undercloud machine. As such, the setup requires that:

* All overcloud machines in the setup must support IPMI
* A management provisioning network is setup for all of the overcloud machines.
  One NIC from every machine needs to be in the same broadcast domain of the
  provisioning network. In the tested environment, this required setting up a new
  VLAN on the switch. Note that you should use the same NIC on each of the
  overcloud machines ( for example: use the second NIC on each overcloud
  machine). This is because during installation we will need to refer to that NIC
  using a single name across all overcloud machines e.g. em2
* The provisioning network NIC should not be the same NIC that you are using
  for remote connectivity to the undercloud machine. During the undercloud
  installation, a openvswitch bridge will be created for Neutron and the
  provisioning NIC will be bridged to the openvswitch bridge. As such,
  connectivity would be lost if the provisioning NIC was also used for remote
  connectivity to the undercloud machine.
* The overcloud machines can PXE boot off the NIC that is on the private VLAN.
  In the tested environment, this required disabling network booting in the BIOS
  for all NICs other than the one we wanted to boot and then ensuring that the
  chosen NIC is at the top of the boot order (ahead of the local hard disk drive
  and CD/DVD drives).
* For each overcloud machine you have: the MAC address of the NIC that will PXE
  boot on the provisioning network the IPMI information for the machine (i.e. IP
  address of the IPMI NIC, IPMI username and password)

Refer to the following diagram for more information

.. image:: ../_images/TripleO_Network_Diagram_.jpg

Setting Up The Undercloud Machine
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Select a machine within the baremetal environment on which to install the
   undercloud.
#. Install RHEL 7.1 x86_64 or CentOS 7 x86_64 on this machine.
#. If needed, create a non-root user with sudo access to use for installing the
   Undercloud::

        sudo useradd stack
        sudo passwd stack  # specify a password
        echo "stack ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/stack
        sudo chmod 0440 /etc/sudoers.d/stack

.. admonition:: RHEL
 :class: rhel

 If using RHEL, register the Undercloud for package installations/updates.

 .. admonition:: RHEL Portal Registration
    :class: portal

    Register the host machine using Subscription Management::

        sudo subscription-manager register --username="[your username]" --password="[your password]"
        # Find this with `subscription-manager list --available`
        sudo subscription-manager attach --pool="[pool id]"
        # Verify repositories are available
        sudo subscription-manager repos --list
        # Enable repositories needed
        sudo subscription-manager repos --enable=rhel-7-server-rpms \
             --enable=rhel-7-server-optional-rpms --enable=rhel-7-server-extras-rpms \
             --enable=rhel-7-server-openstack-6.0-rpms

 .. admonition:: RHEL Satellite Registration
    :class: satellite

    To register the host machine to a Satellite, the following repos must
    be synchronized on the Satellite and enabled for registered systems::

        rhel-7-server-rpms
        rhel-7-server-optional-rpms
        rhel-7-server-extras-rpms
        rhel-7-server-openstack-6.0-rpms

    See the `Red Hat Satellite User Guide`_ for how to configure the system to
    register with a Satellite server. It is suggested to use an activation
    key that automatically enables the above repos for registered systems.

.. _Red Hat Satellite User Guide: https://access.redhat.com/documentation/en-US/Red_Hat_Satellite/


Validations
^^^^^^^^^^^

You can run the ``prep`` validations to verify the hardware. Later in
the process, the validations will be run by the undercloud processes.
Refer to the Ansible section for running directly the validations
over baremetal nodes `validations_no_undercloud`_.


Configuration Files
^^^^^^^^^^^^^^^^^^^

.. _instackenv:

instackenv.json
^^^^^^^^^^^^^^^

Create a JSON file describing your Overcloud baremetal nodes, call it
``instackenv.json`` and place in your home directory. The file should contain
a JSON object with the only field ``nodes`` containing list of node
descriptions.

Each node description should contains required fields:

* ``pm_type`` - driver for Ironic nodes, see `Ironic Hardware Types`_
  for details

* ``pm_addr`` - node BMC IP address (hypervisor address in case of virtual
  environment)

* ``pm_user``, ``pm_password`` - node BMC credentials

Some fields are optional if you're going to use introspection later:

* ``ports`` - list of baremetal port objects, a map specifying the following
  keys: address, physical_network (optional) and local_link_connection
  (optional). Optional for bare metal. Example::

    "ports": [
        {
            "address": "52:54:00:87:c8:2f",
            "physical_network": "physical-network",
            "local_link_connection": {
                "switch_info": "switch",
                "port_id": "gi1/0/11",
                "switch_id": "a6:18:66:33:cb:48"
            }
        }
    ]

* ``cpu`` - number of CPU's in system

* ``arch`` - CPU architecture (common values are ``i386`` and ``x86_64``)

* ``memory`` - memory size in MiB

* ``disk`` - hard driver size in GiB

It is also possible (but optional) to set Ironic node capabilities directly
in the JSON file. This can be useful for assigning node profiles or setting
boot options at registration time:

* ``capabilities`` - Ironic node capabilities.  For example::

    "capabilities": "profile:compute,boot_option:local"

There are also two additional and optional fields that can be used to help a
user identifying machines inside ``instackenv.json`` file:

* ``name`` - name associated to the node, it will appear in the ``Name``
  column while listing nodes

* ``_comment`` to associate a comment to the node (like position, long
  description and so on). Note that this field will not be considered by
  Ironic during the import

Also if you're working in a diverse environment with multiple architectures
and/or platforms within an architecture you may find it necessary to include a
platform field:

* ``platform`` - String paired with images to fine tune image selection

For example::

  {
      "nodes": [
          {
              "name": "node-a",
              "pm_type": "ipmi",
              "ports": [
                  {
                      "address": "fa:16:3e:2a:0e:36",
                      "physical_network": "ctlplane"
                  }
              ],
              "cpu": "2",
              "memory": "4096",
              "disk": "40",
              "arch": "x86_64",
              "pm_user": "admin",
              "pm_password": "password",
              "pm_addr": "10.0.0.8",
              "_comment": "Room 1 - Rack A - Unit 22/24"
          },
          {
              "name": "node-b",
              "pm_type": "ipmi",
              "ports": [
                  {
                      "address": "fa:16:3e:da:39:c9",
                      "physical_network": "ctlplane"
                  }
              ],
              "cpu": "2",
              "memory": "4096",
              "disk": "40",
              "arch": "x86_64",
              "pm_user": "admin",
              "pm_password": "password",
              "pm_addr": "10.0.0.15",
              "_comment": "Room 1 - Rack A - Unit 26/28"
          },
          {
              "name": "node-n",
              "pm_type": "ipmi",
              "ports": [
                  {
                      "address": "fa:16:3e:51:9b:68",
                      "physical_network": "leaf1"
                  }
              ],
              "cpu": "2",
              "memory": "4096",
              "disk": "40",
              "arch": "x86_64",
              "pm_user": "admin",
              "pm_password": "password",
              "pm_addr": "10.0.0.16",
              "_comment": "Room 1 - Rack B - Unit 10/12"
          }
      ]
  }


.. note::
    You don't need to create this file, if you plan on using
    :doc:`../advanced_deployment/node_discovery`.

Ironic Hardware Types
^^^^^^^^^^^^^^^^^^^^^

Ironic *hardware types* provide various level of support for different
hardware. Hardware types, introduced in the Ocata cycle, are a new generation
of Ironic *drivers*. Previously, the word *drivers* was used to refer to what
is now called *classic drivers*. See `Ironic drivers documentation`_ for a full
explanation of similarities and differences between the two types.

Hardware types are enabled in the ``undercloud.conf`` using the
``enabled_hardware_types`` configuration option. Classic drivers are enabled
using the ``enabled_drivers`` option. It is deprecated in the Queens release
cycle and should no longer be used. See the `hardware types migration guide`_
for information on how to migrate existing nodes.

Both hardware types and classic drivers can be equially used in the
``pm_addr`` field of the ``instackenv.json``.

See https://docs.openstack.org/ironic/latest/admin/drivers.html for the most
up-to-date information about Ironic hardware types and hardware
interfaces, but note that this page always targets Ironic git master, not the
release we use.

Generic Hardware Types
~~~~~~~~~~~~~~~~~~~~~~~

* This most generic hardware type is ipmi_. It uses the `ipmitool`_ utility
  to manage a bare metal node, and supports a vast variety of hardware.

  .. admonition:: Stable Branch
     :class: stable

     This hardware type is supported starting with the Pike release. For older
     releases use the functionally equivalent ``pxe_ipmitool`` driver.

  .. admonition:: Virtual
     :class: virtual

     When combined with :doc:`virtualbmc`, this hardware type can be used for
     developing and testing TripleO in a virtual environment as well.

     .. admonition:: Stable Branch
        :class: stable

        Prior to the Ocata release, a special ``pxe_ssh`` driver was used for
        testing Ironic in the virtual environment. This driver connects to the
        hypervisor to conduct management operations on virtual nodes. In case
        of this driver, ``pm_addr`` is a hypervisor address, ``pm_user`` is
        a SSH user name for accessing hypervisor, ``pm_password`` is a private
        SSH key for accessing hypervisor. Note that private key must not be
        encrypted.

        .. warning::
          The ``pxe_ssh`` driver is deprecated and ``pxe_ipmitool`` +
          :doc:`virtualbmc` should be used instead.

* Another generic hardware type is redfish_. It provides support for the
  quite new `Redfish standard`_, which aims to replace IPMI eventually as
  a generic protocol for managing hardware. In addition to the ``pm_*`` fields
  mentioned above, this hardware type also requires setting ``pm_system_id``
  to the full identifier of the node in the controller (e.g.
  ``/redfish/v1/Systems/42``).

  .. admonition:: Stable Branch
     :class: stable

     Redfish support was introduced in the Pike release.

The following generic hardware types are not enabled by default:

* The snmp_ hardware type supports controlling PDUs for power management.
  It requires boot device to be manually configured on the nodes.

* Finally, the ``manual-management`` hardware type (not enabled by default)
  skips power and boot device management completely. It requires manual power
  and boot operations to be done at the right moments, so it's not recommended
  for a generic production.

  .. admonition:: Stable Branch
     :class: stable

     The functional analog of this hardware type before the Queens release
     was the ``fake_pxe`` driver.

Vendor Hardware Types
~~~~~~~~~~~~~~~~~~~~~

TripleO also supports vendor-specific hardware types for some types
of hardware:

* ilo_ targets HPE Proliant Gen 8 and Gen 9 systems.

  .. admonition:: Stable Branch
     :class: stable

     Use the ``pxe_ilo`` classic driver before the Queens release.

* idrac_ targets DELL 12G and newer systems.

  .. admonition:: Stable Branch
     :class: stable

     Use the ``pxe_drac`` classic driver before the Queens release.

The following hardware types are supported but not enabled by default:

* irmc_ targets FUJITSU PRIMERGY servers.

* cisco-ucs-managed_ targets UCS Manager managed Cisco UCS B/C series servers.

* cisco-ucs-standalone_ targets standalone Cisco UCS C series servers.

.. note::
    Contact a specific vendor team if you have problems with any of these
    drivers, as the TripleO team often cannot assist with them.

.. _Ironic drivers documentation: https://docs.openstack.org/ironic/latest/install/enabling-drivers.html
.. _hardware types migration guide: https://docs.openstack.org/ironic/latest/admin/upgrade-to-hardware-types.html
.. _ipmitool: http://sourceforge.net/projects/ipmitool/
.. _Redfish standard: https://www.dmtf.org/standards/redfish
.. _ipmi: https://docs.openstack.org/ironic/latest/admin/drivers/ipmitool.html
.. _redfish: https://docs.openstack.org/ironic/latest/admin/drivers/redfish.html
.. _snmp: https://docs.openstack.org/ironic/latest/admin/drivers/snmp.html
.. _ilo: https://docs.openstack.org/ironic/latest/admin/drivers/ilo.html
.. _idrac: https://docs.openstack.org/ironic/latest/admin/drivers/idrac.html
.. _irmc: https://docs.openstack.org/ironic/latest/admin/drivers/irmc.html
.. _cisco-ucs-managed: https://docs.openstack.org/ironic/latest/admin/drivers/ucs.html
.. _cisco-ucs-standalone: https://docs.openstack.org/ironic/latest/admin/drivers/cimc.html
.. _validations_no_undercloud: ../../validations/ansible.html
