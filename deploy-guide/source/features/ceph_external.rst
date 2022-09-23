Use an external Ceph cluster with the Overcloud
===============================================

|project| supports use of an external Ceph cluster for certain services deployed
in the Overcloud.

Deploying Cinder, Glance, Nova, Gnocchi with an external Ceph RBD service
-------------------------------------------------------------------------

The overcloud may be configured to use an external Ceph RBD service by
enabling a particular environment file when deploying the
Overcloud. For Wallaby and newer include
`environments/external-ceph.yaml`.

For Ocata and earlier use
`environments/puppet-ceph-external.yaml`. For Pike through Victoria
use `environments/ceph-ansible/ceph-ansible-external.yaml` and install
ceph-ansible on the Undercloud as described in
:doc:`../deployment/index`. For Pike through Victoria a Ceph container
is downloaded and executed on Overcloud nodes to use Ceph binaries
only available within the container. These binaries are used to create
the CephX client keyrings on the overcloud. Thus, between Pike and
Victoria it was necessary when preparing to deploy a containerized
overcloud, as described in
:doc:`../deployment/container_image_prepare`, to include the Ceph
container even if that overcloud will only connect to an external Ceph
cluster. Starting in Wallaby neither ceph-ansible or cephadm configure
Ceph clients and instead the tripleo-ansible role tripleo_ceph_client
is used. Thus, it is not necessary to install ceph-ansible nor prepare
a Ceph container when configuring external Ceph in Wallaby and
newer. Simply include `environments/external-ceph.yaml` in the
deployment. All parameters described below remain consistent
regardless of external Ceph configuration method.

Some of the parameters in the above environment files can be overridden::

  parameter_defaults:
    # Enable use of RBD backend in nova-compute
    NovaEnableRbdBackend: true
    # Enable use of RBD backend in cinder-volume
    CinderEnableRbdBackend: true
    # Backend to use for cinder-backup
    CinderBackupBackend: ceph
    # Backend to use for glance
    GlanceBackend: rbd
    # Backend to use for gnocchi-metricsd
    GnocchiBackend: rbd
    # Name of the Ceph pool hosting Nova ephemeral images
    NovaRbdPoolName: vms
    # Name of the Ceph pool hosting Cinder volumes
    CinderRbdPoolName: volumes
    # Name of the Ceph pool hosting Cinder backups
    CinderBackupRbdPoolName: backups
    # Name of the Ceph pool hosting Glance images
    GlanceRbdPoolName: images
    # Name of the Ceph pool hosting Gnocchi metrics
    GnocchiRbdPoolName: metrics
    # Name of the user to authenticate with the external Ceph cluster
    CephClientUserName: openstack

The pools and the CephX user **must** be created on the external Ceph cluster
before deploying the Overcloud. TripleO expects a single user, configured via
CephClientUserName, to have the capabilities to use all the OpenStack pools;
the user could be created with a command like this::

  ceph auth add client.openstack mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rwx pool=images, allow rwx pool=backups, allow rwx pool=metrics'

In addition to the above customizations, the deployer **needs** to provide
at least three required parameters related to the external Ceph cluster::

  parameter_defaults:
    # The cluster FSID
    CephClusterFSID: '4b5c8c0a-ff60-454b-a1b4-9747aa737d19'
    # The CephX user auth key
    CephClientKey: 'AQDLOh1VgEp6FRAAFzT7Zw+Y9V6JJExQAsRnRQ=='
    # The list of Ceph monitors
    CephExternalMonHost: '172.16.1.7, 172.16.1.8, 172.16.1.9'

The above parameters will result in TripleO creating a Ceph
configuration file and cephx keyring in /etc/ceph on every
node which needs to connect to Ceph to use the RBD service.

Configuring Ceph Clients for Multiple External Ceph RBD Services
----------------------------------------------------------------

In Train and newer it's possible to use TripleO to deploy an
overcloud which is capable of using the RBD services of multiple
external Ceph clusters. A separate keyring and Ceph configuration file
is created for each external Ceph cluster in /etc/ceph on every
overcloud node which needs to connect to Ceph. This functionality is
provided by the `CephExternalMultiConfig` parameter.

Do not use `CephExternalMultiConfig` when configuring an overcloud to
use only one external Ceph cluster. Instead follow the example in the
previous section. The example in the previous section and the method
of deploying an internal Ceph cluster documented in
:doc:`deployed_ceph` are mutually exclusive per Heat stack. The
following scenarios are the only supported ones in which
`CephExternalMultiConfig` may be used per Heat stack:

