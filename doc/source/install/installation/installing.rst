Installing the Undercloud
--------------------------

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
      The undercloud is intended to work correctly with SELinux enforcing, and
      cannot be installed to a system with SELinux disabled.  If SELinux
      enforcement must be turned off for some reason, it should instead be set
      to permissive.

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

   .. include:: ../repositories.txt

.. We need to manually continue our list numbering here since the above
  "include" directive breaks the numbering.

3. Install the yum-plugin-priorities package so that the Delorean repository takes precedence over the main RDO repositories::

     sudo yum -y install yum-plugin-priorities

#. Install the TripleO CLI, which will pull in all other necessary packages as dependencies::

    sudo yum install -y python-tripleoclient

   .. admonition:: Ceph
      :class: ceph

      If you intend to deploy Ceph in the overcloud and are running Pike or newer, then install ceph-ansible on the undercloud::

          sudo yum install -y ceph-ansible

#. Copy in the sample configuration file and edit it to reflect your environment::

    cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf

   .. TODO(bnemec): Find a more permanent location for this tool.

   .. note:: There is a tool available that can help with writing a basic
             undercloud.conf:
             `Undercloud Configuration Wizard <http://ucw-bnemec.rhcloud.com/>`_
             It takes some basic information about the intended overcloud
             environment and generates sane values for a number of the important
             options.

#. Run the command to install the undercloud:

   .. admonition:: SSL
      :class: optional

      To deploy an undercloud with SSL, see :doc:`../advanced_deployment/ssl`.

   .. admonition:: Validations
      :class: validations

      :doc:`../validations/validations` will be installed and
      configured during undercloud installation. You can set
      ``enable_validations = false`` in ``undercloud.conf`` to prevent
      that.


   Install the undercloud::

       openstack undercloud install


Once the install has completed, you should take note of the files ``stackrc`` and
``undercloud-passwords.conf``.  You can source ``stackrc`` to interact with the
undercloud via the OpenStack command-line client.  ``undercloud-passwords.conf``
contains the passwords used for each service in the undercloud.  These passwords
will be automatically reused if the undercloud is reinstalled on the same system,
so it is not necessary to copy them to ``undercloud.conf``.

.. note::
    Any passwords set in ``undercloud.conf`` will take precedence over the ones in
    ``undercloud-passwords.conf``.

.. note::
    ``openstack undercloud install`` can be rerun to reapply changes from
    undercloud.conf to the undercloud. Note that this should **not** be done if an
    overcloud has already been deployed or is in progress.

.. note::
   If running ``docker`` commands as a stack user after an undercloud install fail
   with a permission error, log out and log in again. The stack user does get added
   to the docker group during install, but that change gets reflected only after a
   new login.
