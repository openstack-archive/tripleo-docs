Configuring Ceph with Custom Config Settings
============================================

This guide assumes that the undercloud is already installed and ready
to deploy an overcloud and that the appropriate repositories
containing Ceph packages, including ceph-ansible if applicable, have
been enabled and installed as described in
:doc:`../deployment/index`.

Deploying an Overcloud with Ceph
--------------------------------

TripleO can deploy and configure Ceph as if it was a composable
OpenStack service and configure OpenStack services like Nova, Glance,
Cinder, Cinder Backup, and Gnocchi to use it as a storage backend.

TripleO can only deploy one Ceph cluster in the overcloud per Heat
stack. However, within that Heat stack it's possible to configure
an overcloud to communicate with multiple Ceph clusters which are
external to the overcloud. To do this, follow this document to
configure the "internal" Ceph cluster which is part of the overcloud
and also use the `CephExternalMultiConfig` parameter described in the
:doc:`ceph_external` documentation.

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

Configuring nova-compute ephemeral backend per role
---------------------------------------------------

NovaEnableRdbBackend can be configured on a per-role basis allowing compute
hosts to be deployed with a subset using RBD ephemeral disk. The
remaining hosts continue using the default local ephemeral disk.

.. note::

    For best performacne images to be deployed to RBD ephemeral computes should be in RAW format while images to be deployed to local ephemeral computes should be QCOW2 format.


Generate roles_data including the provided ComputeLocalEphemeral and
ComputeRBDEphemeral roles as described in the :ref:`custom_roles`
documentation.

Configure the role counts, for example "nodes.yaml"::

    parameter_defaults:
      ComputeLocalEphemeralCount: 10
      ComputeRBDEphemeralCount: 10

Alternatively the "NovaEnableRbdBackend" parameter can be set as a role
parameter on any Compute role, for example::

    parameter_defaults:
      ComputeParameters:
        NovaEnableRbdBackend: true
      MyCustomComputeParameters:
        NovaEnableRbdBackend: false

Deploy using the per-role ceph-ansible environment file
"environments/ceph-ansible/ceph-ansible-per-role.yaml"::

    openstack overcloud deploy --templates -r my_roles_data.yaml -e nodes.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible-per-role.yaml

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
be duplicated per Ceph storage node and assumes uniform hardware. If
you do not have uniform hardware see :doc:`node_specific_hieradata`.

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

The above example may be used to change any of the defaults found in
`ceph-ansible/group_vars`_.

If a parameter to override is not an available group variable, then
`ceph.conf` sections settings may be set directly using
`CephConfigOverrides` like the following::

  parameter_defaults:
    CephConfigOverrides:
      global:
        max_open_files: 131072
      osd:
        osd_journal_size: 40960

To change the backfill and recovery operations that Ceph uses to
rebalance a cluster, use an example like the following::

  parameter_defaults:
    CephConfigOverrides:
      global:
        osd_recovery_op_priority: 3
        osd_recovery_max_active: 3
        osd_max_backfills: 1

Configuring CephX Keys
----------------------

TripleO will create a Ceph cluster with a CephX key file for OpenStack
RBD client connections that is shared by the Nova, Cinder, Glance and
Gnocchi services to read and write to their pools. Not only will the
keyfile be created but the Ceph cluster will be configured to accept
connections when the key file is used. The file will be named
`/etc/ceph/ceph.client.openstack.keyring` and it will be created
using the following defaults:

* CephClusterName: 'ceph'
* CephClientUserName: 'openstack'
* CephClientKey: This value is randomly genereated per Heat stack. If
  it is overridden the recomendation is to set it to the output of
  `ceph-authtool --gen-print-key`.

If the above values are overridden, the keyring file will have a
different name and different content. E.g. if `CephClusterName` was
set to 'foo' and `CephClientUserName` was set to 'bar', then the
keyring file would be called `foo.client.bar.keyring` and it would
contain the line `[client.bar]`.

