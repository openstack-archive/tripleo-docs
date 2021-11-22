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
or newer. The default version of Ceph deployed by TripleO in Wallaby
is Pacific, regardless of if cephadm or ceph-ansible is used to deploy
it.

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
will be enabled with the appropriate version containing the `cephadm`
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

To deploy an overcloud with a Ceph include the appropriate environment
file as in the example below::

  openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml

If you only wish to deploy Ceph RBD without RGW then use the following
variation of the above::

  openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm-rbd-only.yaml

Do not directly edit the `environments/cephadm/cephadm.yaml`
or `cephadm-rbd-only.yaml` file. If you wish to override the defaults,
as described below in the sections starting with "Overriding", then
place those overrides in a separate `cephadm-overrides.yaml` file and
deploy like this::

  openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml \
    -e cephadm-overrides.yaml

Deploying with the commands above will result in the processes described
in the rest of this document.

Deploying Ceph Before Overcloud Deployment
------------------------------------------

In Wallaby and newer it is possible to provision hardware and deploy
Ceph before deploying the overcloud on the same hardware. This feature
is called "deployed ceph" and it uses the command `openstack overcloud
ceph deploy` which executes the same Ansible roles described
below. For more details see :doc:`deployed_ceph`.

Overview of Ceph Deployment with TripleO and cephadm
----------------------------------------------------

When Ceph is deployed during overcloud configuration or when Ceph is
deployed before overcloud configuration with :doc:`deployed_ceph`,
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
which is distributed to all overcloud nodes which host the Ceph
Mgr and Mon service; only the public key is distributed to nodes
in the Ceph cluster which do not run the Mgr or Mon service. Unlike
the `tripleo-admin` user, this allows the `ceph-admin` user to SSH
from any overcloud node hosting the Mon or Mgr service to any other
overcloud node hosting the Mon or Mgr service. By default these
services run on the controller nodes so this means by default that
Controllers can SSH to each other but other nodes, e.g. CephStorage
nodes, cannot SSH to Controller nodes. `cephadm`_ requires this type
of access in order to scale from more than one Ceph node.

The deployment definition as described TripleO Heat Templates,
e.g. which servers run which services according to composable
roles, will be converted by the tripleo-ansible `ceph_spec_bootstrap`_
module into a `Ceph Service Specification`_ file. The module has the
ability to do this based on the Ansible inventory generated by the
`tripleo-ansible-inventory`. When Ceph is deployed *during* overcloud
configuration by including the cephadm.yaml environment file, the
module uses the Ansible inventory to create the `Ceph Service
Specification`_. In this scenario the default location of the
generated Ceph Service Specification file is
`config-download/<STACK>/cephadm/ceph_spec.yaml`.

The same `ceph_spec_bootstrap`_ module can also generate the Ceph
Service Specification file from a combination of a TripleO roles data
file
(e.g. /usr/share/openstack-tripleo-heat-templates/roles_data.yaml)
and the output of the command
`openstack overcloud node provision --output deployed_metal.yaml`.
When Ceph is deployed *before* overcloud configuration as described in
:doc:`deployed_ceph`, the module uses the deployed_metal.yaml and
roles_data.yaml to create the `Ceph Service Specification`_.

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

After the full Ceph cluster is running, either as a result of
:doc:`deployed_ceph` or by cephadm being triggered during the
overcloud deployment via the `cephadm.yaml` environment file, the
Ceph pools and the cephx keys to access the pools will be created as
defined or overridden as described in the Heat environment examples
below. The information necessary to configure Ceph clients will then
be extracted to `/home/stack/ceph_client.yml` on the undercloud and
passed to the as input to the tripleo-ansible role tripleo_ceph_client
which will then configure the rest of the overcloud to use the new
Ceph cluster as described in the :doc:`ceph_external` documentation.

When `openstack overcloud deploy` is re-run in order to update
the stack, the cephadm bootstrap process is not repeated because
that process is only run if `cephadm list` returns an empty
list. Thus, configuration changes to the running Ceph cluster, outside
of scale up as described below, should be made directly with `Ceph
Orchestrator`_.

