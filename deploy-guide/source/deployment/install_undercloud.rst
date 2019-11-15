Undercloud Installation
=======================

This section contains instructions on how to install the undercloud. For update
or upgrade to a deployed undercloud see undercloud_upgrade_.

.. _undercloud_upgrade: ../post_deployment/upgrade/undercloud.html


.. _install_undercloud:

Installing the Undercloud
--------------------------

.. note::
   Instack-undercloud is deprecated in Rocky cycle. Containerized undercloud
   should be installed instead. See :doc:`undercloud`
   for backward compatibility related information.

.. note::
   Please ensure all your nodes (undercloud, compute, controllers, etc) have
   their internal clock set to UTC in order to prevent any issue with possible
   file future-dated timestamp if hwclock is synced before any timezone offset
   is applied.


#. Log in to your machine (baremetal or VM) where you want to install the
   undercloud as a non-root user (such as the stack user)::

       ssh <non-root-user>@<undercloud-machine>

   .. note::
      If you don't have a non-root user created yet, log in as root and create
      one with following commands::

          sudo useradd stack
          sudo passwd stack  # specify a password

          echo "stack ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/stack
          sudo chmod 0440 /etc/sudoers.d/stack

          su - stack

   .. note::
      The undercloud is intended to work correctly with SELinux enforcing.
      Installatoins with the permissive/disabled SELinux are not recommended.
      The ``undercloud_enable_selinux`` config option controls that setting.

   .. note::
      vlan tagged interfaces must follow the if_name.vlan_id convention, like for
      example: eth0.vlan100 or bond0.vlan120.

   .. admonition:: Baremetal
      :class: baremetal

      Ensure that there is a FQDN hostname set and that the $HOSTNAME environment
      variable matches that value.  The easiest way to do this is to set the
      ``undercloud_hostname`` option in undercloud.conf before running the
      install.  This will allow the installer to configure all of the hostname-
      related settings appropriately.

      Alternatively the hostname settings can be configured manually, but
      this is strongly discouraged.  The manual steps are as follows::

          sudo hostnamectl set-hostname myhost.mydomain
          sudo hostnamectl set-hostname --transient myhost.mydomain

      An entry for the system's FQDN hostname is also needed in /etc/hosts. For
      example, if the system is named *myhost.mydomain*, /etc/hosts should have
      an entry like::

         127.0.0.1   myhost.mydomain myhost


#. Enable needed repositories:

   .. admonition:: RHEL
      :class: rhel

      Enable optional repo::

          sudo yum install -y yum-utils
          sudo yum-config-manager --enable rhelosp-rhel-7-server-opt

   .. include:: ../repositories.rst


#. Install the TripleO CLI, which will pull in all other necessary packages as dependencies::

    sudo yum install -y python-tripleoclient

   .. admonition:: Ceph
      :class: ceph

      If you intend to deploy Ceph in the overcloud, or configure the overcloud to use an external Ceph cluster, and are running Pike or newer, then install ceph-ansible on the undercloud::

          sudo yum install -y ceph-ansible

#. Prepare the configuration file::

    cp /usr/share/python-tripleoclient/undercloud.conf.sample ~/undercloud.conf

   It is backwards compatible with non-containerized instack underclouds.

   .. admonition:: Stable Branch
      :class: stable

      For a non-containerized undercloud, copy in the sample configuration
      file and edit it to reflect your environment::

       cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf

      .. note:: There is a tool available that can help with writing a basic
          ``undercloud.conf``:
          `Undercloud Configuration Wizard <http://ucw.tripleo.org/>`_
          It takes some basic information about the intended overcloud
          environment and generates sane values for a number of the important
          options.

#. (OPTIONAL) Generate configuration for preparing container images

   As part of the undercloud install, an image registry is configured on port
   `8787`.  This is used to increase reliability of overcloud image pulls, and
   minimise overall network transfers.  The undercloud registry will be
   populated with images required by the undercloud by generating the following
   `containers-prepare-parameter.yaml` file and including it in
   ``undercloud.conf:
   container_images_file=$HOME/containers-prepare-parameter.yaml``::

      openstack tripleo container image prepare default \
        --local-push-destination \
        --output-env-file ~/containers-prepare-parameter.yaml

   .. note::
      This command is available since Rocky.

   See :ref:`prepare-environment-containers` for details on using
   `containers-prepare-parameter.yaml` to control what can be done
   during the container images prepare phase of an undercloud install.

   Additionally, ``docker_insecure_registries`` and ``docker_registry_mirror``
   parameters allow to customize container registries via the
   ``undercloud.conf`` file.

