Distributed Multibackend Storage
================================

|project| is able to extend :doc:`distributed_compute_node` to include
distributed image management and persistent storage with the benefits
of using OpenStack and Ceph.

Features
--------

This Distributed Multibackend Storage design extends the architecture
described in :doc:`distributed_compute_node` to support the following
workflow.

- Upload an image to the Central site using `glance image-create`
  command with `--file` and `--store central` parameters.
- Move a copy of the same image to DCN sites using a command like
  `glance image-import <IMAGE-ID> --stores dcn1,dcn2 --import-method
  copy-image`.
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
overcloud as described in :doc:`deployed_ceph`. An "external" Ceph
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
  with a cephx keyring which may be used to access the central ceph
  pools.
- The dcn0 site deploys an internal Ceph cluster called dcn0 with a
  cephx keyring which may be used to access the dcn0 Ceph pools.
  During the same deployment the dcn0 site is also configured
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
as described in :doc:`deployed_ceph` and the `CephExternalMultiConfig`
parameter described in :doc:`ceph_external`.

Decide which cephx key will be used to access remote Ceph clusters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When |project| deploys Ceph it creates a cephx key called openstack and
configures Cinder, Glance, and Nova to use this key. When |project| creates
multiple Ceph clusters, as described in this document, a unique version of
this key is automatically created for each site,
e.g. central.client.openstack.keyring, dcn0.client.openstack.keyring,
and dcn1.client.openstack.keyring. Each site also needs a cephx key to
access the Ceph cluster at another site, and there are two options.

1. Each site shares a copy of its openstack cephx key with the other site.
2. Each site shares a separately created external cephx key with the other
   site, and does not share its own openstack key.

Option 1 allows certain Cinder volume operations to function correctly across
sites. For example, Cinder can back up volumes at DCN sites to the central
site, and restore volume backups to other sites. Offline volume migration can
be used to move volumes from DCN sites to the central site, and from the
central site to DCN sites. Note that online volume migration between sites,
and migrating volumes directly from one DCN site to another DCN site are not
supported.

Option 2 does not support backing up or restoring cinder volumes between the
central and DCN sites, nor does it support offline volume migration between
the sites. However, if a shared external key is ever compromised, it can be
rescinded without affecting the site's own openstack key.

Deployment Steps
----------------

This section shows the deployment commands and associated environment
files of an example DCN deployment with distributed image
management. It is based on the :doc:`distributed_compute_node`
example and does not cover redundant aspects of it such as networking.

This example assumes that the VIPs and Networks have already been
provisioned as described in :doc:`../deployment/network_v2`. We assume
that ``~/deployed-vips-control-plane.yaml`` was created by the output
of `openstack overcloud network vip provision` and that
``~/deployed-network-control-plane.yaml`` was created by the output of
`openstack overcloud network provision`.

Create a separate external Cephx key (optional)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you do not wish to distribute the default cephx key called
openstack, and instead create a cephx key used at external sites, then
follow this section. Otherwise proceed to the next section.
Some cinder volume operations only work when sites are using a common
'openstack' cephx key name. Cross-AZ backups and offline volume
migration are not supported when using a separate external cephx key.

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
:doc:`deployed_ceph` section called `Overriding CephX Keys`.

Create control-plane roles
^^^^^^^^^^^^^^^^^^^^^^^^^^

Generate the roles used for the deployment::

  openstack overcloud roles generate Controller ComputeHCI -o ~/control-plane/control_plane_roles.yaml

If you do not wish to hyper-converge the compute nodes with Ceph OSD
services, then substitute `CephStorage` and `Compute` for `ComputeHCI`.
There should at least three `Controller` nodes and at least three
`CephStorage` or `ComputeHCI` nodes in order to have a redundant Ceph
cluster.

The roles should align to hosts which are provisioned as described in
:doc:`../provisioning/baremetal_provision`. Since each site should
use a separate stack, this example assumes that ``--stack
control-plane`` was passed to the `openstack overcloud node provision`
command and that ``~/deployed-metal-control-plane.yaml`` was the
output of the same command. We also assume that the
``--network-config`` option was used to configure the network
when the hosts were provisioned.


