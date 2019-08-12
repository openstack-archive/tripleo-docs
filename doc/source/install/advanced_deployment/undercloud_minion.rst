Installing a Undercloud Minion
==============================

.. note::
   This is optional functionality that is helpful for large scale related
   deployments.

.. note::
   The minion functionality is only available starting from the Train cycle.

The undercloud can be scaled horizontally by installing and configuring undercloud
minions. The minions can expand the number of heat-engine and ironic-conductors
available the overall undercloud installation.  The undercloud minions can be
added and removed as necessary to scale processing during a deployment.

Installation Steps
------------------

.. note::
   The minion requires an undercloud has been installed. The undercloud
   installation process has two output files that we will need to install the
   minion.

#. Log in to your machine (baremetal or VM) where you want to install the
   minion as a non-root user (such as the stack user)::

       ssh <non-root-user>@<minion-machine>

   .. note::
      If you don't have a non-root user created yet, log in as root and create
      one with following commands::

          sudo useradd stack
          sudo passwd stack  # specify a password

          echo "stack ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/stack
          sudo chmod 0440 /etc/sudoers.d/stack

          su - stack

   .. note::
      The minion is intended to work correctly with SELinux enforcing.
      Installations with the permissive/disabled SELinux are not recommended.
      The ``minion_enable_selinux`` config option controls that setting.

   .. note::
      vlan tagged interfaces must follow the if_name.vlan_id convention, like for
      example: eth0.vlan100 or bond0.vlan120.

#. Enable needed repositories:

   .. admonition:: RHEL
      :class: rhel

      Enable optional repo::

          sudo yum install -y yum-utils
          sudo yum-config-manager --enable rhelosp-rhel-7-server-opt

   .. include:: ../repositories.txt

.. We need to manually continue our list numbering here since the above
  "include" directive breaks the numbering.

3. Install the TripleO CLI, which will pull in all other necessary packages as dependencies::

    sudo yum install -y python-tripleoclient

#. Copy the `tripleo-undercloud-outputs.yaml` and `tripleo-undercloud-passwords.yaml`
   from the undercloud to the node being provisioned as a minion::

    scp tripleo-undercloud-outputs.yaml tripleo-undercloud-passwords.yaml <non-root-user>@<minion-machine>:

#. Prepare the configuration file::

    cp /usr/share/python-tripleoclient/minion.conf.sample ~/minion.conf

   Update the settings in this file to match the desired configuration. The
   options in the minion.conf are similarly configured as the undercloud.conf
   on the undercloud node. It is important to configure the `minion_local_ip`
   and the `minion_local_interface` to match the available interfaces on the
   minion system.

   .. note::
      The minion configured interface and ip must be on the control plane network.

#. Run the command to install the minion:

   To deploy a minion::

    openstack undercloud minion install

#. Verify services

   - Heat Engine

     By default only the heat-engine service is configured. To verify it has
     been configured correctly, run the following on the undercloud::

       source ~/stackrc
       openstack orchestration service list

     Example output::

       (undercloud) [stack@undercloud ~]$ openstack orchestration service list
       +------------------------+-------------+--------------------------------------+------------------------+--------+----------------------------+--------+
       | Hostname               | Binary      | Engine ID                            | Host                   | Topic  | Updated At                 | Status |
       +------------------------+-------------+--------------------------------------+------------------------+--------+----------------------------+--------+
       | undercloud.localdomain | heat-engine | b1af4e18-6859-4b73-b1cf-87674bd0ce1f | undercloud.localdomain | engine | 2019-07-25T23:19:34.000000 | up     |
       | minion.localdomain     | heat-engine | 3a0d7080-06a9-4049-bb00-dbdcafbce0fc | minion.localdomain     | engine | 2019-07-25T23:19:24.000000 | up     |
       | undercloud.localdomain | heat-engine | f6ccea46-2b30-4869-b06f-935c342a9ed6 | undercloud.localdomain | engine | 2019-07-25T23:19:34.000000 | up     |
       | minion.localdomain     | heat-engine | eef759de-f7d3-472a-afbc-878eb6a3b9c0 | minion.localdomain     | engine | 2019-07-25T23:19:24.000000 | up     |
       | minion.localdomain     | heat-engine | 7f076afe-5116-45ad-9f08-aab7fbfda40b | minion.localdomain     | engine | 2019-07-25T23:19:24.000000 | up     |
       | undercloud.localdomain | heat-engine | 038ead61-91f1-4739-8537-df63a9e2c917 | undercloud.localdomain | engine | 2019-07-25T23:19:34.000000 | up     |
       | undercloud.localdomain | heat-engine | f16a4f55-b053-4650-9202-781aef55698e | undercloud.localdomain | engine | 2019-07-25T23:19:36.000000 | up     |
       | minion.localdomain     | heat-engine | e853d9c9-9f75-4958-ad9b-49e4b63b79b2 | minion.localdomain     | engine | 2019-07-25T23:19:24.000000 | up     |
       +------------------------+-------------+--------------------------------------+------------------------+--------+----------------------------+--------+


   - Ironic Conductor

     If the ironic conductor service has been enabled, run the following on the
     undercloud::

       source ~/stackrc
       openstack baremetal conductor list

     Example output::

       (undercloud) [stack@undercloud ~]$ openstack baremetal conductor list
       +------------------------+-----------------+-------+
       | Hostname               | Conductor Group | Alive |
       +------------------------+-----------------+-------+
       | undercloud.localdomain |                 | True  |
       | minion.localdomain     |                 | True  |
       +------------------------+-----------------+-------+

