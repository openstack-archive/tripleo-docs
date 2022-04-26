.. _standalone:

Standalone Containers based Deployment
======================================

.. warning::
   Standalone deployments are currently only supported in Rocky or newer
   versions.

This documentation explains how the underlying framework used by the
Containerized Undercloud deployment mechanism can be reused to deploy a
single node capable of running OpenStack services for development. Optional
instructions for installing Ceph are included as well.

System Requirements for a Standalone Deployment
-----------------------------------------------

   .. include:: ../environments/standalone.rst
       :start-after: .. include_after_header

Deploying a Standalone OpenStack node
-------------------------------------

#. Copy your SSH key to a non-root user on your machine (baremetal or VM)
   where you want to install the standalone services.::

       ssh-copy-id -i ~/.ssh/<your ssh key> <non-root-user>@<machine>

#. Connect to your machine as the non-root user.::

       ssh <non-root-user>@<machine>

#. Ensure a fully qualified hostname has been configured on the host being
   deployed on. For example::

       sudo hostnamectl set-hostname standalone.localdomain
       sudo hostnamectl set-hostname standalone.localdomain --transient

#. Enable needed repositories:

   .. note::
      The Ussuri version of TripleO utilizes Python3 and requires the OS
      to be RHEL 8 or CentOS 8. It is also recommended that Train be installed
      on RHEL 8 or CentOS 8.

   .. admonition:: RHEL
      :class: rhel

      Enable optional repo for RHEL 7::

          sudo yum install -y yum-utils
          sudo yum-config-manager --enable rhelosp-rhel-7-server-opt

   .. include:: ../repositories.rst

#. Install the TripleO CLI, which will pull in all other necessary packages as dependencies.

    For Python3::

       sudo yum install -y python3-tripleoclient

    For Python2.7::

       sudo yum install -y python-tripleoclient

   .. admonition:: Ceph
      :class: ceph

      Install the packages necessary to deploy Ceph.

      .. code-block:: bash

         sudo yum install -y util-linux lvm2 cephadm

      .. admonition:: Victoria or earlier

        .. code-block:: bash

           sudo yum install -y util-linux lvm2 ceph-ansible

#. Generate a file with the default ContainerImagePrepare value::

    openstack tripleo container image prepare default \
      --output-env-file $HOME/containers-prepare-parameters.yaml

   .. note::
      Update containers-prepare-parameters.yaml for your own needs.
      See :ref:`prepare-environment-containers` for more details.

   .. admonition:: Ceph
      :class: ceph

      Create a block device with logical volumes to be used as an OSD.

      .. code-block:: bash

         sudo dd if=/dev/zero of=/var/lib/ceph-osd.img bs=1 count=0 seek=7G
         sudo losetup /dev/loop3 /var/lib/ceph-osd.img
         sudo pvcreate /dev/loop3
         sudo vgcreate vg2 /dev/loop3
         sudo lvcreate -n data-lv2 -l 597 vg2
         sudo lvcreate -n db-lv2 -l 1194 vg2

      Create a systemd service that restores the device on startup.

      .. code-block:: bash

         cat <<EOF > /tmp/ceph-osd-losetup.service
         [Unit]
         Description=Ceph OSD losetup
         After=syslog.target

         [Service]
         Type=oneshot
         ExecStart=/bin/bash -c '/sbin/losetup /dev/loop3 || \
         /sbin/losetup /dev/loop3 /var/lib/ceph-osd.img ; partprobe /dev/loop3'
         ExecStop=/sbin/losetup -d /dev/loop3
         RemainAfterExit=yes

         [Install]
         WantedBy=multi-user.target
         EOF

         sudo mv /tmp/ceph-osd-losetup.service /etc/systemd/system/
         sudo systemctl enable ceph-osd-losetup.service