The `CephExtraKeys` parameter may be used to generate additional key
files containing other key values and should contain a list of maps
where each map describes an each additional key. The syntax of each
map must conform to what the `ceph-ansible/library/ceph_key.py`
Ansible module accepts. The `CephExtraKeys` parameter should be used
like this::

    CephExtraKeys:
      - name: "client.glance"
        caps:
          mgr: "allow *"
          mon: "profile rbd"
          osd: "profile rbd pool=images"
        key: "AQBRgQ9eAAAAABAAv84zEilJYZPNuJ0Iwn9Ndg=="
        mode: "0600"

If the above is used, in addition to the
`ceph.client.openstack.keyring` file, an additional file called
`ceph.client.glance.keyring` will be created which contains::

  [client.glance]
        key = AQBRgQ9eAAAAABAAv84zEilJYZPNuJ0Iwn9Ndg==
        caps mgr = "allow *"
        caps mon = "profile rbd"
        caps osd = "profile rbd pool=images"

The Ceph cluster will also allow the above key file to be used to
connect to the images pool. Ceph RBD clients which are external to the
overcloud could then use this CephX key to connect to the images
pool used by Glance. The default Glance deployment defined in the Heat
stack will continue to use the `ceph.client.openstack.keyring` file
unless that Glance configuration itself is overridden.

Tuning Ceph OSD CPU and Memory
------------------------------

The group variable `ceph_osd_docker_cpu_limit`, which corresponds to
``docker run ... --cpu-quota``, may be overridden depending on the
hardware configuration and the system needs. Below is an example of
setting custom values for this parameter::

  parameter_defaults:
    CephAnsibleExtraConfig:
      ceph_osd_docker_cpu_limit: 1

.. warning:: Overriding the `ceph_osd_docker_memory_limit` variable
             is not recommended. Use of ceph-ansible 3.2 or newer is
             recommended as it will automatically tune this variable
             based on hardware.

.. admonition:: ceph-ansible 3.2 and newer
   :class: ceph

   As of ceph-ansible 3.2, the `ceph_osd_docker_memory_limit` is set
   by default to the max memory of the host in order to ensure Ceph
   does not run out of resources. While it is technically possible to
   override the bluestore `osd_memory_target` by setting it inside of
   the `CephConfigOverrides` directive, it is better to let
   ceph-ansible automatically tune this variable. Such tuning is
   also influenced by the boolean `is_hci` flag. When collocating
   Ceph OSD services on the same nodes which run Nova compute
   services (also known as "hyperconverged deployments"), set
   this variable as in the example below::

      parameter_defaults:
        CephAnsibleExtraConfig:
          is_hci: true

   When using filestore in hyperconverged deployments, include the
   "environments/tuned-ceph-filestore-hci.yaml" enviornment file to
   set a :doc:`tuned profile <tuned>` designed for Ceph filestore.
   Do not use this tuned profile with bluestore.

.. admonition:: ceph-ansible 4.0 and newer
   :class: ceph

   Stein's default Ceph is Nautilus, which introduced the Messenger v2 protocol.
   ceph-ansible 4.0 and newer added a parameter in order to:

   * enable or disable the v1 protocol
   * define the port used to bind the process

   Ceph Nautilus enables both v1 and v2 protocols by default and v1 is maintained
   for backward compatibility.
   To disable v1 protocol, set the variables as in the example below::

      parameter_defaults:
        CephAnsibleExtraConfig:
          mon_host_v1:
            enabled: False


Configure OSD settings with ceph-ansible
----------------------------------------

To specify which block devices will be used as Ceph OSDs, use a
variation of the following::

  parameter_defaults:
    CephAnsibleDisksConfig:
      devices:
        - /dev/sdb
        - /dev/sdc
        - /dev/sdd
        - /dev/nvme0n1
      osd_scenario: lvm
      osd_objectstore: bluestore