Overriding Ceph Configuration Options during deployment
-------------------------------------------------------

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

The above does not apply to :doc:`deployed_ceph`.

Overriding Server Configuration after deployment
------------------------------------------------

To make a Ceph *server* configuration change, after the cluster has
been deployed, use the `ceph config command`_. A '/etc/ceph/ceph.conf'
file is not distributed to all Ceph servers and instead `Ceph's
centralized configuration management`_ is used.

A single '/etc/ceph/ceph.conf' file may be found on the bootstrap node.
The directives under `CephConfigOverrides` are used to create a config
file, e.g. assimilate_ceph.conf, which is passed to `cephadm bootstrap`
with `--config assimilate_ceph.conf` so that those directives are
applied to the new cluster at bootstrap. The option `--output-config
/etc/ceph/ceph.conf` is also passed to the `cephadm bootstrap` command
and that's what creates the `ceph.conf` on the bootstrap node. The
name of the file is `ceph.conf` because the `CephClusterName`
parameter defaults to "ceph". If `CephClusterName` was set to "foo",
then the file would be called `/etc/ceph/foo.conf`.

By default the parameters in `CephConfigOverrides` are only applied to
a new Ceph server at bootstrap. They are ignored during stack updates
because `ApplyCephConfigOverridesOnUpdate` defaults to false. When
`ApplyCephConfigOverridesOnUpdate` is set to true, parameters in
`CephConfigOverrides` are put into a file, e.g. assimilate_ceph.conf,
and a command like `ceph config assimilate-conf -i
assimilate_ceph.conf` is run.

When using :doc:`deployed_ceph` the `openstack overcloud ceph deploy`
command outputs an environment file with
`ApplyCephConfigOverridesOnUpdate` set to true so that services not
covered by deployed ceph, e.g. RGW, can have the configuration changes
that they need applied during overcloud deployment. After the deployed
ceph process has run and then after the overcloud is deployed, it is
recommended to set `ApplyCephConfigOverridesOnUpdate` to false.


Overriding Client Configuration after deployment
------------------------------------------------

To make a Ceph *client* configuration change, update the parameters in
`CephConfigOverrides` and run a stack update. This will not change the
configuration for the Ceph servers unless
`ApplyCephConfigOverridesOnUpdate` is set to true (as described in the
section above). By default it should only change configurations for
the Ceph clients. Examples of Ceph clients include Nova compute
containers, Cinder volume containers, Glance image containers, etc.

The `CephConfigOverrides` directive updates all Ceph client
configuration files on the overcloud in the `CephConfigPath` (which
defaults to /var/lib/tripleo-config/ceph). The `CephConfigPath` is
mounted on the client containers as `/etc/ceph`. The name of the
configuration file is `ceph.conf` because the `CephClusterName`
parameter defaults to "ceph". If `CephClusterName` was set to "foo",
then the file would be called `/etc/ceph/foo.conf`.


Overriding the Ceph Service Specification
-----------------------------------------

All TripleO cephadm deployments rely on a valid `Ceph Service
Specification`_. It is not necessary to provide a service
specification directly as TripleO will generate one dynamically.
However, one may provide their own service specification by disabling
the dynamic spec generation and providing a path to their service
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

The `CephDynamicSpec` and `CephSpecPath` parameters are not available
when using "deployed ceph", but the functionality is available via
the `--ceph-spec` command line option as described in
:doc:`deployed_ceph`.

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
parameter. In the example below all rotating devices will be data
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

The `CephOsdSpec` parameter is not available when using "deployed
ceph", but the same functionality is available via `--osd-spec`
command line option as described in :doc:`deployed_ceph`.

Overriding Ceph Pools and Placement Group values during deployment
------------------------------------------------------------------

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


Regardless of if the :doc:`deployed_ceph` feature is used, pools will
always be created during overcloud deployment as documented above.
Additional pools may also  be created directly via the Ceph command
line tools.

Overriding CRUSH rules
----------------------

To deploy Ceph pools with custom `CRUSH Map Rules`_ use the
`CephCrushRules` parameter to define a list of named rules and
then associate the `rule_name` per pool with the `CephPools`
parameter::

  parameter_defaults:
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