#. (OPTIONAL) Override heat parameters and environment files used for undercloud
   deployment.

   Similarly to overcloud deployments, see :ref:`override-heat-templates` and
   :ref:`custom-template-location`, the ``undercloud.conf: custom_env_files``
   and ``undercloud.conf: templates`` configuration parameters allow to
   use a custom heat templates location and override or specify additional
   information for Heat resources used for undercloud deployment.

   Additionally, the ``undercloud.conf: roles_file`` parameter brings in the
   ultimate flexibility of :ref:`custom_roles` and :ref:`composable_services`.
   This allows you to deploy an undercloud composed of highly customized
   containerized services, with the same workflow that TripleO uses for
   overcloud deployments.

   .. note:: The CLI and configuration interface used to deploy a containerized
       undercloud is the same as that used by 'legacy' non-containerized
       underclouds. As noted above however mechanism by which the undercloud is
       actually deployed is completely changed and what is more, for the first
       time aligns with the overcloud deployment. See the command
       ``openstack tripleo deploy --standalone`` help for details.
       That interface extention for standalone clouds is experimental for Rocky.
       It is normally should not be used directly for undercloud installations.

#. Run the command to install the undercloud:

   .. admonition:: SSL
      :class: optional

      To deploy an undercloud with SSL, see :doc:`../features/ssl`.

   .. admonition:: Validations
      :class: validations

      :doc:`../post_deployment/validations/index` will be installed and
      configured during undercloud installation. You can set
      ``enable_validations = false`` in ``undercloud.conf`` to prevent
      that.

   To deploy an undercloud::

       openstack undercloud install

.. note::
    The undercloud is containerized by default as of Rocky.

.. note::
    It's possible to enable verbose logging with ``--verbose`` option.

.. note::
    To install a deprecated instack undercloud, you'll need to deploy
    with ``--use-heat=False`` option. It only works in Rocky
    as instack-undercloud was retired in Stein.


In Rocky, we will run all the OpenStack services in a moby container runtime
unless the default settings are overwritten.
This command requires 2 services to be running at all times. The first one is a
basic keystone service, which is currently executed by `tripleoclient` itself, the
second one is `heat-all` which executes the templates and installs the services.
The latter can be run on baremetal or in a container (tripleoclient will run it
in a container by default).

Once the install has completed, you should take note of the files ``stackrc`` and
``undercloud-passwords.conf``.  You can source ``stackrc`` to interact with the
undercloud via the OpenStack command-line client.  The ``undercloud-passwords.conf``
file contains the passwords used for each service in the undercloud.  These passwords
will be automatically reused if the undercloud is reinstalled on the same system,
so it is not necessary to copy them to ``undercloud.conf``.

.. note:: Heat installer configuration, logs and state is ephemeral for
    undercloud deployments. Generated artifacts for consequent deployments get
    overwritten or removed (when ``undercloud.conf: cleanup = true``).
    Although, you can still find them stored in compressed files.

Miscellaneous undercloud deployment artifacts, like processed heat templates and
compressed files, can be found in ``undercloud.conf: output_dir`` locations
like ``~/tripleo-heat-installer-templates``.

There is also a compressed file created and placed into the output dir, named as
``undercloud-install-<TS>.tar.bzip2``, where TS represents a timestamp.

Downloaded ansible playbooks and inventory files (see :ref:`config_download`)
used for undercloud deployment are stored in the tempdir
``~/undercloud-ansible-<XXXX>`` by default.

.. note::
    Any passwords set in ``undercloud.conf`` will take precedence over the ones in
    ``undercloud-passwords.conf``.

.. note::
    The undercloud installation command can be rerun to reapply changes from
    ``undercloud.conf`` to the undercloud. Note that this should be done with
    caution if an overcloud has already been deployed or is in progress as some
    configuration changes could affect the overcloud. These changes include but
    are not limited to:

    #. Package repository changes on the undercloud, followed by running the
       installation command could update the undercloud such that further
       management operations are not possible on the overcloud until the
       overcloud update or upgrade procedure is followed.
    #. Reconfiguration of the undercloud container registry if the
       overcloud is using the undercloud as the source for container images.
    #. Networking configuration changes on the undercloud which may affect
       the overcloud's ability to connect to the undercloud for
       instance metadata services.


.. note::
    If running ``docker`` commands as a stack user after an undercloud install fail
    with a permission error, log out and log in again. The stack user does get added
    to the docker group during install, but that change gets reflected only after a
    new login.
