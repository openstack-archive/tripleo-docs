Basic Deployment (CLI)
======================

With these few steps you will be able to simply deploy via |project| to your
environment using our defaults in a few steps.


Prepare Your Environment
------------------------

#. Make sure you have your environment ready and undercloud running:

   * :doc:`../environments/environments`
   * :doc:`../installation/installing`

#. Log into your undercloud (instack) virtual machine as non-root user::

    ssh root@<undercloud-machine>

    su - stack

#. In order to use CLI commands easily you need to source needed environment
   variables::

    source stackrc


Get Images
----------

.. note::

       If you already have images built, perhaps from a previous installation of
       |project|, you can simply copy those image files into your regular user's
       home directory and skip this section.

       If you do this, be aware that sometimes newer versions of |project| do not
       work with older images, so if the deployment fails it may be necessary to
       delete the older images and restart the process from this step.

       The image files required are::

           deploy-ramdisk-ironic.initramfs
           deploy-ramdisk-ironic.kernel
           ironic-python-agent.initramfs
           ironic-python-agent.kernel
           overcloud-full.initrd
           overcloud-full.qcow2
           overcloud-full.vmlinuz

Images must be built prior to doing a deployment. An IPA ramdisk,
deployment ramdisk, and openstack-full image can all be built using
instack-undercloud.

It's recommended to build images on the installed undercloud directly since all
the dependencies are already present.

The following steps can be used to build images. They should be run as the same
non-root user that was used to install the undercloud.


#. Choose image operating system:

   The built images will automatically have the same base OS as the
   running undercloud. To choose a different OS use one of the following
   commands (make sure you have your OS specific content visible):

   .. admonition:: CentOS
      :class: centos

      ::

          export NODE_DIST=centos7

   .. admonition:: RHEL
      :class: rhel

      ::

          export NODE_DIST=rhel7

#. Install the ``current-tripleo`` delorean repo and deps repo into the overcloud images:

    ::

        export USE_DELOREAN_TRUNK=1
        export DELOREAN_TRUNK_REPO="http://trunk.rdoproject.org/centos7/current-tripleo/"
        export DELOREAN_REPO_FILE="delorean.repo"

   .. admonition:: Stable Branch
      :class: stable

      ::

          export DELOREAN_TRUNK_REPO="http://trunk.rdoproject.org/centos7-liberty/current/"


#. Build the required images:


  .. admonition:: RHEL
     :class: rhel

     Download the RHEL 7.1 cloud image or copy it over from a different location,
     for example:
     https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.1/x86_64/product-downloads,
     and define the needed environment variables for RHEL 7.1 prior to running
     ``openstack overcloud image build --all``::

          export DIB_LOCAL_IMAGE=rhel-guest-image-7.1-20150224.0.x86_64.qcow2

  .. admonition:: RHEL Portal Registration
     :class: portal

     To register the image builds to the Red Hat Portal define the following variables::

            export REG_METHOD=portal
            export REG_USER="[your username]"
            export REG_PASSWORD="[your password]"
            # Find this with `sudo subscription-manager list --available`
            export REG_POOL_ID="[pool id]"
            export REG_REPOS="rhel-7-server-rpms rhel-7-server-extras-rpms rhel-ha-for-rhel-7-server-rpms \
                rhel-7-server-optional-rpms rhel-7-server-openstack-6.0-rpms"

  .. admonition:: RHEL Satellite Registration
     :class: satellite

     To register the image builds to a Satellite define the following
     variables. Only using an activation key is supported when registering to
     Satellite, username/password is not supported for security reasons. The
     activation key must enable the repos shown::

            export REG_METHOD=satellite
            # REG_SAT_URL should be in the format of:
            # http://<satellite-hostname>
            export REG_SAT_URL="[satellite url]"
            export REG_ORG="[satellite org]"
            # Activation key must enable these repos:
            # rhel-7-server-rpms
            # rhel-7-server-optional-rpms
            # rhel-7-server-extras-rpms
            # rhel-7-server-openstack-6.0-rpms
            export REG_ACTIVATION_KEY="[activation key]"

  .. admonition:: Source
     :class: source

     Git checkouts of the puppet modules can be used instead of packages. Export the
     following environment variable::

       export DIB_INSTALLTYPE_puppet_modules=source

     It is also possible to use this functionality to use an in-progress review
     as part of the overcloud image build.  See
     :doc:`../advanced_deployment/in_progress_review` for details.

  ::

   openstack overcloud image build --all

  .. note::
    This command will build **overcloud-full** images (\*.qcow2, \*.initrd,
    \*.vmlinuz) and **ironic-python-agent** images (\*.initramfs, \*.kernel)

    To rebuild only a single image, see :doc:`../advanced_deployment/build_single_image`.

