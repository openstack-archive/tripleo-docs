Configuring Ceph with Custom Config Settings
============================================

This guide assumes that the undercloud is already installed and ready
to deploy an overcloud and that the appropriate repositories
containing Ceph packages, including ceph-ansible if applicable, have
been enabled and installed as described in
:doc:`../installation/installing`.

Deploying an Overcloud with Ceph
--------------------------------

TripleO can deploy and configure Ceph as if it was a composable
OpenStack service and configure OpenStack services like Nova, Glance,
Cinder, Cinder Backup, and Gnocchi to use it as a storage backend.

Prior to Pike, TripleO deployed Ceph with `puppet-ceph`_. With the
Pike release it is possible to use TripleO to deploy Ceph with
either `ceph-ansible`_ or puppet-ceph, though puppet-ceph is
deprecated. To deploy Ceph in containers use `ceph-ansible`, for which
only a containerized Ceph deployment is possible. It is not possible
to deploy a containerized Ceph with `puppet-ceph`.

To deploy with Ceph include either of the appropriate environment
files. For puppet-ceph use "environments/puppet-ceph.yaml"
like the following::

    openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/puppet-ceph.yaml

For ceph-ansible use "environments/ceph-ansible/ceph-ansible.yaml"
like the following::

    openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml

In both the puppet-ceph and ceph-ansible examples above, at least one
Ceph storage node is required. The following example will configure
one Ceph storage nodes on servers matching the `ceph-storage`
profile. It will also set the default pool size, the number of times
that an object should be written for data protection, to one. These
`parameter_defaults` may be saved in an environment file
"~/my-ceph-settings.yaml" and added to the deploy commandline::

  parameter_defaults:
    OvercloudCephStorageFlavor: ceph-storage
    CephStorageCount: 1
    CephDefaultPoolSize: 1

The values above are only appropriate for a development or POC
deployment. The default pool size is three but if there are less
than three Ceph OSDs, then the cluster will never reach status
`HEALTH_OK` because it has no place to make additional copies.
Thus, a POC deployment with less than three OSDs should override the
default default pool size. However, a production deployment should
replace both of the ones above with threes, or greater, in order to
have at least three storage nodes and at least three back up copies of
each object at minimum.

Customizing ceph.conf with puppet-ceph
--------------------------------------

Ceph demands for more careful configuration when deployed at scale.

It is possible to override any of the configuration parameters supported by
`puppet-ceph`_ at deployment time via Heat environment files. For example::

  parameter_defaults:
    ExtraConfig:
      ceph::profile::params::osd_journal_size: 2048

will customize the default `osd_journal_size` overriding any default
provided in the `ceph.yaml static hieradata`_.

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

To specify a set of dedicated block devices to use as Ceph OSDs use
the following::

  parameter_defaults:
    ExtraConfig:
      ceph::profile::params::osds:
        '/dev/sdb':
          journal: '/dev/sde'
        '/dev/sdc':
          journal: '/dev/sde'
        '/dev/sdd':
          journal: '/dev/sde'

The above will produce three OSDs which run on `/dev/sdb`, `/dev/sdc`,
and `/dev/sdd` which all journal to `/dev/sde`. This same setup will
be duplicated per Ceph storage node and assumes uniform hardware.

The `parameter_defaults` like the above may be saved in an environment
file "~/my-ceph-settings.yaml" and added to the deploy commandline::

    openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/puppet-ceph.yaml -e ~/my-ceph-settings.yaml

Customizing ceph.conf with ceph-ansible
---------------------------------------

The playbooks provided by `ceph-ansible` are triggered by Mistral
workflow. A new `CephAnsibleExtraConfig` parameter has been added to
the templates and can be used to provide arbitrary config variables
consumed by `ceph-ansible`. The pre-existing template params consumed
by the TripleO Pike release to drive `puppet-ceph` continue to work
and are translated, when possible, into their equivalent
`ceph-ansible` variable.

Global settings in the `ceph.conf` may be set using
`ceph_conf_overrides` like the following::

  CephAnsibleExtraConfig:
    ceph_conf_overrides:
      global:
        journal_size: 2048
        max_open_files: 131072
        osd_pool_default_size: 3
        osd_pool_default_pg_num: 256

`CephAnsibleExtraConfig` isn't just for `ceph.conf` overrides. For
example, to encrypt the data stored on OSDs use the following::

  CephAnsibleExtraConfig:
    dmcrypt: true

The above overrides the defaults found in the
`ceph-ansible/group_vars`_.

To specify a set of dedicated block devices to use as Ceph OSDs use
a variation of the following::

  parameter_defaults:
    CephAnsibleDisksConfig:
      devices:
        - /dev/sdb
        - /dev/sdc
        - /dev/sdd
      raw_journal_devices:
        - /dev/sde
        - /dev/sde
        - /dev/sde
      journal_collocation: false
      raw_multi_journal: true

The above will produce three OSDs which run on `/dev/sdb`, `/dev/sdc`,
and `/dev/sdd` which all journal to `/dev/sde`. This same setup will
be duplicated per Ceph storage node and assumes uniform hardware.

The `parameter_defaults` like the above may be saved in an environment
file "~/my-ceph-settings.yaml" and added to the deploy commandline::

    openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml -e ~/my-ceph-settings.yaml

.. note::

    The :doc:`extra_config` doc has a more details on the usage of the different
    ExtraConfig interfaces.

.. note::

    Deployment with `ceph-ansible` requires that OSDs run on dedicated
    block devices.

.. _`puppet-ceph`: https://github.com/openstack/puppet-ceph
.. _`ceph-ansible`: https://github.com/ceph/ceph-ansible
.. _`ceph.yaml static hieradata`: https://github.com/openstack/tripleo-heat-templates/blob/master/puppet/hieradata/ceph.yaml
.. _`ceph-ansible/group_vars`: https://github.com/ceph/ceph-ansible/tree/master/group_vars
