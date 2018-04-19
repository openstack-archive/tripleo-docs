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

When using ceph-ansible to deploy Ceph in containers, the process
described in the :ref:`prepare-environment-containers` documentation
will configure the deployment to use the appropriate Ceph docker
image. However, it is also possible to override the default docker
image. For example::

  parameter_defaults:
    DockerCephDaemonImage: ceph/daemon:tag-stable-3.0-jewel-centos-7

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

The playbooks provided by `ceph-ansible` are triggered by a Mistral
workflow. A new `CephAnsibleExtraConfig` parameter has been added to
the templates and can be used to provide arbitrary config variables
consumed by `ceph-ansible`. The pre-existing template params consumed
by the TripleO Pike release to drive `puppet-ceph` continue to work
and are translated, when possible, into their equivalent
`ceph-ansible` variable.

For example, to encrypt the data stored on OSDs use the following::

  parameter_defaults:
    CephAnsibleExtraConfig:
      dmcrypt: true

To change the backfill and recovery operations that Ceph uses to
rebalance a cluster, use an example like the following::

  parameter_defaults:
    CephAnsibleExtraConfig:
      osd_recovery_op_priority: 3
      osd_recovery_max_active: 3
      osd_max_backfills: 1

The above example may be used to change any of the defaults found in
`ceph-ansible/group_vars`_.

If a parameter to override is not an
available group variable, then global settings in the `ceph.conf` may
be set directly using `CephConfigOverrides` like the following::

  parameter_defaults:
    CephConfigOverrides:
      max_open_files: 131072

Configure container settings with ceph-ansible
----------------------------------------------

The group variables `ceph_osd_docker_memory_limit`, which corresponds
to `docker run ... --memory`, and `ceph_osd_docker_cpu_limit`, which
corresponds to `docker run ... --cpu-quota`, may be overridden
depending on the hardware configuration and the system needs. Below is
an example of setting custom values to these parameters::

  parameter_defaults:
    CephAnsibleExtraConfig:
      ceph_osd_docker_memory_limit: 3g
      ceph_osd_docker_cpu_limit: 1

Configure OSD settings with ceph-ansible
----------------------------------------

To specify a set of dedicated block devices to use as Ceph OSDs, use
a variation of the following::

  parameter_defaults:
    CephAnsibleDisksConfig:
      devices:
        - /dev/sdb
        - /dev/sdc
        - /dev/sdd
      dedicated_devices:
        - /dev/sde
        - /dev/sde
        - /dev/sde
      osd_scenario: non-collocated

The above will produce three OSDs which run on `/dev/sdb`, `/dev/sdc`,
and `/dev/sdd` which all journal to `/dev/sde`. This same setup will
be duplicated per Ceph storage node and assumes uniform hardware.
If the journals will reside on the same disks as the OSDs then the
above should be changed to the following::

  parameter_defaults:
    CephAnsibleDisksConfig:
      devices:
        - /dev/sdb
        - /dev/sdc
        - /dev/sdd
      osd_scenario: collocated

The number of OSDs in a Ceph deployment should proportionally affect
the number of Ceph PGs per Pool as determined by Ceph's
`pgcalc`_. When the appropriate default pool size and PG number are
determined, the defaults should be overridden using an example like
the following::

  parameter_defaults:
    CephPoolDefaultSize: 3
    CephPoolDefaultPgNum: 128

Customize OpenStack Ceph Pools
------------------------------

In addition to setting the default PG number for each pool created,
each Ceph pool created for OpenStack can have its own PG number.
TripleO supports customization of these values by using a syntax like
the following::

  parameter_defaults:
    CephPools:
      - {"name": backups, "pg_num": 512, "pgp_num": 512}
      - {"name": volumes, "pg_num": 1024, "pgp_num": 1024, "rule_name": 'replicated_rule', "erasure_profile": '', "expected_num_objects": 6000}
      - {"name": vms, "pg_num": 512, "pgp_num": 512}
      - {"name": images, "pg_num": 128, "pgp_num": 128}

