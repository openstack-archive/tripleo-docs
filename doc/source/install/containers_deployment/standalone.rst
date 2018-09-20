Standalone Containers based Deployment
======================================

.. warning::
   This currently is only supported in Rocky or newer versions.

This documentation explains how the underlying framework used by the
Containterized Undercloud deployment mechanism can be reused to deploy a single
node capable of running OpenStack services for development.


Deploying a Standalone Keystone node
------------------------------------

#. Log into your machine (baremetal or VM) where you want to install the
   standalone services on as a non-root user.::

       ssh <non-root-user>@<machine>

#. Enable needed repositories:

   .. admonition:: RHEL
      :class: rhel

      Enable optional repo::

          sudo yum install -y yum-utils
          sudo yum-config-manager --enable rhelosp-rhel-7-server-opt

   .. include:: ../repositories.txt

#. Install the TripleO CLI, which will pull in all other necessary packages as dependencies::

    sudo yum install -y python-tripleoclient

   .. admonition:: Ceph
      :class: ceph

      Install the ceph-ansible package and util-linux.

      .. code-block:: bash

         sudo yum install -y ceph-ansible util-linux

#. Generate a file with the default ContainerImagePrepare value::

    openstack tripleo container image prepare default \
      --output-env-file $HOME/containers-prepare-parameters.yaml

   .. admonition:: Ceph
      :class: ceph

      Create a block device to be used as an OSD.

      .. code-block:: bash

         sudo dd if=/dev/zero of=/var/lib/ceph-osd.img bs=1 count=0 seek=7G
         sudo losetup /dev/loop3 /var/lib/ceph-osd.img

      Create a directory to back up the ceph-ansible fetch directory.

      .. code-block:: bash

         mkdir /root/ceph_ansible_fetch

#. Configure basic standalone parameters which include network configuration
   and some deployment options.

   The following configuration can be used for a system with 2 network
   interfaces. This configuration assumes the first interface is used for
   management and we will only configure the second interface. The deployment
   assumes the second interface has a "public" /24 network which will be used
   for the cloud endpoints and public VM connectivity.

   .. code-block:: bash

      # EXAMPLE: 2 interfaces
      # NIC1 - management NIC (any address, left untouched)
      # NIC2 - OpenStack & Provider network NIC ($INTERFACE configured with $IP, $NETMASK)
      export IP=192.168.24.2
      export NETMASK=24
      export INTERFACE=eth1

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
        NeutronDnsDomain: localdomain
        # re-use ctlplane bridge for public net, defined in the standalone
        # net config (do not change unless you know what you're doing)
        NeutronBridgeMappings: datacentre:br-ctlplane
        NeutronPhysicalBridge: br-ctlplane
        # enable to force metadata for public net
        #NeutronEnableForceMetadata: true
        StandaloneEnableRoutedNetworks: false
        StandaloneHomeDir: $HOME
        StandaloneLocalMtu: 1500
        # Needed if running in a VM, not needed if on baremetal
        StandaloneExtraConfig:
          nova::compute::libvirt::services::libvirt_virt_type: qemu
          nova::compute::libvirt::libvirt_virt_type: qemu
      EOF

   The following configuration can be used for a system with a single network
   interface. This configuration assumes that the interface is shared for
   management and cloud functions. This configuration requires there be at
   least 3 ip addresses available for configuration. 1 ip is used for the
   cloud endpoints, 1 is used for an internal router and 1 is used as a
   floating IP.

   .. code-block:: bash

      # EXAMPLE: 1 interface
      # NIC1 - management, OpenStack, & Provider network ($INTERFACE reconfigured using $IP, $NETMASK, $GATEWAY)
      export IP=192.168.24.2
      export NETMASK=24
      # We need the gateway as we'll be reconfiguring the eth0 interface
      export GATEWAY=192.168.24.1
      export INTERFACE=eth0

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
        NeutronDnsDomain: localdomain
        # re-use ctlplane bridge for public net, defined in the standalone
        # net config (do not change unless you know what you're doing)
        NeutronBridgeMappings: datacentre:br-ctlplane
        NeutronPhysicalBridge: br-ctlplane
        # enable to force metadata for public net
        #NeutronEnableForceMetadata: true
        StandaloneEnableRoutedNetworks: false
        StandaloneHomeDir: $HOME
        StandaloneLocalMtu: 1500
        # Needed if running in a VM, not needed if on baremetal
        StandaloneExtraConfig:
          nova::compute::libvirt::services::libvirt_virt_type: qemu
          nova::compute::libvirt::libvirt_virt_type: qemu
      EOF

   .. admonition:: Ceph
      :class: ceph

      Create an additional environment file which directs ceph-ansible
      to use the block device and fecth directory backup created
      earlier. In the same file pass additional Ceph parameters
      for the OSD scenario and Ceph networks. Set the placement group
      and replica count to values which fit the number of OSDs being
      used, e.g. 32 and 1 are used for testing with only one OSD.

      .. code-block:: bash

         cat <<EOF > $HOME/ceph_parameters.yaml
         parameter_defaults:
           CephAnsibleDisksConfig:
             devices:
               - /dev/loop3
             journal_size: 1024
           LocalCephAnsibleFetchDirectoryBackup: /root/ceph_ansible_fetch
           CephAnsibleExtraConfig:
             osd_scenario: collocated
             osd_objectstore: filestore
             cluster_network: 192.168.24.0/24
             public_network: 192.168.24.0/24
           CephPoolDefaultPgNum: 32
           CephPoolDefaultSize: 1
         EOF

#. Run deploy command:

   .. code-block:: bash

    sudo openstack tripleo deploy \
      --templates \
      --local-ip=$IP/$NETMASK \
      -e /usr/share/openstack-tripleo-heat-templates/environments/standalone.yaml \
      -r /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml \
      -e $HOME/containers-prepare-parameters.yaml \
      -e $HOME/standalone_parameters.yaml \
      --output-dir $HOME \
      --standalone

   .. admonition:: Ceph
      :class: ceph

      Include the Ceph environment files in the deploy command:

      .. code-block:: bash

         sudo openstack tripleo deploy \
           --templates \
           --local-ip=$IP/$NETMASK \
           -e /usr/share/openstack-tripleo-heat-templates/environments/standalone.yaml \
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


Example: 1 NIC, Using Compute with Tenant and Provider Networks
---------------------------------------------------------------

The following example is based on the single NIC configuration and assumes that
the environment had at least 3 total IP addresses available to it. The IPs are
used for the following:

- 1 IP address for the OpenStack services (this is the --local-ip from the
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

- 1 IP address for the OpenStack services (this is the --local-ip from the
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