Regardless of if the :doc:`deployed_ceph` feature is used, custom
CRUSH rules may be created during overcloud deployment as documented
above. CRUSH rules may also be created directly via the Ceph command
line tools.

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

Regardless of if the :doc:`deployed_ceph` feature is used, CephX keys
may be created during overcloud deployment as documented above.
Additional CephX keys may also be created directly via the Ceph
command line tools.

Enabling cephadm debug mode
---------------------------

TripleO can deploy the Ceph cluster enabling the cephadm backend in debug
mode; this is useful for troubleshooting purposes, and can be activated
by using a syntax like the following::

  parameter_defaults:
    CephAdmDebug: true

After step 2, when the Ceph cluster is up and running, after SSH'ing into
one of your controller nodes run::

    sudo cephadm shell ceph -W cephadm --watch-debug

The command above shows a more verbose cephadm execution, and it's useful
to identify potential issues with the deployment of the Ceph cluster.


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

Scenario: Deploy Ceph with TripleO and Metalsmith
-------------------------------------------------

Deploy the hardware as described in :doc:`../provisioning/baremetal_provision`
and include nodes with in the `CephStorage` role. For example, the
following could be the content of ~/overcloud_baremetal_deploy.yaml::

  - name: Controller
    count: 3
    instances:
      - hostname: controller-0
        name: controller-0
      - hostname: controller-1
        name: controller-1
      - hostname: controller-2
        name: controller-2
  - name: CephStorage
    count: 3
    instances:
      - hostname: ceph-0
        name: ceph-0
      - hostname: ceph-1
        name: ceph-2
      - hostname: ceph-2
        name: ceph-2
  - name: Compute
    count: 1
    instances:
      - hostname: compute-0
        name: compute-0

which is passed to the following command::

  openstack overcloud node provision \
    --stack overcloud \
    --output ~/overcloud-baremetal-deployed.yaml \
    ~/overcloud_baremetal_deploy.yaml

If desired at this stage, then Ceph may be deployed early as described
in :doc:`deployed_ceph`. Otherwise Ceph may be deployed during the
overcloud deployment. Either way, as described in
:doc:`../provisioning/baremetal_provision`, pass
~/overcloud_baremetal_deploy.yaml as input, along with
/usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml
and cephadm-overrides.yaml described above, to the `openstack overcloud
deploy` command.

Scenario: Scale Up Ceph with TripleO and Metalsmith
---------------------------------------------------

Modify the ~/overcloud_baremetal_deploy.yaml file described above to
add more CephStorage nodes. In the example below the number of storage
nodes is doubled::

  - name: CephStorage
    count: 6
    instances:
      - hostname: ceph-0
        name: ceph-0
      - hostname: ceph-1
        name: ceph-2
      - hostname: ceph-2
        name: ceph-2
      - hostname: ceph-3
        name: ceph-3
      - hostname: ceph-4
        name: ceph-4
      - hostname: ceph-5
        name: ceph-5

As described in :doc:`../provisioning/baremetal_provision`, re-run the
same `openstack overcloud node provision` command with the updated
~/overcloud_baremetal_deploy.yaml file. This will result in the three
new storage nodes being provisioned and output an updated copy of
~/overcloud-baremetal-deployed.yaml. The updated copy will have the
`CephStorageCount` changed from 3 to 6 and the `DeployedServerPortMap`
and `HostnameMap` will contain the new storage nodes.

After the three new storage nodes are deployed run the same
`openstack overcloud deploy` command as described in the previous
section with updated copy of ~/overcloud-baremetal-deployed.yaml.
The additional Ceph Storage nodes will be added to the Ceph and
the increased capacity will available.

In particular, the following will happen as a result of running
`openstack overcloud deploy`:

- The storage networks and firewall rules will be appropriately
  configured on the new CephStorage nodes
- The ceph-admin user will be created on the new CephStorage nodes
- The ceph-admin user's public SSH key will be distributed to the new
  CephStorage nodes so that cephadm can use SSH to add extra nodes
- If a new host with the Ceph Mon or Ceph Mgr service is being added,
  then the private SSH key will also be added to that node.