Because `/dev/nvme0n1` is in a higher performing device class, e.g.
it is an SSD and the other devices are spinning HDDs, the above will
produce three OSDs which run on `/dev/sdb`, `/dev/sdc`, and
`/dev/sdd` and they will use `/dev/nvme0n1` as a bluestore WAL device.
The `ceph-volume` tool does this by using `the "batch" subcommand`_.
This same setup will be duplicated per Ceph storage node and assumes
uniform hardware. If you do not have uniform hardware see
:doc:`node_specific_hieradata`. If the bluestore WAL data will reside
on the same disks as the OSDs, then the above could be changed to the
following::

  parameter_defaults:
    CephAnsibleDisksConfig:
      devices:
        - /dev/sdb
        - /dev/sdc
        - /dev/sdd
      osd_scenario: lvm
      osd_objectstore: bluestore

The example above configures the devices list using the disk
name, e.g. `/dev/sdb`, based on the `sd` driver. This method of
referring to block devices is not guaranteed to be consistent on
reboots so a disk normally identified by `/dev/sdc` may be named
`/dev/sdb` later. Another way to refer to block devices is `by-path`
which is persistent accross reboots. The `by-path` names for your
disks are in the Ironic introspection data. A utility exists to
generate a Heat environment file from Ironic introspection data
with a devices list for each of the Ceph nodes in a deployment
automatically as described in :doc:`node_specific_hieradata`.

.. warning:: `osd_scenario: lvm` is used above to default new
             deployments to bluestore as configured, by `ceph-volume`,
             and is only available with ceph-ansible 3.2, or newer,
             and with Luminous, or newer. The parameters to support
             filestore with ceph-ansible 3.2 are backwards-compatible
             so existing filestore deployments should not simply have
             their `osd_objectstore` or `osd_scenario` parameters
             changed without taking steps to maintain both backends.

.. admonition:: Filestore or ceph-ansible 3.1 (or older)
    :class: ceph

    Ceph Luminous supports both filestore and bluestore, but bluestore
    deployments require ceph-ansible 3.2, or newer, and `ceph-volume`.
    For older versions, if the `osd_scenario` is either `collocated` or
    `non-collocated`, then ceph-ansible will use the `ceph-disk` tool,
    in place of `ceph-volume`, to configure Ceph's filestore backend
    in place of bluestore. A variation of the above example which uses
    filestore and `ceph-disk` is the following::

       parameter_defaults:
         CephAnsibleDisksConfig:
           devices:
             - /dev/sdb
             - /dev/sdc
             - /dev/sdd
           dedicated_devices:
             - /dev/nvme0n1
             - /dev/nvme0n1
             - /dev/nvme0n1
           osd_scenario: non-collocated
           osd_objectstore: filestore

    The above will produce three OSDs which run on `/dev/sdb`,
    `/dev/sdc`, and `/dev/sdd`, and which all journal to three
    partitions which will be created on `/dev/nvme0n1`. If the
    journals will reside on the same disks as the OSDs, then
    the above should be changed to the following::

       parameter_defaults:
         CephAnsibleDisksConfig:
           devices:
             - /dev/sdb
             - /dev/sdc
             - /dev/sdd
           osd_scenario: collocated
           osd_objectstore: filestore

    It is unsupported to use `osd_scenario: collocated` or
    `osd_scenario: non-collocated` with `osd_objectstore: bluestore`.

Maintaining both Bluestore and Filestore Ceph Backends
------------------------------------------------------

For existing Ceph deployments, it is possible to scale new Ceph
storage nodes which use bluestore while keeping the existing Ceph
storage nodes using filestore.

In order to support both filestore and bluestore in a deployment,
the nodes which use filestore must continue to use the filestore
parameters like the following::

   parameter_defaults:
     CephAnsibleDisksConfig:
       devices:
         - /dev/sdb
         - /dev/sdc
       dedicated_devices:
         - /dev/nvme0n1
         - /dev/nvme0n1
       osd_scenario: non-collocated
       osd_objectstore: filestore

While the nodes which will use bluestore, all of the new nodes, must
use bluestore parameters like the following::

  parameter_defaults:
    CephAnsibleDisksConfig:
      devices:
        - /dev/sdb
        - /dev/sdc
        - /dev/nvme0n1
      osd_scenario: lvm
      osd_objectstore: bluestore

