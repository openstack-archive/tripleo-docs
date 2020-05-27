Distributed Multibackend Storage
================================

In Ussuri and newer, |project| is able to extend
:doc:`distributed_compute_node` to include distributed image
management and persistent storage with the benefits of using
OpenStack and Ceph.

Features
--------

This Distributed Multibackend Storage design extends the architecture
described in :doc:`distributed_compute_node` to support the following
worklow.

- Upload an image to the Central site, and any additional DCN sites
  with storage, concurrently using one command like `glance
  image-create-via-import --stores central,dcn1,dcn3`.
- Move a copy of the same image to additional DCN sites when needed
  using a command like `glance image-import <IMAGE-ID> --stores
  dcn2,dcn4 --import-method copy-image`.
- The image's unique ID will be shared consistently across sites
- The image may be copy-on-write booted on any DCN site as the RBD
  pools for Glance and Nova will use the same local Ceph cluster.
- If the Glance server at each DCN site was configured with write
  access to the Central Ceph cluster as an additional store, then an
  image generated from making a snapshot of an instance running at a
  DCN site may be copied back to the central site and then copied to
  additional DCN sites.
- The same Ceph cluster per site may also be used by Cinder as an RBD
  store to offer local volumes in active/active mode.

In the above workflow the only time RBD traffic crosses the WAN is
when an image is imported or copied between sites. Otherwise all RBD
traffic is local to each site for fast COW boots, and performant IO
to the local Cinder and Nova Ceph pools.

Architecture
------------

The architecture to support the above features has the following
properties.

- A separate Ceph cluster at each availability zone or geographic
  location
- Glance servers at each availability zone or geographic location
- The containers implementing the Ceph clusters may be collocated on
  the same hardware providing compute services, i.e. the compute nodes
  may be hyper-converged, though it is not necessary that they be
  hyper-converged
- It is not necessary to deploy Glance and Ceph at each DCN site, if
  storage services are not needed at that DCN site

In this scenario the Glance service at the central site is configured
with multiple stores such that.

- The central Glance server's default store is the central Ceph
  cluster using the RBD driver
- The central Glance server has additional RBD stores; one per DCN
  site running Ceph

Similarly the Glance server at each DCN site is configured with
multiple stores such that.

- Each DCN Glance server's default store is the DCN Ceph
  cluster that is in the same geographic location.
- Each DCN Glance server is configured with one additional store which
  is the Central RBD Ceph cluster.

Though there are Glance services distributed to multiple sites, the
Glance client for overcloud users should use the public Glance
endpoints at the central site. These endpoints may be determined by
querying the Keystone service, which only runs at the central site,
with `openstack endpoint list`. Ideally all images should reside in
the central Glance and be copied to DCN sites before instances of
those images are booted on DCN sites. If an image is not copied to a
DCN site before it is booted, then the image will be streamed to the
DCN site and then the image will boot as an instance. This happens
because Glance at the DCN site has access to the images store at the
Central ceph cluster. Though the booting of the image will take time
because it has not been copied in advance, this is still preferable
to failing to boot the image.

Stacks
------

In the example deployment three stacks are deployed:

control-plane
   All control plane services including Glance. Includes a Ceph
   cluster named central which is hypercoverged with compute nodes and
   runs Cinder in active/passive mode managed by pacemaker.
dcn0
   Runs Compute, Glance and Ceph services. The Cinder volume service
   is configured in active/active mode and not managed by pacemaker.
   The Compute and Cinder services are deployed in a separate
   availability zone and may also be in a separate geographic
   location.
dcn1
   Deploys the same services as dcn0 but in a different availability
   zone and also in a separate geographic location.

Note how the above differs from the :doc:`distributed_compute_node`
example which splits services at the primary location into two stacks
called `control-plane` and `central`. This example combines the two
into one stack.

During the deployment steps all templates used to deploy the
control-plane stack will be kept on the undercloud in
`/home/stack/control-plane`, all templates used to deploy the dcn0
stack will be kept on the undercloud in `/home/stack/dcn0` and dcn1
will follow the same pattern as dcn0. The sites dcn2, dcn3 and so on
may be created, based on need, by following the same pattern.

Ceph Deployment Types
---------------------

|project| supports two types of Ceph deployments. An "internal" Ceph
deployment is one where a Ceph cluster is deployed as part of the
overcloud as described in :doc:`ceph_config`. An "external" Ceph
deployment is one where a Ceph cluster already exists and an overcloud
is configured to be a client of that Ceph cluster as described in
:doc:`ceph_external`. Ceph external deployments have special meaning
to |project| in the following ways:

- The Ceph cluster was not deployed by |project|
- The OpenStack Ceph client is configured by |project|

The deployment example in this document uses the "external" term to
focus on the second of the above because the client configuration is
important. This example differs from the first of the above because
Ceph was deployed by |project|, however relative to other stacks, it
is an external Ceph cluster because, for the stacks which configure
the Ceph clients, it doesn't matter that the Ceph server came from a
different stack. In this sense, the example in this document uses both
types of deployments as described in the following sequence:

- The central site deploys an internal Ceph cluster called central
  with an additional cephx keyring which may be used to access the
  central ceph pools.
- The dcn0 site deploys an internal Ceph cluster called dcn0 with an
  additional cephx keyring which may be used to access the dcn0 Ceph
  pools. During the same deployment the dcn0 site is also configured
  with the cephx keyring from the previous step so that it is also a
  client of the external Ceph cluster, relative to dcn0, called
  central from the previous step. The `GlanceMultistoreConfig`
  parameter is also used during this step so that Glance will use the
  dcn0 Ceph cluster as an RBD store by default but it will also be
  configured to use the central Ceph cluster as an additional RBD
  backend.