#. Configure basic standalone parameters which include network configuration
   and some deployment options.

   .. warning::
      The standalone deployment requires one network interface on the deployment
      machine  and **that interface will be reconfigured as per the parameters
      you specify below**. The interface you want to use is specified by name
      in the $INTERFACE parameter below and passed as the NeutronPublicInterface
      in the standalone_parameters.yaml. If you only have one interface on your
      machine be advised that it will be reconfigured to have the IP address
      specified in $IP. If that is a remote box you may lose connectivity to it.
      Any other network interfaces are left untouched.

   For the standalone deployment we use a single NIC on the target machine
   which is reconfigured and set as a member of an ovs bridge, **br-ctlplane**. Two
   examples follow which can be copy/pasted as is - depending on your setup.
   You should only have to change the name of the interface to match whatever
   it is called on your system. Ideally you will have two network interfaces,
   so that one is used for the standalone deployment, whilst the other will be
   left untouched. This can be especially important if you are deploying on a
   remote box (e.g. via ssh).

   The following configuration can be used for a system with 2 network
   interfaces. This configuration assumes the first interface is used for
   management and we will only configure the second interface. The deployment
   assumes the second interface has a "public" /24 network which will be used
   for the cloud endpoints and public VM connectivity.

   In addition to the IPs used on eth1, a virtual IP will be added and managed
   by pacemaker. This must be a different address to the other IP as one will
   be bound to by haproxy while the other by backend services on the same.

   .. Note: The following example utilizes 2 interfaces. NIC1 which will serve as
      the management inteface. It can have any address and will be left untouched.
      NIC2 will serve as the OpenStack & Provider network NIC. The following
      exports should be configured for your network and interface.

   .. code-block:: bash

      export IP=192.168.24.2
      export VIP=192.168.24.3
      export NETMASK=24
      export INTERFACE=eth1

   You will now create the standalone_parameters.yaml. The $IP, $VIP, $NETMASK,
   and $INTERFACE will be replaced with the values from the export commands.

   .. code-block:: bash

      cat <<EOF > $HOME/standalone_parameters.yaml
      parameter_defaults:
        CloudName: $IP
        ControlPlaneStaticRoutes: []
        Debug: true
        DeploymentUser: $USER
        DnsServers:
          - 1.1.1.1
          - 8.8.8.8
        DockerInsecureRegistryAddress:
          - $IP:8787
        NeutronPublicInterface: $INTERFACE
        # domain name used by the host
        CloudDomain: localdomain
        NeutronDnsDomain: localdomain
        # re-use ctlplane bridge for public net, defined in the standalone
        # net config (do not change unless you know what you're doing)
        NeutronBridgeMappings: datacentre:br-ctlplane
        NeutronPhysicalBridge: br-ctlplane
        # enable to force metadata for public net
        #NeutronEnableForceMetadata: true
        StandaloneEnableRoutedNetworks: false
        StandaloneHomeDir: $HOME
        InterfaceLocalMtu: 1500
        # Needed if running in a VM, not needed if on baremetal
        NovaComputeLibvirtType: qemu
      EOF

   The following configuration can be used for a system with a single network
   interface. This configuration assumes that the interface is shared for
   management and cloud functions. This configuration requires there be at
   least 3 ip addresses available for configuration. 1 ip is used for the
   cloud endpoints, 1 is used for an internal router and 1 is used as a
   floating IP.

   .. Note: NIC1 will serve as the management, OpenStack and Provider network
      inteface. The exports should be configured for your network and interface.

   .. code-block:: bash

      export IP=192.168.24.2
      export VIP=192.168.24.3
      export NETMASK=24
      export GATEWAY=192.168.24.1
      export INTERFACE=eth0

   You will now create the standalone_parameters.yaml. The $IP, $NETMASK,
   $GATEWAY, and $INTERFACE will be replaced with the values from the export
   commands.

   .. code-block:: bash

      cat <<EOF > $HOME/standalone_parameters.yaml
      parameter_defaults:
        CloudName: $IP
        # default gateway
        ControlPlaneStaticRoutes:
          - ip_netmask: 0.0.0.0/0
            next_hop: $GATEWAY
            default: true
        Debug: true
        DeploymentUser: $USER
        DnsServers:
          - 1.1.1.1
          - 8.8.8.8
        # needed for vip & pacemaker
        KernelIpNonLocalBind: 1
        DockerInsecureRegistryAddress:
          - $IP:8787
        NeutronPublicInterface: $INTERFACE
        # domain name used by the host
        CloudDomain: localdomain
        NeutronDnsDomain: localdomain
        # re-use ctlplane bridge for public net, defined in the standalone
        # net config (do not change unless you know what you're doing)
        NeutronBridgeMappings: datacentre:br-ctlplane
        NeutronPhysicalBridge: br-ctlplane
        # enable to force metadata for public net
        #NeutronEnableForceMetadata: true
        StandaloneEnableRoutedNetworks: false
        StandaloneHomeDir: $HOME
        InterfaceLocalMtu: 1500
        # Needed if running in a VM, not needed if on baremetal
        NovaComputeLibvirtType: qemu
      EOF

   .. admonition:: Ceph
      :class: ceph

      Create an additional environment file which directs the ceph
      deployment tool to use the block device with logical volumes
      created earlier. Set the placement group and replica count to
      values which fit the number of OSDs being used, e.g. 8
      and 1 are used for testing with only one OSD.

      .. code-block:: bash

         cat <<EOF > $HOME/ceph_parameters.yaml
         parameter_defaults:
           CephSpecFqdn: true
           CephOsdSpec:
            data_devices:
              paths:
                - /dev/vg2/data-lv2
           CephConfigOverrides:
             mon:
               mon_warn_on_pool_no_redundancy: false
           CephPoolDefaultPgNum: 8
           CephPoolDefaultSize: 1
         EOF

      .. admonition:: Victoria or earlier

        .. code-block:: bash

           cat <<EOF > $HOME/ceph_parameters.yaml
           parameter_defaults:
             CephAnsibleDisksConfig:
               osd_scenario: lvm
               osd_objectstore: bluestore
               lvm_volumes:
                 - data: data-lv2
                   data_vg: vg2
                   db: db-lv2
                   db_vg: vg2
             CephAnsibleExtraConfig:
               cluster_network: 192.168.24.0/24
               public_network: 192.168.24.0/24
             CephPoolDefaultPgNum: 8
             CephPoolDefaultSize: 1
           EOF

