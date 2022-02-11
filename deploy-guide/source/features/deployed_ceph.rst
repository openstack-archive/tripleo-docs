Deployed Ceph
=============

In Wallaby and newer it is possible to provision hardware and deploy
Ceph before deploying the overcloud on the same hardware.

Deployed Ceph Workflow
----------------------

As described in the :doc:`../deployment/network_v2` the ``overcloud
deploy`` command was extended so that it can run all of the following
as separate steps:

#. Create Networks
#. Create Virtual IPs
#. Provision Baremetal Instances
#. Deploy Ceph
#. Create the overcloud Ephemeral Heat stack
#. Run Config-Download and the deploy-steps playbook

This document covers the "Deploy Ceph" step above. For details on the
other steps see :doc:`../deployment/network_v2`.

The "Provision Baremetal Instances" step outputs a YAML file
describing the deployed baremetal, for example::

  openstack overcloud node provision \
          -o ~/deployed_metal.yaml \
          ...

The deployed_metal.yaml file can be passed as input to the ``openstack
overcloud ceph deploy`` command, which in turn outputs a YAML file
describing the deployed Ceph cluster, for example::

  openstack overcloud ceph deploy \
          ~/deployed_metal.yaml \
          -o ~/deployed_ceph.yaml \
          ...

Both the deployed_metal.yaml and deployed_ceph.yaml files may then be
passed as input to the step to "Create the overcloud Ephemeral Heat
stack", for example::

  openstack overcloud deploy --templates \
          -e ~/deployed_metal.yaml \
          -e ~/deployed_ceph.yaml \
          ...

While the overcloud is being deployed the data in the
deployed_ceph.yaml file will be used to configure the OpenStack
clients to connect to the Ceph cluster as well as configure the Ceph
cluster to host OpenStack.

The above workflow is called "Deployed Ceph" because Ceph is already
deployed when the overcloud is configured.

Deployed Ceph Scope
-------------------

The "Deployed Ceph" feature deploys a Ceph cluster ready to serve RBD
by calling the same TripleO Ansible roles described in :doc:`cephadm`.
When the "Deployed Ceph" process is over you should expect to find the
following:

- The CephMon, CephMgr, and CephOSD services are running on all nodes
  which should have those services
- It's possible to SSH into a node with the CephMon service and run
  `sudo cepham shell`
- All OSDs should be running unless there were environmental issues
  (e.g. disks were not cleaned)
- A ceph configuration file and client admin keyring file in /etc/ceph
  of overcloud nodes with the CephMon service
- The Ceph cluster is ready to serve RBD

You should not expect the following after "Deployed Ceph" has run:

- No pools or cephx keys for OpenStack will be created yet
- No CephDashboard, CephRGW or CephMds services will be running yet

The above will be configured during overcloud deployment by the
`openstack overcloud deploy` command as they were prior to the
"Deployed Ceph" feature. The reasons for this are the following:

- The Dashboard and RGW services need to integrate with haproxy which
  is deployed with the overcloud
- The list of pools to create and their respective cephx keys are a
  function of which OpenStack clients (e.g. Nova, Cinder, etc) will be
  used so they must be in the overcloud definition. Thus, they are
  created during overcloud deployment

During the overcloud deployment the above resources will be
created in Ceph by the TripleO Ansible roles described in
:doc:`cephadm` using the client admin keyring file and the
``~/deployed_ceph.yaml`` file output by `openstack overcloud ceph
deploy`. Because these resources are created directly on the Ceph
cluster with admin level access, "Deployed Ceph" is different from
the "External Ceph" feature described in :doc:`ceph_external`.

The main benefits of using "Deployed Ceph" are the following:

- Use cephadm to deploy Ceph on the hardware managed by TripleO
  without having to write your own cephadm spec file (though you may
  provide your own if you wish)
- Focus on debugging the basic Ceph deployment without debugging the
  overcloud deployment at the same time
- Fix any Ceph deployment problems directly using either Ansible or
  the Ceph orchestrator tools before starting the overcloud deployment
- Have the benefits above while maintaining hyperconverged support by
  using a tested workflow