- The dcn1 site is deployed the same way as the dcn0 site and the
  pattern may be continued for as many DCN sites as necessary.
- The central site is then updated so that in addition to having an
  internal Ceph deployment for the cluster called central, it is also
  configured with multiple external ceph clusters, relative to the
  central site, for each DCN site. This is accomplished by passing
  the cephx keys which were created during each DCN site deployment
  as input to the stack update. During the stack update the
  `GlanceMultistoreConfig` parameter is added so that Glance will
  continue to use the central Ceph cluster as an RBD store by
  default but it will also be configured to use each DCN Ceph cluster
  as an additional RBD backend.

The above sequence is possible by using the `CephExtraKeys` parameter
as described in :doc:`ceph_config` and the `CephExternalMultiConfig`
parameter described in :doc:`ceph_external`.

Deployment Steps
----------------

This section shows the deployment commands and associated environment
files of an example DCN deployment with distributed image
management. It is based on the :doc:`distributed_compute_node`
example and does not cover redundant aspects of it such as networking.

Create extra Ceph key
^^^^^^^^^^^^^^^^^^^^^

Create ``/home/stack/control-plane/ceph_keys.yaml`` with contents like
the following::

  parameter_defaults:
    CephExtraKeys:
        - name: "client.external"
          caps:
            mgr: "allow *"
            mon: "profile rbd"
            osd: "profile rbd pool=vms, profile rbd pool=volumes, profile rbd pool=images"
          key: "AQD29WteAAAAABAAphgOjFD7nyjdYe8Lz0mQ5Q=="
          mode: "0600"

The key should be considered sensitive and may be randomly generated
with the following command::

  python3 -c 'import os,struct,time,base64; key = os.urandom(16); header = struct.pack("<hiih", 1, int(time.time()), 0, len(key)) ; print(base64.b64encode(header + key).decode())'

Passing `CephExtraKeys`, as above, during deployment will result in a
Ceph cluster with pools which may be accessed by the cephx user
"client.external". The same parameters will be used later when the
DCN overclouds are configured as external Ceph clusters. For more
information on the `CephExtraKeys` parameter see the document
:doc:`ceph_config` section called `Configuring CephX Keys`.

Create control-plane roles
^^^^^^^^^^^^^^^^^^^^^^^^^^

Generate the roles used for the deployment::

  openstack overcloud roles generate Controller ComputeHCI -o ~/control-plane/control_plane_roles.yaml

To determine the number of nodes per role create
``~/control-plane/roles-counts.yaml`` with the following::

  parameter_defaults:
    ControllerCount: 3
    ComputeHCICount: 3

If you do not wish to hyper-converge the compute nodes with Ceph OSD
services, then substitute `CephStorage` for `ComputeHCI` and increment
the number of `Compute` nodes. There should at least three
`Controller` nodes and at least three `CephStorage` or `ComputeHCI`
nodes in order to have a redundant Ceph cluster.

Deploy the control-plane stack
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Deploy the control-plane stack::

  openstack overcloud deploy \
         --stack control-plane \
         --templates /usr/share/openstack-tripleo-heat-templates/ \
         -r ~/control-plane/control_plane_roles.yaml \
         -n ~/network-data.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/net-multiple-nics.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/network-environment.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/disable-telemetry.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/podman.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/disable-swift.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
         -e ~/control-plane/role-counts.yaml \
         -e ~/control-plane/ceph.yaml \
         -e ~/control-plane/ceph_keys.yaml

The network related files are included to make the example complete
but are not displayed in this document. For more information on
configuring networks with distributed compute nodes see
:doc:`distributed_compute_node`.

The ``environments/ceph-ansible/ceph-ansible.yaml`` results in
ceph-ansible deploying Ceph as part of the ``control-plane`` stack.
This file also contains both `NovaEnableRbdBackend: true` and
`GlanceBackend: rbd`. When both of these settings are used, the Glance
`image_import_plugins` setting will contain `image_conversion`. With
this setting enabled commands like `glance image-create-via-import`
with `--disk-format qcow2` will result in the image being converted
into a raw format, which is optimal for the Ceph RBD driver. If
you need to disable image conversion you may override the
`GlanceImageImportPlugin` parameter. For example::

   parameter_defaults:
     GlanceImageImportPlugin: []

The ``ceph.yaml`` file contains the following which sets the name of
the Ceph cluster to "central"::

  parameter_defaults:
    CephClusterName: central

The ``ceph.yaml`` file should also contain additional parameters like
`CephAnsibleDisksConfig`, `CephPoolDefaultSize`,
`CephPoolDefaultPgNum` to configure the Ceph cluster relative to the
available hardware as described in :doc:`ceph_config`.

The ``environments/disable-swift.yaml`` file was passed to disable
Swift simply because an object storage system is not needed for this
example. However, if an object storage system is desired at the
Central site, substitute ``environments/ceph-ansible/ceph-rgw.yaml``
in its place to configure Ceph RGW.

The ``environments/cinder-backup.yaml`` file is not used in this
deployment. It's possible to enable the Cinder-backup service by using
this file but it will only write to the backups pool of the central
Ceph cluster.

The ``~/control-plane/ceph_keys.yaml`` and
``~/control-plane/role-counts.yaml`` files were created in the
previous sections.

