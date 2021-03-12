Deploying Ceph with cephadm
===========================

TripleO can deploy and configure Ceph as if it was a composable
OpenStack service and configure OpenStack services like Nova, Glance,
Cinder, and Cinder Backup to use its RBD interface as a storage
backend as well as configure Ceph's RGW service as the backend for
OpenStack object storage. Both Ceph and OpenStack containers can also
run "hyperconverged" on the same container host.

This guide assumes that the undercloud is already installed and ready
to deploy an overcloud as described in :doc:`../deployment/index`.

Limitations
-----------

TripleO deployments of Ceph with cephadm_ are only supported in Wallaby
or newer and are currently limited to only Ceph RBD and RGW
services. Support for deployment of CephFS with Manila and Ceph
Dashboard via TripleO's cephadm integration are not yet available in
Wallaby though these services may be deployed with ceph-ansible as
described in the :doc:`ceph_config` documentation. The default version
of Ceph deployed by TripleO in Wallaby is Pacific, regardless of if
cephadm or ceph-ansible is used to deploy it.

TripleO can only deploy one Ceph cluster in the overcloud per Heat
stack. However, within that Heat stack it's possible to configure
an overcloud to communicate with multiple Ceph clusters which are
external to the overcloud. To do this, follow this document to
configure the "internal" Ceph cluster which is part of the overcloud
and also use the `CephExternalMultiConfig` parameter described in the
:doc:`ceph_external` documentation.

Prerequisite: Ensure the Ceph container is available
----------------------------------------------------

Before deploying Ceph follow the 
:ref:`prepare-environment-containers` documentation so
the appropriate Ceph container image is used.
The output of the `openstack tripleo container image prepare`
command should contain a line like the following::

  ContainerCephDaemonImage: undercloud.ctlplane.mydomain.tld:8787/ceph-ci/daemon:v6.0.0-stable-6.0-pacific-centos-8-x86_64
  
Prerequisite: Ensure the cephadm package is installed
-----------------------------------------------------

The `cephadm` package needs to be installed on at least one node in
the overcloud in order to bootstrap the first node of the Ceph
cluster.

The `cephadm` package is pre-built into the overcloud-full image.
The `tripleo_cephadm` role will also use Ansible's package module
to ensure it is present. If `tripleo-repos` is passed the `ceph`
argument for Wallaby or newer, then the CentOS SIG Ceph repository
will be enabled with the appropriate version containg the `cephadm`
package, e.g. for Wallaby the ceph-pacific repository is enabled.

Prerequisite: Ensure Disks are Clean
------------------------------------

cephadm does not reformat the OSD disks and expect them to be clean to
complete successfully. Consequently, when reusing the same nodes (or
disks) for new deployments, it is necessary to clean the disks before
every new attempt. One option is to enable the automated cleanup
functionality in Ironic, which will zap the disks every time that a
node is released. The same process can be executed manually or only
for some target nodes, see `cleaning instructions in the Ironic documentation`_.


Deploying Ceph During Overcloud Deployment
------------------------------------------

To deploy an overcloud with a Ceph RBD and RGW server include the
appropriate environment file as in the example below::

    openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml

Do not directly edit the `environments/cephadm/cephadm.yaml` file.
If you wish to override the defaults, as described below in the
sections starting with "Overriding", then place those overrides
in a separate `cephadm-overrides.yaml` file and deploy like this::

    openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml -e cephadm-overrides.yaml

Deploying with the commands above will result in the process described
in the next section.

Overview of Ceph Deployment with TripleO and cephadm
----------------------------------------------------

TripleO will use Ansible automate the process described in the
`cephadm`_ documentation to bootstrap a new cluster. It will
bootstrap a single Ceph monitor and manager on one server
(by default on the first Controller node) and then add the remaining
servers to the cluster by using Ceph orchestrator to apply a `Ceph
Service Specification`_. The Ceph Service Specification is generated
automatically based on TripleO composable roles and most of the
existing THT parameters remain backwards compatible. During stack
updates the same bootstrap process is not executed.

Details of Ceph Deployment with TripleO and cephadm
---------------------------------------------------

After the hardware is provisioned, the user `ceph-admin` is created
on the overcloud nodes. The `ceph-admin` user has one set of public
and private SSH keys created on the undercloud (in
/home/stack/.ssh/ceph-admin-id_rsa.pub and .ssh/ceph-admin-id_rsa)
which is distributed to all overcloud nodes. Unlike the
`tripleo-admin` user, this allows the `ceph-admin` user to SSH from
any overcloud node to any other overcloud node. `cephadm`_ requires
this type of access in order to scale from more than one Ceph node.

The deployment definition as described TripleO Heat Templates,
e.g. which servers run which services according to composable
roles, will be converted by the tripleo-ansible `ceph_spec_bootstrap`
module into a `Ceph Service Specification`_ file. The module has the
ability to do this based on the Ansible inventory generated by the
`tripleo-ansible-inventory` but it can also generate the Ceph Service
Specification file from a combination of a TripleO roles data file
(e.g. /usr/share/openstack-tripleo-heat-templates/roles_data.yaml)
and the output of the command
`openstack overcloud node provision --output deployed_metal.yaml`.
The default location of the generated Ceph Service Specification
file is in `config-download/<STACK>/cephadm/ceph_spec.yaml`.

