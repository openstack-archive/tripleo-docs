Deploying Manila in the Overcloud
=================================

This guide assumes that your undercloud is already installed and ready to
deploy an overcloud with Manila enabled.

Deploying the Overcloud with the Internal Ceph Backend
------------------------------------------------------
Ceph deployed by TripleO can be used as a Manila share backend. Make sure that
Ceph, Ceph MDS and Manila Ceph environment files are included when deploying the
Overcloud::

    openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/storage-environment.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/services/ceph-mds.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/manila-cephfsnative-config.yaml

Network Isolation
~~~~~~~~~~~~~~~~~
When mounting a ceph share from a user instance, the user instance needs access
to the Ceph public network. When mounting a ceph share from a user instance,
the user instance needs access to the Ceph public network, which in TripleO
maps to the Overcloud storage network.  In an Overcloud which uses isolated
networks the tenant network and storage network are isolated from one another
so user instances cannot reach the Ceph public network unless the cloud
administrator creates a provider network in neutron that maps to the storage
network and exposes access to it.

Before deploying Overcloud make sure that there is a bridge for storage network
interface. If single NIC with VLANs network configuration is used (as in
``/usr/share/openstack-tripleo-heat-templates/network/config/single-nic-vlans/``)
then by default ``br-ex`` bridge is used for storage network and no additional
customization is required for Overcloud deployment. If a dedicated interface is
used for storage network (as in
``/usr/share/openstack-tripleo-heat-templates/network/config/multiple-nics/``)
then update storage interface for each node type (controller, compute, ceph) to
use bridge. The following interface definition::

    - type: interface
        name: nic2
        use_dhcp: false
        addresses:
        - ip_netmask:
            get_param: StorageIpSubnet

should be replaced with::

    - type: ovs_bridge
      name: br-storage
      use_dhcp: false
      addresses:
      - ip_netmask:
          get_param: StorageIpSubnet
      members:
      - type: interface
        name: nic2
        use_dhcp: false
        primary: true

And pass following parameters when deploying Overcloud to allow Neutron to map
provider networks to the storage bridge::

      parameter_defaults:
          NeutronBridgeMappings: datacentre:br-ex,storage:br-storage
          NeutronFlatNetworks: datacentre,storage

When Overcloud is deployed, create a provider network which can be used to
access storage network.

* If single NIC with VLANs is used, then the provider network is mapped
  to the default datacentre network::

      neutron net-create storage --shared --provider:physical_network \
        datacentre --provider:network_type vlan --provider:segmentation_id 30

      neutron subnet-create --name storage-subnet \
        --allocation-pool start=172.16.1.100,end=172.16.1.120 \
        --enable-dhcp storage 172.16.1.0/24

* If a custom bridge was used for storage network interface (``br-storage`` in
  the example above) then provider network is mapped to the network specified
  by ``NeutronBridgeMappings`` parameter (``storage`` network in the example
  above)::

      neutron net-create storage --shared --provider:physical_network storage \
        --provider:network_type flat

      neutron subnet-create --name storage-subnet \
        --allocation-pool start=172.16.1.200,end=172.16.1.220 --enable-dhcp \
        storage 172.16.1.0/24 --no-gateway

.. note::
    Allocation pool should not overlap with storage network
    pool used for storage nodes (``StorageAllocationPools`` parameter).
    You may also need to shrink storage nodes pool size to reserve more IPs
    for tenants using the provider network.

.. note::

    Make sure that subnet CIDR matches storage network CIDR (``StorageNetCidr``
    parameter)and
    segmentation_id matches VLAN ID for the storage network traffic
    (``StorageNetworkVlanID`` parameter).

Then Ceph shares can be accessed from a user instance by adding the provider
network to the instance.

.. note::

    Cloud-init by default configures only first network interface to use DHCP
    which means that user intances will not have network interface for storage
    network autoconfigured. You can configure it manually or use
    `dhcp-all-interfaces <https://docs.openstack.org/developer/diskimage-builder/elements/dhcp-all-interfaces/README.html>`_.

Deploying the Overcloud with an External Backend
------------------------------------------------
.. note::

    The :doc:`template_deploy` doc has a more detailed explanation of the
    following steps.

#. Copy the Manila driver-specific configuration file to your home directory:

     - Generic driver::

          sudo cp /usr/share/openstack-tripleo-heat-templates/environments/manila-generic-config.yaml ~

     - NetApp driver::

         sudo cp /usr/share/openstack-tripleo-heat-templates/environments/manila-netapp-config.yaml ~

#. Edit the permissions (user is typically ``stack``)::

    sudo chown $USER ~/manila-*-config.yaml
    sudo chmod 755 ~/manila-*-config.yaml


#. Edit the parameters in this file to fit your requirements.
    - If you're using the generic driver, ensure that the service image
      details correspond to the service image you intend to load.
    - Ensure that the following line is changed::

       OS::TripleO::ControllerExtraConfigPre: /usr/share/openstack-tripleo-heat-templates/puppet/extraconfig/pre_deploy/controller/manila-[generic or netapp].yaml


#. Continue following the TripleO instructions for deploying an overcloud.
   Before entering the command to deploy the overcloud, add the environment
   file that you just configured as an argument::

    openstack overcloud deploy --templates -e ~/manila-[generic or netapp]-config.yaml

#. Wait for the completion of the overcloud deployment process.


Creating the Share
------------------

.. note::

    The following steps will refer to running commands as an admin user or a
    tenant user. Sourcing the ``overcloudrc`` file will authenticate you as
    the admin user. You can then create a tenant user and use environment
    files to switch between them.

#. Upload a service image:

   .. note::

       This step is only required for the generic driver.

   Download a Manila service image to be used for share servers and upload it
   to Glance so that Manila can use it [tenant]::

       glance image-create --name manila-service-image --disk-format qcow2 --container-format bare --file manila_service_image.qcow2

#. Create a share network to host the shares:

   - Create the overcloud networks. The :doc:`../basic_deployment/basic_deployment_cli`
     doc has a more detailed explanation about creating the network
     and subnet. Note that you may also need to perform the following
     steps to get Manila working::

       neutron router-create router1
       neutron router-interface-add router1 [subnet id]

   - List the networks and subnets [tenant]::

       neutron net-list && neutron subnet-list

   - Create a share network (typically using the private default-net net/subnet)
     [tenant]::

       manila share-network-create --neutron-net-id [net] --neutron-subnet-id [subnet]

#. Create a new share type (yes/no is for specifying if the driver handles
   share servers) [admin]::

    manila type-create [name] [yes/no]

#. Create the share [tenant]::

    manila create --share-network [share net ID] --share-type [type name] [nfs/cifs] [size of share]


Accessing the Share
-------------------

#. To access the share, create a new VM on the same Neutron network that was
   used to create the share network::

    nova boot --image [image ID] --flavor [flavor ID] --nic net-id=[network ID] [name]

#. Allow access to the VM you just created::

    manila access-allow [share ID] ip [IP address of VM]

#. Run ``manila list`` and ensure that the share is available.

#. Log into the VM::

    ssh [user]@[IP]

.. note::

    You may need to configure Neutron security rules to access the
    VM. That is not in the scope of this document, so it will not be covered
    here.

5. In the VM, execute::

    sudo mount [export location] [folder to mount to]

6. Ensure the share is mounted by looking at the bottom of the output of the
   ``mount`` command.

7. That's it - you're ready to start using Manila!