Deploy the central Ceph cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Use the `openstack overcloud ceph deploy` command as described in
:doc:`deployed_ceph` to deploy the central Ceph cluster::

  openstack overcloud ceph deploy \
          ~/deployed-metal-control-plane.yaml \
          --output ~/control-plane/deployed-ceph-control-plane.yaml \
          --config ~/control-plane/initial-ceph.conf \
          --container-image-prepare ~/containers-prepare-parameter.yaml \
          --network-data ~/network-data.yaml \
          --roles-data ~/control-plane/control_plane_roles.yaml \
          --cluster central \
          --stack control-plane

The output of the above command,
``--output ~/control-plane/deployed-ceph-control-plane.yaml``, will be
used when deploying the overcloud in the next section.

The ``--config ~/control-plane/initial-ceph.conf`` is optional and
may be used for initial Ceph configuration. If the Ceph cluster
will be hyper-converged with compute services then create this file
like the following so Ceph will not consume memory that Nova compute
instances will need::

    $ cat <<EOF > ~/control-plane/initial-ceph.conf
    [osd]
    osd_memory_target_autotune = true
    osd_numa_auto_affinity = true
    [mgr]
    mgr/cephadm/autotune_memory_target_ratio = 0.2
    EOF
    $

The ``--container-image-prepare`` and ``--network-data`` options are
included to make the example complete but are not displayed in this
document. Both are necessary so that ``cephadm`` can download the Ceph
container from the undercloud and so that the correct storage networks
are used.

Passing ``--stack control-plane`` directs the above command to use the
working directory (e.g. ``$HOME/overcloud-deploy/<STACK>``) which was
created by `openstack overcloud node provision`. This directory
contains the Ansible inventory and is where generated files from the
Ceph deployment will be stored.

Passing ``--cluster central`` changes the name of Ceph cluster. As
multiple Ceph clusters will be deployed, each is given a separate
name. This name is inherited in the cephx key and configuration files.

After Ceph is deployed, confirm that the central admin cephx key and
Ceph configuration file have been configured on one of the
controllers::

    [root@oc0-controller-0 ~]# ls -l /etc/ceph/
    -rw-------. 1 root root  63 Mar 26 21:49 central.client.admin.keyring
    -rw-r--r--. 1 root root 177 Mar 26 21:49 central.conf
    [root@oc0-controller-0 ~]#

From one of the controller nodes confirm that the `cephadm shell`
functions when passed these files::

  cephadm shell --config /etc/ceph/central.conf \
                --keyring /etc/ceph/central.client.admin.keyring

Deploy the control-plane stack
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Deploy the control-plane stack::

  openstack overcloud deploy \
         --stack control-plane \
         --templates /usr/share/openstack-tripleo-heat-templates/ \
         -r ~/control-plane/control_plane_roles.yaml \
         -n ~/network-data.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/network-environment.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/podman.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm-rbd-only.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/cinder-backup.yaml \
         -e ~/control-plane/deployed-ceph-control-plane.yaml \
         -e ~/control-plane/ceph_keys.yaml \
         -e ~/deployed-vips-control-plane.yaml \
         -e ~/deployed-network-control-plane.yaml \
         -e ~/deployed-metal-control-plane.yaml \
         -e ~/control-plane/glance.yaml


Passing ``-e ~/control-plane/ceph_keys.yaml`` is only required if you
followed the optional section called "Create a separate external Cephx
key (optional)". If you are using the openstack keyring, then you may
pass the ``environments/cinder-backup.yaml`` to deploy the
cinder-backup service at the central site. The cinder-backup service
running in the central site will be able to back up volumes located at
DCN sites as long as all sites use the default 'openstack' cephx key
name. DCN volumes cannot be backed up to the central site if the
deployment uses a separate 'external' cephx key.

The network related files are included to make the example complete
but are not displayed in this document. For more information on
configuring networks with distributed compute nodes see
:doc:`distributed_compute_node`.