- An updated Ceph spec will be generated and installed on the
  bootstrap node, i.e. /home/ceph-admin/specs/ceph_spec.yaml on the
  bootstrap node will contain new entries for the new CephStorage
  nodes.
- The cephadm bootstrap process will be skipped because `cephadm ls`
  will indicate that Ceph containers are already running.
- The updated spec will be applied and cephadm will schedule the new
  nodes to join the cluster.

Scenario: Scale Down Ceph with TripleO and Metalsmith
-----------------------------------------------------

.. warning:: This procedure is only possible if the Ceph cluster has
             the capacity to lose OSDs.

Before using TripleO to remove hardware which is part of a Ceph
cluster, use Ceph orchestrator to deprovision the hardware gracefully.
This example uses commands from the `OSD Service Documentation for
cephadm`_ to remove the OSDs, and their host, before using TripleO
to scale down the Ceph storage nodes.

Start a Ceph shell as described in "Accessing the Ceph Command Line"
above and identify the OSDs to be removed by server. In the following
example we will identify the OSDs of the host ceph-2::

  [ceph: root@oc0-controller-0 /]# ceph osd tree
  ID  CLASS  WEIGHT   TYPE NAME            STATUS  REWEIGHT  PRI-AFF
  -1         0.58557  root default
  ... <redacted>
  -7         0.19519      host ceph-2
   5    hdd  0.04880          osd.5            up   1.00000  1.00000
   7    hdd  0.04880          osd.7            up   1.00000  1.00000
   9    hdd  0.04880          osd.9            up   1.00000  1.00000
  11    hdd  0.04880          osd.11           up   1.00000  1.00000
  ... <redacted>
  [ceph: root@oc0-controller-0 /]#

As per the example above the ceph-2 host has OSDs 5,7,9,11 which can
be removed by running `ceph orch osd rm 5 7 9 11`. For example::

  [ceph: root@oc0-controller-0 /]# ceph orch osd rm 5 7 9 11
  Scheduled OSD(s) for removal
  [ceph: root@oc0-controller-0 /]# ceph orch osd rm status
  OSD_ID  HOST        STATE     PG_COUNT  REPLACE  FORCE  DRAIN_STARTED_AT
  7       ceph-2      draining  27        False    False  2021-04-23 21:35:51.215361
  9       ceph-2      draining  8         False    False  2021-04-23 21:35:49.111500
  11      ceph-2      draining  14        False    False  2021-04-23 21:35:50.243762
  [ceph: root@oc0-controller-0 /]#

Use `ceph orch osd rm status` to check the status::

  [ceph: root@oc0-controller-0 /]# ceph orch osd rm status
  OSD_ID  HOST        STATE                    PG_COUNT  REPLACE  FORCE  DRAIN_STARTED_AT
  7       ceph-2      draining                 34        False    False  2021-04-23 21:35:51.215361
  11      ceph-2      done, waiting for purge  0         False    False  2021-04-23 21:35:50.243762
  [ceph: root@oc0-controller-0 /]#

Only proceed if `ceph orch osd rm status` returns no output.

Remove the host with `ceph orch host rm <HOST>`. For example::

  [ceph: root@oc0-controller-0 /]# ceph orch host rm ceph-2
  Removed host 'ceph-2'
  [ceph: root@oc0-controller-0 /]#

Now that the host and OSDs have been logically removed from the Ceph
cluster proceed to remove the host from the overcloud as described in
the "Scaling Down" section of :doc:`../provisioning/baremetal_provision`.

Scenario: Deploy Hyperconverged Ceph
------------------------------------

Use a command like the following to create a `roles.yaml` file
containing a standard Controller role and a ComputeHCI role::

  openstack overcloud roles generate Controller ComputeHCI -o ~/roles.yaml

The ComputeHCI role is a Compute node which also runs co-located Ceph
OSD daemons. This kind of service co-location is referred to as HCI,
or hyperconverged infrastructure. See the :doc:`composable_services`
documentation for details on roles and services.

