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

#. Configure standalone parameters which include container discovery, network
   configuration, and some deployment options.

   The following configuration can be used for a system with 2 network
   interfaces. This configuration assumes the first interface is used for
   management and we will only configure the second interface. The deployment
   assumes the second interface has a "public" /24 network which will be used
   for the cloud endpoints and public VM connectivity.

   .. code-block:: bash

      # EXAMPLE: 2 interfaces
      # TODO(aschultz): This still assumes a bunch of undercloud stuff that may
      # not be needed anymore. Will need to clean this up.

      export IP=192.168.24.2
      export NETMASK=24
      export INTERFACE=eth1

      cat <<EOF > $HOME/standalone_parameters.yaml
      parameter_defaults:
        CertmongerCA: local
        CloudName: $IP
        ContainerImagePrepare:
        - set:
            ceph_image: daemon
            ceph_namespace: docker.io/ceph
            ceph_tag: v3.0.3-stable-3.0-luminous-centos-7-x86_64
            name_prefix: centos-binary-
            name_suffix: ''
            namespace: docker.io/tripleomaster
            neutron_driver: null
            tag: current-tripleo
          tag_from_label: rdo_version
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
        # re-use ctlplane bridge for public net
        NeutronBridgeMappings: datacentre:br-ctlplane
        NeutronPhysicalBridge: br-ctlplane
        # enable to force metadata for public net
        #NeutronEnableForceMetadata: true
        StandaloneEnableRoutedNetworks: false
        StandaloneHomeDir: $HOME
        StandaloneLocalMtu: 1500
        # Needed if running in a VM
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
      # TODO(aschultz): This still assumes a bunch of undercloud stuff that may
      # not be needed anymore. Will need to clean this up.
      export IP=192.168.24.2
      export NETMASK=24
      # We need the gateway as we'll be reconfiguring the eth0 interface
      export GATEWAY=192.168.24.1
      export INTERFACE=eth0

      cat <<EOF > $HOME/standalone_parameters.yaml
      parameter_defaults:
        CertmongerCA: local
        CloudName: $IP
        ContainerImagePrepare:
        - set:
            ceph_image: daemon
            ceph_namespace: docker.io/ceph
            ceph_tag: v3.0.3-stable-3.0-luminous-centos-7-x86_64
            name_prefix: centos-binary-
            name_suffix: ''
            namespace: docker.io/tripleomaster
            neutron_driver: null
            tag: current-tripleo
          tag_from_label: rdo_version
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
        # re-use ctlplane bridge for public net
        NeutronBridgeMappings: datacentre:br-ctlplane
        NeutronPhysicalBridge: br-ctlplane
        # enable to force metadata for public net
        #NeutronEnableForceMetadata: true
        StandaloneEnableRoutedNetworks: false
        StandaloneHomeDir: $HOME
        StandaloneLocalMtu: 1500
        # Needed if running in a VM
        StandaloneExtraConfig:
          nova::compute::libvirt::services::libvirt_virt_type: qemu
          nova::compute::libvirt::libvirt_virt_type: qemu
      EOF

#. Run deploy command::

    sudo openstack tripleo deploy \
      --templates \
      --local-ip=$IP/$NETMASK \
      -e /usr/share/openstack-tripleo-heat-templates/environments/standalone.yaml \
      -r /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml \
      -e $HOME/standalone_parameters.yaml \
      --output-dir $HOME \
      --standalone

#. Check the deployed OpenStack Services

   At the end of the deployment, a clouds.yaml configuration file is placed in
   the /root/.config/openstack folder. This can be used with the openstack
   client to query the OpenStack services.

   .. code-block:: bash

     export OS_CLOUD=standalone
     openstack endpoint list