* One external Ceph cluster configured, as described in previous
  section, in addition to multiple external Ceph clusters configured
  via `CephExternalMultiConfig`.

* One internal Ceph cluster, as described in :doc:`deployed_ceph` in
  addition to multiple external ceph clusters configured via
  `CephExternalMultiConfig`.

The `CephExternalMultiConfig` parameter is used like this::

  CephExternalMultiConfig:
    - cluster: 'ceph2'
      fsid: 'af25554b-42f6-4d2b-9b9b-d08a1132d3e8'
      external_cluster_mon_ips: '172.18.0.5,172.18.0.6,172.18.0.7'
      keys:
        - name: "client.openstack"
          caps:
            mgr: "allow *"
            mon: "profile rbd"
            osd: "profile rbd pool=volumes, profile rbd pool=backups, profile rbd pool=vms, profile rbd pool=images"
          key: "AQCwmeRcAAAAABAA6SQU/bGqFjlfLro5KxrB1Q=="
          mode: "0600"
      dashboard_enabled: false
    - cluster: 'ceph3'
      fsid: 'e2cba068-5f14-4b0f-b047-acf375c0004a'
      external_cluster_mon_ips: '172.18.0.8,172.18.0.9,172.18.0.10'
      keys:
        - name: "client.openstack"
          caps:
            mgr: "allow *"
            mon: "profile rbd"
            osd: "profile rbd pool=volumes, profile rbd pool=backups, profile rbd pool=vms, profile rbd pool=images"
          key: "AQCwmeRcAAAAABAA6SQU/bGqFjlfLro5KxrB2Q=="
          mode: "0600"
      dashboard_enabled: false

The above, in addition to the parameters from the previous section,
will result in an overcloud with the following files in /etc/ceph:

* ceph.client.openstack.keyring
* ceph.conf
* ceph2.client.openstack.keyring
* ceph2.conf
* ceph3.client.openstack.keyring
* ceph3.conf

The first two files which start with `ceph` will be created based on
the parameters discussed in the previous section. The next two files
which start with `ceph2` will be created based on the parameters from
the first list item within the `CephExternalMultiConfig` parameter
(e.g. `cluster: ceph2`). The last two files which start with `ceph3`
will be created based on the parameters from the last list item within
the `CephExternalMultiConfig` parameter (e.g. `cluster: ceph3`).

The last four files in the list which start with `ceph2` or `ceph3`
will also contain parameters found in the first two files which
start with `ceph` except where those parameters intersect. When
there's an intersection those parameters will be overridden with the
values from the `CephExternalMultiConfig` parameter. For example there
will only be one FSID in each Ceph configuration file with the
following values per file:

* ceph.conf will have `fsid = 4b5c8c0a-ff60-454b-a1b4-9747aa737d19`
  (as seen in the previous section)
* ceph2.conf will have `fsid = af25554b-42f6-4d2b-9b9b-d08a1132d3e8`
* ceph3.conf will have `fsid = e2cba068-5f14-4b0f-b047-acf375c0004a`

However, if the `external_cluster_mon_ips` key was not set within
the `CephExternalMultiConfig` parameter, then all three Ceph
configuration files would contain `mon host = 172.16.1.7, 172.16.1.8,
172.16.1.9`, as seen in the previous section. Thus, it is necessary to
override the `external_cluster_mon_ips` key within each list item of
the `CephExternalMultiConfig` parameter because each external Ceph
cluster will have its own set of unique monitor IPs.

The `CephExternalMultiConfig` and `external_cluster_mon_ips` keys map
one to one but have different names because each element of the
`CephExternalMultiConfig` list should contain a map of keys and values
directly supported by ceph-ansible. See `ceph-ansible/group_vars`_ for
an example of all possible keys.

The following parameters are the minimum necessary to configure an
overcloud to connect to an external ceph cluster:

* cluster: The name of the configuration file and key name prefix.
  This name defaults to "ceph" so if this parameter is not overridden
  there will be a name collision. It is not relevant if the
  external ceph cluster's name is already "ceph". For client role
  configuration this parameter is only used for setting a unique name
  for the configuration and key files.
* fsid: The FSID of the external ceph cluster.
* external_cluster_mon_ips: The list of monitor IPs of the external
  ceph cluster as a single string where each IP is comma delimited.
  If the external Ceph cluster is using both the v1 and v2 MSGR
  protocol this value may look like '[v2:10.0.0.1:3300,
  v1:10.0.0.1:6789], [v2:10.0.0.2:3300, v1:10.0.0.2:6789],
  [v2:10.0.0.3:3300, v1:10.0.0.3:6789]'.