In summary, `openstack overcloud ceph deploy` deploys the Ceph cluster
while `openstack overcloud deploy` (and the commands that follow)
deploy OpenStack and configure that Ceph cluster to be used by
OpenStack.

Deployed Ceph Command Line Interface
------------------------------------

The command line interface supports the following options::

  $ openstack overcloud ceph deploy --help
  usage: openstack overcloud ceph deploy [-h] -o <deployed_ceph.yaml>
                                         [-y] [--skip-user-create]
                                         [--cephadm-ssh-user CEPHADM_SSH_USER]
                                         [--stack STACK]
                                         [--working-dir WORKING_DIR]
                                         [--roles-data ROLES_DATA]
                                         [--network-data NETWORK_DATA]
                                         [--public-network-name PUBLIC_NETWORK_NAME]
                                         [--cluster-network-name CLUSTER_NETWORK_NAME]
                                         [--config CONFIG]
                                         [--ceph-spec CEPH_SPEC | --osd-spec OSD_SPEC | --crush-hierarchy CRUSH_HIERARCHY]
                                         [--ceph-vip CEPH_SERVICES]
                                         [--container-image-prepare CONTAINER_IMAGE_PREPARE]
                                         [--container-namespace CONTAINER_NAMESPACE]
                                         [--container-image CONTAINER_IMAGE]
                                         [--container-tag CONTAINER_TAG]
                                         [--registry-url REGISTRY_URL]
                                         [--registry-username REGISTRY_USERNAME]
                                         [--registry-password REGISTRY_PASSWORD]
                                         <deployed_baremetal.yaml>

  positional arguments:
    <deployed_baremetal.yaml>
                          Path to the environment file output from "openstack
                          overcloud node provision".

  optional arguments:
    -h, --help            show this help message and exit
    -o <deployed_ceph.yaml>, --output <deployed_ceph.yaml>
                          The path to the output environment file describing the
                          Ceph deployment to pass to the overcloud deployment.
    -y, --yes             Skip yes/no prompt before overwriting an existing
                          <deployed_ceph.yaml> output file (assume yes).
    --skip-user-create    Do not create the cephadm SSH user. This user is
                          necessary to deploy but may be created in a separate
                          step via 'openstack overcloud ceph user enable'.
    --skip-hosts-config   Do not update /etc/hosts on deployed servers. By
                          default this is configured so overcloud nodes can
                          reach each other and the undercloud by name.
    --skip-container-registry-config
                          Do not update /etc/containers/registries.conf on
                          deployed servers. By default this is configured so
                          overcloud nodes can pull containers from the
                          undercloud registry.
    --cephadm-ssh-user CEPHADM_SSH_USER
                          Name of the SSH user used by cephadm. Warning: if this
                          option is used, it must be used consistently for every
                          'openstack overcloud ceph' call. Defaults to 'ceph-
                          admin'. (default=Env: CEPHADM_SSH_USER)
    --stack STACK
                          Name or ID of heat stack (default=Env:
                          OVERCLOUD_STACK_NAME)
    --working-dir WORKING_DIR
                          The working directory for the deployment where all
                          input, output, and generated files will be stored.
                          Defaults to "$HOME/overcloud-deploy/<stack>"
    --roles-data ROLES_DATA
                          Path to an alternative roles_data.yaml. Used to decide
                          which node gets which Ceph mon, mgr, or osd service
                          based on the node's role in <deployed_baremetal.yaml>.
    --network-data NETWORK_DATA
                          Path to an alternative network_data.yaml. Used to
                          define Ceph public_network and cluster_network. This
                          file is searched for networks with name_lower values
                          of storage and storage_mgmt. If none found, then
                          search repeats but with service_net_map_replace in
                          place of name_lower. Use --public-network-name or
                          --cluster-network-name options to override name of the
                          searched for network from storage or storage_mgmt to a
                          customized name. If network_data has no storage
                          networks, both default to ctlplane. If found network
                          has >1 subnet, they are all combined (for routed
                          traffic). If a network has ipv6 true, then the
                          ipv6_subnet is retrieved instead of the ip_subnet, and
                          the Ceph global ms_bind_ipv4 is set false and the
                          ms_bind_ipv6 is set true. Use --config to override
                          these defaults if desired.
    --public-network-name PUBLIC_NETWORK_NAME
                          Name of the network defined in network_data.yaml which
                          should be used for the Ceph public_network. Defaults
                          to 'storage'.
    --cluster-network-name CLUSTER_NETWORK_NAME
                          Name of the network defined in network_data.yaml which
                          should be used for the Ceph cluster_network. Defaults
                          to 'storage_mgmt'.
    --config CONFIG
                          Path to an existing ceph.conf with settings to be
                          assimilated by the new cluster via 'cephadm bootstrap
                          --config'
    --ceph-spec CEPH_SPEC
                          Path to an existing Ceph spec file. If not provided a
                          spec will be generated automatically based on --roles-
                          data and <deployed_baremetal.yaml>
    --ceph-vip CEPH_SERVICES
                          Path to an existing Ceph services/network mapping file
    --single-host-defaults
                          Adjust configuration defaults to suit a single-host
                          Ceph cluster.
    --osd-spec OSD_SPEC
                          Path to an existing OSD spec file. Mutually exclusive
                          with --ceph-spec. If the Ceph spec file is generated
                          automatically, then the OSD spec in the Ceph spec file
                          defaults to {data_devices: {all: true}} for all
                          service_type osd. Use --osd-spec to override the
                          data_devices value inside the Ceph spec file.
    --crush-hierarchy CRUSH_HIERARCHY
                          Path to an existing crush hierarchy spec file.
    --container-image-prepare CONTAINER_IMAGE_PREPARE
                          Path to an alternative
                          container_image_prepare_defaults.yaml. Used to control
                          which Ceph container is pulled by cephadm via the
                          ceph_namespace, ceph_image, and ceph_tag variables in
                          addition to registry authentication via
                          ContainerImageRegistryCredentials.

  container-image-prepare overrides:
    The following options may be used to override individual values set via
    --container-image-prepare. If the example variables below were set the
    image would be concatenated into quay.io/ceph/ceph:latest and a custom
    registry login would be used.

    --container-namespace CONTAINER_NAMESPACE
                          e.g. quay.io/ceph
    --container-image CONTAINER_IMAGE
                          e.g. ceph
    --container-tag CONTAINER_TAG
                          e.g. latest
    --registry-url REGISTRY_URL
    --registry-username REGISTRY_USERNAME
    --registry-password REGISTRY_PASSWORD

  This command is provided by the python-tripleoclient plugin.
  $