The ``environments/cephadm/cephadm-rbd-only.yaml`` results in
additional configuration of ceph for the ``control-plane`` stack. It
creates the pools for the OpenStack services being deployed and
creates the cephx keyring for the `openstack` cephx user and
distributes the keys and conf files so OpenStack can be a client of
the Ceph cluster. RGW is not deployed simply because an object storage
system is not needed for this example. However, if an object storage
system is desired at the Central site, substitute
``environments/cephadm/cephadm.yaml`` for
``environments/cephadm/cephadm-rbd-only.yaml`` and Ceph RGW will also
be configured at the central site.

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

The ``glance.yaml`` file sets the following to configue the local Glance backend::

  parameter_defaults:
    GlanceShowMultipleLocations: true
    GlanceEnabledImportMethods: web-download,copy-image
    GlanceBackend: rbd
    GlanceBackendID: central
    GlanceStoreDescription: 'central rbd glance store'

The ``environments/cinder-backup.yaml`` file is not used in this
deployment. It's possible to enable the Cinder-backup service by using
this file but it will only write to the backups pool of the central
Ceph cluster.

All files matching ``deployed-*.yaml`` should have been created in the
previous sections.

The optional ``~/control-plane/ceph_keys.yaml`` file was created in
the previous sections.

Extract overcloud control-plane and Ceph configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Once the overcloud control plane has been deployed, data needs to be
retrieved from it to pass as input values into the separate DCN
deployment.

The Heat export file is created automatically within the working
directory as described in :doc:`distributed_compute_node`. Confirm
this file was created for the control-plane as it will be used in the
next section::

  stat ~/overcloud-deploy/control-plane/control-plane-export.yaml

Use the `openstack overcloud export ceph` command to create
``~/central_ceph_external.yaml``::

  openstack overcloud export ceph \
          --stack control-plane \
          --output-file ~/central_ceph_external.yaml

By default the ``~/central_ceph_external.yaml`` file created from the
command above will contain the contents of cephx file
central.client.openstack.keyring. This document uses the convention of
calling the file "external" because it's for connecting to a Ceph
cluster (central) which is external and deployed before dcn0 which
contains is only internal and deployed during the dcn0 deployment.
If you do not wish to distribute central.client.openstack.keyring
and chose to create an external cephx keyring called "external" as
described in the optional cephx section above, then use the following
following command instead to create ``~/central_ceph_external.yaml``::

  openstack overcloud export ceph \
          --stack control-plane \
          --cephx-key-client-name external \
          --output-file ~/central_ceph_external.yaml

The ``--cephx-key-client-name external`` option passed to the
``openstack overcloud export ceph`` command results in the external
key, created during deployment and defined in
`/home/stack/control-plane/ceph_keys.yaml`, being extracted from
config-download. If the ``--cephx-key-client-name`` is not passed,
then the default cephx client key called `openstack` will be
extracted.

The genereated ``~/central_ceph_external.yaml`` should look something
like the following::

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
cluster.

The ``openstack overcloud export ceph`` command will obtain all of the
values from the config-download directory of the stack specified by
`--stack` option. All values are extracted from the
``cephadm/ceph_client.yml`` file. This file is genereated when
config-download executes the export tasks from the tripleo-ansible
role `tripleo_cephadm`. It should not be necessary to extract these
values manually as the ``openstack overcloud export ceph`` command
will genereate a valid YAML file with `CephExternalMultiConfig`
populated for all stacks passed with the `--stack` option.

The `ceph_conf_overrides` section of the file genereated by ``openstack
overcloud export ceph`` should look like the following::

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
``/etc/ceph/central.conf``. This directive is necessary so that the
Glance client called by Nova on all DCN nodes, which will be deployed
in the next section, know which keyring to use so they may connect to
the central Ceph cluster.

It is necessary to always pass `dashboard_enabled: false` when using
`CephExternalMultiConfig` as the Ceph dashboard cannot be deployed
when configuring an overcloud as a client of an external Ceph cluster.
Thus the ``openstack overcloud export ceph`` command adds this option.

For more information on the `CephExternalMultiConfig` parameter see
:doc:`ceph_external`.