To resolve this difference, use :doc:`node_specific_hieradata` to
map the filestore node's machine unique UUID to the filestore
parameters, so that only those nodes are passed the filestore
parmaters, and then set the default Ceph parameters, e.g. those
found in `~/my-ceph-settings.yaml`, to the bluestore parameters.

An example of what the `~/my-node-settings.yaml` file, as described in
:doc:`node_specific_hieradata`, might look like for two nodes which
will keep using filestore is the following::

  parameter_defaults:
    NodeDataLookup:
      00000000-0000-0000-0000-0CC47A6EFDCC:
        devices:
          - /dev/sdb
          - /dev/sdc
        dedicated_devices:
          - /dev/nvme0n1
          - /dev/nvme0n1
        osd_scenario: non-collocated
        osd_objectstore: filestore
      00000000-0000-0000-0000-0CC47A6F13FF:
        devices:
          - /dev/sdb
          - /dev/sdc
        dedicated_devices:
          - /dev/nvme0n1
          - /dev/nvme0n1
        osd_scenario: non-collocated
        osd_objectstore: filestore

Be sure to set every existing Ceph filestore server to the filestore
parameters by its machine unique UUID. If the above is not done and
the default parameter is set to `osd_scenario=lvm` for the existing
nodes which were configured with `ceph-disk`, then these OSDs will not
start after a restart of the systemd unit or a system reboot.

The example above, makes bluestore the new default and filestore an
exception per node. An alternative approach is to keep the default of
filestore and `ceph-disk` and use :doc:`node_specific_hieradata` for
adding new nodes which use bluestore and `ceph-volume`. A benefit of
this is that there wouldn't be any configuration change for existing
nodes. However, every scale operation with Ceph nodes would require
the use of :doc:`node_specific_hieradata`. While the example above,
of making filestore and `ceph-disk` the per-node exception, requires
more work up front, it simplifies future scale up when completed. If
the cluster will be migrated to all bluestore, through node scale down
and scale up, then the amount of items in `~/my-node-settings.yaml`
could be reduced for each scale down and scale up operation until the
full cluster uses bluestore.

Customize Ceph Placement Groups per OpenStack Pool
--------------------------------------------------

The number of OSDs in a Ceph deployment should proportionally affect
the number of Ceph PGs per Pool as determined by Ceph's
`pgcalc`_. When the appropriate default pool size and PG number are
determined, the defaults should be overridden using an example like
the following::

  parameter_defaults:
    CephPoolDefaultSize: 3
    CephPoolDefaultPgNum: 128

In addition to setting the default PG number for each pool created,
each Ceph pool created for OpenStack can have its own PG number.
TripleO supports customization of these values by using a syntax like
the following::

  parameter_defaults:
    CephPools:
      - {"name": backups, "pg_num": 512, "pgp_num": 512, "application": rbd}
      - {"name": volumes, "pg_num": 1024, "pgp_num": 1024, "application": rbd, "rule_name": 'replicated_rule', "erasure_profile": '', "expected_num_objects": 6000}
      - {"name": vms, "pg_num": 512, "pgp_num": 512, "application": rbd}
      - {"name": images, "pg_num": 128, "pgp_num": 128, "application": rbd}

In the above example, PG numbers for each pool differ based on the
OpenStack use case from `pgcalc`_. The example above also passes
additional options as described in the `ceph osd pool create`_
documentation to the volumes pool used by Cinder. A TripleO validation
(described in `Validating Ceph Configuration`_) may be used to verify
that the PG numbers satisfy Ceph's PG overdose protection check before
the deployment starts.

Customizing crushmap using device classes
-----------------------------------------

Since Luminous, Ceph introduces a new `device classes` feature with the
purpose of automating one of the most common reasons crushmaps are
directly edited.
Device classes are a new property for OSDs visible by running `ceph osd
tree` and observing the class column, which should default correctly to
each device's hardware capability (hdd, ssd or nvme).
This feature is useful because Ceph CRUSH rules can restrict placement
to a specific device class. For example, they make it easy to create a
"fast" pool that distributes data only over SSDs.
To do this, one simply needs to specify in the pool definition which
device class should be used.
This is simpler than directly editing the CRUSH map itself.
There is no need for the operator to specify the device class for each
disk added into the cluster: with this new functionality, ceph is able
to autodetect the disk type (exposed by Linux kernel), placing it in
the right category.
For this reason the old way of specifying which block devices will be
used as Ceph OSDs is still valid::

    CephAnsibleDisksConfig:
        devices:
          - /dev/sdb
          - /dev/sdc
          - /dev/sdd
        osd_scenario: lvm
        osd_objectstore: bluestore

