Configuring Ceph with Custom Config Settings
============================================

This guide assumes that your undercloud is already installed and ready to
deploy an overcloud.

Customizing ceph.conf
---------------------

Ceph demands for more careful configuration when deployed at scale.

It is possible to override any of the configuration parameters supported by
`puppet-ceph`_ at deployment time via Heat environment files. For example::

  parameter_defaults:
    ExtraConfig:
      ceph::profile::params::osd_journal_size: 2048

will customize the default `osd_journal_size` overriding any default
provided in the `ceph.yaml static hieradata`_.

Remember to add such an environment file to the deploy commandline::

    openstack overcloud deploy --templates --ceph-storage-scale <number of nodes> -e /usr/share/openstack-tripleo-heat-templates/environments/storage-environment.yaml -e ~/my-ceph-settings.yaml

It is also possible to provide arbitrary stanza/key/value lines for `ceph.conf`
using the special `ceph::conf` configuration class. For example by using::

  parameter_defaults:
    ExtraConfig:
      ceph::conf::args:
        global/max_open_files:
          value: 131072
        global/my_setting:
          value: my_value

the resulting `ceph.conf` file should be populated with the following::

  [global]
  max_open_files: 131072
  my_setting: my_value

.. note::

    The :doc:`extra_config` doc has a more details on the usage of the different
    ExtraConfig interfaces.

.. _`puppet-ceph`: https://github.com/openstack/puppet-ceph
.. _`ceph.yaml static hieradata`: https://github.com/openstack/tripleo-heat-templates/blob/master/puppet/hieradata/ceph.yaml