Create extra Ceph key for dcn0 (optional)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you do not wish for the central site to use the openstack keyring
generated for the dcn0 site, then create ``~/dcn0/ceph_keys.yaml``
with content like the following::

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
connect to the dcn0 images pool.

Override Glance defaults for dcn0
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Create ``~/dcn0/glance.yaml`` with content like the following::

  parameter_defaults:
    GlanceShowMultipleLocations: true
    GlanceEnabledImportMethods: web-download,copy-image
    GlanceBackend: rbd
    GlangeBackendID: dcn0
    GlanceStoreDescription: 'dcn0 rbd glance store'
    GlanceMultistoreConfig:
      central:
        GlanceBackend: rbd
        GlanceStoreDescription: 'central rbd glance store'
        CephClusterName: central

In the above example the `CephClientUserName` is not set because it
uses the default of 'openstack' and thus the openstack cephx key is
used. If you choose to create and distribute separate cephx keys as
described in the optional cephx section, then add this line to this
file so that it looks like the following::

  parameter_defaults:
    GlanceShowMultipleLocations: true
    GlanceEnabledImportMethods: web-download,copy-image
    GlanceBackend: rbd
    GlanceStoreDescription: 'dcn0 rbd glance store'
    GlanceMultistoreConfig:
      central:
        GlanceBackend: rbd
        GlanceStoreDescription: 'central rbd glance store'
        CephClusterName: central
        CephClientUserName: 'external'

The `CephClientUserName` should only be set to "external" if an
additional key which was passed with `CephExtraKeys` to the
control-plane stack had a name of "client.external".

The `GlanceEnabledImportMethods` parameter is used to override the
default of 'web-download' to also include 'copy-image', which is
necessary to support the workflow described earlier.

By default Glance on the dcn0 node will use the RBD store of the
dcn0 Ceph cluster. The `GlanceMultistoreConfig` parameter is then used
to add an additional store of type RBD called `central` which uses
the Ceph cluster deployed by the control-plane stack so the
`CephClusterName` is set to "central".

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

If you do not wish to hyper-converge the compute nodes with Ceph OSD
services, then substitute `DistributedCompute` for
`DistributedComputeHCI` and `DistributedComputeScaleOut` for
`DistributedComputeHCIScaleOut`, and add `CephAll` nodes (which host
both the Mon, Mgr and OSD services).

Both the `DistributedCompute` and `DistributedComputeHCI` roles
contain `CinderVolumeEdge` and `Etcd` service for running Cinder
in active/active mode but this service will not be enabled unless
the `environments/dcn-storage.yaml` environment file is included in the
deploy command. If the `environments/dcn.yaml` is used in its place,
then the CinderVolumeEdge service will remain disabled.

The `DistributedCompute` role contains the `GlanceApiEdge` service so
that the Compute service uses its the local Glance and local Ceph
server at the dcn0 site. The `DistributedComputeScaleOut` contains the
`HAproxyEdge` service so that any compute instances booting on the
`DistributedComputeScaleOut` node proxy their request for images to the
Glance services running on the `DistributedCompute` nodes. It is only
necessary to deploy the `ScaleOut` roles if more than three
`DistributedComputeHCI` or `DistributedCompute` nodes are necessary.
Three are needed for the Cinder active/active service and if
applicable the Ceph Monitor and Manager services.

The roles should align to hosts which are deployed as described in
:doc:`../provisioning/baremetal_provision`. Since each site should
use a separate stack, this example assumes that ``--stack
dcn0`` was passed to the `openstack overcloud node provision`
command and that ``~/deployed-metal-dcn0.yaml`` was the
output of the same command. We also assume that the
``--network-config`` option was used to configure the network when the
hosts were provisioned.

Deploy the dcn0 Ceph cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Use the `openstack overcloud ceph deploy` command as described in
:doc:`deployed_ceph` to deploy the first DCN Ceph cluster::

  openstack overcloud ceph deploy \
          ~/deployed-metal-dcn0.yaml \
          --output ~/dcn0/deployed-ceph-dcn0.yaml \
          --config ~/dcn0/initial-ceph.conf \
          --container-image-prepare ~/containers-prepare-parameter.yaml \
          --network-data ~/network-data.yaml \
          --roles-data ~/dcn0/dcn_roles.yaml \
          --cluster dcn0 \
          --stack dcn0