However, if the operator would like to force a specific device to
belong to a specific class, the `crush_device_class` property is
provided and the device list defined above can be changed into::

    CephAnsibleDisksConfig:
         lvm_volumes:
            - data: '/dev/sdb'
              crush_device_class: 'hdd'
            - data: '/dev/sdc'
              crush_device_class: 'sdd'
            - data: '/dev/sdd'
              crush_device_class: 'hdd'
        osd_scenario: lvm
        osd_objectstore: bluestore

.. note::

    crush_device_class property is optional and can be omitted. Ceph is
    able to `autodect` the type of disk, so this option can be used for
    advanced users or to fake/force the disk type.

After the device list is defined, the next step is to set some additional
parameters to properly generate the ceph-ansible variables; in TripleO
there are no explicitly exposed parameters to integrate this feature,
however, the ceph-ansible expected parameters can be generated through
`CephAnsibleExtraConfig`::

    CephAnsibleExtraConfig:
        crush_rule_config: true
        create_crush_tree: true
        crush_rules:
          - name: HDD
            root: default
            type: host
            class: hdd
            default: true
          - name: SSD
            root: default
            type: host
            class: ssd
            default: false

As seen in the example above, in order to properly generate the
crushmap hierarchy used by device classes, the `crush_rule_config` and
`create_crush_tree` booleans should be enabled. These booleans will
trigger the ceph-ansible playbook related to the crushmap customization,
and the rules associated to the device classes will be generated
according to the `crush_rules` array.  This allows the ceph cluster to
build a shadow hierarchy which reflects the specified rules.
Finally, as described in the customize placement group section, TripleO
supports the customization of pools; in order to tie a specific pool to
a device class, the `rule_name` option should be added as follows::

    CephPools:
      - name: fastpool
        pg_num: 8
        rule_name: SSD
        application: rbd

By adding this rule, we can make sure `fastpool` will follow the SSD
rule which is defined for the ssd device class and it can be configured
and used as a second (fast) tier to manage cinder volumes.

Customizing crushmap using node specific overrides
--------------------------------------------------

With device classes the ceph cluster can expose different storage
tiers with no need to manually edit the crushmap.
However, if device classes are not sufficient, the creation of a
specific crush hierarchy (e.g., host, rack, row, etc.), adding or
removing extra layers (e.g., racks) on the crushmap is still valid
and can be done via :doc:`node_specific_hieradata`.
NodeDataLookup playbook is able to generate node spec overrides using
the following syntax::

    NodeDataLookup: {"SYSTEM_UUID": {"osd_crush_location": {"root": "$MY_ROOT", "rack": "$MY_RACK", "host": "$OVERCLOUD_NODE_HOSTNAME"}}}

Generate NodeDataLookup manually can be error-prone. For this reason
TripleO provides the `make_ceph_disk`_ utility to build a JSON file
to get started, then it can be modified adding the `osd_crush_location`
properties dictionary with the syntax described above.

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

In the example above, the OVERCLOUD_HOSTS variable should be set to
the IPs of the overcloud hosts which will be Ceph servers or which
will host Ceph clients (e.g. Nova, Cinder, Glance, Gnocchi, Manila,
etc.). The `enable-ssh-admin.sh` script configures a user on the
overcloud nodes that Ansible uses to configure Ceph.

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


Adding Ceph Dashboard to a Overcloud deployment
------------------------------------------------

Starting from Ceph Nautilus the ceph dashboard component is available and
fully automated by TripleO.
To deploy the ceph dashboard include the ceph-dashboard.yaml environment
file as in the following example::

    openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-dashboard.yaml