In the above example, PG numbers for each pool differ based on the
OpenStack use case from `pgcalc`_. The example above also passes
additional options as described in the `ceph osd pool create`_
documentation to the volumes pool used by Cinder.

Override Ansible run options
----------------------------

TripleO runs the ceph-ansible `site-docker.yml.sample` playbook by
default. The values in this playbook should be overridden as described
in this document and the playbooks themselves should not be modified.
However, it is possible to specify which playbook is run using the
following parameter::

  parameter_defaults:
    CephAnsiblePlaybook: /usr/share/ceph-ansible/site-docker.yml.sample

For each TripleO Ceph deployment, the above playbook's output is logged
to `/var/log/mistral/ceph-install-workflow.log`. The default verbosity
of the playbook run is 0. The example below sets the verbosity to 3::

  parameter_defaults:
    CephAnsiblePlaybookVerbosity: 3

During the playbook run temporary files, like the Ansible inventory
and the ceph-ansible parameters that are passed as overrides as
described in this document, are stored on the undercloud in a
directory that matches the pattern `/tmp/ansible-mistral-action*`.
This directory is deleted at the end of each Mistral workflow which
triggers the playbook run. However, the temporary files are not
deleted when the verbosity is greater than 0. This option is helpful
when debugging.

The Ansible environment variables may be overridden using an example
like the following::

  parameter_defaults:
    CephAnsibleEnvironmentVariables:
      ANSIBLE_SSH_RETRIES: '6'
      DEFAULT_FORKS: '25'

In the above example, the number of SSH retries is increased from the
default to prevent timeouts. Ansible's fork number is automatically
limited to the number of possible hosts at runtime. TripleO uses
ceph-ansible to configure Ceph clients in addition to Ceph servers so
when deploying a large number of compute nodes ceph-ansible may
consume a lot of memory on the undercloud. Lowering the fork count
will reduce the memory footprint while the Ansible playbook is running
at the expense of the number of hosts configured in parallel.

Applying ceph-ansible customizations to a overcloud deployment
--------------------------------------------------------------

The desired options from the ceph-ansible examples above to customize
the ceph.conf, container, OSD or Ansible options may be combined under
one `parameter_defaults` setting and saved in an environment file
"~/my-ceph-settings.yaml" and added to the deploy commandline::

    openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml -e ~/my-ceph-settings.yaml

Already Deployed Servers and ceph-ansible
-----------------------------------------

When using ceph-ansible and :doc:`deployed_server`, it is necessary
to run commands like the following from the undercloud before
deployment::

    export OVERCLOUD_HOSTS="192.168.1.8 192.168.1.42"
    bash /usr/share/openstack-tripleo-heat-templates/deployed-server/scripts/enable-ssh-admin.sh
    for h in $OVERCLOUD_HOSTS ; do
        ssh $h -l stack "sudo groupadd ceph -g 64045 ; sudo useradd ceph -u 64045 -g ceph"
    done

In the example above, the OVERCLOUD_HOSTS variable should be set to
the IPs of the overcloud hosts which will be Ceph servers or which
will host Ceph clients (e.g. Nova, Cinder, Glance, Gnocchi, Manila,
etc.). The `enable-ssh-admin.sh` script configures a user on the
overcloud nodes that Ansible uses to configure Ceph. The `for`
loop creates the Ceph user on the relevant overcloud hosts.

.. note::

   Both puppet-ceph and ceph-ansible do not reformat the OSD disks and
   expect them to be clean to complete successfully. Consequently, when reusing
   the same nodes (or disks) for new deployments, it is necessary to clean the
   disks before every new attempt. One option is to enable the automated
   cleanup functionality in Ironic, which will zap the disks every time that a
   node is released. The same process can be executed manually or only for some
   target nodes, see `cleaning instructions in the Ironic doc`.

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
.. _`pgcalc`: http://ceph.com/pgcalc
.. _`ceph osd pool create`: http://docs.ceph.com/docs/jewel/rados/operations/pools/#create-a-pool
.. _`cleaning instructions in the Ironic doc`: https://docs.openstack.org/ironic/latest/admin/cleaning.html