Run `openstack overcloud ceph deploy --help` in your own environment
to see the latest options which you have available.


Ceph Configuration Options
--------------------------

Any initial Ceph configuration options may be passed to a new cluster
by putting them in a standard ini-style configuration file and using
`cephadm bootstrap --config` option. The exact same option is passed
through to cephadm with `openstack overcloud ceph deploy --config`::

  $ cat <<EOF > initial-ceph.conf
  [global]
  osd crush chooseleaf type = 0
  EOF
  $ openstack overcloud ceph deploy --config initial-ceph.conf ...

The `deployed_ceph.yaml` Heat environment file output by `openstack
overcloud ceph deploy` has `ApplyCephConfigOverridesOnUpdate` set to
true. This means that services not covered by deployed ceph, e.g. RGW,
can have the configuration changes that they need applied during
overcloud deployment. After the deployed ceph process has run and
then after the overcloud is deployed, it is recommended to update the
`deployed_ceph.yaml` Heat environment file, or similar, to set
`ApplyCephConfigOverridesOnUpdate` to false. Any subsequent Ceph
configuration changes should then be made by the `ceph config
command`_. For more information on the `CephConfigOverrides` and
`ApplyCephConfigOverridesOnUpdate` parameters see :doc:`cephadm`.

It is supported to pass through the `cephadm --single-host-defaults`
option, which configures a Ceph cluster to run on a single host::

  openstack overcloud ceph deploy --single-host-defaults


Ceph Spec Options
-----------------