Extract overcloud control-plane and Ceph configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Use the `openstack overcloud export` command to create
``~/control-plane-export.yaml`` as described in
:doc:`distributed_compute_node`::

  openstack overcloud export \
          --config-download-dir /var/lib/mistral/control-plane/ \
          --stack control-plane \
          --output-file ~/control-plane-export.yaml

In the above example `--config-download-dir` may be at a different
location if you deployed with a manual config-download as described in
:doc:`../deployment/ansible_config_download`.

Create ``~/central_ceph_external.yaml`` with content like the following::

  parameter_defaults:
    CephExternalMultiConfig:
      - cluster: "central"
        fsid: "3161a3b4-e5ff-42a0-9f53-860403b29a33"
        external_cluster_mon_ips: "172.16.11.84, 172.16.11.87, 172.16.11.92"
        keys:
          - name: "client.external"
            caps:
              mgr: "allow *"
              mon: "profile rbd"
              osd: "profile rbd pool=vms, profile rbd pool=volumes, profile rbd pool=images"
            key: "AQD29WteAAAAABAAphgOjFD7nyjdYe8Lz0mQ5Q=="
            mode: "0600"
        dashboard_enabled: false
        ceph_conf_overrides:
          client:
            keyring: /etc/ceph/central.client.external.keyring

The `CephExternalMultiConfig` section of the above is used to
configure any DCN node as a Ceph client of the central Ceph
cluster. All of the values, except `external_cluster_mon_ips`, for
this section may be obtained from the directory specified by
`--config-download-dir` when the `openstack overcloud export` command
was run. Based on the example provided in this document, the relevant
file with the desired values is
``/var/lib/mistral/control-plane/ceph-ansible/group_vars/all.yml``.

For example, the `fsid` and `cluster` (name) may be found like this::

  cd /var/lib/mistral/control-plane/ceph-ansible/group_vars/
  grep fsid: all.yml
  grep name: all.yml

The `keys` section should contain the same list item that was passed
to the `CephExtraKeys` parameter in the first step of this procedure.
The key value may also be obtained by looking at the key used by the
`client.external` name in the `openstack_keys` list found in
``all.yml``. For example, ``all.yml`` should contain something which
looks like the following and the second item in the list should be
used to get the value because it has `name` set to `client.external`::

  openstack_keys:
  -   caps:
        mgr: allow *
        mon: profile rbd
        osd: profile rbd pool=vms, profile rbd pool=volumes, profile rbd pool=images
      key: AQDIl2teAAAAABAAtFuRHdcS8v3+kk9Y6RzehA==
      mode: '0600'
      name: client.openstack
  -   caps:
        mgr: allow *
        mon: profile rbd
        osd: profile rbd pool=vms, profile rbd pool=volumes, profile rbd pool=images
      key: AQD29WteAAAAABAAphgOjFD7nyjdYe8Lz0mQ5Q==
      mode: '0600'
      name: client.external

To determine the value for `external_cluster_mon_ips` to provide
within the `CephExternalMultiConfig` parameter, use the inventory
generated by config-download and select the storage IP or storage
hostname of every node which runs the `CephMons` service. Based on the
example provided in this document, the relevant inventory file is
``/var/lib/mistral/control-plane/inventory.yaml``. For example the
nodes in the `Controller` role run the `CephMon` service by default
on IPs in the storage network so the IPs will be in a section which
looks like this::

  Controller:
    hosts:
      control-plane-controller-0:
        ansible_host: 192.168.24.16
        ...
        storage_hostname: control-plane-controller-0.storage.localdomain
        storage_ip: 172.16.11.84
        ...
      control-plane-controller-1:
        ansible_host: 192.168.24.22
        ...
        storage_hostname: control-plane-controller-1.storage.localdomain
        storage_ip: 172.16.11.87
        ...

The `storage_ip` from each host should be combined in a comma delimited
list. In this example the parameter is set to
`external_cluster_mon_ips: "172.16.11.84, 172.16.11.87,
172.16.11.92"`. If necessary, the inventory may be regenerated by
running the `tripleo-ansible-inventory` command as described in
:doc:`../deployment/ansible_config_download`.

The `ceph_conf_overrides` section should look like the following::

        ceph_conf_overrides:
          client:
            keyring: /etc/ceph/central.client.external.keyring

The above will result in the following lines in
``/etc/ceph/central.conf`` on all DCN nodes which interact with
the central Ceph cluster::

  [client]
  keyring = /etc/ceph/central.client.external.keyring

The name of the external Ceph cluster, relative to the DCN nodes,
is `central` so the relevant Ceph configuration file is called
``/etc/ceph/central.conf``. Optionally, the path to the key may be
confirmed by looking directly on any node running the `CephMon`
service on the control-plane stack if desired. If the conventions in
this document are followed, then it should remain consistent. This
directive is necessary so that the Glance service on all DCN nodes,
which will be deployed in the next section, knows which keyring to use
when connecting to the central Ceph cluster.

It is necessary to always pass `dashboard_enabled: false` when using
`CephExternalMultiConfig` as the Ceph dashboard cannot be deployed
when configuring an overcloud as a client of an external Ceph cluster.

For more information on the `CephExternalMultiConfig` parameter see
:doc:`ceph_external`.

