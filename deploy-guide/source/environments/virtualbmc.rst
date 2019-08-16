VirtualBMC
==========

VirtualBMC_ is a small CLI that allows users to create a virtual BMC_
to manage a virtual machines using the IPMI_ protocol, similar to how real
bare metal machines are managed. It can be used to to enable testing
bare metal deployments in completely virtual environments.

.. admonition:: Stable Branch
  :class: stable

  Ironic also ships a ``pxe_ssh`` driver that can be used for that purpose,
  but it has been deprecated and its use is discouraged.

.. warning::
   VirtualBMC is not meant for production environments.

Installation
------------

VirtualBMC is available from RDO repositories starting with the Ocata release::

    sudo yum install -y python-virtualbmc

It is usually installed and used on the hypervisor where the virtual machines
reside.

.. _create-vbmc:

Creating virtual BMC
--------------------

Every virtual machine needs its own virtual BMC. Create it with::

    vbmc add <domain> --port 6230 --username admin --password password

.. note::
   You need to use a different port for each domain. Port numbers
   lower than 1025 requires the user to have root privilege in the system.

.. note::
   For **tripleo-quickstart** you may have to specify
   ``--libvirt-uri=qemu:///session``.

Start the virtual BMCs::

    vbmc start <domain>

.. warning::
   This step has to be repeated after virtual host reboot.

Test the virtual BMC to see if it's working. For example, to power on
the virtual machine do::

    ipmitool -I lanplus -U admin -P password -H 127.0.0.1 -p 6230 power on

Enrolling virtual machines
--------------------------

In the undercloud, populate the ``instackenv.json`` with new "bare metals"
in a similar way to real bare metal machines (see :ref:`instackenv`) with
two exceptions:

* set ``pm_port`` to the port you specified when `Creating virtual BMC`

* populate ``mac`` field even if you plan to use introspection

For example:

.. code-block:: json

  {
      "nodes": [
          {
              "pm_type": "ipmi",
              "mac": [
                  "00:0a:f2:88:12:aa"
              ],
              "pm_user": "admin",
              "pm_password": "password",
              "pm_addr": "172.16.0.1",
              "pm_port": "6230",
              "name": "compute-0"
          }
      ]
  }

Migrating from pxe_ssh to VirtualBMC
------------------------------------

If you already have a virtual cloud deployed and want to migrate from the
deprecated ``pxe_ssh`` driver to ``ipmi`` using VirtualBMC,
follow `Creating virtual BMC`_, then update the existing nodes to change
their drivers and certain driver properties:

.. code-block:: bash

    openstack baremetal node set $NODE_UUID_OR_NAME \
        --driver ipmi \
        --driver-info ipmi_address=<IP address of the virthost> \
        --driver-info ipmi_port=<Virtual BMC port> \
        --driver-info ipmi_username="admin" \
        --driver-info ipmi_password="password"

.. admonition:: Stable Branch
   :class: stable

   For the Ocata release, use ``pxe_ipmitool`` driver instead of ``ipmi``.

In the case of bare metal service in the overcloud, you will first have to
configure the deployment to include the pxe_ipmitool driver, then rerun the
deployment command,
for example:

.. code-block:: yaml

 parameter_defaults:
   IronicEnabledDrivers:
       - pxe_ipmitool
       - pxe_ssh


Before updating to Pike release, make sure to remove the pxe_ssh driver from the
deployment configuration, as it will be removed from Ironic, then rerun
the deployment command,
for example:

.. code-block:: yaml

 parameter_defaults:
   IronicEnabledDrivers:
       - pxe_ipmitool

To validate after updating deployment and verify everything is populated properly:

.. code-block:: bash

    openstack baremetal node validate $NODE_UUID_OR_NAME | grep power

.. _VirtualBMC: https://opendev.org/openstack/virtualbmc
.. _IPMI: https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface
.. _BMC: https://en.wikipedia.org/wiki/Baseboard_management_controller