The roles file, described in the next section, and the output of
`openstack overcloud node provision` are passed to the
`ceph_spec_bootstrap`_ Ansible module to create a `Ceph Service
Specification`_. The `openstack overcloud ceph deploy` command does
this automatically so that a spec does usually need to be generated
separately. However, it is possible to generate a ceph spec before
deployment with the following command::

  $ openstack overcloud ceph spec --help
  usage: openstack overcloud ceph spec [-h] -o <ceph_spec.yaml> [-y]
                                       [--stack STACK]
                                       [--working-dir WORKING_DIR]
                                       [--roles-data ROLES_DATA]
                                       [--mon-ip MON_IP] [--standalone]
                                       [--osd-spec OSD_SPEC | --crush-hierarchy CRUSH_HIERARCHY]
                                       [<deployed_baremetal.yaml>]

  positional arguments:
    <deployed_baremetal.yaml>
                          Path to the environment file output from "openstack
                          overcloud node provision". This argument may be
                          excluded only if --standalone is used.

  optional arguments:
    -h, --help            show this help message and exit
    -o <ceph_spec.yaml>, --output <ceph_spec.yaml>
                          The path to the output cephadm spec file to pass to
                          the "openstack overcloud ceph deploy --ceph-spec
                          <ceph_spec.yaml>" command.
    -y, --yes             Skip yes/no prompt before overwriting an existing
                          <ceph_spec.yaml> output file (assume yes).
    --stack STACK
                          Name or ID of heat stack (default=Env:
                          OVERCLOUD_STACK_NAME)
    --working-dir WORKING_DIR
                          The working directory for the deployment where all
                          input, output, and generated files will be stored.
                          Defaults to "$HOME/overcloud-deploy/<stack>"
    --roles-data ROLES_DATA
                          Path to an alternative roles_data.yaml. Used to decide
                          which node gets which Ceph mon, mgr, or osd service
                          based on the node's role in <deployed_baremetal.yaml>.
    --mon-ip MON_IP
                          IP address of the first Ceph monitor. Only available
                          with --standalone.
    --standalone          Create a spec file for a standalone deployment. Used
                          for single server development or testing environments.
    --osd-spec OSD_SPEC
                          Path to an existing OSD spec file. When the Ceph spec
                          file is generated its OSD spec defaults to
                          {data_devices: {all: true}} for all service_type osd.
                          Use --osd-spec to override the data_devices value
                          inside the Ceph spec file.
    --crush-hierarchy CRUSH_HIERARCHY
                          Path to an existing crush hierarchy spec file.
  $

The spec file may then be edited if desired and passed directly like
this::

  openstack overcloud ceph deploy \
          deployed_metal.yaml \
          -o deployed_ceph.yaml \
          --ceph-spec ~/ceph_spec.yaml

By default the spec instructs cephadm to use all available disks
(excluding the disk where the operating system is installed) as OSDs.
The syntax it uses to do this is the following::

  data_devices:
    all: true

In the above example, the `data_devices` key is valid for any `Ceph
Service Specification`_ whose `service_type` is "osd". Other OSD
service types, as found in the `Advanced OSD Service
Specifications`_, may be set by using the ``--osd-spec`` option.

If the file ``osd_spec.yaml`` contains the following::

  data_devices:
    rotational: 1
  db_devices:
    rotational: 0

and the following command is run::

  openstack overcloud ceph deploy \
          deployed_metal.yaml \
          -o deployed_ceph.yaml \
          --osd-spec osd_spec.yaml

Then all rotating devices will be data devices and all non-rotating
devices will be used as shared devices (wal, db). This is because when
the dynamic Ceph service specification is built, whatever is in the
file referenced by ``--osd-spec`` will be appended to the section of
the specification if the `service_type` is "osd". The same
``--osd-spec`` is available to the `openstack overcloud ceph spec`
command.

Ceph VIP Options
----------------

The `--ceph-vip` option may be used to reserve a VIP for each Ceph service
specified by the 'service/network' mapping defined as input.
A generic ceph service mapping can be something like the following::

  ---
  ceph_services:
    - service: ceph_nfs
      network: storage_cloud_0
    - service: ceph_rgw
      network: storage_cloud_0