Create extra Ceph key for dcn0
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Create ``~/dcn0/ceph_keys.yaml`` with content like the following::

  parameter_defaults:
    CephExtraKeys:
      - name: "client.external"
        caps:
          mgr: "allow *"
          mon: "profile rbd"
          osd: "profile rbd pool=vms, profile rbd pool=volumes, profile rbd pool=images"
        key: "AQBO/mteAAAAABAAc4mVMTpq7OFtrPlRFqN+FQ=="
        mode: "0600"

The `CephExtraKeys` section of the above should follow the same
pattern as the first step of this procedure. It should use a
new key, which should be considered sensitive and can be randomly
generated with the same Python command from the first step. This same
key will be used later when Glance on the central site needs to
connect to dcn0 "images".

Override Glance defaults for dcn0
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Create ``~/dcn0/glance.yaml`` with content like the following::

  parameter_defaults:
    GlanceShowMultipleLocations: true
    GlanceEnabledImportMethods: web-download,copy-image
    GlanceBackend: rbd
    GlanceStoreDescription: 'dcn0 rbd glance store'
    GlanceMultistoreConfig:
      central:
        GlanceBackend: rbd
        GlanceStoreDescription: 'central rbd glance store'
        CephClientUserName: 'external'
        CephClusterName: central

The `GlanceEnabledImportMethods` parameter is used to override the
default of 'web-download' to also include 'copy-image', which is
necessary to support the workflow described earlier.

By default Glance on the dcn0 node will use the RBD store of the
dcn0 Ceph cluster. The `GlanceMultistoreConfig` parameter is then used
to add an additional store of type RBD called `central` which uses
the Ceph cluster deployed by the control-plane stack so the
`CephClusterName` is set to "central". The `CephClientUserName` is set
to "external" because the additional key which was passed with
`CephExtraKeys` to the control-plane stack had a name of
"client.external".

Create DCN roles for dcn0
^^^^^^^^^^^^^^^^^^^^^^^^^

Generate the roles used for the deployment::

  openstack overcloud roles generate DistributedComputeHCI DistributedComputeHCIScaleOut -o ~/dcn0/dcn_roles.yaml

The `DistributedComputeHCI` role includes the default compute
services, the cinder volume service, and also includes the Ceph Mon,
Mgr, and OSD services for deploying a Ceph cluster at the distributed
site. Using this role, both the compute services and Ceph services are
deployed on the same nodes, enabling a hyper-converged infrastructure
for persistent storage at the distributed site. When Ceph is used,
there must be a minimum of three `DistributedComputeHCI` nodes. This
role also includes a Glance server, provided by the `GlanceApiEdge`
service with in the `DistributedComputeHCI` role. The Nova compute
service of each node in the `DistributedComputeHCI` role is configured
by default to use its local Glance server.

`DistributedComputeHCIScaleOut` role is like the `DistributedComputeHCI`
role but does not run the Ceph Mon and Mgr service. It offers the Ceph
OSD service however, so it may be used to scale up storage and compute
services at each DCN site after the minimum of three
`DistributedComputeHCI` nodes have been deployed. There is no
`GlanceApiEdge` service in the `DistributedComputeHCIScaleOut` role but
in its place the Nova compute service of the role is configured by
default to connect to a local `HaProxyEdge` service which in turn
proxies image requests to the Glance servers running on the
`DistributedComputeHCI` roles.

To determine the number of each nodes create ``~/dcn0/roles-counts.yaml``
with the following::

  parameter_defaults:
    ControllerCount: 0
    ComputeCount: 0
    DistributedComputeHCICount: 3
    DistributedComputeHCIScaleOutCount: 1

If you do not wish to hyper-converge the compute nodes with Ceph OSD
services, then substitute `DistributedCompute` for
`DistributedComputeHCI`, `DistributedComputeScaleOut` for
`DistributedComputeHCIScaleOut`, and add `CephStorage` nodes. The
`DistributedCompute` role contains the `GlanceApiEdge` service so that
the Compute service uses its the local Glance and local Ceph server at
the dcn0 site. The `DistributedComputeScaleOut` contains the
`HAproxyEdge` service so that any compute instances booting on the
`DistributedComputeScaleOut` node proxy their request for images to the
Glance services running on the `DistributedCompute` nodes. It is only
necessary to deploy the `ScaleOut` roles if more than three
`DistributedComputeHCI` or `DistributedCompute` nodes are necessary.
Unlike the `DistributedComputeHCI` role, there is no minimum number of
`DistributedCompute` required.

Deploy the dcn0 stack
^^^^^^^^^^^^^^^^^^^^^

Deploy the dcn0 stack::

    openstack overcloud deploy \
         --stack dcn0 \
         --templates /usr/share/openstack-tripleo-heat-templates/ \
         -r ~/dcn0/dcn_roles.yaml \
         -n ~/network-data.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/net-multiple-nics.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/network-environment.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/disable-telemetry.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/podman.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/dcn-hci.yaml \
         -e ~/control-plane-export.yaml \
         -e ~/central_ceph_external.yaml \
         -e ~/dcn0/dcn_ceph_keys.yaml \
         -e ~/dcn0/role-counts.yaml \
         -e ~/dcn0/ceph.yaml \
         -e ~/dcn0/az.yaml \
         -e ~/dcn0/glance.yaml

The network related files are included to make the example complete
but are not displayed in this document. For more information on
configuring networks with distributed compute nodes see
:doc:`distributed_compute_node`.