The output of the above command,
``--output ~/dcn0/deployed-ceph-dcn0.yaml``, will be
used when deploying the overcloud in the next section.

The ``--config ~/dcn0/initial-ceph.conf`` is optional and
may be be used for initial Ceph configuration. If the Ceph cluster
will be hyper-converged with compute services then create this file
like the following so Ceph will not consume memory that Nova compute
instances will need::

    $ cat <<EOF > ~/dcn0/initial-ceph.conf
    [osd]
    osd_memory_target_autotune = true
    osd_numa_auto_affinity = true
    [mgr]
    mgr/cephadm/autotune_memory_target_ratio = 0.2
    EOF
    $

The ``--container-image-prepare`` and ``--network-data`` options are
included to make the example complete but are not displayed in this
document. Both are necessary so that ``cephadm`` can download the Ceph
container from the undercloud and so that the correct storage networks
are used.

Passing ``--stack dcn0`` directs the above command to use the
working directory (e.g. ``$HOME/overcloud-deploy/<STACK>``) which was
created by `openstack overcloud node provision`. This directory
contains the Ansible inventory and is where generated files from the
Ceph deployment will be stored.

Passing ``--cluster dcn0`` changes the name of Ceph cluster. As
multiple Ceph clusters will be deployed, each is given a separate
name. This name is inherited in the cephx key and configuration files.

After Ceph is deployed, confirm that the dcn0 admin cephx key and
Ceph configuration file have been configured in ``/etc/ceph``.
Ensure the `cephadm shell` functions when passed these files::

  cephadm shell --config /etc/ceph/dcn0.conf \
                --keyring /etc/ceph/dcn0.client.admin.keyring


Deploy the dcn0 stack
^^^^^^^^^^^^^^^^^^^^^

Deploy the dcn0 stack::

    openstack overcloud deploy \
         --stack dcn0 \
         --templates /usr/share/openstack-tripleo-heat-templates/ \
         -r ~/dcn0/dcn_roles.yaml \
         -n ~/network-data.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/network-environment.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/podman.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm-rbd-only.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/dcn-storage.yaml \
         -e ~/overcloud-deploy/control-plane/control-plane-export.yaml \
         -e ~/central_ceph_external.yaml \
         -e ~/dcn0/deployed-ceph-dcn0.yaml \
         -e ~/dcn0/dcn_ceph_keys.yaml \
         -e deployed-vips-dcn0.yaml \
         -e deployed-network-dcn0.yaml \
         -e deployed-metal-dcn0.yaml \
         -e ~/dcn0/az.yaml \
         -e ~/dcn0/glance.yaml

Passing ``-e ~/dcn0/dcn_ceph_keys.yaml`` is only required if you
followed the optional section called "Create extra Ceph key for dcn0
(optional)".

The network related files are included to make the example complete
but are not displayed in this document. For more information on
configuring networks with distributed compute nodes see
:doc:`distributed_compute_node`.

The ``environments/cinder-volume-active-active.yaml`` file is NOT used
to configure Cinder active/active on the DCN site because
``environments/dcn-storage.yaml`` contains the same parameters. The
``environments/dcn-storage.yaml`` file is also used to configure the
`GlanceApiEdge` and `HAproxyEdge` edge services. If you are not using
hyper-converged Ceph, then use ``environments/dcn.yaml`` instead.
Both ``environments/dcn-storage.yaml`` and ``environments/dcn.yaml`` use
`NovaCrossAZAttach: False` to override the Nova configuration `[cinder]`
`cross_az_attach` setting from its default of `true`. This setting
should be `false` for all nodes in the dcn0 stack so that volumes
attached to an instance must be in the same availability zone in
Cinder as the instance availability zone in Nova. This is useful when
booting an instance from a volume on DCN nodes because Nova will
attempt to create a volume using the same availability zone as what is
assigned to the instance.

