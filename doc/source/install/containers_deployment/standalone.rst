Standalone Containers based Deployment
======================================

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

#. Specify required parameters for network configuration::

    # TODO(aschultz): This still assumes a bunch of undercloud stuff that may
    # not be needed anymore. Will need to clean this up.

    export NETWORK=192.168.24
    export IP=$NETWORK.2
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
      DnsServers: ''
      DockerInsecureRegistryAddress:
      - $IP:8787
      MasqueradeNetworks:
        $NETWORK.0/24:
        - $NETWORK.0/24
      NeutronPublicInterface: $INTERFACE
      StandaloneCtlplaneLocalSubnet: ctlplane-subnet
      StandaloneCtlplaneSubnets:
        ctlplane-subnet:
          DhcpRangeEnd: $NETWORK.40
          DhcpRangeStart: $NETWORK.20
          NetworkCidr: $NETWORK.0/24
          NetworkGateway: $IP
      StandaloneEnableRoutedNetworks: false
      StandaloneHomeDir: $HOME
      StandaloneLocalMtu: 1500
    EOF

#. Run deploy command::

    sudo openstack tripleo deploy \
      --templates \
      --local-ip=$IP \
      -e /usr/share/openstack-tripleo-heat-templates/environments/standalone.yaml \
      -r /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml \
      -e $HOME/standalone_parameters.yaml \
      --output-dir $HOME \
      --standalone

#. Validate Keystone services

   You can validate the Keystone is running by fetching a token::

    # validate keystone
    ADMIN_PASS=$(egrep "^[[:space:]]+AdminPassword:" $HOME/tripleo-undercloud-passwords.yaml | awk '{print $2}')

    KEYSTONE_PAYLOAD=$(cat <<EOF
    { "auth": {
        "identity": {
          "methods": ["password"],
          "password": {
            "user": {
              "name": "admin",
              "domain": { "id": "default" },
              "password": "$ADMIN_PASS"
            }
          }
        }
      }
    }
    EOF
    )
    curl -i \
      -H "Content-Type: application/json" \
      -d "$KEYSTONE_PAYLOAD" \
      "http://$IP:5000/v3/auth/tokens" ; echo

