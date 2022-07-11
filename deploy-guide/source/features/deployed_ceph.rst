Deploying Ceph with TripleO
===========================

In Wallaby and newer it is possible to have TripleO provision hardware
and deploy Ceph before deploying the overcloud on the same hardware.

Deployed Ceph Workflow
----------------------

As described in the :doc:`../deployment/network_v2` the ``openstack
overcloud`` command was extended so that it can run all of the
following as separate steps:

#. Create Networks
#. Create Virtual IPs
#. Provision Baremetal Instances
#. Deploy Ceph
#. Create the overcloud Ephemeral Heat stack
#. Run Config-Download and the deploy-steps playbook

This document covers the "Deploy Ceph" step above. It also covers how
to configure the overcloud deployed in the subsequent steps to use the
Ceph cluster. For details on the earlier steps see
:doc:`../deployment/network_v2`.

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
and CephFS by calling TripleO Ansible roles which execute the
`cephadm` command. When the "Deployed Ceph" process is over you should
expect to find the following:

- The CephMon, CephMgr and CephOSD services are running on all nodes
  which should have those services as defined by the
  :doc:`composable_services` interface
- If desired, the CephMds and CephNFS service will also be deployed
  and running (this feature is not available in Wallaby however).
- It's possible to SSH into a node with the CephMon service and run
  `sudo cephadm shell`
- All OSDs should be running unless there were environmental issues
  (e.g. disks were not cleaned)
- A ceph configuration file and client admin keyring file in /etc/ceph
  of overcloud nodes with the CephMon service
- The Ceph cluster is ready to serve RBD

You should not expect the following after "Deployed Ceph" has run:

- No pools or cephx keys for OpenStack will be created yet
- No CephDashboard or CephRGW services will be running yet

The above will be configured during overcloud deployment by the
`openstack overcloud deploy` command as they were prior to the
"Deployed Ceph" feature. The reasons for this are the following:

- The Dashboard and RGW services need to integrate with haproxy which
  is deployed with the overcloud
- The list of pools to create and their respective cephx keys are a
  function of which OpenStack clients (e.g. Nova, Cinder, etc) will be
  used so they must be in the overcloud definition. Thus, they are
  created during overcloud deployment

During the overcloud deployment, the above resources will be created
in Ceph using the client admin keyring file and the
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

Multiple Ceph clusters per deployment
-------------------------------------

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

See "Container Options" options below for more details.

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


Deployed Ceph Command Line Interface
------------------------------------