#. Run the deploy command:

   .. code-block:: bash

    sudo openstack tripleo deploy \
      --templates \
      --local-ip=$IP/$NETMASK \
      --control-virtual-ip $VIP \
      -e /usr/share/openstack-tripleo-heat-templates/environments/standalone/standalone-tripleo.yaml \
      -r /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml \
      -e $HOME/containers-prepare-parameters.yaml \
      -e $HOME/standalone_parameters.yaml \
      --output-dir $HOME

   .. admonition:: Ceph
      :class: ceph

      Include the Ceph environment files in the deploy command:

      .. code-block:: bash

         sudo openstack tripleo deploy \
           --templates \
           --local-ip=$IP/$NETMASK \
           --control-virtual-ip $VIP \
           -e /usr/share/openstack-tripleo-heat-templates/environments/standalone/standalone-tripleo.yaml \
           -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml \
           -r /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml \
           -e $HOME/containers-prepare-parameters.yaml \
           -e $HOME/standalone_parameters.yaml \
           -e $HOME/ceph_parameters.yaml \
           --output-dir $HOME

      .. admonition:: Victoria or earlier

        .. code-block:: bash

           sudo openstack tripleo deploy \
             --templates \
             --local-ip=$IP/$NETMASK \
             --control-virtual-ip $VIP \
               -e /usr/share/openstack-tripleo-heat-templates/environments/standalone/standalone-tripleo.yaml \
             -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
             -r /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml \
             -e $HOME/containers-prepare-parameters.yaml \
             -e $HOME/standalone_parameters.yaml \
             -e $HOME/ceph_parameters.yaml \
             --output-dir $HOME \
             --standalone

#. Check the deployed OpenStack Services

   At the end of the deployment, a clouds.yaml configuration file is placed in
   the /root/.config/openstack folder. This can be used with the openstack
   client to query the OpenStack services.

   .. code-block:: bash

     export OS_CLOUD=standalone
     openstack endpoint list