After the `ceph-admin` user is created, `ceph_spec.yaml` is copied
to the bootstrap host. The bootstrap host will be the first host
in the `ceph_mons` group of the inventory generated by the
`tripleo-ansible-inventory` command. By default this is the first
controller node.

Ansible will then interact only with the bootstrap host. It will run
the `cephadm` commands necessary to bootstrap a small Ceph cluster on
the bootstrap node and then run `ceph orch apply -i ceph_spec.yaml`
and `cephadm` will use the `ceph-admin` account and SSH keys to add
the other nodes.

After the full Ceph cluster is running the Ceph pools and the cephx
keys to access the pools will be created as defined or overridden as
described in the Heat environment examples below. The information
necessary to configure Ceph clients will then be extracted to
`/home/stack/ceph_client.yml` on the undercloud and passed to the
as input to the tripleo-ansible role tripleo_ceph_client which will
then configure the rest of the overcloud to use the new Ceph cluster
as described in the :doc:`ceph_external` documentation.

When `openstack overcloud deploy` is re-run in order to update
the stack, the cephadm bootstrap process is not repeated because
that process is only run if `cephadm list` returns an empty
list. Thus, configuration changes to the running Ceph cluster
should be made directly with `Ceph Orchestrator`_.

Overriding Ceph Configuration Options
-------------------------------------

To override the keys and values of the Ceph configuration
database, which has been traditionally stored in the Ceph
configuration file, e.g. `/etc/ceph/ceph.conf`, use the
`CephConfigOverrides` parameter. For example, if the
`cephadm-overrides.yaml` file referenced in the example `openstack
overcloud deploy` command in the previous section looked like the
following::

  parameter_defaults:
    CephConfigOverrides:
      mon:
        mon_warn_on_pool_no_redundancy: false

Then the Ceph monitors would be configured with the above parameter
and a command like the following could confirm it::

  [stack@standalone ~]$ sudo cephadm shell -- ceph config dump | grep warn
  Inferring fsid 65e8d744-eaec-4ff1-97be-2551d452426d
  Inferring config /var/lib/ceph/65e8d744-eaec-4ff1-97be-2551d452426d/mon.standalone.localdomain/config
  Using recent ceph image quay.ceph.io/ceph-ci/daemon@sha256:6b3c720e58ae84b502bd929d808ba63a1e9b91f710418be9df3ee566227546c0
    mon                                       advanced  mon_warn_on_pool_no_redundancy     false
  [stack@standalone ~]$

In the above example the configuration group is 'mon' for the Ceph
monitor. The supported configuration groups are 'global', 'mon',
'mgr', 'osd', 'mds', and 'client'. If no group is provided, then the
default configuration group is 'global'.

Overriding the Ceph Service Specification
-----------------------------------------

All TripleO cephadm deployments rely on a valid `Ceph Service
Specification`_. It is not necessary to provide a service
specification directly as TripleO will generate one dynamically.
However, one may provide their own service specification by disabling
the dynmaic spec generation and providing a path to their service
specification as shown in the following::

  parameter_defaults:
    CephDynamicSpec: false
    CephSpecPath: /home/stack/cephadm_spec.yaml

The `CephDynamicSpec` parameter defaults to true. The `CephSpecPath`
defaults to "{{ playbook_dir }}/cephadm/ceph_spec.yaml", where the
value of "{{ playbook_dir }}" is controlled by config-download.
If `CephDynamicSpec` is true and `CephSpecPath` is set to a valid
path, then the spec will be created at that path before it is used to
deploy Ceph.

Overriding which disks should be OSDs
-------------------------------------

The `Advanced OSD Service Specifications`_ should be used to define
how disks are used as OSDs.

By default all available disks (excluding the disk where the operating
system is installed) are used as OSDs. This is because the
`CephOsdSpec` parameter defaults to the following::

      data_devices:
        all: true

In the above example, the `data_devices` key is valid for any `Ceph
Service Specification`_ whose `service_type` is "osd". Other OSD
service types, as found in the `Advanced OSD Service
Specifications`_, may be set by overriding the `CephOsdSpec`
paramter. In the example below all rotating devices will be data
devices and all non-rotating devices will be used as shared devices
(wal, db) following::

  parameter_defaults:
    CephOsdSpec:
      data_devices:
        rotational: 1
      db_devices:
        rotational: 0

When the dynamic Ceph service specification is built (whenever
`CephDynamicSpec` is true) whatever is in the `CephOsdSpec` will
be appended to that section of the specification if the `service_type`
is "osd".

If `CephDynamicSpec` is false, then the OSD definition can also be
placed directly in the `Ceph Service Specification`_ located at the
path defined by `CephSpecPath` as described in the previous section.

The :doc:`node_specific_hieradata` feature is not supported by the
cephadm integration but the `Advanced OSD Service Specifications`_ has
a `host_pattern` parameter which specifies which host to target for
certain `data_devices` definitions, so the equivalent functionality is
available but with the new syntax. When using this option consider
setting `CephDynamicSpec` to false and defining a custom specification
which is passed to TripleO by setting the `CephSpecPath`.

