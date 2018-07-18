.. _basic-deployment-cli:

Basic Deployment (CLI)
======================

These steps document a basic deployment with |project| in an environment using
the project defaults.

.. note::

      Beginning in the Rocky release, Ansible is used to deploy the software
      configuration of the overcloud nodes using a feature called
      **config-download**. While there are no necessary changes to the default
      deployment commands, there are several differences to the deployer
      experience.

      It's recommended to review these differences as documented at
      :doc:`../advanced_deployment/ansible_config_download_differences`

      **config-download** is fully documented at
      :doc:`../advanced_deployment/ansible_config_download`


Prepare Your Environment
------------------------

#. Make sure you have your environment ready and undercloud running:

   * :doc:`../environments/environments`
   * :doc:`../installation/installing`

#. Log into your undercloud virtual machine and become the non-root user (stack
   by default)::

    ssh root@<undercloud-machine>

    su - stack

#. In order to use CLI commands easily you need to source needed environment
   variables::

    source stackrc

.. _basic-deployment-cli-get-images:

Get Images
----------

.. note::

       If you already have images built, perhaps from a previous installation of
       |project|, you can simply copy those image files into your non-root user's
       home directory and skip this section.

       If you do this, be aware that sometimes newer versions of |project| do not
       work with older images, so if the deployment fails it may be necessary to
       delete the older images and restart the process from this step.

       Alternatively, images are available via RDO at
       https://images.rdoproject.org/master which offers images from both the
       CentOS Build System (cbs) and RDO Trunk (called rdo_trunk or delorean).
       However this mirror is slow so if you experience slow download speeds
       you should skip to building the images instead.

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

          export OS_YAML="/usr/share/openstack-tripleo-common/image-yaml/overcloud-images-rhel7.yaml"


#. Install the ``current-tripleo`` delorean repository and deps repository:

   .. include:: ../repositories.txt


3. Export environment variables

   ::

        export DIB_YUM_REPO_CONF="/etc/yum.repos.d/delorean*"

   .. admonition:: Ceph
      :class: ceph

      ::

         export DIB_YUM_REPO_CONF="$DIB_YUM_REPO_CONF /etc/yum.repos.d/tripleo-centos-ceph-jewel.repo"

   .. admonition:: Stable Branch
      :class: stable

      .. admonition:: Newton
         :class: newton

         ::

            export STABLE_RELEASE="newton"

         .. admonition:: Ceph
            :class: ceph

            ::

               export DIB_YUM_REPO_CONF="$DIB_YUM_REPO_CONF /etc/yum.repos.d/tripleo-centos-ceph-jewel.repo"

      .. admonition:: Ocata
         :class: ocata

         ::

            export STABLE_RELEASE="ocata"

         .. admonition:: Ceph
            :class: ceph

            ::

               export DIB_YUM_REPO_CONF="$DIB_YUM_REPO_CONF /etc/yum.repos.d/tripleo-centos-ceph-jewel.repo"


#. Build the required images:


   .. admonition:: RHEL
      :class: rhel

      Download the RHEL 7.4 cloud image or copy it over from a different location,
      for example:
      ``https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.4/x86_64/product-software``,
      and define the needed environment variables for RHEL 7.4 prior to running
      ``tripleo-build-images``::

          export DIB_LOCAL_IMAGE=rhel-server-7.4-x86_64-kvm.qcow2

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

   ::

       openstack overcloud image build

   ..

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

.. _basic-deployment-cli-upload-images:

Upload Images
-------------

Load the images into the containerized undercloud Glance::

    openstack overcloud image upload


To upload a single image, see :doc:`../post_deployment/upload_single_image`.

If working with multiple atchitectures and/or plaforms with an architecure these
attributes can be specified at upload time as in::

    openstack overcloud image upload
    openstack overcloud image upload --arch x86_64 \
        --httpboot /var/lib/ironic/httpboot/x86_64
    openstack overcloud image upload --arch x86_64 --platform SNB \
        --httpboot /var/lib/ironic/httpboot/x86_64-SNB

.. note::

    Adding --httpboot is optional but suggested if you need to ensure that the
    ``agent`` images are unique within your environment.