#. Cleanup a deployment

   If you want to remove the services and files installed by Standalone after
   a deployment failure, or just to re-deploy from scratch, you can run the
   following script:

   .. code-block:: bash

     #!/bin/bash
     echo "Tearing down TripleO environment"
     if type pcs &> /dev/null; then
         sudo pcs cluster destroy
     fi
     if type podman &> /dev/null; then
         echo "Removing podman containers and images (takes times...)"
         sudo podman rm -af
         sudo podman rmi -af
     fi
     sudo rm -rf \
         /var/lib/tripleo-config \
         /var/lib/config-data /var/lib/container-config-scripts \
         /var/lib/container-puppet \
         /var/lib/heat-config \
         /var/lib/image-serve \
         /var/lib/containers \
         /etc/systemd/system/tripleo* \
         /var/lib/mysql/* \
         /etc/openstack
     rm -rf ~/.config/openstack
     sudo systemctl daemon-reload

Manual deployments with ansible
-------------------------------

With the ``--output-only`` option enabled, the installation stops before Ansible
playbooks would be normally executed. Instead, it only creates a Heat stack,
then downloads the ansible deployment data and playbooks to ``--output-dir`` for
the manual execution.

.. note::
   When updating the existing standalone installation, keep in mind the
   special cases described in :ref:`notes-for-stack-updates`. There is an
   additional case for the ``--force-stack-update`` flag that might need to be
   used, when in the ``--output-only`` mode.  That is when you cannot know the
   results of the actual deployment before ansible has started.

Example: 1 NIC, Using Compute with Tenant and Provider Networks
---------------------------------------------------------------

The following example is based on the single NIC configuration and assumes that
the environment had at least 3 total IP addresses available to it. The IPs are
used for the following:

- 1 IP address for the OpenStack services (this is the ``--local-ip`` from the
  deploy command)
- 1 IP used as a Virtual Router to provide connectivity to the Tenant network
  is used for the OpenStack services (is automatically assigned in this example)
- The remaining IP addresses (at least 1) are used for Floating IPs on the
  provider network.

The following is an example post deployment launching of a VM using the
private tenant network and the provider network.

#. Create helper variables for the configuration::

    # standalone with tenant networking and provider networking
    export OS_CLOUD=standalone
    export GATEWAY=192.168.24.1
    export STANDALONE_HOST=192.168.24.2
    export PUBLIC_NETWORK_CIDR=192.168.24.0/24
    export PRIVATE_NETWORK_CIDR=192.168.100.0/24
    export PUBLIC_NET_START=192.168.24.4
    export PUBLIC_NET_END=192.168.24.5
    export DNS_SERVER=1.1.1.1

#. Initial Nova and Glance setup::

    # nova flavor
    openstack flavor create --ram 512 --disk 1 --vcpu 1 --public tiny
    # basic cirros image
    wget https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
    openstack image create cirros --container-format bare --disk-format qcow2 --public --file cirros-0.4.0-x86_64-disk.img
    # nova keypair for ssh
    ssh-keygen
    openstack keypair create --public-key ~/.ssh/id_rsa.pub default

#. Setup a simple network security group::

    # create basic security group to allow ssh/ping/dns
    openstack security group create basic
    # allow ssh
    openstack security group rule create basic --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0
    # allow ping
    openstack security group rule create --protocol icmp basic
    # allow DNS
    openstack security group rule create --protocol udp --dst-port 53:53 basic

#. Create Neutron Networks::

    openstack network create --external --provider-physical-network datacentre --provider-network-type flat public
    openstack network create --internal private
    openstack subnet create public-net \
        --subnet-range $PUBLIC_NETWORK_CIDR \
        --no-dhcp \
        --gateway $GATEWAY \
        --allocation-pool start=$PUBLIC_NET_START,end=$PUBLIC_NET_END \
        --network public
    openstack subnet create private-net \
        --subnet-range $PRIVATE_NETWORK_CIDR \
        --network private

#. Create Virtual Router::

    # create router
    # NOTE(aschultz): In this case an IP will be automatically assigned
    # out of the allocation pool for the subnet.
    openstack router create vrouter
    openstack router set vrouter --external-gateway public
    openstack router add subnet vrouter private-net

#. Create floating IP::

    # create floating ip
    openstack floating ip create public

#. Launch Instance::

    # launch instance
    openstack server create --flavor tiny --image cirros --key-name default --network private --security-group basic myserver

#. Assign Floating IP::

    openstack server add floating ip myserver <FLOATING_IP>

#. Test SSH::

    # login to vm
    ssh cirros@<FLOATING_IP>


Networking Details
~~~~~~~~~~~~~~~~~~

Here's a basic diagram of where the connections occur in the system for this
example::

     +-------------------------------------------------------+
     |Standalone Host                                        |
     |                                                       |
     |              +----------------------------+           |
     |              |          vrouter           |           |
     |              |                            |           |
     |              +------------+ +-------------+           |
     |              |192.168.24.4| |             |           |
     |              |192.168.24.3| |192.168.100.1|           |
     |              +---------+------+-----------+           |
     |      +-------------+   |      |                       |
     |      |  myserver   |   |      |                       |
     |      |192.168.100.2|   |      |                       |
     |      +-------+-----+   |    +-+                       |
     |              |         |    |                         |
     |              |         |    |                         |
     |             ++---------+----+-+   +-----------------+ |
     |             |     br-int      +---+   br-ctlplane   | |
     |             |                 |   |  192.168.24.2   | |
     |             +------+----------+   +--------+--------+ |
     |                    |                       |          |
     |             +------+----------+            |          |
     |             |     br-tun      |            |          |
     |             |                 |            |          |
     |             +-----------------+       +----+---+      |
     |                                       |  eth0  |      |
     +---------------------------------------+----+---+------+
                                                  |
                                                  |
                                          +-------+-----+
                                          |   switch    |
                                          +-------------+

Example: 1 NIC, Using Compute with Provider Network
---------------------------------------------------

The following example is based on the single NIC configuration and assumes that
the environment had at least 4 total IP addresses available to it. The IPs are
used for the following:

- 1 IP address for the OpenStack services (this is the ``--local-ip`` from the
  deploy command)
- 1 IP used as a Virtual Router to provide connectivity to the Tenant network
  is used for the OpenStack services
- 1 IP used for DHCP on the provider network
- The remaining IP addresses (at least 1) are used for Floating IPs on the
  provider network.

The following is an example post deployment launching of a VM using the
private tenant network and the provider network.

#. Create helper variables for the configuration::

    # standalone with provider networking
    export OS_CLOUD=standalone
    export GATEWAY=192.168.24.1
    export STANDALONE_HOST=192.168.24.2
    export VROUTER_IP=192.168.24.3
    export PUBLIC_NETWORK_CIDR=192.168.24.0/24
    export PUBLIC_NET_START=192.168.24.4
    export PUBLIC_NET_END=192.168.24.5
    export DNS_SERVER=1.1.1.1

#. Initial Nova and Glance setup::

    # nova flavor
    openstack flavor create --ram 512 --disk 1 --vcpu 1 --public tiny
    # basic cirros image
    wget https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
    openstack image create cirros --container-format bare --disk-format qcow2 --public --file cirros-0.4.0-x86_64-disk.img
    # nova keypair for ssh
    ssh-keygen
    openstack keypair create --public-key ~/.ssh/id_rsa.pub default

#. Setup a simple network security group::

    # create basic security group to allow ssh/ping/dns
    openstack security group create basic
    # allow ssh
    openstack security group rule create basic --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0
    # allow ping
    openstack security group rule create --protocol icmp basic
    # allow DNS
    openstack security group rule create --protocol udp --dst-port 53:53 basic

#. Create Neutron Networks::

    openstack network create --external --provider-physical-network datacentre --provider-network-type flat public
    openstack subnet create public-net \
        --subnet-range $PUBLIC_NETWORK_CIDR \
        --gateway $GATEWAY \
        --allocation-pool start=$PUBLIC_NET_START,end=$PUBLIC_NET_END \
        --network public \
        --host-route destination=169.254.169.254/32,gateway=$VROUTER_IP \
        --host-route destination=0.0.0.0/0,gateway=$GATEWAY \
        --dns-nameserver $DNS_SERVER

#. Create Virtual Router::

    # vrouter needed for metadata route
    # NOTE(aschultz): In this case we're creating a fixed IP because we need
    # to create a manual route in the subnet for the metadata service
    openstack router create vrouter
    openstack port create --network public --fixed-ip subnet=public-net,ip-address=$VROUTER_IP vrouter-port
    openstack router add port vrouter vrouter-port

#. Launch Instance::

    # launch instance
    openstack server create --flavor tiny --image cirros --key-name default --network public --security-group basic myserver

#. Test SSH::

    # login to vm
    ssh cirros@<VM_IP>

Networking Details
~~~~~~~~~~~~~~~~~~

Here's a basic diagram of where the connections occur in the system for this
example::

    +----------------------------------------------------+
    |Standalone Host                                     |
    |                                                    |
    |    +------------+   +------------+                 |
    |    |  myserver  |   |  vrouter   |                 |
    |    |192.168.24.4|   |192.168.24.3|                 |
    |    +---------+--+   +-+----------+                 |
    |              |        |                            |
    |          +---+--------+----+   +-----------------+ |
    |          |     br-int      +---+   br-ctlplane   | |
    |          |                 |   |  192.168.24.2   | |
    |          +------+----------+   +--------+--------+ |
    |                 |                       |          |
    |          +------+----------+            |          |
    |          |     br-tun      |            |          |
    |          |                 |            |          |
    |          +-----------------+       +----+---+      |
    |                                    |  eth0  |      |
    +------------------------------------+----+---+------+
                                              |
                                              |
                                      +-------+-----+
                                      |   switch    |
                                      +-------------+

Example: 2 NIC, Using Compute with Tenant and Provider Networks
---------------------------------------------------------------

The following example is based on the dual NIC configuration and assumes that
the environment has an entire IP range available to it on the provider network.
We are assuming the following would be reserved on the provider network:

- 1 IP address for a gateway on the provider network
- 1 IP address for OpenStack Endpoints
- 1 IP used as a Virtual Router to provide connectivity to the Tenant network
  is used for the OpenStack services (is automatically assigned in this example)
- The remaining IP addresses (at least 1) are used for Floating IPs on the
  provider network.

The following is an example post deployment launching of a VM using the
private tenant network and the provider network.

#. Create helper variables for the configuration::

    # standalone with tenant networking and provider networking
    export OS_CLOUD=standalone
    export GATEWAY=192.168.24.1
    export STANDALONE_HOST=192.168.0.2
    export PUBLIC_NETWORK_CIDR=192.168.24.0/24
    export PRIVATE_NETWORK_CIDR=192.168.100.0/24
    export PUBLIC_NET_START=192.168.0.3
    export PUBLIC_NET_END=192.168.24.254
    export DNS_SERVER=1.1.1.1

#. Initial Nova and Glance setup::

    # nova flavor
    openstack flavor create --ram 512 --disk 1 --vcpu 1 --public tiny
    # basic cirros image
    wget https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
    openstack image create cirros --container-format bare --disk-format qcow2 --public --file cirros-0.4.0-x86_64-disk.img
    # nova keypair for ssh
    ssh-keygen
    openstack keypair create --public-key ~/.ssh/id_rsa.pub default

#. Setup a simple network security group::

    # create basic security group to allow ssh/ping/dns
    openstack security group create basic
    # allow ssh
    openstack security group rule create basic --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0
    # allow ping
    openstack security group rule create --protocol icmp basic
    # allow DNS
    openstack security group rule create --protocol udp --dst-port 53:53 basic

#. Create Neutron Networks::

    openstack network create --external --provider-physical-network datacentre --provider-network-type flat public
    openstack network create --internal private
    openstack subnet create public-net \
        --subnet-range $PUBLIC_NETWORK_CIDR \
        --no-dhcp \
        --gateway $GATEWAY \
        --allocation-pool start=$PUBLIC_NET_START,end=$PUBLIC_NET_END \
        --network public
    openstack subnet create private-net \
        --subnet-range $PRIVATE_NETWORK_CIDR \
        --network private

#. Create Virtual Router::

    # create router
    # NOTE(aschultz): In this case an IP will be automatically assigned
    # out of the allocation pool for the subnet.
    openstack router create vrouter
    openstack router set vrouter --external-gateway public
    openstack router add subnet vrouter private-net

#. Create floating IP::

    # create floating ip
    openstack floating ip create public

#. Launch Instance::

    # launch instance
    openstack server create --flavor tiny --image cirros --key-name default --network private --security-group basic myserver

#. Assign Floating IP::

    openstack server add floating ip myserver <FLOATING_IP>

#. Test SSH::

    # login to vm
    ssh cirros@<FLOATING_IP>

Networking Details
~~~~~~~~~~~~~~~~~~

Here's a basic diagram of where the connections occur in the system for this
example::

    +---------------------------------------------------------------------+
    |Standalone Host                                                      |
    |                                                                     |
    |            +----------------------------+                           |
    |            |          vrouter           |                           |
    |            |                            |                           |
    |            +------------+ +-------------+                           |
    |            |192.168.24.4| |             |                           |
    |            |192.168.24.3| |192.168.100.1|                           |
    |            +---------+------+-----------+                           |
    |    +-------------+   |      |                                       |
    |    |  myserver   |   |      |                                       |
    |    |192.168.100.2|   |      |                                       |
    |    +-------+-----+   |    +-+                                       |
    |            |         |    |                                         |
    |           ++---------+----+-+   +-----------------+                 |
    |           |     br-int      +---+   br-ctlplane   |                 |
    |           |                 |   |  192.168.24.2   |                 |
    |           +------+----------+   +------------+----+                 |
    |                  |                           |                      |
    |           +------+----------+                |                      |
    |           |     br-tun      |                |                      |
    |           |                 |                |                      |
    |           +-----------------+                |       +----------+   |
    |                                        +-----+---+   |   eth0   |   |
    |                                        |  eth1   |   | 10.0.1.4 |   |
    +----------------------------------------+-----+---+---+-----+----+---+
                                                   |             |
                                                   |             |
                                            +------+------+      |
                                            |   switch    +------+
                                            +-------------+

Example: 2 nodes, 2 NIC, Using remote Compute with Tenant and Provider Networks
-------------------------------------------------------------------------------

The following example uses two nodes and the split control plane
method to simulate a distributed edge computing deployment. The first
Heat stack deploys a controller node which could run in a Centralized
Data Center. The second Heat stack deploys a second node which could
run at another location on the Aggregation Edge Layer. The second node
runs the nova-compute service, Ceph, and the cinder-volume service.
Both nodes use the networking configuration found in the 2 NIC, Using
Compute with Tenant and Provider Network example.

Deploy the central controller node
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To deploy the first node, follow the Deploying a Standalone OpenStack
node section described earlier in the document but also include the
following parameters:

.. code-block:: yaml

    parameter_defaults:
      GlanceBackend: swift
      StandaloneExtraConfig:
        oslo_messaging_notify_use_ssl: false
        oslo_messaging_rpc_use_ssl: false

The above configures the Swift backend for Glance so that images are
pulled by the remote compute node over HTTP and ensures that Oslo
messaging does not use SSL for RPC and notifications. Note that in a
production deployment this will result in sending unencrypted traffic
over WAN connections.

When configuring the network keep in mind that it will be necessary
for both standalone systems to be able to communicate with each
other. E.g. the $IP for the first node will be in the endpoint map
that later will be extracted from the first node and passed as a
parameter to the second node for it to access its endpoints. In this
standalone example both servers share an L2 network. In a production
edge deployment it may be necessary instead to route.

When deploying the first node with ``openstack tripleo deploy``, pass
the ``--keep-running`` option so the Heat processes continue to run.

Extract deployment information from the controller node
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Heat processes were kept running in the previous step because
this allows the Heat stack to be queried after the deployment in order
to extract parameters that the second node's deployment will need as
input. To extract these parameters into separate files in a directory,
(e.g. `DIR=export_control_plane`), which may then be exported to the
second node, run the following:

.. code-block:: bash

  unset OS_CLOUD
  export OS_AUTH_TYPE=none
  export OS_ENDPOINT=http://127.0.0.1:8006/v1/admin

  openstack stack output show standalone EndpointMap --format json \
  | jq '{"parameter_defaults": {"EndpointMapOverride": .output_value}}' \
  > $DIR/endpoint-map.json

  openstack stack output show standalone HostsEntry -f json \
  | jq -r '{"parameter_defaults":{"ExtraHostFileEntries": .output_value}}' \
  > $DIR/extra-host-file-entries.json

In addition to the above create a file in the same directory,
e.g. `$DIR/oslo.yaml`, containing Oslo overrides for the second
compute node:

.. code-block:: yaml

  parameter_defaults:
    StandaloneExtraConfig:
      oslo_messaging_notify_use_ssl: false
      oslo_messaging_rpc_use_ssl: false

In addition to the parameters above, add the
`oslo_messaging_notify_password` and `oslo_messaging_rpc_password`
parameters. Their values may be extracted from
`/etc/puppet/hieradata/service_configs.json` on the first node. The
following command will do this for you:

.. code-block:: bash

  sudo egrep "oslo.*password" /etc/puppet/hieradata/service_configs.json \
  | sed -e s/\"//g -e s/,//g >> $DIR/oslo.yaml

In addition to the above, you need to create $DIR/$HOME/export_control_plane/all-nodes-extra-map-data.json
which will contain the following AllNodesExtraMapData. You first need to locate the
group_vars generated by tripleo-ansible, located in the config-download directory.

Then you can generate the correct Heat environment with the following command:

.. code-block:: bash

   STANDALONE_LATEST=$(find $HOME/standalone-ansible-* -type d -printf "%T@ %p\n" | sort -n | cut -d' ' -f 2- | tail -n 1)
   python3 -c "import json; t = {'parameter_defaults': {'AllNodesExtraMapData': json.loads(open('$HOME/$STANDALONE_LATEST/group_vars/overcloud.json').read()) }}; print(t)" > $DIR/all-nodes-extra-map-data.json

Set a copy of the first node's passwords aside for the second node:

.. code-block:: bash

  cp $HOME/tripleo-standalone-passwords.yaml $DIR/passwords.yaml

Put a copy of the directory containing the extracted information,
e.g. `$DIR`, on the second node to be deployed.

Deploy the remote compute node
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On a second node, follow the procedure at the beginning of this
document to deploy a standalone OpenStack node with Ceph up to the
point where you have the following files:

- `$HOME/standalone_parameters.yaml`
- `$HOME/containers-prepare-parameters.yaml`
- `$HOME/ceph_parameters.yaml`

When setting the `$IP` of the second node, keep in mind that it should
have a way to reach the endpoints of the first node as found in the
endpoint-map.json, which was extracted from the first node.

Create an environment file, e.g. `$HOME/standalone_edge.yaml`, with the
following content:

.. code-block:: yaml

  resource_registry:
    OS::TripleO::Services::CACerts: OS::Heat::None
    OS::TripleO::Services::CinderApi: OS::Heat::None
    OS::TripleO::Services::CinderScheduler: OS::Heat::None
    OS::TripleO::Services::Clustercheck: OS::Heat::None
    OS::TripleO::Services::HAproxy: OS::Heat::None
    OS::TripleO::Services::Horizon: OS::Heat::None
    OS::TripleO::Services::Keystone: OS::Heat::None
    OS::TripleO::Services::Memcached: OS::Heat::None
    OS::TripleO::Services::MySQL: OS::Heat::None
    OS::TripleO::Services::NeutronApi: OS::Heat::None
    OS::TripleO::Services::NeutronDhcpAgent: OS::Heat::None
    OS::TripleO::Services::NovaApi: OS::Heat::None
    OS::TripleO::Services::NovaConductor: OS::Heat::None
    OS::TripleO::Services::NovaConsoleauth: OS::Heat::None
    OS::TripleO::Services::NovaIronic: OS::Heat::None
    OS::TripleO::Services::NovaMetadata: OS::Heat::None
    OS::TripleO::Services::NovaPlacement: OS::Heat::None
    OS::TripleO::Services::NovaScheduler: OS::Heat::None
    OS::TripleO::Services::NovaVncProxy: OS::Heat::None
    OS::TripleO::Services::OsloMessagingNotify: OS::Heat::None
    OS::TripleO::Services::OsloMessagingRpc: OS::Heat::None
    OS::TripleO::Services::Redis: OS::Heat::None
    OS::TripleO::Services::SwiftProxy: OS::Heat::None
    OS::TripleO::Services::SwiftStorage: OS::Heat::None
    OS::TripleO::Services::SwiftRingBuilder: OS::Heat::None

  parameter_defaults:
    CinderRbdAvailabilityZone: edge1
    GlanceBackend: swift
    GlanceCacheEnabled: true

The above file disables additional resources which
`/usr/share/openstack-tripleo-heat-templates/environments/standalone/standalone-tripleo.yaml`
does not disable since it represents a compute node which will consume
those resources from the earlier deployed controller node. It also
sets the Glance blackened to Swift and enables Glance caching so that
after images are pulled from the central node once, they do not need
to be pulled again. Finally the above sets the Cinder RBD availability
zone a separate availability zone for the remote compute and cinder
volume service.

Deploy the second node with the following:

.. code-block:: bash

    sudo openstack tripleo deploy \
        --templates \
        --local-ip=$IP/$NETMASK \
        -r /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml \
        -e /usr/share/openstack-tripleo-heat-templates/environments/standalone/standalone-tripleo.yaml \
        -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
        -e $HOME/containers-prepare-parameters.yaml \
        -e $HOME/standalone_parameters.yaml \
        -e $HOME/ceph_parameters.yaml \
        -e $HOME/standalone_edge.yaml \
        -e $HOME/export_control_plane/passwords.yaml \
        -e $HOME/export_control_plane/endpoint-map.json \
        -e $HOME/export_control_plane/all-nodes-extra-map-data.json \
        -e $HOME/export_control_plane/extra-host-file-entries.json \
        -e $HOME/export_control_plane/oslo.yaml \
        --output-dir $HOME

The example above assumes that ``export_control_plane`` is the name
of the directory which contains the content extracted from the
controller node.

Discover the remote compute node from the central controller node
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

After completing the prior steps, the `openstack` command will only
work on the central node because of how the ``OS_CLOUD`` environment
variable works with that nodes /root/.config/openstack folder, which
in turn assumes that keystone is running the central node and not
the edge nodes. To run `openstack` commands on edge nodes, override
the auth URL to point to keystone on the central node.

On the central controller node run the following command to discover
the new compute node:

.. code-block:: bash

  sudo docker exec -it nova_api nova-manage cell_v2 discover_hosts --verbose

List the available zones, hosts, and hypervisors and look for the new node:

.. code-block:: bash

    export OS_CLOUD=standalone
    openstack availability zone list
    openstack host list
    openstack hypervisor list

Take note of the zone and host list so that you can use that
information to schedule an instance on the new compute node. The
following example shows the result of deploying two new external
compute nodes::

  [root@overcloud0 ~]# sudo docker exec -it nova_api nova-manage cell_v2 discover_hosts --verbose
  Found 2 cell mappings.
  Skipping cell0 since it does not contain hosts.
  Getting computes from cell 'default': 631301c8-1744-4beb-8aa0-6a90aef6cd2d
  Checking host mapping for compute host 'overcloud0.localdomain': 0884a9fc-9ef6-451c-ab22-06f825484e5e
  Checking host mapping for compute host 'overcloud1.localdomain': 00fb920d-ef12-4a2a-9aa4-ba987d8a5e17
  Creating host mapping for compute host 'overcloud1.localdomain': 00fb920d-ef12-4a2a-9aa4-ba987d8a5e17
  Checking host mapping for compute host 'overcloud2.localdomain': 3e3a3cd4-5959-405a-b632-0b64415c43f2
  Creating host mapping for compute host 'overcloud2.localdomain': 3e3a3cd4-5959-405a-b632-0b64415c43f2
  Found 2 unmapped computes in cell: 631301c8-1744-4beb-8aa0-6a90aef6cd2d
  [root@overcloud0 ~]# openstack hypervisor list
  +----+------------------------+-----------------+--------------+-------+
  | ID | Hypervisor Hostname    | Hypervisor Type | Host IP      | State |
  +----+------------------------+-----------------+--------------+-------+
  |  1 | overcloud0.example.com | QEMU            | 192.168.24.2 | up    |
  |  2 | overcloud1.example.com | QEMU            | 192.168.24.7 | up    |
  |  3 | overcloud2.example.com | QEMU            | 192.168.24.8 | up    |
  +----+------------------------+-----------------+--------------+-------+
  [root@overcloud0 ~]#

Note that the hostnames of the hypervisors above were set prior to the
deployment.

On the central controller node run the following to create a host
aggregate for a remote compute node:

.. code-block:: bash

  openstack aggregate create HA-edge1 --zone edge1
  openstack aggregate add host HA-edge1 overcloud1.localdomain

To test, follow the example from "2 NIC, Using remote Compute with
Tenant and Provider Networks", except when creating the instance use
the `--availability-zone` option to schedule the instance on the new
remote compute node:

.. code-block:: bash

  openstack server create --flavor tiny --image cirros \
  --key-name demokp --network private --security-group basic \
  myserver --availability-zone edge1

On the first node, run the following command to create a volume on the
second node:

.. code-block:: bash

  openstack volume create --size 1 --availability-zone edge1 myvol

On the second node, verify that the instance is running locally and
and that the Cinder volume was created on the local Ceph server::

  [root@overcloud1 ~]# docker exec nova_libvirt virsh list
   Id    Name                           State
  ----------------------------------------------------
   1     instance-00000001              running

  [root@overcloud1 ~]# docker exec -ti ceph-mon rbd -p volumes ls -l
  NAME                                        SIZE PARENT FMT PROT LOCK
  volume-f84ae4f5-cc25-4ed4-8a58-8b1408160e03 1GiB          2
  [root@overcloud1 ~]#

Topology Details
~~~~~~~~~~~~~~~~

Here's a basic diagram of where the connections occur in the system for this
example::

  +-------------------------+         +-------------------------+
  |standalone|compute|edge|1|         |standalone|compute|edge|2|
  +-----------------------+-+         +-+-----------------------+
                          |             |
                     +----+-------------+----------+
                     |standalone|controller|central|
                     +-----------------------------+