When collocating Nova Compute and Ceph OSD services boundaries can be
set to reduce contention for CPU and Memory between the two services.
This is possible by adding parameters to `cephadm-overrides.yaml` like
the following::

  parameter_defaults:
    CephHciOsdType: hdd
    CephHciOsdCount: 4
    CephConfigOverrides:
      osd:
        osd_memory_target_autotune: true
        osd_numa_auto_affinity: true
      mgr:
        mgr/cephadm/autotune_memory_target_ratio: 0.2

The `CephHciOsdType` and `CephHciOsdCount` parameters are used by the
Derived Parameters workflow to tune the Nova scheduler to not allocate
a certain amount of memory and CPU from the hypervisor to virtual
machines so that Ceph can use them instead. See the
:doc:`derived_parameters` documentation for details. If you do not use
Derived Parameters workflow, then at least set the
`NovaReservedHostMemory` to the number of OSDs multipled by 5 GB per
OSD per host.

The `CephConfigOverrides` map passes Ceph OSD parameters to limit the
CPU and memory used by the OSDs.

The `osd_memory_target_autotune`_ is set to true so that the OSD
daemons will adjust their memory consumption based on the
`osd_memory_target` config option. The `autotune_memory_target_ratio`
defaults to 0.7. So 70% of the total RAM in the system is the starting
point, from which any memory consumed by non-autotuned Ceph daemons
are subtracted, and then the remaining memory is divided by the OSDs
(assuming all OSDs have `osd_memory_target_autotune` true). For HCI
deployments the `mgr/cephadm/autotune_memory_target_ratio` can be set
to 0.2 so that more memory is available for the Nova Compute
service. This has the same effect as setting the ceph-ansible `is_hci`
parameter to true.

A two NUMA node system can host a latency sensitive Nova workload on
one NUMA node and a Ceph OSD workload on the other NUMA node. To
configure Ceph OSDs to use a specific NUMA node (and not the one being
used by the Nova Compute workload) use either of the following Ceph
OSD configurations:

- `osd_numa_node` sets affinity to a numa node (-1 for none)
- `osd_numa_auto_affinity` automatically sets affinity to the NUMA
  node where storage and network match

If there are network interfaces on both NUMA nodes and the disk
controllers are NUMA node 0, then use a network interface on NUMA node
0 for the storage network and host the Ceph OSD workload on NUMA
node 0. Then host the Nova workload on NUMA node 1 and have it use the
network interfaces on NUMA node 1. Setting `osd_numa_auto_affinity`,
to true, as in the example `cephadm-overrides.yaml` file above, should
result in this configuration. Alternatively, the `osd_numa_node` could
be set directly to 0 and `osd_numa_auto_affinity` could be unset so
that it will default to false.

When a hyperconverged cluster backfills as a result of an OSD going
offline, the backfill process can be slowed down. In exchange for a
slower recovery, the backfill activity has less of an impact on
the collocated Compute workload. Ceph Pacific has the following
defaults to control the rate of backfill activity::

  parameter_defaults:
    CephConfigOverrides:
      osd:
        osd_recovery_op_priority: 3
        osd_max_backfills: 1
        osd_recovery_max_active_hdd: 3
        osd_recovery_max_active_ssd: 10

It is not necessary to pass the above as they are the default values,
but if these values need to be deployed with different values modify
an example like the above before deployment. If the values need to be
adjusted after the deployment use `ceph config set osd <key> <value>`.

Deploy the overcloud as described in "Scenario: Deploy Ceph with
TripleO and Metalsmith" but use the `-r` option to include generated
`roles.yaml` file and the `-e` option with the
`cephadm-overrides.yaml` file containing the HCI tunings described
above.

The examples above may be used to tune a hyperconverged system during
deployment. If the values need to be changed after deployment, then
use the `ceph orchestrator` command to set them directly.

After deployment start a Ceph shell as described in "Accessing the
Ceph Command Line" and confirm the above values were applied. For
example, to check that the NUMA and memory target auto tuning run
commands lke this::

  [ceph: root@oc0-controller-0 /]# ceph config dump | grep numa
    osd                                             advanced  osd_numa_auto_affinity                 true
  [ceph: root@oc0-controller-0 /]# ceph config dump | grep autotune
    osd                                             advanced  osd_memory_target_autotune             true
  [ceph: root@oc0-controller-0 /]# ceph config get mgr mgr/cephadm/autotune_memory_target_ratio
  0.200000
  [ceph: root@oc0-controller-0 /]#