For each service added to the list above, a virtual IP on the specified
network is created to be used as `frontend_vip` of the ingress daemon.
When no subnet is specified, a default `<network>_subnet` pattern is used.
If the subnet does not follow the `<network>_subnet` pattern, a subnet for
the VIP may be specified per service::

  ---
  ceph_services:
    - service: ceph_nfs
      network: storage_cloud_0
    - service: ceph_rgw
      network: storage_cloud_0
      subnet: storage_leafX

When the `subnet` parameter is provided, it will be used by the
`tripleo_service_vip` Ansible module, otherwise the default pattern is followed.
This feature also supports the fixed_ips mode. When fixed IPs are defined, the
module is able to use that input to reserve the VIP on that network. A valid
input can be something like the following::

  ---
  fixed: true
  ceph_services:
    - service: ceph_nfs
      network: storage_cloud_0
      ip_address: 172.16.11.159
    - service: ceph_rgw
      network: storage_cloud_0
      ip_address: 172.16.11.160

When the boolean `fixed` is set to True, the subnet pattern is ignored, and
a sanity check on the user input is performed, looking for the `ip_address`
keys associated to the specified services. If the `fixed` keyword is missing,
the subnet pattern is followed. When the environment file containing the
'ceph service/network' mapping described above is created, it can be passed
to the ceph deploy command via the `--ceph-vip` option::

  openstack overcloud ceph deploy \
          deployed_metal.yaml \
          -o deployed_ceph.yaml \
          --ceph-vip ~/ceph_services.yaml


Crush Hierarchy Options
-----------------------

As described in the previous section, the `ceph_spec_bootstrap`_ Ansible
module is used to generate the Ceph related spec file which is applied
using the Ceph orchestrator tool.
During the Ceph OSDs deployment, a custom crush hierarchy can be defined
and passed using the ``--crush-hierarchy`` option.
As per `Ceph Host Management`_, by doing this the `location` attribute is
added to the Hosts spec.
The location attribute will only affect the initial CRUSH location
Subsequent changes of the location property will be ignored. Also, removing
a host will not remove any CRUSH generated bucket.


Example: Apply a custom crush hierarchy to the deployed OSDs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If the file ``crush_hierarchy.yaml`` contains something like the following::

    ---
    ceph_crush_hierarchy:
      ceph-0:
        root: default
        rack: r0
      ceph-1:
        root: default
        rack: r1
      ceph-2:
        root: default
        rack: r2

and the following command is run::

    openstack overcloud ceph deploy \
            deployed_metal.yaml \
            -o deployed_ceph.yaml \
            --osd-spec osd_spec.yaml \
            --crush-hierarchy crush_hierarchy.yaml

Then the Ceph cluster will bootstrap with the following Ceph OSD layout::


    [ceph: root@ceph-0 /]# ceph osd tree
    ID  CLASS  WEIGHT   TYPE NAME                  STATUS  REWEIGHT  PRI-AFF
    -1         0.02939  root default
    -3         0.00980      rack r0
    -2         0.00980          host ceph-node-00
     0    hdd  0.00980              osd.0              up   1.00000  1.00000
    -5         0.00980      rack r1
    -4         0.00980          host ceph-node-01
     1    hdd  0.00980              osd.1              up   1.00000  1.00000
    -7         0.00980      rack r2
    -6         0.00980          host ceph-node-02
     2    hdd  0.00980              osd.2              up   1.00000  1.00000


.. note::

    Device classes are automatically detected by Ceph, but crush rules are associated to pools
    and they still be defined using the CephCrushRules parameter during the overcloud deployment.
    Additional details can be found in the `Overriding crush rules`_ section.

Service Placement Options
-------------------------

The Ceph services defined in the roles_data.yaml file as described in
:doc:`composable_services` determine which baremetal node runs which
service. By default the Controller role has the CephMon and CephMgr
service while the CephStorage role has the CephOSD service. Most
composable services require Heat output in order to determine how
services are configured, but not the Ceph services. Thus, the
roles_data.yaml file remains authoritative for Ceph service placement
even though the "Deployed Ceph" process happens before Heat is run.

It is only necessary to use the `--roles-file` option if the default
roles_data.yaml file is not being used. For example if you intend to
deploy hyperconverged nodes, then you want the predeployed compute
nodes to be in the ceph spec with the "osd" label and for the
`service_type` "osd" to have a placement list containing a list of the
compute nodes. To do this generate a custom roles file as described in
:doc:`composable_services` like this::

  openstack overcloud roles generate Controller ComputeHCI > custom_roles.yaml