Upload Images
-------------

Load the images into the undercloud Glance::

    openstack overcloud image upload


Register Nodes
--------------

Register nodes for your deployment with Ironic::

    openstack baremetal import --json instackenv.json

.. note::
   It's not recommended to delete nodes and/or rerun this command after
   you have proceeded to the next steps. Particularly, if you start introspection
   and then re-register nodes, you won't be able to retry introspection until
   the previous one times out (1 hour by default). If you are having issues
   with nodes after registration, please follow
   :ref:`node_registration_problems`.

.. note::
   By default Ironic will not sync the power state of the nodes,
   because in our HA (high availability) model Pacemaker is the
   one responsible for controlling the power state of the nodes
   when fencing.  If you are using a non-HA setup and want Ironic
   to take care of the power state of the nodes please change the
   value of the "force_power_state_during_sync" configuration option
   in the /etc/ironic/ironic.conf file to "True" and restart the
   openstack-ironic-conductor service.

   Also, note that if ``openstack undercloud install`` is re-run the value of
   the "force_power_state_during_sync" configuration option will be set back to
   the default, which is "False".

Assign kernel and ramdisk to nodes::

    openstack baremetal configure boot

.. note::
    If your hardware has several hard drives, it's highly recommended that you
    specify the exact device to be used during introspection and deployment
    as a root device. Please see :ref:`root_device` for details.

    If you don't specify the root device explicitly, any device may be picked.
    Also the device chosen automatically is NOT guaranteed to be the same
    across rebuilds. Make sure to wipe the previous installation before
    rebuilding in this case.

.. _introspection:

Introspect Nodes
----------------

Introspect hardware attributes of nodes::

    openstack baremetal introspection bulk start

.. note:: **Introspection has to finish without errors.**
   The process can take up to 5 minutes for VM / 15 minutes for baremetal. If
   the process takes longer, see :ref:`introspection_problems`.

.. note:: If you need to introspect just a single node, see
   :doc:`../advanced_deployment/introspect_single_node`

Flavor Details
--------------

The undercloud will have a number of default flavors created at install time.
In most cases these flavors do not need to be modified, but they can be if
desired.  By default, all overcloud instances will be booted with the
``baremetal`` flavor, so all baremetal nodes must have at least as much
memory, disk, and cpu as that flavor.

In addition, there are profile-specific flavors created which can be used with
the profile-matching feature.  For more details on deploying with profiles,
see :doc:`../advanced_deployment/profile_matching`.

Configure a nameserver for the Overcloud
----------------------------------------

Overcloud nodes can have a nameserver configured in order to resolve
hostnames via DNS. The nameserver is defined in the undercloud's neutron
subnet. If needed, define the nameserver to be used for the environment::

    # List the available subnets
    neutron subnet-list
    neutron subnet-update <subnet-uuid> --dns-nameserver <nameserver-ip>

.. note::
   A public DNS server, such as 8.8.8.8 or the undercloud DNS name server
   can be used if there is no internal DNS server.

.. admonition:: Virtual
   :class: virtual

   In virtual environments, the libvirt default network DHCP server address,
   typically 192.168.122.1, can be used as the overcloud nameserver.

Deploy the Overcloud
--------------------

By default 1 compute and 1 control node will be deployed, with networking
configured for the virtual environment.  To customize this, see the output of::

    openstack help overcloud deploy

.. admonition:: Ceph
  :class: ceph

  When deploying Ceph it is necessary to specify the number of Ceph OSD nodes
  to be deployed and to provide some additional parameters to enable usage
  of Ceph for Glance, Cinder, Nova or all of them. To do so, use the
  following arguments when deploying::

      --ceph-storage-scale <number of nodes> -e /usr/share/openstack-tripleo-heat-templates/environments/storage-environment.yaml

  By default when Ceph is enabled the Cinder LVM back-end is disabled. This
  behavior may be changed passing::

      --cinder-lvm

.. admonition:: RHEL Satellite Registration
  :class: satellite

  To register the Overcloud nodes to a Satellite add the following flags
  to the deploy command::

         --rhel-reg --reg-method satellite --reg-org <ORG ID#> --reg-sat-url <satellite URL> --reg-activation-key <KEY>

  .. note::

      Only using an activation key is supported when registering to
      Satellite, username/password is not supported for security reasons.
      The activation key must enable the following repos:

      rhel-7-server-rpms

      rhel-7-server-optional-rpms

      rhel-7-server-extras-rpms

      rhel-7-server-openstack-6.0-rpms