Overriding Ceph placement group values
--------------------------------------

The default cephadm deployment as triggered by TripleO has
`Autoscaling Placement Groups`_ enabled. Thus, it is not necessary to
use `pgcalc`_ and hard code a PG number per pool.

However, the interfaces described in the :doc:`ceph_config`
for configuring the placement groups per pool remain backwards
compatible. For example, to set the default pool size and default PG
number per pool use an example like the following::

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
      - {"name": volumes, "pg_num": 1024, "pgp_num": 1024, "application": rbd}
      - {"name": vms, "pg_num": 512, "pgp_num": 512, "application": rbd}
      - {"name": images, "pg_num": 128, "pgp_num": 128, "application": rbd}

Overriding CRUSH rules
----------------------

To deploy Ceph pools with custom `CRUSH Map Rules`_ use the
`CephCrushRules` parameter to define a list of named rules and
then associate the `rule_name` per pool with the `CephPools`
parameter::

  paramter_defaults:
    CephCrushRules:
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
    CephPools:
      - {'name': 'slow_pool', 'rule_name': 'HDD', 'application': 'rbd'}
      - {'name': 'fast_pool', 'rule_name': 'SSD', 'application': 'rbd'}


Overriding CephX Keys
---------------------

TripleO will create a Ceph cluster with a CephX key file for OpenStack
RBD client connections that is shared by the Nova, Cinder, and Glance
services to read and write to their pools. Not only will the keyfile
be created but the Ceph cluster will be configured to accept
connections when the key file is used. The file will be named
`ceph.client.openstack.keyring` and it will be stored in `/etc/ceph`
within the containers, but on the container host it will be stored in
a location defined by a TripleO exposed parameter which defaults to
`/var/lib/tripleo-config/ceph`.

The keyring file is created using the following defaults:

* CephClusterName: 'ceph'
* CephClientUserName: 'openstack'
* CephClientKey: This value is randomly generated per Heat stack. If
  it is overridden the recommendation is to set it to the output of
  `ceph-authtool --gen-print-key`.

If the above values are overridden, the keyring file will have a
different name and different content. E.g. if `CephClusterName` was
set to 'foo' and `CephClientUserName` was set to 'bar', then the
keyring file would be called `foo.client.bar.keyring` and it would
contain the line `[client.bar]`.

The `CephExtraKeys` parameter may be used to generate additional key
files containing other key values and should contain a list of maps
where each map describes an additional key. The syntax of each
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

Accessing the Ceph Command Line
-------------------------------

After step 2 of the overcloud deployment is completed you can login to
check the status of your Ceph cluster. By default the Ceph Monitor
containers will be running on the Controller nodes. After SSH'ing into
one of your controller nodes run `sudo cephadm shell`. An example of
what you might see is below::

  [stack@standalone ~]$ sudo cephadm shell
  Inferring fsid 65e8d744-eaec-4ff1-97be-2551d452426d
  Inferring config /var/lib/ceph/65e8d744-eaec-4ff1-97be-2551d452426d/mon.standalone.localdomain/config
  Using recent ceph image quay.ceph.io/ceph-ci/daemon@sha256:6b3c720e58ae84b502bd929d808ba63a1e9b91f710418be9df3ee566227546c0
  [ceph: root@standalone /]# ceph -s
    cluster:
      id:     65e8d744-eaec-4ff1-97be-2551d452426d
      health: HEALTH_OK

    services:
      mon: 1 daemons, quorum standalone.localdomain (age 61m)
      mgr: standalone.localdomain.saojan(active, since 61m)
      osd: 1 osds: 1 up (since 61m), 1 in (since 61m)
      rgw: 1 daemon active (1 hosts, 1 zones)

    data:
      pools:   8 pools, 201 pgs
      objects: 315 objects, 24 KiB
      usage:   19 MiB used, 4.6 GiB / 4.7 GiB avail
      pgs:     201 active+clean

  [ceph: root@standalone /]#

If you need to make updates to your Ceph deployment use the `Ceph
Orchestrator`_.


.. _`cephadm`: https://docs.ceph.com/en/latest/cephadm/index.html
.. _`cleaning instructions in the Ironic documentation`: https://docs.openstack.org/ironic/latest/admin/cleaning.html
.. _`Ceph Orchestrator`: https://docs.ceph.com/en/latest/mgr/orchestrator/
.. _`Ceph Service Specification`: https://docs.ceph.com/en/latest/cephadm/service-management/#orchestrator-cli-service-spec
.. _`Advanced OSD Service Specifications`: https://docs.ceph.com/en/latest/cephadm/osd/#drivegroups
.. _`Autoscaling Placement Groups`: https://docs.ceph.com/en/latest/rados/operations/placement-groups/
.. _`pgcalc`: http://ceph.com/pgcalc
.. _`CRUSH Map Rules`: https://docs.ceph.com/en/latest/rados/operations/crush-map-edits/?highlight=ceph%20crush%20rules#crush-map-rules