The command above will include the ceph dashboard related services and
generates all the `ceph-ansible` required variables to trigger the playbook
execution for both deployment and configuration of this component.
When the deployment has been completed the Ceph dashboard containers,
including prometheus and grafana, will be running on the controller nodes
and will be accessible using the port 3100 for grafana and 9092 for prometheus;
since this service is only internal and doesn’t listen on the public vip, users
can reach both grafana and the exposed ceph dashboard using the controller
provisioning network vip on the specified port (8444 is the default for a generic
overcloud deployment).
The resulting deployment will be composed by an external stack made by grafana,
prometheus, alertmanager, node-exporter containers and the ceph dashboard mgr
module that acts as the backend for this external stack, embedding the grafana
layouts and showing the ceph cluster specific metrics coming from prometheus.
The Ceph Dashboard frontend is fully integrated with the tls-everywhere framework,
hence providing the tls environments files will trigger the certificate request for
both grafana and the ceph dashboard: the generated crt and key files are then passed
to ceph-ansible.
The Ceph Dashboard admin user role is set to `read-only` mode by default for safe
monitoring of the Ceph cluster. To permit an admin user to have elevated privileges
to alter elements of the Ceph cluster with the Dashboard, the operator can change the
default.
For this purpose, TripleO exposes a parameter that can be used to change the Ceph
Dashboard admin default mode.
Log in to the undercloud as `stack` user and create the `ceph_dashboard_admin.yaml`
environment file with the following content::

  parameter_defaults:
     CephDashboardAdminRO: false

Run the overcloud deploy command to update the existing stack and include the environment
file created with all other environment files that are already part of the existing
deployment::

    openstack overcloud deploy  --templates -e <existing_overcloud_environment_files> -e ceph_dashboard_admin.yml

The ceph dashboard will also work with composable networks.
In order to isolate the monitoring access for security purposes, operators can
take advantage of composable networks and access the dashboard through a separate
network vip. By doing this, it's not necessary to access the provisioning network
and separate authorization profiles may be implemented.
To deploy the overcloud with the ceph dashboard composable network we need first
to generate the controller specific role created for this scenario::

    openstack overcloud roles generate -o /home/stack/roles_data.yaml ControllerStorageDashboard Compute BlockStorage ObjectStorage CephStorage

Finally, run the overcloud deploy command including the new generated `roles_data.yaml`
and the `network_data_dashboard.yaml` file that will trigger the generation of this
new network.
The final overcloud command must look like the following::

    openstack overcloud deploy --templates -r /home/stack/roles_data.yaml -n /usr/share/openstack-tripleo-heat-templates/network_data_dashboard.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml -e ~/my-ceph-settings.yaml

Validating Ceph Configuration
-----------------------------

The tripleo-validations framework contains validations for Ceph
which may be run before deployment to save time debugging possible
failures.

Create an inventory on the undercloud which refers to itself::

  echo "undercloud ansible_connection=local" > inventory

Set Ansible environment variables::

  BASE="/usr/share/openstack-tripleo-validations"
  export ANSIBLE_RETRY_FILES_ENABLED=false
  export ANSIBLE_KEEP_REMOTE_FILES=1
  export ANSIBLE_CALLBACK_PLUGINS="${BASE}/callback_plugins"
  export ANSIBLE_ROLES_PATH="${BASE}/roles"
  export ANSIBLE_LOOKUP_PLUGINS="${BASE}/lookup_plugins"
  export ANSIBLE_LIBRARY="${BASE}/library"

See what Ceph validations are available::

  ls $BASE/playbooks | grep ceph

Run a Ceph validation with command like the following::

  ansible-playbook -i inventory $BASE/playbooks/ceph-ansible-installed.yaml

For Stein and newer it is possible to run validations using the
`openstack tripleo validator run` command with a syntax like the
following::

  openstack tripleo validator run --validation ceph-ansible-installed

The `ceph-ansible-installed` validation warns if the `ceph-ansible`
RPM is not installed on the undercloud. This validation is also run
automatically during deployment unless validations are disabled.