.. admonition:: SSL
   :class: ssl

   To deploy an overcloud with SSL, see :doc:`../advanced_deployment/ssl`.

Run the deploy command, including any additional parameters as necessary::

  openstack overcloud deploy --templates [additional parameters]

.. admonition:: Virtual
   :class: virtual

   When deploying the Compute node in a virtual machine, add ``--libvirt-type
   qemu`` or launching instances on the deployed overcloud will fail.

.. note::
   To deploy an overcloud with multiple controllers and achieve HA
   you must provision at least 3 controller nodes, specify an NTP
   server and enable Pacemaker as cluster resource manager. To do
   so add the following arguments to the deploy command::

    --control-scale 3 -e /usr/share/openstack-tripleo-heat-templates/environments/puppet-pacemaker.yaml --ntp-server pool.ntp.org

.. note::

   To deploy the overcloud with network isolation, bonds, and/or custom
   network interface configurations, instead follow the workflow here to
   deploy: :doc:`../advanced_deployment/network_isolation`

.. note::

   Previous versions of the client had many parameters defaulted. Some of these
   parameters are now pulling defaults directly from the Heat templates. In
   order to override these parameters, one should use an environment file to
   specify these overrides, via 'parameter_defaults'.

   The parameters that controlled these parameters will be deprecated in the
   client, and eventually removed in favor of using environment files.


Post-Deployment
---------------


Access the Overcloud
^^^^^^^^^^^^^^^^^^^^

``openstack overcloud deploy`` generates an overcloudrc file appropriate for
interacting with the deployed overcloud in the current user's home directory.
To use it, simply source the file::

    source ~/overcloudrc

To return to working with the undercloud, source the stackrc file again::

    source ~/stackrc


Setup the Overcloud network
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Initial networks in Neutron in the Overlcoud need to be created for tenant
instances. The following are example commands to create the initial networks.
Edit the address ranges, or use the necessary neutron commands to match the
environment appropriately. This assumes a dedicated interface or native VLAN::


    neutron net-create nova --router:external --provider:network_type flat \
      --provider:physical_network datacentre
    neutron subnet-create --name nova --disable-dhcp \
      --allocation-pool start=172.16.23.140,end=172.16.23.240 \
      --gateway 172.16.23.251 nova 172.16.23.128/25

The example shows naming the network "nova" because that will make tempest
tests to pass, based on the default floating pool name set in nova.conf. You
can confirm that the network was created with::

    neutron net-list

Sample output of the command::

    +--------------------------------------+-------------+-------------------------------------------------------+
    | id                                   | name        | subnets                                               |
    +--------------------------------------+-------------+-------------------------------------------------------+
    | d474fe1f-222d-4e32-802b-cde86e746a2a | nova        | 01c5f621-1e0f-4b9d-9c30-7dc59592a52f 172.16.23.128/25 |
    +--------------------------------------+-------------+-------------------------------------------------------+

To use a VLAN, the following example should work. Customize the address ranges
and VLAN id based on the environment::

    neutron net-create nova --router:external --provider:network_type vlan \
      --provider:physical_network datacentre --provider:segmentation_id 195
    neutron subnet-create --name nova --disable-dhcp \
      --allocation-pool start=172.16.23.140,end=172.16.23.240 \
      --gateway 172.16.23.251 nova 172.16.23.128/25


Validate the Overcloud
^^^^^^^^^^^^^^^^^^^^^^
Source the ``overcloudrc`` file::

    source ~/overcloudrc

Create a directory for Tempest (eg. naming it ``tempest``)::

    mkdir ~/tempest
    cd ~/tempest

Tempest expects the tests it discovers to be in the current working directory.
Set it up accordingly::

    /usr/share/openstack-tempest-liberty/tools/configure-tempest-directory

The ``~/tempest-deployer-input.conf`` file was created during deployment and
contains deployment specific settings. Use that file to configure
Tempest::

    tools/config_tempest.py --deployer-input ~/tempest-deployer-input.conf \
                            --debug --create \
                            identity.uri $OS_AUTH_URL \
                            identity.admin_password $OS_PASSWORD

Run Tempest::

    tools/run-tests.sh

.. note:: The full Tempest test suite might take hours to run on a single CPU.


Redeploy the Overcloud
^^^^^^^^^^^^^^^^^^^^^^

The overcloud can be redeployed when desired.

#. First, delete any existing Overcloud::

    heat stack-delete overcloud

#. Confirm the Overcloud has deleted. It may take a few minutes to delete::

    # This command should show no stack once the Delete has completed
    heat stack-list

#. Although not required, introspection can be rerun::

    openstack baremetal introspection bulk start

#. Deploy the Overcloud again::

    openstack overcloud deploy --templates