and then pass that roles file like this::

  openstack overcloud ceph deploy \
          deployed_metal.yaml \
          -o deployed_ceph.yaml \
          --roles-data custom_roles.yaml

After running the above the compute nodes should have running OSD
containers and when the overcloud is deployed Nova compute services
will then be set up on the same hosts.

If you wish to generate the ceph spec with the modified placement
described above before the ceph deployment, then the same roles
file may be passed to the 'openstack overcloud ceph spec' command::

  openstack overcloud ceph spec \
          --stack overcloud \
          --roles-data custom_roles.yaml \
          --output ceph_spec.yaml \
          deployed_metal.yaml

In the above example the `--stack` is used in order to find the
working directory containing the Ansible inventory which was created
when `openstack overcloud node provision` was run.

Network Options
---------------

The storage networks defined in the network_data.yaml file as
described in :doc:`custom_networks` determine which networks
Ceph is configured to use. When using network isolation, the
standard is for TripleO to deploy two storage networks which
map to the two Ceph networks in the following way:

* ``storage`` - Storage traffic, the Ceph ``public_network``,
  e.g. Nova compute nodes use this network for RBD traffic to the Ceph
  cluster.

* ``storage_mgmt`` - Storage management traffic (such as replication
  traffic between storage nodes), the Ceph ``cluster_network``,
  e.g. Ceph OSDs use this network to replicate data.

``openstack overcloud ceph deploy`` will use the network_data.yaml
file specified by the ``--network-data`` option to determine which
networks should be used for the ``public_network`` and
``cluster_network``. It assumes these networks are named ``storage``
and ``storage_mgmt`` in the network_data.yaml file unless a different
name should be used as indicated by the ``--public-network-name`` and
``--cluster-network-name`` options.

It is necessary to use the ``--network-data`` option when deploying
with network isolation. Otherwise the default network, i.e. the
ctlplane network on the undercloud (192.168.24.0/24), will be used for
both the ``public_network`` and ``cluster_network``.


Example: Multiple subnets with custom network names
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If network_data.yaml contains the following::

    - name: StorageMgmtCloud0
      name_lower: storage_mgmt_cloud_0
      service_net_map_replace: storage_mgmt
      subnets:
        storage_mgmt_cloud_0_subnet12:
          ip_subnet: '172.16.12.0/24'
        storage_mgmt_cloud_0_subnet13:
          ip_subnet: '172.16.13.0/24'
    - name: StorageCloud0
      name_lower: storage_cloud_0
      service_net_map_replace: storage
      subnets:
        storage_cloud_0_subnet14:
          ip_subnet: '172.16.14.0/24'
        storage_cloud_0_subnet15:
          ip_subnet: '172.16.15.0/24'

Then the Ceph cluster will bootstrap with an initial Ceph
configuration containing the following::

  [global]
  public_network = '172.16.14.0/24,172.16.15.0/24'
  cluster_network = '172.16.12.0/24,172.16.13.0/24'
  ms_bind_ipv4 = True
  ms_bind_ipv6 = False

This is because the TripleO client will see that though the
``name_lower`` value does not match ``storage`` or ``storage_mgmt``
(they match the custom names ``storage_cloud_0`` and
``storage_mgmt_cloud_0`` instead), those names do match the
``service_net_map_replace`` values. If ``service_net_map_replace``
is in the network_data.yaml, then it is not necessary to use the
``--public-network-name`` and ``--cluster-network-name``
options. Alternatively the ``service_net_map_replace`` key could have
been left out and the ``--public-network-name`` and
``--cluster-network-name`` options could have been used instead. Also,
because multiple subnets are used they are concatenated and it is
assumed that there is routing between the subnets. If there was no
``subnets`` key, in the network_data.yaml file, then the client would
have looked instead for the single ``ip_subnet`` key for each network.

By default the Ceph globals `ms_bind_ipv4` is set `true` and
`ms_bind_ipv6` is set `false`.