* dashboard_enabled: Always set this value to false when using
  `CephExternalMultiConfig`. It ensures that the Ceph Dashboard is not
  installed. It is not supported to use ceph-ansible dashboard roles
  to communicate with an external Ceph cluster so not passing this
  parameter with a value of false within `CephExternalMultiConfig`
  will result in a failed deployment because the default value of true
  will be used.
* keys: This is a list of maps where each map defines CephX keys which
  OpenStack clients will use to connect to an external Ceph cluster.
  As stated in the previous section, the pools and the CephX user must
  be created on the external Ceph cluster before deploying the
  overcloud. The format of each map is the same as found in
  ceph-ansible. Thus, if the external Ceph cluster was deployed by
  ceph-ansible, then the deployer of that cluster could share that map
  with the TripleO deployer so that it could be used as a list item of
  `CephExternalMultiConfig`. Similarly, the `CephExtraKeys` parameter,
  described in the :doc:`deployed_ceph` documentation, has the same
  syntax.

Deploying Manila with an External CephFS Service
------------------------------------------------

If choosing to configure Manila with Ganesha as NFS gateway for CephFS,
with an external Ceph cluster, then add `environments/manila-cephfsganesha-config.yaml`
to the list of environment files used to deploy the overcloud and also
configure the following parameters::

  parameter_defaults:
    ManilaCephFSDataPoolName: manila_data
    ManilaCephFSMetadataPoolName: manila_metadata
    ManilaCephFSCephFSAuthId: 'manila'
    CephManilaClientKey: 'AQDLOh1VgEp6FRAAFzT7Zw+Y9V6JJExQAsRnRQ=='

Which represent the data and metadata pools in use by the MDS for
the CephFS filesystems, the CephX keyring to use and its secret.

Like for the other services, the pools and keyring must be created on the
external Ceph cluster before attempting the deployment of the overcloud.
The keyring should look like the following::

  ceph auth add client.manila mgr "allow *" mon "allow r, allow command 'auth del', allow command 'auth caps', allow command 'auth get', allow command 'auth get-or-create'" mds "allow *" osd "allow rw"

Compatibility Options
---------------------

As of the Train release TripleO will install Ceph Nautilus. If the
external Ceph cluster uses the Hammer release instead, pass the
following parameters to enable backward compatibility features::

  parameter_defaults:
    ExtraConfig:
      ceph::profile::params::rbd_default_features: '1'

Deployment of an Overcloud with External Ceph
---------------------------------------------

Finally add the above environment files to the deploy commandline. For
Wallaby and newer use::

  openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/external-ceph.yaml -e ~/my-additional-ceph-settings.yaml

For Train use::

  openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible-external.yaml -e ~/my-additional-ceph-settings.yaml

Standalone Ansible Roles for External Ceph
------------------------------------------

To configure an overcloud to use an external Ceph cluster, a directory
(e.g. /etc/ceph) in the overcloud containers should be populated with
Ceph configuration files and overcloud services (e.g. Nova) should be
configured to use those files. Tripleo provides Ansible roles to do
this standalone without tripleo-heat-templates or config-download.

Single Ceph Cluster
^^^^^^^^^^^^^^^^^^^

The `tripleo_ceph_client_files` Ansible role copies files from a
source directory (`tripleo_ceph_client_files_source`) on the host
where Ansible is run to a destination directory
(`tripleo_ceph_client_config_home`) on the overcloud nodes.
The user must create and populate the
`tripleo_ceph_client_files_source` directory with actual Ceph
configuration and cephx key files before running the role. For
example::

  $ ls -l /home/stack/ceph_files/
  total 16
  -rw-r--r--. 1 stack stack 245 Nov 14 13:40 ceph.client.openstack.keyring
  -rw-r--r--. 1 stack stack 173 Nov 14 13:40 ceph.conf

If the above directory exists on the host where the `ansible-playbook`
command is run, then the `tripleo_ceph_client_files_source` parameter
should be set to `/home/stack/ceph_files/`. The optional parameter
`tripleo_ceph_client_config_home` defaults to
`/var/lib/tripleo-config/ceph` since OpenStack containers will bind
mount this directory to `/etc/ceph`. The `tripleo_nova_libvirt`
Ansible role will add a secret key to libvirt so that it uses the
cephx key put in place by the `tripleo_ceph_client_files` role; it
does this if either `tripleo_nova_libvirt_enable_rbd_backend` or
`tripleo_cinder_enable_rbd_backend` are true. When these roles
are used to configure a compute node the following `group_vars` should
be set::

  tripleo_ceph_client_files_source: /home/stack/ceph_files
  tripleo_ceph_client_config_home: /var/lib/tripleo-config/ceph
  tripleo_nova_libvirt_enable_rbd_backend: true
  tripleo_cinder_enable_rbd_backend: true