.. admonition:: Prior to Rocky release
  :class: stable

  Before Rocky, the undercloud isn't containerized by default. Hence
  you should use the ``/httpboot/*`` paths instead.

This will create 3 sets of images with in the undercloud image service for later
use in deployment, see :doc:`../environments/baremetal`

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

If there is information from previous deployments on the nodes' disks, it is
recommended to at least remove the partitions and partition table(s). See
:doc:`../advanced_deployment/cleaning` for information on how to do it.

Finally, if you want your nodes to boot in the UEFI mode, additional steps may
have to be taken - see :doc:`../advanced_deployment/uefi_boot` for details.

.. warning::
   It's not recommended to delete nodes and/or rerun this command after
   you have proceeded to the next steps. Particularly, if you start introspection
   and then re-register nodes, you won't be able to retry introspection until
   the previous one times out (1 hour by default). If you are having issues
   with nodes after registration, please follow
   :ref:`node_registration_problems`.

Another approach to enrolling node is
:doc:`../advanced_deployment/node_discovery`.

.. _introspection:

Introspect Nodes
----------------


.. admonition:: Validations
   :class: validations

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

.. _basic-deployment-cli-configure-namserver:

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

.. _deploy-the-overcloud:

Deploy the Overcloud
--------------------

.. admonition:: Validations
   :class: validations

   Before you start the deployment, you may want to run the
   ``pre-deployment`` validations::

     openstack workflow execution create tripleo.validations.v1.run_groups '{"group_names": ["pre-deployment"]}'

   Then verify the results as described in :ref:`running_validation_group`.


By default 1 compute and 1 control node will be deployed, with networking
configured for the virtual environment.  To customize this, see the output of::

    openstack help overcloud deploy

.. admonition:: Swap
   :class: optional

   Swap files or partitions can be installed as part of an Overcloud deployment.
   For adding swap files there is no restriction besides having
   4GB available on / (by default). When using a swap partition,
   the partition must exist and be tagged as `swap1` (by default).
   To deploy a swap file or partition in each Overcloud node use one
   of the following arguments when deploying::

      -e /usr/share/openstack-tripleo-heat-templates/environments/enable-swap-partition.yaml
      -e /usr/share/openstack-tripleo-heat-templates/environments/enable-swap.yaml

.. admonition:: Ceph
  :class: ceph

  When deploying Ceph with dedicated CephStorage nodes to host the CephOSD
  service it is necessary to specify the number of CephStorage nodes
  to be deployed and to provide some additional parameters to enable usage
  of Ceph for Glance, Cinder, Nova or all of them. To do so, use the
  following arguments when deploying::

      --ceph-storage-scale <number of nodes> -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml

  When deploying Ceph without dedicated CephStorage nodes, opting for an HCI
  architecture instead, where the CephOSD service is colocated with the
  NovaCompute service on the Compute nodes, use the following arguments::

      -e /usr/share/openstack-tripleo-heat-templates/environments/hyperconverged-ceph.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml

  The `hyperconverged-ceph.yaml` environment file will also enable a port on the
  `StorageMgmt` network for the Compute nodes. This will be the Ceph private
  network and the Compute NIC templates have to be configured to use that, see
  :doc:`../advanced_deployment/network_isolation` for more details on how to do
  it.

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
   :class: optional

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
   :class: validations

   After the deployment finishes, you can run the ``post-deployment``
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


Add entries to /etc/hosts
^^^^^^^^^^^^^^^^^^^^^^^^^

In cases where the overcloud hostnames are not already resolvable with DNS,
entries can be added to /etc/hosts to make them resolvable. This is
particularly convenient on the undercloud. The Heat stack provides an output
value that can be appended to /etc/hosts easily. Run the following command to
get the output value and add it to /etc/hosts wherever the hostnames should
be resolvable::

    openstack stack output show overcloud HostsEntry -f value -c output_value


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

Check the `Tempest`_ documentation on how to run tempest.

.. _tempest: ../basic_deployment/tempest.html

Redeploy the Overcloud
^^^^^^^^^^^^^^^^^^^^^^

The overcloud can be redeployed when desired.

#. First, delete any existing Overcloud::

    openstack overcloud delete overcloud

#. Confirm the Overcloud has deleted. It may take a few minutes to delete::

    # This command should show no stack once the Delete has completed
    openstack stack list

#. It is recommended that you delete existing partitions from all nodes before
   redeploying, see :doc:`../advanced_deployment/cleaning` for details.

#. Deploy the Overcloud again::

    openstack overcloud deploy --templates
