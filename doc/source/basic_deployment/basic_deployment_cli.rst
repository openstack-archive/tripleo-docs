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

       Alternatively, images are available via RDO at
       http://buildlogs.centos.org/centos/7/cloud/x86_64/tripleo_images/

       The image files required are::

           ironic-python-agent.initramfs
           ironic-python-agent.kernel
           overcloud-full.initrd
           overcloud-full.qcow2
           overcloud-full.vmlinuz

Images must be built prior to doing a deployment. An IPA ramdisk and
openstack-full image can all be built using tripleo-common.

It's recommended to build images on the installed undercloud directly since all
the dependencies are already present, but this is not a requirement.

The following steps can be used to build images. They should be run as the same
non-root user that was used to install the undercloud. If the images are not
created on the undercloud, one should use a non-root user.


#. Choose image operating system:

   .. admonition:: CentOS
      :class: centos

      The image build with no arguments will build CentOS 7. It will include the
      common YAML of
      ``/usr/share/openstack-tripleo-common/image-yaml/overcloud-images.yaml``
      and the CentOS YAML at
      ``/usr/share/openstack-tripleo-common/image-yaml/overcloud-images-centos7.yaml``.

   .. admonition:: RHEL
      :class: rhel

      The common YAML is
      ``/usr/share/openstack-tripleo-common/image-yaml/overcloud-images.yaml``.
      It must be specified along with the following.

      The default YAML for RHEL is
      ``/usr/share/openstack-tripleo-common/image-yaml/overcloud-images-rhel7.yaml``

      ::

          export OS_YAML="/usr/share/openstack-tripleo-common/image-yaml/overcloud-images/rhel7.yaml"


#. Install the ``current-tripleo`` delorean repository and deps repository:

.. include:: ../repositories.txt


3. Export environment variables

    ::

        export DIB_YUM_REPO_CONF="/etc/yum.repos.d/delorean*"

   .. admonition:: Ceph
      :class: ceph

      ::

         export DIB_YUM_REPO_CONF="$DIB_YUM_REPO_CONF /etc/yum.repos.d/CentOS-Ceph-Jewel.repo"

   .. admonition:: Stable Branch
      :class: stable

      .. admonition:: Mitaka
         :class: mitaka

         ::

            STABLE_RELEASE="mitaka"

         .. admonition:: Ceph
            :class: ceph

            ::

               export DIB_YUM_REPO_CONF="$DIB_YUM_REPO_CONF /etc/yum.repos.d/CentOS-Ceph-Hammer.repo"

      .. admonition:: Newton
         :class: newton

         ::

            STABLE_RELEASE="newton"

         .. admonition:: Ceph
            :class: ceph

            ::

               export DIB_YUM_REPO_CONF="$DIB_YUM_REPO_CONF /etc/yum.repos.d/CentOS-Ceph-Jewel.repo"


#. Build the required images:


  .. admonition:: RHEL
     :class: rhel

     Download the RHEL 7.1 cloud image or copy it over from a different location,
     for example:
     ``https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.1/x86_64/product-downloads``,
     and define the needed environment variables for RHEL 7.1 prior to running
     ``tripleo-build-images``::

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

     .. admonition:: Ceph
        :class: ceph

        If using Ceph, additional channels need to be added to `REG_REPOS`.
        Enable the appropriate channels for the desired release, as indicated below.
        Do not enable any other channels not explicitly marked for that release.

        ::

           rhel-7-server-rhceph-2-mon-rpms
           rhel-7-server-rhceph-2-osd-rpms
           rhel-7-server-rhceph-2-tools-rpms

        .. admonition:: Mitaka
           :class: mitaka

           ::

              rhel-7-server-rhceph-1.3-mon-rpms
              rhel-7-server-rhceph-1.3-osd-rpms
              rhel-7-server-rhceph-1.3-tools-rpms

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
            # rhel-7-server-rhceph-{2,1.3}-mon-rpms
            # rhel-7-server-rhceph-{2,1.3}-osd-rpms
            # rhel-7-server-rhceph-{2,1.3}-tools-rpms
            export REG_ACTIVATION_KEY="[activation key]"

  .. admonition:: Source
     :class: source

     Git checkouts of the puppet modules can be used instead of packages. Export the
     following environment variable::

       export DIB_INSTALLTYPE_puppet_modules=source

     It is also possible to use this functionality to use an in-progress review
     as part of the overcloud image build.  See
     :doc:`../developer/in_progress_review` for details.

  ::

    openstack overcloud image build

  .. admonition:: RHEL
     :class: rhel

     ::

        openstack overcloud image build --config-file /usr/share/openstack-tripleo-common/image-yaml/overcloud-images.yaml --config-file $OS_YAML


  See the help for ``openstack overcloud image build`` for further options.

  The YAML files are cumulative. Order on the command line is important. The
  packages, elements, and options sections will append. All others will overwrite
  previously read values.

  .. note::
    This command will build **overcloud-full** images (\*.qcow2, \*.initrd,
    \*.vmlinuz) and **ironic-python-agent** images (\*.initramfs, \*.kernel)

    In order to build specific images, one can use the ``--image-name`` flag
    to ``openstack overcloud image build``. It can be specified multiple times.