Example: IPv6
^^^^^^^^^^^^^

If network_data.yaml contains the following::

  - name: Storage
    ipv6: true
    ipv6_subnet: fd00:fd00:fd00:3000::/64
    name_lower: storage
  - name: StorageMgmt
    ipv6: true
    ipv6_subnet: fd00:fd00:fd00:4000::/64
    name_lower: storage_mgmt

Then the Ceph cluster will bootstrap with an initial Ceph
configuration containing the following::

  [global]
  public_network = fd00:fd00:fd00:3000::/64
  cluster_network = fd00:fd00:fd00:4000::/64
  ms_bind_ipv4 = False
  ms_bind_ipv6 = True

Because the storage networks in network_data.yaml contain `ipv6:
true`, the ipv6_subet values are extracted and the Ceph globals
`ms_bind_ipv4` is set `false` and `ms_bind_ipv6` is set `true`.
It is not supported to have the ``public_network`` use IPv4 and
the ``cluster_network`` use IPv6 or vice versa.

Example: Directly setting network and ms_bind options
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If the above defaults are not desired, then it's possible to create
an initial-ceph.conf with the ``public_network``, ``cluster_network``,
``ms_bind_ipv4``, and ``ms_bind_ipv6`` options set to whatever values
are desired and pass it as in the example below::

  $ cat <<EOF > initial-ceph.conf
  [global]
  public_network = 'fd00:fd00:fd00:3000::/64,172.16.14.0/24'
  cluster_network = 'fd00:fd00:fd00:4000::/64,172.16.12.0/24'
  ms_bind_ipv4 = true
  ms_bind_ipv6 = true
  EOF
  $ openstack overcloud ceph deploy \
    --config initial-ceph.conf --network-data network_data.yaml

The above assumes that network_data.yaml contains the following::

  - name: Storage
    ipv6_subnet: fd00:fd00:fd00:3000::/64
    ip_subnet: 172.16.14.0/24
    name_lower: storage
  - name: StorageMgmt
    ipv6_subnet: fd00:fd00:fd00:4000::/64
    ip_subnet: 172.16.12.0/24
    name_lower: storage_mgmt

The above settings are experimental and untested.

When using the ``--config`` option as above it is still
important to ensure the TripleO ``storage`` and ``storage_mgmt``
network names map to the correct ``public_network`` and
``cluster_network`` since that is how the initial boot address is
determined.

The ``--config`` option ignores automatically generated initial
configuration options, i.e. the ``public_network``,
``cluster_network``, and Ceph ms_bind options which would normally
be retrieved from network_data.yaml. These variables all need to be
set explicitly in the file passed as an argument to ``--config``.

SSH User Options
----------------

Cephadm must use SSH to connect to all remote Ceph hosts that it
manages. The "Deployed Ceph" feature creates an account and SSH key
pair on all Ceph nodes in the overcloud and passes this information
to cephadm so that it uses this account instead of creating its own.
The `openstack overcloud ceph deploy` command will automatically
create this user and distribute their SSH keys. It's also possible
to create this user and distribute the associated keys in a separate
step by running `openstack overcloud ceph user enable` and then when
calling `openstack overcloud ceph deploy` with the
`--skip-user-create` option. By default the user is called
`ceph-admin` though both commands support the `--cephadm-ssh-user`
option to set a different name. If this option is used though, it must
be used consistently for every `openstack overcloud ceph` call.

The `openstack overcloud ceph user disable --fsid <FSID>` command
may be run after `openstack overcloud ceph deploy` has been run
to disable cephadm so that it may not be used to administer the
Ceph cluster and no `ceph orch ...` CLI commands will function.
This will also prevent Ceph node overcloud scale operations though
the Ceph cluster will still be able to read and write data. This same
command will also remove the public and private SSH keys of the
cephadm SSH user on overclouds which host Ceph. The "ceph user enable"
option may then be used to re-distribute the public and private SSH
keys of the cephadm SSH user and re-enable the cephadm mgr module.
`openstack overcloud ceph user enable` will only re-enable the cephadm
mgr module if it is passed the FSID with the `--fsid <FSID>` option.
The FSID may be found in the deployed_ceph.yaml Heat environment file
which is generated by the `openstack overcloud ceph deploy -o
deployed_ceph.yaml` command.