The ``environments/cinder-volume-active-active.yaml`` file is NOT used
to configure Cinder active/active on the DCN site because
``environments/dcn-hci.yaml`` contains the same parameters. The
``environments/dcn-hci.yaml`` file is also used to configure the
`GlanceApiEdge` and `HAproxyEdge` edge services. If you are not using
hyper-converged Ceph, then use ``environments/dcn.yaml`` instead.
Both ``environments/dcn-hci.yaml`` and ``environments/dcn.yaml`` use
`NovaCrossAZAttach: False` to override the Nova configuration `[cinder]`
`cross_az_attach` setting from its default of `true`. This setting
should be `false` for all nodes in the dcn0 stack so that volumes
attached to an instance must be in the same availability zone in
Cinder as the instance availability zone in Nova. This is useful when
booting an instance from a volume on DCN nodes because Nova will
attempt to create a volume using the same availability zone as what is
assigned to the instance.

The ``~/dcn0/ceph.yaml`` file contains the following which sets the
name of the ceph cluster to "dcn0"::

  parameter_defaults:
    CephClusterName: dcn0

The ``~/dcn0/ceph.yaml`` file should also contain additional
parameters like `CephAnsibleDisksConfig`, `CephPoolDefaultSize`,
`CephPoolDefaultPgNum` to configure the Ceph cluster relative to
the available hardware as described in :doc:`ceph_config`.

The ``~/dcn0/az.yaml`` file contains the following::

  parameter_defaults:
    ManageNetworks: false
    NovaComputeAvailabilityZone: dcn0
    CinderStorageAvailabilityZone: dcn0
    CinderVolumeCluster: dcn0

`CinderVolumeCluster` is the name of the Cinder active/active cluster
which is deployed per DCN site. The above setting overrides the
default of "dcn" to "dcn0" found in `environments/dcn-hci.yaml`. See
:doc:`distributed_compute_node` for details on the other parameters
above.

The ``~/control-plane-export.yaml``, ``~/dcn0/dcn_ceph_keys.yaml``,
``~/dcn0/glance.yaml``, and ``role-counts.yaml`` files were created in
the previous steps. The ``~/central_ceph_external.yaml`` file should
also have been created in a previous step. Deployment with this file
is only necessary if images on DCN sites will be pushed back to the
central site so that they may then be shared with other DCN sites.
This may be useful for sharing snapshots between sites.

Deploy additional DCN sites
^^^^^^^^^^^^^^^^^^^^^^^^^^^

All of the previous sections which were done for dcn0 may be repeated
verbatim except with "dcn1" substituted for "dcn0" and a new cephx key
should be generated for each DCN site as described under `Create extra
Ceph key`. Other than that, the same process may be continued to
deploy as many DCN sites as needed. Once all of the desired DCN sites
have been deployed proceed to the next section. The
``~/control-plane-export.yaml`` and ``~/central_ceph_external.yaml``
which were created earlier may be reused for each DCN deployment and
do not need to be recreated. The roles in the previous section were
created specifically for dcn0 to allow for variations between DCN
sites.

Update central site to use additional Ceph clusters as Glance stores
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Once all of the desired DCN sites are deployed the central site needs
to be updated so that the central Glance service may push images to
the DCN sites.

In this example only one additional DCN site, dcn1, has been deployed
as indicated by the list of undercloud Heat stacks::

  $ openstack stack list -c "Stack Name" -c "Stack Status"
  +---------------+-----------------+
  | Stack Name    | Stack Status    |
  +---------------+-----------------+
  | dcn1          | CREATE_COMPLETE |
  | dcn0          | CREATE_COMPLETE |
  | control-plane | CREATE_COMPLETE |
  +---------------+-----------------+
  $

Create ``~/control-plane/glance_update.yaml`` with content like the
following::

  parameter_defaults:
    GlanceShowMultipleLocations: true
    GlanceEnabledImportMethods: web-download,copy-image
    GlanceBackend: rbd
    GlanceStoreDescription: 'central rbd glance store'
    CephClusterName: central
    GlanceMultistoreConfig:
      dcn0:
        GlanceBackend: rbd
        GlanceStoreDescription: 'dcn0 rbd glance store'
        CephClientUserName: 'external'
        CephClusterName: dcn0
      dcn1:
        GlanceBackend: rbd
        GlanceStoreDescription: 'dcn1 rbd glance store'
        CephClientUserName: 'external'
        CephClusterName: dcn1

The above will configure the Glance service running on the Controllers
to use two additional stores called "dcn0" and "dcn1".

Create ``~/control-plane/dcn_ceph_external.yaml`` with content like the
following::

  parameter_defaults:
    CephExternalMultiConfig:
      - cluster: "dcn0"
        fsid: "539e2b96-316e-4c23-b7df-035a3037ddd1"
        external_cluster_mon_ips: "172.16.11.61, 172.16.11.64, 172.16.11.66"
        keys:
          - name: "client.external"
            caps:
              mgr: "allow *"
              mon: "profile rbd"
              osd: "profile rbd pool=vms, profile rbd pool=volumes, profile rbd pool=images"
            key: "AQBO/mteAAAAABAAc4mVMTpq7OFtrPlRFqN+FQ=="
            mode: "0600"
        dashboard_enabled: false
        ceph_conf_overrides:
          client:
            keyring: /etc/ceph/dcn0.client.external.keyring
      - cluster: "dcn1"
        fsid: "7504a91e-5a0f-4408-bb55-33c3ee2c67e9"
        external_cluster_mon_ips: "172.16.11.182, 172.16.11.185, 172.16.11.187"
        keys:
          - name: "client.external"
            caps:
              mgr: "allow *"
              mon: "profile rbd"
              osd: "profile rbd pool=vms, profile rbd pool=volumes, profile rbd pool=images"
            key: "AQACCGxeAAAAABAAHocX/cnygrVnLBrKiZHJfw=="
            mode: "0600"
        dashboard_enabled: false
        ceph_conf_overrides:
          client:
            keyring: /etc/ceph/dcn1.client.external.keyring