The ``~/dcn0/az.yaml`` file contains the following::

  parameter_defaults:
    ManageNetworks: false
    NovaComputeAvailabilityZone: dcn0
    CinderStorageAvailabilityZone: dcn0
    CinderVolumeCluster: dcn0

`CinderVolumeCluster` is the name of the Cinder active/active cluster
which is deployed per DCN site. The above setting overrides the
default of "dcn" to "dcn0" found in `environments/dcn-storage.yaml`. See
:doc:`distributed_compute_node` for details on the other parameters
above.

The ``~/overcloud-deploy/control-plane/control-plane-export.yaml``,
``~/dcn0/dcn_ceph_keys.yaml``, ``~/dcn0/glance.yaml``, and
``role-counts.yaml`` files were created in the previous steps. The
``~/central_ceph_external.yaml`` file should also have been created in
a previous step. Deployment with this file is only necessary if images
on DCN sites will be pushed back to the central site so that they may
then be shared with other DCN sites. This may be useful for sharing
snapshots between sites.

All files matching ``deployed-*.yaml`` should have been created in the
previous sections.

Deploy additional DCN sites
^^^^^^^^^^^^^^^^^^^^^^^^^^^

All of the previous sections which were done for dcn0 may be repeated
verbatim except with "dcn1" substituted for "dcn0" and a new cephx key
should be generated for each DCN site as described under `Create extra
Ceph key`. Other than that, the same process may be continued to
deploy as many DCN sites as needed. Once all of the desired DCN sites
have been deployed proceed to the next section. The
``~/overcloud-deploy/control-plane/control-plane-export.yaml`` and ``~/central_ceph_external.yaml``
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

Create ``~/control-plane/glance-dcn-stores.yaml`` with content like the
following::

  parameter_defaults:
    GlanceMultistoreConfig:
      dcn0:
        GlanceBackend: rbd
        GlanceStoreDescription: 'dcn0 rbd glance store'
        CephClusterName: dcn0
      dcn1:
        GlanceBackend: rbd
        GlanceStoreDescription: 'dcn1 rbd glance store'
        CephClusterName: dcn1

In the above example the `CephClientUserName` is not set because it
uses the default of 'openstack' and thus the openstack cephx key is
used. If you choose to create and distribute separate cephx keys as
described in the optional cephx section, then add this line to this
file per DCN site so that it looks like the following::

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

The `CephClientUserName` should only be set to "external" if an
additional key which was passed with `CephExtraKeys` to the
DCN stacks had a name of "client.external". The above will configure
the Glance service running on the Controllers to use two additional
stores called "dcn0" and "dcn1".

Use the `openstack overcloud export ceph` command to create
``~/control-plane/dcn_ceph_external.yaml``::

  openstack overcloud export ceph \
          --stack dcn0,dcn1 \
          --output-file ~/control-plane/dcn_ceph_external.yaml

In the above example a coma-delimited list of Heat stack names is
provided to the ``--stack`` option. Pass as many stacks as necessary
for all deployed DCN sites so that the configuration data to connect
to every DCN Ceph cluster is extracted into the single genereated
``dcn_ceph_external.yaml`` file.

If you created a separate cephx key called external on each DCN ceph
cluster with ``CephExtraKeys``, then use the following variation of
the above command instead::

  openstack overcloud export ceph \
          --stack dcn0,dcn1 \
          --cephx-key-client-name external \
          --output-file ~/control-plane/dcn_ceph_external.yaml

Create ``~/control-plane/dcn_ceph_external.yaml`` should have content
like the following::

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

For more information on the `CephExternalMultiConfig` parameter see
:doc:`ceph_external`.

The number of lines in the ``~/control-plane/glance-dcn-stores.yaml`` and
``~/control-plane/dcn_ceph_external.yaml`` files will be proportional to
the number of DCN sites deployed.

Run the same `openstack overcloud deploy --stack control-plane ...`
command which was run in the previous section but also include the
the ``~/control-plane/glance-dcn-stores.yaml`` and
``~/control-plane/dcn_ceph_external.yaml`` files with a `-e`. When the
stack update is complete, proceed to the next section.

