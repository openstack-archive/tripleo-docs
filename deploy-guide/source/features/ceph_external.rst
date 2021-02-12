Use an external Ceph cluster with the Overcloud
===============================================

|project| supports use of an external Ceph cluster for certain services deployed
in the Overcloud.

Deploying Cinder, Glance, Nova, Gnocchi with an external Ceph RBD service
-------------------------------------------------------------------------

The overcloud may be configured to use an external Ceph RBD service by
enabling a particular environment file when deploying the
Overcloud. For Ocata and earlier use
`environments/puppet-ceph-external.yaml`. For Pike and newer, use
`environments/ceph-ansible/ceph-ansible-external.yaml` and install
ceph-ansible on the Undercloud as described in
:doc:`../deployment/index`. For Pike and newer a Ceph container is
downloaded and executed on Overcloud nodes to use Ceph binaries only
available within the container. These binaries are used to create
the CephX client keyrings on the overcloud. Thus it is necessary when
preparing to deploy a containerized overcloud, as described in
:doc:`../deployment/container_image_prepare`, to include the Ceph
container even if that overcloud will only connect to an external Ceph
cluster.

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
:doc:`ceph_config` are mutually exclusive per Heat stack. The
following scenarios are the only supported ones in which
`CephExternalMultiConfig` may be used per Heat stack:

* One external Ceph cluster configured, as described in previous
  section, in addition to multiple external Ceph clusters configured
  via `CephExternalMultiConfig`.

* One internal Ceph cluster, as described in :doc:`ceph_config` in
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
  described in the :doc:`ceph_config` documentation, has the same
  syntax.

Deploying Manila with an External CephFS Service
------------------------------------------------

If chosing to configure Manila with Ganesha as NFS gateway for CephFS,
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

Configuring Already Deployed Servers to use External Ceph
---------------------------------------------------------

When using ceph-ansible and :doc:`deployed_server`, it is necessary
to run commands like the following from the undercloud before
deployment::

    export OVERCLOUD_HOSTS="192.168.1.8 192.168.1.42"
    bash /usr/share/openstack-tripleo-heat-templates/deployed-server/scripts/enable-ssh-admin.sh
    for h in $OVERCLOUD_HOSTS ; do
        ssh $h -l stack "sudo groupadd ceph -g 64045 ; sudo useradd ceph -u 64045 -g ceph"
    done

In the example above, the OVERCLOUD_HOSTS variable should be set to
the IPs of the overcloud hosts which will be Ceph clients (e.g. Nova,
Cinder, Glance, Gnocchi, Manila, etc.). The `enable-ssh-admin.sh`
script configures a user on the overcloud nodes that Ansible uses to
configure Ceph. The `for` loop creates the Ceph user on the relevant
overcloud hosts.

.. note::

    If the overcloud is named differently than the default ("overcloud"),
    then you'll have to set the OVERCLOUD_PLAN variable as well

Deployment of an Overcloud with External Ceph
---------------------------------------------

Finally add the above environment files to the deploy commandline. For
Ocata and earlier::

  openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/puppet-ceph-external.yaml -e ~/my-additional-ceph-settings.yaml

For Pike and later::

  openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible-external.yaml -e ~/my-additional-ceph-settings.yaml

.. _`ceph-ansible/group_vars`: https://github.com/ceph/ceph-ansible/tree/master/group_vars