We can then confirm that a specific OSD, e.g. osd.11, inherited those
values with commands like this::

  [ceph: root@oc0-controller-0 /]# ceph config get osd.11 osd_memory_target
  4294967296
  [ceph: root@oc0-controller-0 /]# ceph config get osd.11 osd_memory_target_autotune
  true
  [ceph: root@oc0-controller-0 /]# ceph config get osd.11 osd_numa_auto_affinity
  true
  [ceph: root@oc0-controller-0 /]#

To confirm that the default backfill values are set for the same
example OSD, use commands like this::

  [ceph: root@oc0-controller-0 /]# ceph config get osd.11 osd_recovery_op_priority
  3
  [ceph: root@oc0-controller-0 /]# ceph config get osd.11 osd_max_backfills
  1
  [ceph: root@oc0-controller-0 /]# ceph config get osd.11 osd_recovery_max_active_hdd
  3
  [ceph: root@oc0-controller-0 /]# ceph config get osd.11 osd_recovery_max_active_ssd
  10
  [ceph: root@oc0-controller-0 /]#

The above example assumes that :doc:`deployed_ceph` is not used.

Add the Ceph Dashboard to a Overcloud deployment
------------------------------------------------

During the overcloud deployment most of the Ceph daemons can be added and
configured.
To deploy the ceph dashboard include the ceph-dashboard.yaml environment
file as in the following example::

    openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/ceph-dashboard.yaml

The command above will include the ceph dashboard related services and
generates all the `cephadm` required variables to render the monitoring
stack related spec that can be applied against the deployed Ceph cluster.
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
The Ceph Dashboard backend services run on the specified `CephDashboardNetwork`
and `CephGrafanaNetwork`, while the high availability is realized by haproxy and
Pacemaker.
The Ceph Dashboard frontend is fully integrated with the tls-everywhere framework,
hence providing the tls environments files will trigger the certificate request for
both grafana and the ceph dashboard: the generated crt and key files are then
configured by cephadm, resulting in a key-value pair within the Ceph orchestrator,
which is able to mount the required files to the dashboard related containers.
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

    openstack overcloud deploy --templates -r /home/stack/roles_data.yaml -n /usr/share/openstack-tripleo-heat-templates/network_data_dashboard.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml -e ~/my-ceph-settings.yaml

.. _`cephadm`: https://docs.ceph.com/en/latest/cephadm/index.html
.. _`cleaning instructions in the Ironic documentation`: https://docs.openstack.org/ironic/latest/admin/cleaning.html
.. _`Ceph Orchestrator`: https://docs.ceph.com/en/latest/mgr/orchestrator/
.. _`ceph config command`: https://docs.ceph.com/en/latest/man/8/ceph/#config
.. _`Ceph's centralized configuration management`: https://ceph.io/community/new-mimic-centralized-configuration-management/
.. _`Ceph Service Specification`: https://docs.ceph.com/en/octopus/mgr/orchestrator/#orchestrator-cli-service-spec
.. _`ceph_spec_bootstrap`: https://docs.openstack.org/tripleo-ansible/latest/modules/modules-ceph_spec_bootstrap.html
.. _`Advanced OSD Service Specifications`: https://docs.ceph.com/en/octopus/cephadm/drivegroups/
.. _`Autoscaling Placement Groups`: https://docs.ceph.com/en/latest/rados/operations/placement-groups/
.. _`pgcalc`: http://ceph.com/pgcalc
.. _`CRUSH Map Rules`: https://docs.ceph.com/en/latest/rados/operations/crush-map-edits/?highlight=ceph%20crush%20rules#crush-map-rules
.. _`OSD Service Documentation for cephadm`: https://docs.ceph.com/en/latest/cephadm/services/osd/
.. _`osd_memory_target_autotune`: https://docs.ceph.com/en/latest/cephadm/services/osd/#automatically-tuning-osd-memory