DCN using only External Ceph Clusters (optional)
------------------------------------------------

A possible variation of the deployment described above is one in which
Ceph is not deployed by director but is external to director as
described in :doc:`ceph_external`. Each site must still use a Ceph
cluster which is in the same physical location in order to address
latency requirements but that Ceph cluster does not need to be
deployed by director as in the examples above. In this configuration
Ceph services may not be hyperconverged with the Compute and
Controller nodes. The example in this section makes the following
assumptions:

- A separate Ceph cluster at the central site called central
- A separate Ceph cluster at the dcn0 site called dcn0
- A separate Ceph cluster at each dcnN site called dcnN for any other
  DCN sites

For each Ceph cluster listed above the following command has been
run::

  ceph auth add client.openstack mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rwx pool=images'

For the central site you may optionally append `, allow rwx
pool=backups, allow rwx pool=metrics` to the above command if you will
be using the Cinder backup or Telemetry services. Either way, the
above command will return a Ceph client key which should be saved in
an environment file to set the value of `CephClientKey`. The
environment file should be named something like
external-ceph-<SITE>.yaml (e.g. external-ceph-central.yaml,
external-ceph-dcn0.yaml, external-ceph-dcn1.yaml, etc.) and should
contain values like the following::

  parameter_defaults:
    # The cluster FSID
    CephClusterFSID: '4b5c8c0a-ff60-454b-a1b4-9747aa737d19'
    # The CephX user auth key
    CephClientKey: 'AQDLOh1VgEp6FRAAFzT7Zw+Y9V6JJExQAsRnRQ=='
    # The list of IPs or hostnames of the Ceph monitors
    CephExternalMonHost: '172.16.1.7, 172.16.1.8, 172.16.1.9'
    # The desired name of the generated key and conf files
    CephClusterName: central

The above will not result in creating a new Ceph cluster but in
configuring a client to connect to an existing one, though the
`CephClusterName` variable should still be set so that the
configuration files are named based on the variable's value,
e.g. /etc/ceph/central.conf. The above example might be used for
the central site but for the dcn1 site, `CephClusterName` should be
set to "dcn1". Naming the cluster after its planned availability zone
is a strategy to keep the names consistent. Whatever name is supplied
will result in the Ceph configuration file in /etc/ceph/ having that
name, e.g. /etc/ceph/central.conf, /etc/ceph/dcn0.conf,
/etc/ceph/dcn1.conf, etc. and central.client.openstack.keyring,
dcn0.client.openstack.keyring, etc. The name should be unique so as to
avoid file overwrites. If the name is not set it will default to
"ceph".

In each `openstack overcloud deploy` command in the previous sections
replace ``environments/cephadm/cephadm-rbd-only.yaml`` with
``environments/external-ceph.yaml`` and replace the
``deployed-ceph-<SITE>.yaml`` with ``external-ceph-<SITE>.yaml`` as
described above.

Thus, for a three stack deployment the following will be the case.

- The initial deployment of the cental stack is configured with one
  external Ceph cluster called central, which is the default store for
  Cinder, Glance, and Nova. We will refer to this as the central
  site's "primary external Ceph cluster".

- The initial deployment of the dcn0 stack is configured
  with its own primary external Ceph cluster called dcn0  which is the
  default store for the Cinder, Glance, and Nova services at the dcn0
  site. It is also configured with the secondary external Ceph cluster
  central.

- Each subsequent dcnN stack has its own primary external Ceph cluster
  and a secondary Ceph cluster which is central.

- After every DCN site is deployed, the central stack is updated so
  that in addition to its primary external Ceph cluster, "central", it
  has multiple secondary external Ceph clusters. This stack update
  will also configure Glance to use the additional secondary external
  Ceph clusters as additional stores.

In the example above, each site must have a primary external Ceph
cluster and each secondary external Ceph cluster is configured by
using the `CephExternalMultiConfig` parameter described in
:doc:`ceph_external`.