.. note::

       If you want to use whole disk images with TripleO, please see :doc:`../advanced_deployment/whole_disk_images`.

Upload Images
-------------

Load the images into the undercloud Glance::

    openstack overcloud image upload

To upload a single image, see :doc:`../advanced_deployment/upload_single_image`.

Register Nodes
--------------

Register and configure nodes for your deployment with Ironic::

    openstack overcloud node import instackenv.json

The file to be imported may be either JSON, YAML or CSV format, and
the type is detected via the file extension (json, yaml, csv).
The file format is documented in :ref:`instackenv`.

The nodes status will be set to ``manageable`` by default, so that
introspection may later be run. To also run introspection and make the
nodes available for deployment in one step, the following flags can be
used::

    openstack overcloud node import --introspect --provide instackenv.json

.. admonition:: Stable Branch
   :class: stable

   .. admonition:: Mitaka
      :class: mitaka

      For TripleO release Mitaka, the import command is::

          openstack baremetal import instackenv.json

      The following command must be run after registration to assign the
      deployment kernel and ramdisk to all nodes::

          openstack baremetal configure boot

Starting with the Newton release you can take advantage of the ``enroll``
provisioning state - see :doc:`../advanced_deployment/node_states` for details.

If your hardware has several hard drives, it's highly recommended that you
specify the exact device to be used during introspection and deployment
as a root device. Please see :ref:`root_device` for details.

.. warning::
   If you don't specify the root device explicitly, any device may be picked.
   Also the device chosen automatically is **NOT** guaranteed to be the same
   across rebuilds. Make sure to wipe the previous installation before
   rebuilding in this case.

.. warning::
   It's not recommended to delete nodes and/or rerun this command after
   you have proceeded to the next steps. Particularly, if you start introspection
   and then re-register nodes, you won't be able to retry introspection until
   the previous one times out (1 hour by default). If you are having issues
   with nodes after registration, please follow
   :ref:`node_registration_problems`.

.. _introspection:

Introspect Nodes
----------------


.. admonition:: Validations

   Once the undercloud is installed, you can run the
   ``pre-introspection`` validations::

     openstack workflow execution create tripleo.validations.v1.run_groups '{"group_names": ["pre-introspection"]}'

   Then verify the results as described in :ref:`running_validation_group`.

Nodes must be in the ``manageable`` provisioning state in order to run
introspection. Introspect hardware attributes of nodes with::

    openstack overcloud node introspect --all-manageable

Nodes can also be specified individually by UUID. The ``--provide``
flag can be used in order to move the nodes automatically to the
``available`` provisioning state once the introspection is finished,
making the nodes available for deployment.
::

   openstack overcloud node introspect --all-manageable --provide

.. admonition:: Stable Branch
   :class: stable

   .. admonition:: Mitaka
      :class: mitaka

      For TripleO release Mitaka, the introspection command is::

          openstack baremetal introspection bulk start

.. note:: **Introspection has to finish without errors.**
   The process can take up to 5 minutes for VM / 15 minutes for baremetal. If
   the process takes longer, see :ref:`introspection_problems`.

