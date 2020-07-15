Updating puppet-tripleo
-----------------------

.. include:: ../../links.rst

The puppet manifests that currently define overcloud node configuration are
moved from the tripleo-heat-templates to new puppet-tripleo class definitions
as part of the composable services approach. In next iterations, all service
configuration should be moved also to puppet-tripleo.
This section considers the addition of the ntp definition to puppet-tripleo.

Folder structure convention
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Services should be defined in the services folder, depending on the service
purpose.
::

  manifests
    profile/base      ---> To host all services not using pacemaker.
      time            ---> Specific folder for time services (NTP, timezone, Chrony among others).
        ntp.pp        ---> Puppet manifest to configure the service.

.. note::

  For further information related to the current folders manifests structure
  refer to the `puppet-tripleo repository`_.

Adding the puppet manifest
~~~~~~~~~~~~~~~~~~~~~~~~~~

This step will reference how the puppet logic should be organized in
puppet-tripleo.

Inside the manifests folder, add the service manifest following the folder
structure (``manifests/profile/base/time/ntp.pp``) as:
::

  class tripleo::profile::base::time::ntp (
    #We get the configuration step in which we can choose which steps to execute
    $step          = hiera('step'),
  ) {
    #step assigned for core modules.
    #(Check for further references about the configuration steps)
    #https://opendev.org/openstack/tripleo-heat-templates/src/branch/master/puppet/services/README.rst
    if ($step >= 2){
      #We will call the NTP puppet class and assign our configuration values.
      #If needed additional Puppet packages can be added/installed by using the repo tripleo-puppet-elements
      if count($ntpservers) > 0 {
        include ::ntp
      }
    }
  }

If users have followed all the previous steps, they should be able to configure
their services using the composable services within roles guidelines.