The `CephExternalMultiConfig` section of the above is used to
configure the Glance service at the central site as a Ceph client of
all of the Ceph clusters of the DCN sites; that is "dcn0" and "dcn1"
in this example. This will be possible because the central nodes will
have the following files created:

- /etc/ceph/dcn0.conf
- /etc/ceph/dcn0.client.external.keyring
- /etc/ceph/dcn1.conf
- /etc/ceph/dcn1.client.external.keyring

All of the values under `CephExternalMultiConfig`, except
`external_cluster_mon_ips`, for this section may be obtained from the
config-download directory as described in
:doc:`../deployment/ansible_config_download`. Based on the examples in
this document the relevant files with the desired values are
``/var/lib/mistral/dcn0/ceph-ansible/group_vars/all.yml`` and
``/var/lib/mistral/dcn1/ceph-ansible/group_vars/all.yml``.

For example, the `fsid` and `cluster` (name) for dcn0 may be found
like this::

  cd /var/lib/mistral/dcn0/ceph-ansible/group_vars/
  grep fsid: all.yml
  grep name: all.yml

The `keys` section for dcn0 should contain the same list item that was
passed to the `CephExtraKeys` parameter in an earlier step of this
procedure. The key value may also be obtained by looking at the key
used by the `client.external` name in the `openstack_keys` list found in
``all.yml``. For example, ``all.yml`` should contain something which
looks like the following and the second item in the list should be
used to get the value because it has `name` set to `client.external`::

  openstack_keys:
  -   caps:
          mgr: allow *
          mon: profile rbd
          osd: profile rbd pool=vms, profile rbd pool=volumes, profile rbd pool=images
      key: AQB7/mteAAAAABAAZzufVwFpSN4Hg2TCsR5AfA==
      mode: '0600'
      name: client.openstack
  -   caps:
          mgr: allow *
          mon: profile rbd
          osd: profile rbd pool=vms, profile rbd pool=volumes, profile rbd pool=images
      key: AQBO/mteAAAAABAAc4mVMTpq7OFtrPlRFqN+FQ==
      mode: '0600'
      name: client.external

To determine the value for `external_cluster_mon_ips` to provide
within the `CephExternalMultiConfig` parameter for dcn0, use the
inventory generated by config-download and select the storage IP or
storage hostname of every node which runs the `CephMons` service.
Based on the example provided in this document, the relevant inventory
file is ``/var/lib/mistral/dcn0/inventory.yaml``. For example
the nodes in the `DistributedComputeHCI` role run the `CephMon`
service by default on IPs in the storage network so the IPs will be in
a section which looks like this::

  DistributedComputeHCI:
    hosts:
      dcn0-distributedcomputehci-0:
        ansible_host: 192.168.24.20
        ...
        storage_hostname: dcn0-distributedcomputehci-0.storage.localdomain
        storage_ip: 172.16.11.61
        ...
      dcn0-distributedcomputehci-1:
        ansible_host: 192.168.24.25
        ...
        storage_hostname: dcn0-distributedcomputehci-1.storage.localdomain
        storage_ip: 172.16.11.64
        ...

The `storage_ip` from each host should be combined in a comma delimited
list. In this example the parameter is set to
`external_cluster_mon_ips: "172.16.11.61, 172.16.11.64, 172.16.11.66"`.
If necessary, the inventory may be regenerated by running the
`tripleo-ansible-inventory` command as described in
:doc:`../deployment/ansible_config_download`.

The `ceph_conf_overrides` section should look like the following::

        ceph_conf_overrides:
          client:
            keyring: /etc/ceph/dcn0.client.external.keyring

The above will result in the following lines in
``/etc/ceph/dcn0.conf`` on the central nodes which interact with
the dcn0 Ceph cluster::

  [client]
  keyring = /etc/ceph/dcn0.client.external.keyring

The name of the external Ceph cluster, relative to the central node,
is `dcn0` so the relevant Ceph configuration file is called
``/etc/ceph/dcn0.conf``. Optionally, the path to the key may be
confirmed by looking directly on any node running the `CephMon`
service on the dcn0 stack if desired. If the conventions in this
document are followed, then it should remain consistent. This
directive is necessary so that the Glance service on central site
knows which keyring to use when connecting to the central Ceph
cluster.

It is necessary to always pass `dashboard_enabled: false` when using
`CephExternalMultiConfig` as the Ceph dashboard cannot be deployed
when configuring an overcloud as a client of an external Ceph cluster.

The second item in the `CephExternalMultiConfig` list which starts
with `cluster: "dcn1"` may have its values determined exactly as they
were determined for `cluster: "dcn0"`, except the relevant data should
come from ``/var/lib/mistral/dcn1/ceph-ansible/group_vars/all.yml``
and ``/var/lib/mistral/dcn1/inventory.yaml``. The same pattern may be
continued for additional DCN sites which the central site wishes to
use as an additional Glance store.

For more information on the `CephExternalMultiConfig` parameter see
:doc:`ceph_external`.

The number of lines in the ``~/control-plane/glance_update.yaml`` and
``~/control-plane/glance_update.yaml`` files will be proportional to
the number of DCN sites deployed.