Ceph Placement Group Validation
-------------------------------

Ceph will refuse to take certain actions if they are harmful to the
cluster. E.g. if the placement group numbers are not correct for the
amount of available OSDs, then Ceph will refuse to create pools which
are required for OpenStack. Rather than wait for the deployment to
reach the point where Ceph is going to be configured only to find out
that the deployment failed because the parameters were not correct,
you may run a validation before deployment starts to quickly determine
if Ceph will create your OpenStack pools based on the overrides which
will be passed to the overcloud.

.. note::

   Unless there are at least 8 OSDs, the TripleO defaults will
   cause the deployment to fail unless you modify the CephPools,
   CephPoolDefaultSize, or CephPoolDefaultPgNum parameters. This
   validation will help you find the appropriate values.

To run the `ceph-pg` validation, configure your environment as
described in the previous section but also run the following
command to switch Ansible's `hash_behaviour` from `replace`
(the default) to `merge`. This is done to make Ansible behave
the same way that TripleO Heat Templates behaves when multiple
environment files are passed with the `-e @file.yaml` syntax::

  export ANSIBLE_HASH_BEHAVIOUR=merge

Then use a command like the following::

  ansible-playbook -i inventory $BASE/playbooks/ceph-pg.yaml -e @ceph.yaml -e num_osds=36

The `num_osds` parameter is required. This value should be the number
of expected OSDs that will be in the Ceph deployment. It should be
equal to the number of devices and lvm_volumes under
`CephAnsibleDisksConfig` multiplied by the number of nodes running the
`CephOSD` service (e.g. nodes in the CephStorage role, nodes in the
ComputeHCI role, and any custom roles, etc.). This value should also
be adjusted to compensate for the number of OSDs used by nodes with
node-specific overrides as covered earlier in this document.

In the above example, `ceph.yaml` should be the same file passed to
the overcloud deployment, e.g. `opesntack overcloud deploy ... -e
ceph.yaml`, as covered earlier in this document. As many files as
required may be passed using `-e @file.yaml` in order to get the
following parameters passed to the `ceph-pg` validation.

* CephPoolDefaultSize
* CephPoolDefaultPgNum
* CephPools

If the above parameters are not passed, then the TripleO defaults will
be used for the parameters above.

The above example is based only on Ceph pools created for RBD. If Ceph
RGW and/or Manila via NFS Ganesha is also being deployed, then simply
pass the same environment files for enabling these services you would
as if you were running `openstack overcloud deploy`. For example::

  export THT=/usr/share/openstack-tripleo-heat-templates/
  ansible-playbook -i inventory $BASE/playbooks/ceph-pg.yaml \
    -e @$THT/environments/ceph-ansible/ceph-rgw.yaml \
    -e @$THT/environments/ceph-ansible/ceph-mds.yaml \
    -e @$THT/environments/manila-cephfsganesha-config.yaml \
    -e @ceph.yaml -e num_osds=36

In the above example, the validation will simulate the creation of the
pools required for the RBD, RGW and MDS services and the validation
will fail if the placement group numbers are not correct.

.. _`puppet-ceph`: https://github.com/openstack/puppet-ceph
.. _`ceph-ansible`: https://github.com/ceph/ceph-ansible
.. _`ceph.yaml static hieradata`: https://github.com/openstack/tripleo-heat-templates/blob/master/puppet/hieradata/ceph.yaml
.. _`ceph-ansible/group_vars`: https://github.com/ceph/ceph-ansible/tree/master/group_vars
.. _`the "batch" subcommand`: http://docs.ceph.com/docs/master/ceph-volume/lvm/batch
.. _`pgcalc`: http://ceph.com/pgcalc
.. _`ceph osd pool create`: http://docs.ceph.com/docs/jewel/rados/operations/pools/#create-a-pool
.. _`cleaning instructions in the Ironic doc`: https://docs.openstack.org/ironic/latest/admin/cleaning.html
.. _`make_ceph_disk`: https://github.com/openstack/tripleo-heat-templates/blob/master/tools/make_ceph_disk_list.py