The `tripleo_ceph_client_files` role may then be included in a
playbook as follows in order to configure a standalone compute node to
use a single Ceph cluster::

    - name: configure ceph client
      import_role:
        name: tripleo_ceph_client_files

In order for Nova to use the Ceph cluster, the `libvirt` section of
the `nova.conf` file should be configured. The `tripleo_nova_compute`
role `tripleo_nova_compute_config_overrides` variable may be set as
follows in the inventory to set the `libvirt` values along with
others::

  Compute:
    vars:
      tripleo_nova_compute_config_overrides:
        libvirt:
          images_rbd_ceph_conf: /etc/ceph/ceph.conf
          images_rbd_glance_copy_poll_interval: '15'
          images_rbd_glance_copy_timeout: '600'
          images_rbd_glance_store_name: default_backend
          images_rbd_pool: vms
          images_type: rbd
          rbd_secret_uuid: 604c9994-1d82-11ed-8ae5-5254003d6107
          rbd_user: openstack

TripleO's convention is to set the `rbd_secret_uuid` to the FSID of
the Ceph cluster. The FSID should be in the ceph.conf file. The
`tripleo_nova_libvirt` role will use `virsh secret-*` commands so that
libvirt can retrieve the cephx secret using the FSID as a key. This
can be confirmed after running Ansible with `podman exec
nova_virtsecretd virsh secret-get-value $FSID`.

The `tripleo_ceph_client_files` role only supports the _configure_
aspect of the standalone tripleo-ansible roles because it just
configures one or more pairs of files on its target nodes. Thus, the
`import_role` example above could be placed in a playbook file like
`deploy-tripleo-openstack-configure.yml`, before the roles for
`tripleo_nova_libvirt` and `tripleo_nova_compute` are imported.

Multiple Ceph Clusters
^^^^^^^^^^^^^^^^^^^^^^

To configure more than one Ceph backend include the
`tripleo_ceph_client_files` role from the single cluster example
above. Populate the `tripleo_ceph_client_files_source` directory with
all of the ceph configuration and cephx key files For example::

  $ ls -l /home/stack/ceph_files/
  total 16
  -rw-r--r--. 1 stack stack 213 Nov 14 13:41 ceph2.client.openstack.keyring
  -rw-r--r--. 1 stack stack 228 Nov 14 13:41 ceph2.conf
  -rw-r--r--. 1 stack stack 245 Nov 14 13:40 ceph.client.openstack.keyring
  -rw-r--r--. 1 stack stack 173 Nov 14 13:40 ceph.conf

For multiple Ceph clusters, the `tripleo_nova_libvirt` role expects a
`tripleo_cinder_rbd_multi_config` Ansible variable like this::

  tripleo_cinder_rbd_multi_config:
    ceph2:
      CephClusterName: ceph2
      CephClientUserName: openstack

It is not necessary to put the default Ceph cluster (named "ceph" from
the single node example) in `tripleo_cinder_rbd_multi_config`. Only
the additional clusters (e.g. ceph2) and name their keys so they
match the `CephClusterName`. In the above example, the
`CephClusterName` value "ceph2" matches the "ceph2.conf" and
"ceph2.client.openstack.keyring". Also, the `CephClientUserName` value
"openstack" matches "ceph2.client.openstack.keyring". The
`tripleo_nova_libvirt` Ansible role uses the
`tripleo_cinder_rbd_multi_config` map as a guide to know which libvirt
secrets to create and which cephx keys to make available within the
Nova containers.

If the combined examples above from the single cluster section for
the primary cluster "ceph" and this section for the seconary Ceph
cluster "ceph2" are used, then the directory defined by
`tripleo_ceph_client_config_home` will be populated with four files:
`ceph.conf`, `ceph2.conf`, `ceph.client.openstack.keyring` and
`ceph2.client.openstack.keyring`, which will be mounted into the Nova
containers and two libvirt secrets will be created for each cephx
key. To add more Ceph clusters, extend the list
`tripleo_cinder_rbd_multi_config` and populate
`tripleo_ceph_client_files_source` with additional files.

.. _`ceph-ansible/group_vars`: https://github.com/ceph/ceph-ansible/tree/master/group_vars