The command line interface supports the following options::

  $ openstack overcloud ceph deploy --help
  usage: openstack overcloud ceph deploy [-h] -o <deployed_ceph.yaml> [-y]
                                         [--skip-user-create]
                                         [--skip-hosts-config]
                                         [--skip-container-registry-config]
                                         [--cephadm-ssh-user CEPHADM_SSH_USER]
                                         [--stack STACK]
                                         [--working-dir WORKING_DIR]
                                         [--roles-data ROLES_DATA]
                                         [--network-data NETWORK_DATA]
                                         [--public-network-name PUBLIC_NETWORK_NAME]
                                         [--cluster-network-name CLUSTER_NETWORK_NAME]
                                         [--cluster CLUSTER] [--mon-ip MON_IP]
                                         [--config CONFIG]
                                         [--cephadm-extra-args CEPHADM_EXTRA_ARGS]
                                         [--force]
                                         [--ansible-extra-vars ANSIBLE_EXTRA_VARS]
                                         [--ceph-client-username CEPH_CLIENT_USERNAME]
                                         [--ceph-client-key CEPH_CLIENT_KEY]
                                         [--skip-cephx-keys]
                                         [--ceph-vip CEPH_VIP]
                                         [--daemons DAEMONS]
                                         [--single-host-defaults]
                                         [--ceph-spec CEPH_SPEC | --osd-spec OSD_SPEC]
                                         [--crush-hierarchy CRUSH_HIERARCHY]
                                         [--standalone]
                                         [--container-image-prepare CONTAINER_IMAGE_PREPARE]
                                         [--cephadm-default-container]
                                         [--container-namespace CONTAINER_NAMESPACE]
                                         [--container-image CONTAINER_IMAGE]
                                         [--container-tag CONTAINER_TAG]
                                         [--registry-url REGISTRY_URL]
                                         [--registry-username REGISTRY_USERNAME]
                                         [--registry-password REGISTRY_PASSWORD]
                                         [<deployed_baremetal.yaml>]

  positional arguments:
    <deployed_baremetal.yaml>
                          Path to the environment file output from "openstack
                          overcloud node provision". This argument may be
                          excluded only if --ceph-spec is used.

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
    --stack STACK         Name or ID of heat stack (default=Env:
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
    --cluster CLUSTER     Name of the Ceph cluster. If set to 'foo', then the
                          files /etc/ceph/<FSID>/foo.conf and
                          /etc/ceph/<FSID>/foo.client.admin.keyring will be
                          created. Otherwise these files will use the name
                          'ceph'. Changing this means changing command line
                          calls too, e.g. 'ceph health' will become 'ceph
                          --cluster foo health' unless export CEPH_ARGS='--
                          cluster foo' is used.
    --mon-ip MON_IP       IP address of the first Ceph monitor. If not set, an
                          IP from the Ceph public_network of a server with the
                          mon label from the Ceph spec is used. IP must already
                          be active on server.
    --config CONFIG       Path to an existing ceph.conf with settings to be
                          assimilated by the new cluster via 'cephadm bootstrap
                          --config'
    --cephadm-extra-args CEPHADM_EXTRA_ARGS
                          String of extra parameters to pass cephadm. E.g. if
                          --cephadm-extra-args '--log-to-file --skip-prepare-
                          host', then cephadm boostrap will use those options.
                          Warning: requires --force as not all possible options
                          ensure a functional deployment.
    --force               Run command regardless of consequences.
    --ansible-extra-vars ANSIBLE_EXTRA_VARS
                          Path to an existing Ansible vars file which can
                          override any variable in tripleo-ansible. If '--
                          ansible-extra-vars vars.yaml' is passed, then
                          'ansible-playbook -e @vars.yaml ...' is used to call
                          tripleo-ansible Ceph roles. Warning: requires --force
                          as not all possible options ensure a functional
                          deployment.
    --ceph-client-username CEPH_CLIENT_USERNAME
                          Name of the cephx user. E.g. if 'openstack' is used,
                          then 'ceph auth get client.openstack' will return a
                          working user with key and capabilities on the deployed
                          Ceph cluster. Ignored unless tripleo_cephadm_pools is
                          set via --ansible-extra-vars. If this parameter is not
                          set and tripleo_cephadm_keys is set via --ansible-
                          extra-vars, then 'openstack' will be used. Used to set
                          CephClientUserName in --output.
    --ceph-client-key CEPH_CLIENT_KEY
                          Value of the cephx key. E.g.
                          'AQC+vYNXgDAgAhAAc8UoYt+OTz5uhV7ItLdwUw=='. Ignored
                          unless tripleo_cephadm_pools is set via --ansible-
                          extra-vars. If this parameter is not set and
                          tripleo_cephadm_keys is set via --ansible-extra-vars,
                          then a random key will be generated. Used to set
                          CephClientKey in --output.
    --skip-cephx-keys     Do not create cephx keys even if tripleo_cephadm_pools
                          is set via --ansible-extra-vars. If this option is
                          used, then even the defaults of --ceph-client-key and
                          --ceph-client-username are ignored, but the pools
                          defined via --ansible-extra-vars are still be created.
    --ceph-vip CEPH_VIP   Path to an existing Ceph services/network mapping
                          file.
    --daemons DAEMONS     Path to an existing Ceph daemon options definition.
    --single-host-defaults
                          Adjust configuration defaults to suit a single-host
                          Ceph cluster.
    --ceph-spec CEPH_SPEC
                          Path to an existing Ceph spec file. If not provided a
                          spec will be generated automatically based on --roles-
                          data and <deployed_baremetal.yaml>. The
                          <deployed_baremetal.yaml> parameter is optional only
                          if --ceph-spec is used.
    --osd-spec OSD_SPEC   Path to an existing OSD spec file. Mutually exclusive
                          with --ceph-spec. If the Ceph spec file is generated
                          automatically, then the OSD spec in the Ceph spec file
                          defaults to {data_devices: {all: true}} for all
                          service_type osd. Use --osd-spec to override the
                          data_devices value inside the Ceph spec file.
    --crush-hierarchy CRUSH_HIERARCHY
                          Path to an existing crush hierarchy spec file.
    --standalone          Use single host Ansible inventory. Used only for
                          development or testing environments.
    --container-image-prepare CONTAINER_IMAGE_PREPARE
                          Path to an alternative
                          container_image_prepare_defaults.yaml. Used to control
                          which Ceph container is pulled by cephadm via the
                          ceph_namespace, ceph_image, and ceph_tag variables in
                          addition to registry authentication via
                          ContainerImageRegistryCredentials.
    --cephadm-default-container
                          Use the default continer defined in cephadm instead of
                          container_image_prepare_defaults.yaml. If this is
                          used, 'cephadm bootstrap' is not passed the --image
                          parameter.

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
  ms_cluster_mode: secure
  ms_service_mode: secure
  ms_client_mode: secure
  EOF
  $ openstack overcloud ceph deploy --config initial-ceph.conf ...

The above example shows how to configure the messenger v2 protocol to
use a secure mode that encrypts all data passing over the network.

The `deployed_ceph.yaml` Heat environment file output by `openstack
overcloud ceph deploy` has `ApplyCephConfigOverridesOnUpdate` set to
true. This means that services not covered by deployed ceph, e.g. RGW,
can have the configuration changes that they need applied during
overcloud deployment. After the deployed ceph process has run and
then after the overcloud is deployed, it is recommended to update the
`deployed_ceph.yaml` Heat environment file, or similar, to set
`ApplyCephConfigOverridesOnUpdate` to false. Any subsequent Ceph
configuration changes should then be made by the `ceph config
command`_.

It is supported to pass through the `cephadm --single-host-defaults`
option, which configures a Ceph cluster to run on a single host::

  openstack overcloud ceph deploy --single-host-defaults

Any option available from running `cephadm bootstrap --help` may be
passed through `openstack overcloud ceph deploy` with the
`--cephadm-extra-args` argument. For example::

    openstack overcloud ceph deploy --force \
      --cephadm-extra-args '--log-to-file --skip-prepare-host' \
      ...

When the above is run the following will be run on the cephadm
bootstrap node (the first controller node by default) on the
overcloud::

  cephadm bootstrap --log-to-file --skip-prepare-host ...

The `--force` option is required when using `--cephadm-extra-args`
because not all possible options ensure a functional deployment.

Placement Groups (PGs)
----------------------

When Ceph is initially deployed with `openstack overcloud ceph deploy`
the PG and replica count settings are not changed from Ceph's own
defaults unless their parameters (osd_pool_default_size,
osd_pool_default_pg_num, osd_pool_default_pgp_num) are included in an
initial Ceph configuration file which can be passed with the --config
option. These settings may also be modified after `openstack overcloud
ceph deploy`.

The deprecated Heat paramters `CephPoolDefaultSize` and
`CephPoolDefaultPgNum` no longer have any effect as these
configurations are not made during overcloud deployment.
However, during overcloud deployment pools are created and
both the target_size_ratio or pg_num per pool may be set at that
point. See the "Ceph Pool Options" section for more details.

Ceph Name Options
-----------------

To deploy with a different cluster name than the default of "ceph" use
the ``--cluster`` option::

  openstack overcloud ceph deploy \
          --cluster central \
          ...

The above will result in keyrings and Ceph configuration files being
created with the name passed to cluster, for example::

  [root@oc0-controller-0 ~]# ls -l /etc/ceph/
  total 16
  -rw-------. 1 root root  63 Mar 26 21:49 central.client.admin.keyring
  -rw-------. 1  167  167 201 Mar 26 22:17 central.client.openstack.keyring
  -rw-------. 1  167  167 134 Mar 26 22:17 central.client.radosgw.keyring
  -rw-r--r--. 1 root root 177 Mar 26 21:49 central.conf
  [root@oc0-controller-0 ~]#

When `cephadm shell` is run on an overcloud node like the above, Ceph
commands might return the error ``monclient: get_monmap_and_config
cannot identify monitors to contact`` because the default "ceph" name
is not used. Thus, if the ``--cluster`` is used when deploying Ceph,
then use options like the following to run `cephadm shell` after
deployment::

  cephadm shell --config /etc/ceph/central.conf \
                --keyring /etc/ceph/central.client.admin.keyring

Another solution is to use the following before running ceph commands::

  cephadm shell --mount /etc/ceph:/etc/ceph
  export CEPH_ARGS='--cluster central'

After using either of the above standard Ceph commands should work
within the cephadm shell container.

Ceph Spec Options
-----------------

The roles file, described in the next section, and the output of
`openstack overcloud node provision` are passed to the
`ceph_spec_bootstrap`_ Ansible module to create a `Ceph Service
Specification`_. The `openstack overcloud ceph deploy` command does
this automatically so that a spec does not usually need to be
generated separately. However, it is possible to generate a ceph spec
before deployment with the following command::

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

Overriding which disks should be OSDs
-------------------------------------

The `Advanced OSD Service Specifications`_ should be used to define
how disks are used as OSDs.

By default all available disks (excluding the disk where the operating
system is installed) are used as OSDs. This is because the default
spec has the following::

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

Another possible variation of the ``osd_spec.yaml`` file is the
following::

  data_devices:
    model: 'SAMSUNG'
  osds_per_device: 2

In the above example we use the `model` under `data_devices` to only
create OSDs on that model of device and we configure two OSDs per
device. Note that `osds_per_device` does not go under `data_devices`
but on the same level. The above was created after using the following
command on an already running system to find a way to refer only to a
particular device. In this case the ATA_INTEL devices are not
configured as OSDs but the three SAMSUNG devices are configured as
OSDs::

  [ceph: root@controller-0 /]# ceph orch device ls | grep cephstorage-2
  cephstorage-2  /dev/nvme0n1  ssd   SAMSUNG MZ1LW960HMJP-00003_S2X6NY0KB00178              960G             Insufficient space (<10 extents) on vgs, LVM detected, locked
  cephstorage-2  /dev/nvme1n1  ssd   SAMSUNG MZQLB960HAJR-00007_S437NA0N506009              960G             Insufficient space (<10 extents) on vgs, LVM detected, locked
  cephstorage-2  /dev/nvme2n1  ssd   SAMSUNG MZQLB960HAJR-00007_S437NA0N506027              960G  Yes
  cephstorage-2  /dev/sdb      ssd   ATA_INTEL_SSDSC2KB960G8_PHYF039400YC960CGN             960G  Yes
  cephstorage-2  /dev/sdc      ssd   ATA_INTEL_SSDSC2KB960G8_PHYF039402F2960CGN             960G  Yes
  [ceph: root@controller-0 /]#

The :doc:`node_specific_hieradata` feature is not supported by the
cephadm integration but the `Advanced OSD Service Specifications`_ has
a `host_pattern` parameter which specifies which host to target for
certain `data_devices` definitions, so the equivalent functionality is
available but with the new syntax.

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


Deploy additional daemons
-------------------------
A new option `--daemons` for the `openstack overcloud ceph deploy` command has
been added and may be used to define additional Ceph daemons that are deployed
during the Ceph provisioning process.
This option requires a data structure which defines the services to be
deployed::

  ceph_nfs:
    cephfs_data: 'manila_data'
    cephfs_metadata: 'manila_metadata'
  ceph_rgw: {}
  ceph_ingress:
    tripleo_cephadm_haproxy_container_image: undercloud.ctlplane.mydomain.tld:8787/ceph/haproxy:2.3
    tripleo_cephadm_keepalived_container_image: undercloud.ctlplane.mydomain.tld:8787/ceph/keepalived:2.5.1

For each service added to the data structure above, additional options can be
defined and passed as extra_vars to the tripleo-ansible flow. If no option is
specified, the default values provided by the cephadm tripleo-ansible role will
be used.


Example: deploy HA Ceph NFS daemon
----------------------------------
Cephadm is able to deploy and manage the lifecycle of a highly available
ceph-nfs daemon, called `CephIngress`_, which uses haproxy and keepalived. The
`--daemon` option described in the previous section, provides:

#. a stable, VIP managed by keepalived used to access the NFS service
#. fail-over between hosts in case of failure
#. load distribution across multiple NFS gateways through Haproxy

To deploy a cephadm managed ceph-nfs daemon with the related ingress service,
create a `ceph_daemons.yaml` spec file with the following definition::

  ceph_nfs:
    cephfs_data: 'manila_data'
    cephfs_metadata: 'manila_metadata'
  ceph_ingress:
    tripleo_cephadm_haproxy_container_image: undercloud.ctlplane.mydomain.tld:8787/ceph/haproxy:2.3
    tripleo_cephadm_keepalived_container_image: undercloud.ctlplane.mydomain.tld:8787/ceph/keepalived:2.5.1

When the environment file containing the services definition described above is
created, it can be passed to the ceph deploy command via the `--daemon`
option::

  openstack overcloud ceph deploy \
          deployed_metal.yaml \
          -o deployed_ceph.yaml \
          --ceph-vip ~/ceph_services.yaml \
          --daemon ~/ceph_daemons.yaml

.. note::
   A VIP must be reserved for the ceph_nfs service and passed to the command
   above. For further information on reserving a VIP for a Ceph service, see
   `Ceph VIP Options`_.


Crush Hierarchy Options
-----------------------

The `ceph_spec_bootstrap`_ Ansible module is used to generate the Ceph
related spec file which is applied using the Ceph orchestrator tool.
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

    Device classes are automatically detected by Ceph, but crush rules
    are associated to pools and they still be defined using the
    CephCrushRules parameter during the overcloud deployment. Additional
    details can be found in the "Overriding CRUSH rules" section below.

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

Then the Ceph cluster will have the following parameters set::

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

By default the Ceph global `ms_bind_ipv4` is set `true` and
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

Then the Ceph cluster will have the following parameters set::

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

If the examples above are not sufficient for your Ceph network needs,
then it's possible to create an initial-ceph.conf with the four
parameters ``public_network``, ``cluster_network``, ``ms_bind_ipv4``,
and ``ms_bind_ipv6`` options set to whatever values are desired.

When using the ``--config`` option it is still important to ensure the
TripleO ``storage`` and ``storage_mgmt`` network names map to the
correct ``public_network`` and ``cluster_network`` so that the rest of
the deployment is consistent.

The four parameters, ``public_network``, ``cluster_network``,
``ms_bind_ipv4``, and ``ms_bind_ipv6``, are always set in the Ceph
cluster (with `ceph config set global`) from the ``--network-data``
file unless those parameters are explicitly set in the ``--config``
file. In that case the values in the ``--network-data`` file are not
set directly in the Ceph cluster though other aspects of the overcloud
deployment treat the ``--network-data`` file as authoritative
(e.g. when Ceph RGW is set) so both sources should be consistent if
the ``--config`` file has any of these four parameters.

An example of setting the four parameters in the initial Ceph
configuration is below::

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

The above settings, which mix IPv4 and IPv6, are experimental and
untested.

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

Creating Pools and CephX keys before overcloud deployment (Optional)
--------------------------------------------------------------------

By default `openstack overcloud ceph deploy` does not create Ceph
pools or cephx keys to access those pools. Later during overcloud
deployment the pools and cephx keys are created based on which Heat
environment files are passed. For most cases only pools for Cinder
(volumes), Nova (vms), and Glance (images) are created but if the
Heat environment file to configure additional services are passed,
e.g. cinder-backup, then the required pools are created. This is
covered in more detail in the next section of this document.

It is not necessary to create pools and cephx keys before overcloud
deployment but it is possible. The Ceph pools can be created when
`openstack overcloud ceph deploy` is run by using the option
--ansible-extra-vars to set the tripleo_cephadm_pools variable used
by tripleo-ansible's tripleo_cephadm role.

Create an Ansible extra vars file defining the desired pools::

  cat <<EOF > tripleo_cephadm_ansible_extra_vars.yaml
  ---
  tripleo_cephadm_pools:
    - name: vms
      pg_autoscale_mode: True
      target_size_ratio: 0.3
      application: rbd
    - name: volumes
      pg_autoscale_mode: True
      target_size_ratio: 0.5
      application: rbd
    - name: images
      target_size_ratio: 0.2
      pg_autoscale_mode: True
      application: rbd
  tripleo_ceph_client_vars: /home/stack/overcloud-deploy/overcloud/cephadm/ceph_client.yml
  EOF

The pool names 'vms', 'volumes', and 'images' used above are
recommended since those are the default names that the overcloud
deployment will use when "openstack overcloud deploy" is run, unless
the Heat parameters NovaRbdPoolName, CinderRbdPoolName, and
GlanceRbdPoolName are overridden respectively.

In the above example, tripleo_ceph_client_vars is used to direct Ansible
to save the generated ceph_client.yml file in a cephadm subdirectory of
the working directory. The tripleo_cephadm role will ensure this directory
exists before creating the file. If `openstack overcloud export ceph` is
going to be used, it will expect the Ceph client file to be in this location,
based on the stack name (e.g. overcloud).

Deploy the Ceph cluster with Ansible extra vars::

  openstack overcloud ceph deploy \
          deployed-metal-overcloud.yaml \
          -y -o deployed-ceph-overcloud.yaml \
          --force \
          --ansible-extra-vars tripleo_cephadm_ansible_extra_vars.yaml

After Ceph is deployed, the pools should be created and an openstack cephx
key will also be created to access all of those pools. The contents of
deployed-ceph-overcloud.yaml will also have the pool and cephx key
Heat environment parameters set so the overcloud will use the same
values.

When the tripleo_cephadm_pools variable is set, the Tripleo client will
create a tripleo_cephadm_keys tripleo-ansible variable structure with
the client name "openstack" and a generated cephx key like the following::

  tripleo_cephadm_keys:
    - name: client.openstack
      key: AQC+vYNXgDAgAhAAc8UoYt+OTz5uhV7ItLdwUw==
      mode: '0600'
      caps:
        mgr: allow *
        mon: profile rbd
        osd: profile rbd pool=vms, profile rbd pool=volumes, profile rbd pool=images

It is not recommended to define tripleo_cephadm_keys in the Ansible extra vars file.
If you prefer to set the key username to something other than "openstack" or prefer
to pass your own cephx client key (e.g. AQC+vYNXgDAgAhAAc8UoYt+OTz5uhV7ItLdwUw==),
then use following parameters::

  --ceph-client-username (default: openstack)
  --ceph-client-key (default: auto generates a valid cephx key)

Both of the above parameters are ignored unless tripleo_cephadm_pools is set via
--ansible-extra-vars. If tripleo_cephadm_pools is set then a cephx key to access
all of the pools will always be created unless --skip-cephx-keys is used.

If you wish to re-run 'openstack overcloud ceph deploy' for any
reason and have created-cephx keys in previous runs, then you may use
the --ceph-client-key parameter from the previous run to prevent a new
key from being generated. The key value can be found in the file which
is output from he previous run (e.g. --output <deployed_ceph.yaml>).

If any of the above parameters are used, then the generated deployed Ceph output
file (e.g. --output <deployed_ceph.yaml>) will contain the values of the above
variables mapped to their TripleO Heat template environment variables to ensure a
consistent overcloud deployment::

  CephPools: {{ tripleo_cephadm_pools }}
  CephClientConfigVars: {{ tripleo_ceph_client_vars }}
  CephClientKey: {{ ceph_client_username }}
  CephClientUserName: {{ ceph_client_key }}

The CephPools Heat parameter above has always supported idempotent
updates. It will be pre-populated with the pools from
tripleo_cephadm_pools after Ceph is deployed. The deployed_ceph.yaml
which is output can also be updated so that additional pools can be
created when the overcloud is deployed. The Heat parameters above are
described in more detail in the rest of this document.

Environment files to configure Ceph during Overcloud deployment
---------------------------------------------------------------

After `openstack overcloud ceph deploy` has run and output the
`deployed_ceph.yaml` file, this file and other Heat environment
files should be passed to the `openstack overcloud deploy`
command::

  openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml \
    -e deployed_ceph.yaml

The above will make the following modifications to the Ceph cluster
while the overcloud is being deployed:

- Execute cephadm to add the Ceph RADOS Gateway (RGW) service
- Configure HAProxy as a front end for RGW
- Configure Keystone so RGW behaves like the OpenStack object service
- Create Pools for both RGW and RBD services
- Create an openstack client cephx keyring for Nova, Cinder, Glance to
  access RBD

The information necessary to configure Ceph clients will then
be extracted to `/home/stack/ceph_client.yml` on the undercloud and
passed to the as input to the tripleo-ansible role tripleo_ceph_client
which will then configure the rest of the overcloud to use the new
Ceph cluster as described in the :doc:`ceph_external` documentation.
  
If you only wish to deploy Ceph RBD without RGW then use the following
variation of the above::

  openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm-rbd-only.yaml \
    -e deployed_ceph.yaml

Do not directly edit the `environments/cephadm/cephadm.yaml`
or `cephadm-rbd-only.yaml` file. If you wish to override the defaults,
as described below in the sections starting with "Overriding", then
place those overrides in a separate `cephadm-overrides.yaml` file and
deploy like this::

  openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml \
    -e deployed_ceph.yaml \
    -e ceph-overrides.yaml 

Applying Ceph server configuration during overcloud deployment
--------------------------------------------------------------

The `deployed_ceph.yaml` file output by `openstack overcloud ceph deploy`
has the paramter `ApplyCephConfigOverridesOnUpdate` set to true so
that Ceph services not deployed by `openstack overcloud ceph deploy`,
e.g. RGW, can be configured during initial overcloud deployment. After
both Ceph and the overcloud have been deployed, edit the
`deployed_ceph.yaml` file and set `ApplyCephConfigOverridesOnUpdate`
to false. All Ceph server configuration changes should then be made
using `Ceph Orchestrator`_.

It is technically possible to set `ApplyCephConfigOverridesOnUpdate`
to true and use `CephConfigOverrides` to override Ceph *server*
configurations during stack updates. When this happens, parameters in
`CephConfigOverrides` are put into a file, e.g. assimilate_ceph.conf,
and a command like `ceph config assimilate-conf -i
assimilate_ceph.conf` is run.

Regardless of the value of the `ApplyCephConfigOverridesOnUpdate`
boolean, if `openstack overcloud deploy` is re-run in order to update
the stack, the cephadm bootstrap process is not repeated because
that process is only run if `cephadm list` returns an empty list.

Applying Ceph client configuration during overcloud deployment
--------------------------------------------------------------

To make a Ceph *client* configuration change, update the parameters in
`CephConfigOverrides` and run a stack update. This will not
change the configuration for the Ceph servers unless
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

Ceph Pool Options
-----------------

When `openstack overcloud deploy` is run a pool is created for each
OpenStack service depending on if that service is enabled by including
its Heat environment. For example, a command like the following will
result in pools for Nova (vms), Cinder (volumes) and Glance (images)
being created::

  openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm-rbd-only.yaml

If `-e environments/cinder-backup.yaml` included in the above command
then a pool called backups would also be created.

By default each pool will have Ceph`s pg_autoscale_mode enabled so it
is not necessary to directly set a PG number per pool. However, even
with this mode enabled it is recommended to set a `target_size_ratio`
(or pg_num) per pool in order to minimize data rebalancing. For more
information on pg_autoscale_mode see `Autoscaling Placement Groups`_.

To control the target_size_ratio per pool, create a Heat environment
file like pools.yaml with the following content and include it in the
`openstack overcloud deploy` command with a `-e pools.yaml`::

      CephPools:
        - name: volumes
          target_size_ratio: 0.4
          application: rbd
        - name: images
          target_size_ratio: 0.1
          application: rbd
        - name: vms
          target_size_ratio: 0.3
          application: rbd

In the above example it is assumed that the percentage of data used
per service will be Cinder volumes 40%, Glance images 10% and Nova vms
30% (with 20% of space free for other pools). It is worthwhile to set
these values based on your expected usage (e.g. maybe 40% is not right
for your usecase). If you do not override the CephPools parameter,
then each pool will have Ceph's default PG number. Though the
autoscaler will adjust this number automatically over time based on
usage, the data will be moved within the cluster as a result which
will use computational resources.

If you prefer to set a PG number instead of a target size ratio, then
replace `target_size_ratio` in the example above with ‘pg_num’ and
supply a different integer per pool (e.g. 512 for volumes, 128 for
images, etc.) based on your expected usage.

Overriding CRUSH rules
----------------------

To deploy Ceph pools with custom CRUSH Map Rules use the
`CephCrushRules` parameter to define a list of named rules and then
associate the `rule_name` per pool with the `CephPools` parameter::

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

CRUSH rules may be created during overcloud deployment as documented
above. CRUSH rules may also be created directly via the Ceph command
line tools.

Overriding CephX Keys
---------------------

During overcloud deployment, TripleO will create a Ceph cluster with a
CephX key file for OpenStack RBD client connections that is shared by
the Nova, Cinder, and Glance services to read and write to their
pools. Not only will the keyfile be created but the Ceph cluster will
be configured to accept connections when the key file is used. The
file will be named `ceph.client.openstack.keyring` and it will be
stored in `/etc/ceph` within the containers, but on the container host
it will be stored in a location defined by a TripleO exposed parameter
which defaults to `/var/lib/tripleo-config/ceph`.

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

Add the Ceph Dashboard to a Overcloud deployment
------------------------------------------------

During the overcloud deployment most of the Ceph daemons can be added and
configured.
To deploy the ceph dashboard include the ceph-dashboard.yaml environment
file as in the following example::

    openstack overcloud deploy --templates \
      -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml \
      -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/ceph-dashboard.yaml

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

    openstack overcloud deploy --templates \
      -e <existing_overcloud_environment_files> \
      -e ceph_dashboard_admin.yml

The ceph dashboard will also work with composable networks.
In order to isolate the monitoring access for security purposes, operators can
take advantage of composable networks and access the dashboard through a separate
network vip. By doing this, it's not necessary to access the provisioning network
and separate authorization profiles may be implemented.

To deploy the overcloud with the ceph dashboard composable network we need first
to generate the controller specific role created for this scenario::

    openstack overcloud roles generate \
      -o /home/stack/roles_data.yaml \
      ControllerStorageDashboard Compute \
      BlockStorage ObjectStorage CephStorage

Finally, run the overcloud deploy command including the new generated `roles_data.yaml`
and the `network_data_dashboard.yaml` file that will trigger the generation of this
new network.

The final overcloud command must look like the following::

    openstack overcloud deploy --templates \
      -r /home/stack/roles_data.yaml \
      -n /usr/share/openstack-tripleo-heat-templates/network_data_dashboard.yaml \
      -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml \
      -e ~/my-ceph-settings.yaml

Scenario: Deploy Ceph with TripleO and Metalsmith and then Scale Up
-------------------------------------------------------------------

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

Ceph may then be deployed with `openstack overcloud ceph deploy`.
As described in :doc:`../provisioning/baremetal_provision`, pass
~/overcloud_baremetal_deploy.yaml as input, along with
/usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml
and any Ceph Overrides described in the rest of this document, to the
`openstack overcloud deploy` command.

To scale up, modify the ~/overcloud_baremetal_deploy.yaml file
described above to add more CephStorage nodes. In the example below
the number of storage nodes is doubled::

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
the increased capacity will available. It is not necessary to run
`openstack overcloud ceph deploy` to scale up. 

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

Start a Ceph shell and identify the OSDs to be removed by server. In
the following example we will identify the OSDs of the host ceph-2::

  [root@oc0-controller-0 ~]# cephadm shell
  ...
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

First we need to update the Ceph spec defined in the server. Otherwise
the Ceph Manager will try to recreate the OSDs when we remove them
with `ceph orch osd rm`. Export the Ceph spec, edit it, and reapply
the modified Ceph spec::

  [ceph: root@oc0-controller-0 /]# ceph orch ls --export > spec.yml
  [ceph: root@oc0-controller-0 /]# vi spec.yml
  [ceph: root@oc0-controller-0 /]# ceph orch apply -i spec.yml
  [ceph: root@oc0-controller-0 /]#

In the step where the spec is edited, update the `service_type: osd`
hosts list to remove the "ceph-2" host and remove the `service_type:
host` entry for the "ceph-2" host.

As per `ceph osd tree`, the ceph-2 host has OSDs 5,7,9,11 which can
be removed by running `ceph orch osd rm 5 7 9 11`. For example::

  [ceph: root@oc0-controller-0 /]# ceph orch osd rm --zap 5 7 9 11
  Scheduled OSD(s) for removal
  [ceph: root@oc0-controller-0 /]# ceph orch osd rm status
  OSD_ID  HOST        STATE     PG_COUNT  REPLACE  FORCE  DRAIN_STARTED_AT
  7       ceph-2      draining  27        False    False  2021-04-23 21:35:51.215361
  9       ceph-2      draining  8         False    False  2021-04-23 21:35:49.111500
  11      ceph-2      draining  14        False    False  2021-04-23 21:35:50.243762
  [ceph: root@oc0-controller-0 /]#

The `--zap` option is used to clean the disks so that they can be
easily added back to the ceph cluster if necessary. Use `ceph orch osd
rm status` to check the status::

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

When collocating Nova Compute and Ceph OSD services, boundaries can be
set to reduce contention for CPU and Memory between the two services.
To limit Ceph for HCI, create an initial Ceph configuration file with
the following::

  $ cat <<EOF > initial-ceph.conf
  [osd]
  osd_memory_target_autotune = true
  osd_numa_auto_affinity = true
  [mgr]
  mgr/cephadm/autotune_memory_target_ratio = 0.2
  EOF
  $

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
to true, as in the `initial-ceph.conf` file above, should result in
this configuration. Alternatively, the `osd_numa_node` could be set
directly to 0 and `osd_numa_auto_affinity` could be unset so that it
will default to false.

When a hyperconverged cluster backfills as a result of an OSD going
offline, the backfill process can be slowed down. In exchange for a
slower recovery, the backfill activity has less of an impact on
the collocated Compute workload. Ceph Pacific has the following
defaults to control the rate of backfill activity::

        osd_recovery_op_priority = 3
        osd_max_backfills = 1
        osd_recovery_max_active_hdd = 3
        osd_recovery_max_active_ssd = 10

It is not necessary to pass the above in an initial ceph.conf as they
are the default values, but if these values need to be deployed with
different values modify an example like the above and add it to the
initial Ceph configuration file before deployment. If the values need
to be adjusted after the deployment use `ceph config set osd <key>
<value>`.

To limit Nova resources add parameters to `ceph-overrides.yaml`
like the following but modify the NovaReservedHostMemory to suit your
environment::

  parameter_defaults:
    ComputeHCIParameters:
      NovaReservedHostMemory: 75000

The `NovaReservedHostMemory` above overrides the default value of
`reserved_host_memory_mb` in /etc/nova/nova.conf. The value may be
set so that the Nova scheduler does not give memory to a virtual
machine that a Ceph OSD on the same server will need. The example
above reserves 5 GB per OSD for 10 OSDs per host in addition to the
default reserved memory for the hypervisor. In an IOPS-optimized
cluster performance can be improved by reserving more memory per OSD.
The 5 GB number is provided as a starting point which can be further
tuned if necessary.

Deploy Ceph with `openstack overcloud ceph deploy` and be sure to
pass the initial Ceph configuration file with Ceph HCI tunings. Then
deploy the overcloud with `openstack overcloud deploy` and the as
described in "Scenario: Deploy Ceph with TripleO and Metalsmith" but
use the `-r` option to include generated `roles.yaml` file and the
`-e` option with the `ceph-overrides.yaml` file containing the Nova
HCI tunings described above.

The examples above may be used to tune a hyperconverged system during
deployment. If the values need to be changed after deployment, then
use the `ceph orchestrator` command to set them directly.

After deployment start a Ceph shell and confirm the above values were
applied. For example, to check that the NUMA and memory target auto
tuning run commands lke this::

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

On the compute node, verify that the `reserved_host_memory_mb` was
applied::

  $ sudo podman exec -ti nova_compute /bin/bash
  # grep reserved_host_memory_mb /etc/nova/nova.conf
  reserved_host_memory_mb=75000
  #

.. _`cephadm`: https://docs.ceph.com/en/latest/cephadm/index.html
.. _`cleaning instructions in the Ironic documentation`: https://docs.openstack.org/ironic/latest/admin/cleaning.html
.. _`ceph config command`: https://docs.ceph.com/en/latest/man/8/ceph/#config
.. _`ceph_spec_bootstrap`: https://docs.openstack.org/tripleo-ansible/latest/modules/modules-ceph_spec_bootstrap.html
.. _`Ceph Service Specification`: https://docs.ceph.com/en/octopus/mgr/orchestrator/#orchestrator-cli-service-spec
.. _`Advanced OSD Service Specifications`: https://docs.ceph.com/en/octopus/cephadm/drivegroups/
.. _`Ceph Host Management`: https://docs.ceph.com/en/latest/cephadm/host-management/#setting-the-initial-crush-location-of-host
.. _`Overriding crush rules`: https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/features/cephadm.html#overriding-crush-rules
.. _`CephIngress`: https://docs.ceph.com/en/pacific/cephadm/services/nfs/#high-availability-nfs
.. _`Ceph Orchestrator`: https://docs.ceph.com/en/latest/mgr/orchestrator/
.. _`Autoscaling Placement Groups`: https://docs.ceph.com/en/latest/rados/operations/placement-groups/
.. _`OSD Service Documentation for cephadm`: https://docs.ceph.com/en/latest/cephadm/services/osd/
.. _`osd_memory_target_autotune`: https://docs.ceph.com/en/latest/cephadm/services/osd/#automatically-tuning-osd-memory