The `CephExternalMultiConfig` parameter must be manually configured
because the `openstack overcloud export ceph` command can only export
Ceph configuration information from clusters which it has deployed.
However, the `ceph auth add` command and `external-ceph-<SITE>.yaml`
site file described above contain all of the information necessary
to populate the `CephExternalMultiConfig` parameter.

If the external Ceph cluster at each DCN site has the default name of
"ceph", then you should still define a unique cluster name within the
`CephExternalMultiConfig` parameter like the following::

  parameter_defaults:
    CephExternalMultiConfig:
      - cluster: dcn1
        ...
      - cluster: dcn2
        ...

The above will result in dcn1.conf, dcn2.conf, etc, being created in
/etc/ceph on the control-plane nodes so that Glance is able to use
the correct Ceph configuration file per image store. If each
`cluster:` parameter above were set to "ceph", then the configuration
for each cluster would overwrite the file defined in the previous
configuration, so be sure to use a unique cluster name matching the
planned name of the availability zone.

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
  | stores   | [{"default": "true", "id": "central", "description": "central rbd glance         |
  |          | store"}, {"id": "http", "read-only": "true"}, {"id": "dcn0", "description":      |
  |          | "dcn0 rbd glance store"}, {"id": "dcn1", "description": "dcn1 rbd glance         |
  |          | store"}]                                                                         |
  +----------+----------------------------------------------------------------------------------+

Assuming an image like `cirros-0.4.0-x86_64-disk.img` is in the
current directory, convert the image from QCOW2 format to RAW format
using a command like the following:

.. code-block:: bash

  qemu-img convert -f qcow2 -O raw cirros-0.4.0-x86_64-disk.img cirros-0.4.0-x86_64-disk.raw

Create an image in Glance default store at the central site as seen
in the following example:

.. code-block:: bash

  glance image-create \
  --disk-format raw --container-format bare \
  --name cirros --file cirros-0.4.0-x86_64-disk.raw \
  --store central

Alternatively, if the image is not in the current directory but in
qcow2 format on a web server, then it may be imported and converted in
one command by running the following:

.. code-block:: bash

  glance --verbose image-create-via-import --disk-format qcow2 --container-format bare --name cirros --uri http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img --import-method web-download --stores central

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

Copy the image from the default store to the dcn0 and dcn1 stores:

.. code-block:: bash

  glance image-import $ID --stores dcn0,dcn1 --import-method copy-image

Confirm a copy of the image is in each store by looking at the image properties:

.. code-block:: bash

  $ openstack image show $ID | grep properties
  | properties       | direct_url='rbd://d25504ce-459f-432d-b6fa-79854d786f2b/images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076/snap', locations='[{u'url': u'rbd://d25504ce-459f-432d-b6fa-79854d786f2b/images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076/snap', u'metadata': {u'store': u'central'}}, {u'url': u'rbd://0c10d6b5-a455-4c4d-bd53-8f2b9357c3c7/images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076/snap', u'metadata': {u'store': u'dcn0'}}, {u'url': u'rbd://8649d6c3-dcb3-4aae-8c19-8c2fe5a853ac/images/8083c7e7-32d8-4f7a-b1da-0ed7884f1076/snap', u'metadata': {u'store': u'dcn1'}}]', os_glance_failed_import='', os_glance_importing_to_stores='', os_hash_algo='sha512', os_hash_value='b795f047a1b10ba0b7c95b43b2a481a59289dc4cf2e49845e60b194a911819d3ada03767bbba4143b44c93fd7f66c96c5a621e28dff51d1196dae64974ce240e', os_hidden='False', stores='central,dcn0,dcn1' |

The `stores` key, which is the last item in the properties map is set
to 'central,dcn0,dcn1'.

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
configured by deploying with `environments/dcn-storage.yaml`. If the
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
  glance image-import $IMAGE_ID --stores central --import-method copy-image

After the above is run the output of `openstack image show
$IMAGE_ID -f value -c properties` should contain a JSON data structure
whose key called `stores` should looke like "dcn0,central" as
the image will also exist in the "central" backend which stores its
data on the central Ceph cluster. The same image at the Central site
may then be copied to other DCN sites, booted in the vms or volumes
pool, and snapshotted so that the same process may repeat.