.. warning::
   Disabling cephadm will disable all Ceph management features
   described in this document. The `openstack overcloud ceph user
   disable` command is not recommended unless you have a good reason
   to disable cephadm.

Both the `openstack overcloud ceph user enable` and `openstack
overcloud ceph user disable` commands require the path to an existing
Ceph spec file to be passed as an argument. This is necessary in order
to determine which hosts require the cephadm SSH user and which of
those hosts require the private SSH key. Only hosts with the _admin
label get the private SSH since they need to be able to SSH into other
Ceph hosts. In the average deployment with three monitor nodes this is
three hosts. All other Ceph hosts only get the public key added to the
users authorized_keys file.

See the "Ceph Spec Options" options of this document for where to find
this file or how to automatically generate one before Ceph deployment
if you plan to call `openstack overcloud ceph user enable` before
calling `openstack overcloud ceph deploy`. See `openstack overcloud
ceph user enable --help` and `openstack overcloud ceph user disable
--help` for more information.

Container Options
-----------------

As described in :doc:`../deployment/container_image_prepare` the
undercloud may be used as a container registry for ceph containers
and there is a supported syntax to download containers from
authenticated registries.

By default `openstack overcloud ceph deploy` will pull the Ceph
container in the default ``container_image_prepare_defaults.yaml``
file. If a `push_destination` is defined in this file, then the
overcloud will be configured so it can access the local registry in
order to download the Ceph container. This means that `openstack
overcloud ceph deploy` will modify the overcloud's ``/etc/hosts``
and ``/etc/containers/registries.conf`` files; unless the
`--skip-hosts-config` and `--skip-container-registry-config` options
are used or a `push_destination` is not defined.

The version of the Ceph used in each OpenStack release changes per
release and can be seen by running a command like this::

  egrep "ceph_namespace|ceph_image|ceph_tag" \
    /usr/share/tripleo-common/container-images/container_image_prepare_defaults.yaml

The `--container-image-prepare` option can be used to override which
``container_image_prepare_defaults.yaml`` file is used. If a version
of this file called ``custom_container_image_prepare.yaml`` is
modified to contain syntax like the following::

  ContainerImageRegistryCredentials:
    quay.io/ceph-ci:
      quay_username: quay_password

Then when a command like the following is run::

  openstack overcloud ceph deploy \
          deployed_metal.yaml \
          -o deployed_ceph.yaml \
          --container-image-prepare custom_container_image_prepare.yaml

The credentials will be extracted from the file and the tripleo
ansible role to bootstrap Ceph will be executed like this::

  cephadm bootstrap
   --registry-url quay.io/ceph-ci
   --registry-username quay_username
   --registry-password quay_password
   ...

The syntax of the container image prepare file can also be ignored and
instead the following command line options may be used instead::

  --container-namespace CONTAINER_NAMESPACE
                        e.g. quay.io/ceph
  --container-image CONTAINER_IMAGE
                        e.g. ceph
  --container-tag CONTAINER_TAG
                        e.g. latest
  --registry-url REGISTRY_URL
  --registry-username REGISTRY_USERNAME
  --registry-password REGISTRY_PASSWORD

If a variable above is unused, then it defaults to the ones found in
the default ``container_image_prepare_defaults.yaml`` file. In other
words, the above options are overrides.

.. _`ceph config command`: https://docs.ceph.com/en/latest/man/8/ceph/#config
.. _`ceph_spec_bootstrap`: https://docs.openstack.org/tripleo-ansible/latest/modules/modules-ceph_spec_bootstrap.html
.. _`Ceph Service Specification`: https://docs.ceph.com/en/octopus/mgr/orchestrator/#orchestrator-cli-service-spec
.. _`Advanced OSD Service Specifications`: https://docs.ceph.com/en/octopus/cephadm/drivegroups/
.. _`Ceph Host Management`: https://docs.ceph.com/en/latest/cephadm/host-management/#setting-the-initial-crush-location-of-host
.. _`Overriding crush rules`: https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/features/cephadm.html#overriding-crush-rules