.. note:: If you need to introspect just a single node, see
   :doc:`../advanced_deployment/introspect_single_node`

Provide Nodes
-------------

Only nodes in the ``available`` provisioning state can be deployed to
(see :doc:`../advanced_deployment/node_states` for details).  To move
nodes from ``manageable`` to ``available`` the following command can be
used::

        openstack overcloud node provide --all-manageable

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
    openstack subnet list
    openstack subnet set <subnet-uuid> --dns-nameserver <nameserver-ip>

.. admonition:: Stable Branch
   :class: stable

    For Mitaka release and older, the subnet commands are executed within the
    `neutron` command::

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

.. admonition:: Validations

   Before you start the deployment, you may want to run the
   ``pre-deployment`` validations::

     openstack workflow execution create tripleo.validations.v1.run_groups '{"group_names": ["pre-deployment"]}'

   Then verify the results as described in :ref:`running_validation_group`.


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

To deploy an overcloud with multiple controllers and achieve HA,
follow :doc:`../advanced_deployment/high_availability`.

.. admonition:: Virtual
   :class: virtual

   When deploying the Compute node in a virtual machine
   without nested guest support, add  ``--libvirt-type qemu``
   or launching instances on the deployed overcloud will fail.

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

.. admonition:: Validations

  After the deployment finished, you can run the ``post-deployment``
  validations::

    openstack workflow execution create tripleo.validations.v1.run_groups '{"group_names": ["post-deployment"]}'

  Then verify the results as described in :ref:`running_validation_group`.


Access the Overcloud
^^^^^^^^^^^^^^^^^^^^

``openstack overcloud deploy`` generates an overcloudrc file appropriate for
interacting with the deployed overcloud in the current user's home directory.
To use it, simply source the file::

    source ~/overcloudrc

To return to working with the undercloud, source the ``stackrc`` file again::

    source ~/stackrc


Setup the Overcloud network
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Initial networks in Neutron in the overcloud need to be created for tenant
instances. The following are example commands to create the initial networks.
Edit the address ranges, or use the necessary ``neutron`` commands to match the
environment appropriately. This assumes a dedicated interface or native VLAN::

    openstack network create public --external --provider-network-type flat \
    --provider-physical-network datacentre
    openstack subnet create --allocation-pool start=172.16.23.140,end=172.16.23.240 \
    --network public --gateway 172.16.23.251 --no-dhcp --subnet-range \
    172.16.23.128/25 public

The example shows naming the network "public" because that will allow tempest
tests to pass, based on the default floating pool name set in ``nova.conf``. 
You can confirm that the network was created with::

    openstack network list

Sample output of the command::

    +--------------------------------------+----------+--------------------------------------+
    | ID                                   | Name     | Subnets                              |
    +--------------------------------------+----------+--------------------------------------+
    | 4db8dd5d-fab5-4ea9-83e5-bdedbf3e9ee6 | public   | 7a315c5e-f8e2-495b-95e2-48af9442af01 |
    +--------------------------------------+----------+--------------------------------------+

To use a VLAN, the following example should work. Customize the address ranges
and VLAN id based on the environment::

    openstack network create public --external --provider-network-type vlan \
    --provider-physical-network datacentre --provider-segment 195 \
    openstack subnet create --allocation-pool start=172.16.23.140,end=172.16.23.240 \
    --network public --no-dhcp --gateway 172.16.23.251 \
    --subnet-range 172.16.23.128/25 public


Validate the Overcloud
^^^^^^^^^^^^^^^^^^^^^^
Source the ``overcloudrc`` file::

    source ~/overcloudrc

Create a directory for Tempest (eg. naming it ``tempest``)::

    mkdir ~/tempest
    cd ~/tempest

Tempest expects the tests it discovers to be in the current working directory.
Set it up accordingly::

    /usr/share/openstack-tempest-*/tools/configure-tempest-directory

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

#. It is recommended that you delete existing partitions from all nodes before
   redeploying. Starting with TripleO Ocata, you can use existing workflows -
   see :ref:`cleaning` for details.

#. Deploy the Overcloud again::

    openstack overcloud deploy --templates
