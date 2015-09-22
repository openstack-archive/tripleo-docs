Installing the Undercloud
==========================

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

   .. admonition:: Baremetal
      :class: baremetal

      Ensure that there is a FQDN hostname set and that the $HOSTNAME environment
      variable matches that value.

      Use ``hostnamectl`` to set a hostname if needed::

          sudo hostnamectl set-hostname myhost.mydomain
          sudo hostnamectl set-hostname --transient myhost.mydomain

      An entry for the system's FQDN hostname is also needed in /etc/hosts. For
      example, if the system is named *myhost.mydomain*, /etc/hosts should have
      an entry like::

         127.0.0.1   myhost.mydomain


#. Enable needed repositories:

  .. admonition:: RHEL
     :class: rhel

     Enable optional repo::

         sudo yum install -y yum-utils
         sudo yum-config-manager --enable rhelosp-rhel-7-server-opt

.. include:: ../repositories.txt


.. We need to manually continue our list numbering here since the above
  "include" directive breaks the numbering.
3. Install the TripleO CLI, which will pull in all other necessary packages as dependencies::

    sudo yum install -y python-rdomanager-oscplugin


#. Run the script to install the undercloud:

  .. admonition:: Baremetal
     :class: baremetal

     Copy in the sample configuration file and edit it to reflect your environment::

        cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf

  .. admonition:: Source
     :class: source

     Git checkouts of the puppet modules can be used instead of packages. Export the
     following environment variable::

       export DIB_INSTALLTYPE_puppet_modules=source

  .. admonition:: Source
     :class: source

     To use a git checkout for only a specific module, export the following variable::

       export DIB_INSTALLTYPE_puppet_tripleo=source

     Replace ``puppet_tripleo`` with the name of the puppet module to be installed
     from source, replacing any -'s with _'s.

     To use a pending review for a module, set its installtype to source as
     described above, then also export the following variables::

       export DIB_REPOLOCATION_puppet_tripleo=https://review.openstack.org/openstack/puppet-tripleo
       export DIB_REPOREF_puppet_tripleo=refs/changes/30/223330/1

     This time replace the name of the module in the variable name and the review URL.
     The correct value for the ``reporef`` can be found in the ``Download`` section
     of the Gerrit UI.  Look for a string that matches the format of the example above.

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