Run the same `openstack overcloud deploy --stack control-plane ...`
command which was run in the previous section but also include the
the ``~/control-plane/glance_update.yaml`` and
``~/control-plane/dcn_ceph_external.yaml`` files with a `-e`. When the
stack update is complete, proceed to the next section.

Confirm images may be copied between sites
------------------------------------------

Ensure you have Glance 3.0.0 or newer as provided by the
`python3-glanceclient` RPM:

.. code-block:: bash

  $ glance --version
  3.0.0

Authenticate to the control-plane using the RC file generated
by the stack from the first deployment which contains Keystone.
In this example the stack was called "control-plane" so the file
to source beofre running Glance commands will be called
"control-planerc".

Confirm the expected stores are available:

.. code-block:: bash

  $ glance stores-info
  +----------+----------------------------------------------------------------------------------+
  | Property | Value                                                                            |
  +----------+----------------------------------------------------------------------------------+
  | stores   | [{"default": "true", "id": "default_backend", "description": "central rbd glance |
  |          | store"}, {"id": "http", "read-only": "true"}, {"id": "dcn0", "description":      |
  |          | "dcn0 rbd glance store"}, {"id": "dcn1", "description": "dcn1 rbd glance         |
  |          | store"}]                                                                         |
  +----------+----------------------------------------------------------------------------------+

Create an image and import it into the default backend at central as
well as the dcn0 backend, by listing both stores with the `--stores`
option, as seen in the following example:

.. code-block:: bash

  glance --verbose image-create-via-import --disk-format qcow2 --container-format bare --name cirros --uri http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img --import-method web-download --stores default_backend,dcn0

.. note:: The example above assumes that Glance image format
          conversion is enabled. Thus, even though `--disk-format` is
          set to `qcow2`, which is the format of the image file, Glance
          will convert and store the image in raw format after it's
          uploaded because the raw format is the optimal setting for
          Ceph RBD. The conversion may be confirmed by running
          `glance image-show <ID> | grep disk_format` after the image
          is uploaded.

Set an environment variable to the ID of the newly created image:

.. code-block:: bash

  ID=$(openstack image show cirros -c id -f value)

Copy the image from the default store to the dcn1 store:

.. code-block:: bash

  glance image-import $ID --stores dcn1 --import-method copy-image

Confirm a copy of the image is in each store by looking at the image properties:

.. code-block:: bash

  $ openstack image show $ID | grep properties
  | properties       | direct_url='rbd://d25504ce-459f-432d-b6fa-79854d786f2b/images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076/snap', locations='[{u'url': u'rbd://d25504ce-459f-432d-b6fa-79854d786f2b/images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076/snap', u'metadata': {u'store': u'default_backend'}}, {u'url': u'rbd://0c10d6b5-a455-4c4d-bd53-8f2b9357c3c7/images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076/snap', u'metadata': {u'store': u'dcn0'}}, {u'url': u'rbd://8649d6c3-dcb3-4aae-8c19-8c2fe5a853ac/images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076/snap', u'metadata': {u'store': u'dcn1'}}]', os_glance_failed_import='', os_glance_importing_to_stores='', os_hash_algo='sha512', os_hash_value='b795f047a1b10ba0b7c95b43b2a481a59289dc4cf2e49845e60b194a911819d3ada03767bbba4143b44c93fd7f66c96c5a621e28dff51d1196dae64974ce240e', os_hidden='False', stores='default_backend,dcn0,dcn1' |

The `stores` key, which is the last item in the properties map is set
to 'default_backend,dcn0,dcn1'.

On further inspection the `direct_url` key is set to::

  rbd://d25504ce-459f-432d-b6fa-79854d786f2b/images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076/snap

Which contains 'd25504ce-459f-432d-b6fa-79854d786f2b', the FSID of the
central Ceph cluster, the name of the pool, 'images', followed by
'8083c7e7-32d8-4f7a-b1da-0ed7884f1076', the Glance image ID and name
of the Ceph object.

The properties map also contains `locations` which is set to similar
RBD paths for the dcn0 and dcn1 cluster with their respective FSIDs
and pool names. Note that the Glance image ID is consistent in all RBD
paths.

If the image were deleted with `glance image-delete`, then the image
would be removed from all three RBD stores to ensure consistency.
However, if the glanceclient is >3.1.0, then an image may be deleted
from a specific store only by using a syntax like `glance
stores-delete --store <store_id> <image_id>`.

Optionally, run the following on any Controller node from the
control-plane stack:

.. code-block:: bash

  sudo podman exec ceph-mon-$(hostname) rbd --cluster central -p images ls -l

Run the following on any DistributedComputeHCI node from the dcn0 stack:

.. code-block:: bash

  sudo podman exec ceph-mon-$(hostname) rbd --id external --keyring /etc/ceph/dcn0.client.external.keyring --conf /etc/ceph/dcn0.conf -p images ls -l

Run the following on any DistributedComputeHCI node from the dcn1 stack:

.. code-block:: bash

  sudo podman exec ceph-mon-$(hostname) rbd --id external --keyring /etc/ceph/dcn1.client.external.keyring --conf /etc/ceph/dcn1.conf -p images ls -l

The results in all cases should produce output like the following::

  NAME                                      SIZE   PARENT FMT PROT LOCK
  8083c7e7-32d8-4f7a-b1da-0ed7884f1076      44 MiB          2
  8083c7e7-32d8-4f7a-b1da-0ed7884f1076@snap 44 MiB          2 yes

When an ephemeral instance is COW booted from the image a similar
command in the vms pool should show the same parent image:

.. code-block:: bash

  $ sudo podman exec ceph-mon-$(hostname) rbd --id external --keyring /etc/ceph/dcn1.client.external.keyring --conf /etc/ceph/dcn1.conf -p vms ls -l
  NAME                                      SIZE  PARENT                                           FMT PROT LOCK
  2b431c77-93b8-4edf-88d9-1fd518d987c2_disk 1 GiB images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076@snap   2      excl
  $


Confirm image-based volumes may be booted as DCN instances
----------------------------------------------------------

An instance with a persistent root volume may be created on a DCN
site by using the active/active Cinder service at the DCN site.
Assuming the Glance image created in the previous step is available,
identify the image ID and pass it to `openstack volume create` with
the `--image` option to create a volume based on that image.

.. code-block:: bash

  IMG_ID=$(openstack image show cirros -c id -f value)
  openstack volume create --size 8 --availability-zone dcn0 pet-volume-dcn0 --image $IMG_ID

Once the volume is created identify its volume ID and pass it to
`openstack server create` with the `--volume` option. This example
assumes a flavor, key, security group and network have already been
created.

.. code-block:: bash

  VOL_ID=$(openstack volume show -f value -c id pet-volume-dcn0)
  openstack server create --flavor tiny --key-name dcn0-key --network dcn0-network --security-group basic --availability-zone dcn0 --volume $VOL_ID pet-server-dcn0

It is also possible to issue one command to have Nova ask Cinder
to create the volume before it boots the instance by passing the
`--image` and `--boot-from-volume` options as in the shown in the
example below:

.. code-block:: bash

  openstack server create --flavor tiny --image $IMG_ID --key-name dcn0-key --network dcn0-network --security-group basic --availability-zone dcn0 --boot-from-volume 4 pet-server-dcn0

The above will only work if the Nova `cross_az_attach` setting
of the relevant compute node is set to `false`. This is automatically
configured by deploying with `environments/dcn-hci.yaml`. If the
`cross_az_attach` setting is `true` (the default), then the volume
will be created from the image not in the dcn0 site, but on the
default central site (as verified with the `rbd` command on the
central Ceph cluster) and then the instance will fail to boot on the
dcn0 site. Even if `cross_az_attach` is `true`, it's still possible to
create an instance from a volume by using `openstack volume create`
and then `openstack server create` as shown earlier.

Optionally, after creating the volume from the image at the dcn0
site and then creating an instance from the existing volume, verify
that the volume is based on the image by running the `rbd` command
within a ceph-mon container on the dcn0 site to list the volumes pool.

.. code-block:: bash

  $ sudo podman exec ceph-mon-$HOSTNAME rbd --cluster dcn0 -p volumes ls -l
  NAME                                      SIZE  PARENT                                           FMT PROT LOCK
  volume-28c6fc32-047b-4306-ad2d-de2be02716b7 8 GiB images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076@snap   2      excl
  $

The following commands may be used to create a Cinder snapshot of the
root volume of the instance.

.. code-block:: bash

  openstack server stop pet-server-dcn0
  openstack volume snapshot create pet-volume-dcn0-snap --volume $VOL_ID --force
  openstack server start pet-server-dcn0

In the above example the server is stopped to quiesce data for clean
a snapshot. The `--force` option is necessary when creating the
snapshot because the volume status will remain "in-use" even when the
server is shut down. When the snapshot is completed start the
server. Listing the contents of the volumes pool on the dcn0 Ceph
cluster should show the snapshot which was created and how it is
connected to the original volume and original image.

.. code-block:: bash

  $ sudo podman exec ceph-mon-$HOSTNAME rbd --cluster dcn0 -p volumes ls -l
  NAME                                                                                      SIZE  PARENT                                           FMT PROT LOCK
  volume-28c6fc32-047b-4306-ad2d-de2be02716b7                                               8 GiB images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076@snap   2      excl
  volume-28c6fc32-047b-4306-ad2d-de2be02716b7@snapshot-a1ca8602-6819-45b4-a228-b4cd3e5adf60 8 GiB images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076@snap   2 yes
  $

Confirm image snapshots may be created and copied between sites
---------------------------------------------------------------

A new image called "cirros-snapshot" may be created at the dcn0 site
from the instance created in the previous section by running the
following commands.

.. code-block:: bash

  NOVA_ID=$(openstack server show pet-server-dcn0 -f value -c id)
  openstack server stop $NOVA_ID
  openstack server image create --name cirros-snapshot $NOVA_ID
  openstack server start $NOVA_ID

In the above example the instance is stopped to quiesce data for clean
a snapshot image and is then restarted after the image has been
created. The output of `openstack image show $IMAGE_ID -f value -c
properties` should contain a JSON data structure whose key called
`stores` should only contain "dcn0" as that is the only store
which has a copy of the new cirros-snapshot image.

The new image may then by copied from the dcn0 site to the central
site, which is the default backend for Glance.

.. code-block:: bash

  IMAGE_ID=$(openstack image show cirros-snapshot -f value -c id)
  glance image-import $IMAGE_ID --stores default_backend --import-method copy-image

After the above is run the output of `openstack image show
$IMAGE_ID -f value -c properties` should contain a JSON data structure
whose key called `stores` should looke like "dcn0,default_backend" as
the image will also exist in the "default_backend" which stores its
data on the central Ceph cluster. The same image at the Central site
may then be copied to other DCN sites, booted in the vms or volumes
pool, and snapshotted so that the same process may repeat.
