.. _ffu-docs:

Fast Forward Upgrade - Upgrading from Newton to Queens
======================================================

Upgrading a TripleO deployment from Newton to Queens is done by first
executing a minor update in both undercloud and overcloud, to ensure that the
system is using the latest Newton release. After that, the undercloud is
upgraded to the target version Queens. This will then be used to upgrade the
overcloud.

.. note::

   Before upgrading the undercloud to Queens, make sure you have created a valid
   backup of the current undercloud and overcloud. The complete backup
   procedure can be found on:
   :doc:`undercloud backup<../install/controlplane_backup_restore/00_index>`

Undercloud FFU upgrade
----------------------

.. note::

   Fast Forward Upgrade testing cannot cover all possible deployment
   configurations. Before performing the Fast Forward Upgrade of the undercloud
   in production, test it in a matching staging environment, and create a backup
   of the undercloud in the production environment. Please refer to
   :doc:`undercloud backup<../install/controlplane_backup_restore/01_undercloud_backup>`
   for proper documentation on undercloud backups.

The undercloud FFU upgrade consists of 3 consecutive undercloud upgrades to
Ocata, Pike and Queens.

Undercloud upgrade to Ocata
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   ## Install tripleo-repos
   TRIPLEO_REPOS_RPM=$(curl -L --silent https://trunk.rdoproject.org/centos7/current/ | grep python2-tripleo-repos | awk -F "href" {'print $2'} | awk -F '"' {'print $2'})
   sudo yum localinstall -y https://trunk.rdoproject.org/centos7/current/${TRIPLEO_REPOS_RPM}

   ## Deploy repos via tripleo-repos
   sudo tripleo-repos -b ocata current ceph

   ## Pre-upgrade stop services and update specific packages
   sudo systemctl stop openstack-* neutron-* httpd
   sudo yum update -y instack-undercloud openstack-puppet-modules openstack-tripleo-common python-tripleoclient
   openstack undercloud upgrade

Undercloud upgrade to Pike
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   ## Deploy repos via tripleo-repos
   sudo tripleo-repos -b pike current ceph

   ## Update tripleoclient and install ceph-ansible
   sudo yum -y install ceph-ansible
   sudo yum -y update python-tripleoclient
   openstack undercloud upgrade

Undercloud upgrade to Queens
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   ## Deploy repos via tripleo-repos
   sudo tripleo-repos -b queens current ceph

   ## Update tripleoclient
   sudo yum -y update python-tripleoclient
   openstack undercloud upgrade

Maintaining the system while the undercloud is on Queens and overcloud on Newton
--------------------------------------------------------------------------------

After upgrading undercloud to Queens, the system is expected to be stable and
allow normal management operations of the overcloud nodes that are still on
Newton. However, to ensure that compatibility, several steps need to be
performed.

1. You need to use the newer introspection images, because of incompatible
   changes in the newer versions of ironic client.

   .. code-block:: bash

      mkdir /home/stack/images
      cd /home/stack/images
      wget https://images.rdoproject.org/queens/delorean/current-tripleo/ironic-python-agent.tar
      tar -xvf ironic-python-agent.tar

      source /home/stack/stackrc
      openstack overcloud image upload --image-path /home/stack/images/ \
      --update-existing

2. Remember to keep the old Newton templates. When the undercloud is upgraded,
   the new Queens templates are installed. The Newton templates can be used to
   perform any needed configuration or management of the overcloud nodes. Be
   sure that you have copied your old templates. Or if you didn't have a local
   copy, clone the Newton templates under a new directory:

   .. code-block:: bash

      git clone -b  stable/newton \
      https://git.openstack.org/openstack/tripleo-heat-templates tripleo-heat-templates-newton

3. Use a new `plan-environment.yaml` file. As undercloud CLI calls have been
   upgraded, they will request that file. It needs to be on
   /home/stack/tripleo-heat-templates-newton, and have the following content:


   .. code-block:: yaml

      version: 1.0

      name: overcloud
      description: >
        Default Deployment plan
      template: overcloud.yaml
      passwords: {}
      environments:
        - path: overcloud-resource-registry-puppet.yaml

   Create a new docker-ha.yaml env file, based on the puppet-pacemaker one:

   .. code-block:: bash

      cp /home/stack/tripleo-heat-templates-newton/environments/puppet-pacemaker.yaml \
      /home/stack/tripleo-heat-templates-newton/environments/docker-ha.yaml

   Create an empty docker.yaml env file, replacing the one that is currently on
   newton:

   .. code-block:: bash

      : > /home/stack/tripleo-heat-templates-newton/environments/docker.yaml

   After all these steps have been performed, the Queens undercloud can be used
   successfully to provide and manage a Newton overcloud.

Upgrading the overcloud from Newton to Queens
---------------------------------------------

.. note::

   Generic Fast Forward Upgrade testing in the overcloud cannot cover all
   possible deployment configurations. Before performing Fast Forward Upgrade
   testing in the overcloud, test it in a matching staging environment, and
   create a backup of the production environment (your controller nodes and your
   workloads).

The Queens upgrade workflow essentially consists of the following steps:

#. `Prepare your environment - get container images`_, backup.
   Generate any environment files you need for the upgrade such as the
   references to the latest container images or commands used to switch repos.

#. `openstack overcloud ffwd-upgrade prepare`_ $OPTS.
   Run a heat stack update to generate the upgrade playbooks.

#. `openstack overcloud ffwd-upgrade run`_. Run the ffwd upgrade tasks on all
   nodes.

#. `openstack overcloud upgrade run`_ $OPTS.
   Run the upgrade on specific nodes or groups of nodes. Repeat until all nodes
   are successfully upgraded.

#. `openstack overcloud ceph-upgrade run`_ $OPTS. (optional)
   Not necessary unless a TripleO managed Ceph cluster is deployed in the
   overcloud; this step performs the upgrade of the Ceph cluster.

#. `openstack overcloud ffwd-upgrade converge`_ $OPTS.
   Finally run a heat stack update, unsetting any upgrade specific variables
   and leaving the heat stack in a healthy state for future updates.

.. _queens-upgrade-dev-docs: https://docs.openstack.org/tripleo-docs/latest/install/developer/upgrades/major_upgrade.html # WIP @ https://review.openstack.org/#/c/569443/

Prepare your environment - Get container images
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When moving from Newton to Queens, the setup will be changing from baremetal to
containers. So as a part of the upgrade the container images for the target
release should be downloaded to the Undercloud.
Please see the `openstack overcloud container image prepare`
:doc:`../install/containers_deployment/overcloud` for more information.

The output of this step will be a Heat environment file that contains
references to the latest container images. You will need to pass this file
into the **upgrade prepare** command using the -e flag to include the
generated file.

You may want to populate a local docker registry in your undercloud, to make the
deployment faster and more reliable. In that case you need to use the 8787 port,
and the ip needs to be the `local_ip` parameter from the `undercloud.conf` file.

.. code-block:: bash

   openstack overcloud container image prepare \
   --namespace=192.0.2.1:8787/tripleoqueens --tag current-tripleo \
   --output-env-file /home/stack/container-default-parameters.yaml \
   --output-images-file overcloud_containers.yaml <OPTIONS> \
   --push-destination 192.0.2.1:8787

In place of the `<OPTIONS>` token should go all parameters that you used with
previous `openstack overcloud deploy` command.

After that, upload your images.

.. code-block:: bash

   openstack overcloud container image upload \
   --config-file /home/stack/overcloud_containers.yaml \
   -e /home/stack/container-default-parameters.yaml

Prepare your environment - New templates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You will also need to create an environment file to override the
`FastForwardCustomRepoScriptContent` and `FastForwardRepoType`
tripleo-heat-templates parameters, that can be used to switch the yum repos in
use by the nodes during the upgrade.
This will likely be the same commands that were used to switch repositories
on the undercloud.

.. code-block:: yaml

   cat <<EOF > init-repo.yaml
   parameter_defaults:
     FastForwardRepoType: custom-script
     FastForwardCustomRepoScriptContent: |
       set -e
       case $1 in
         ocata)
           <code to install ocata repo here>
           ;;
         pike)
           <code to install pike repo here>
           ;;
         queens)
           <code to install queens repo here>
           ;;
         *)
           echo "unknown release $1" >&2
           exit 1
       esac
       yum clean all
   EOF

The resulting init-repo.yaml will then be passed into the upgrade prepare using
the -e option.

.. _Upgradeinitcommand: https://github.com/openstack/tripleo-heat-templates/blob/1d9629ec0b3320bcbc5a4150c8be19c6eb4096eb/puppet/role.role.j2.yaml#L468-L493

You will also need to create a cli_opts_params.yaml file, that will contain the
number of nodes for each role, and the flavor to be used. See that sample:

.. code-block:: bash

   cat <<EOF > cli_opts_params.yaml
     parameter_defaults:
       ControllerCount: 3
       ComputeCount: 1
       CephStorageCount: 1
       NtpServer: clock.redhat.com
   EOF

Prepare your environment - Adapt your templates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Before running Fast Forward Upgrade, it is important that you ensure that the
custom templates that you are using in your deploy (Newton version), are
adapted to the syntax needed for the new stable release (Queens version).
Please check the annex in this document, and the changelogs of all the
different versions to get a detailed list of the templates that need to be
changed.


openstack overcloud ffwd-upgrade prepare
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

   Before running the overcloud upgrade prepare ensure you have a valid backup
   of the current state, including the **undercloud** since there will be a
   Heat stack update performed here. The complete backup procedure can be
   found on:
   :doc:`undercloud backup<../install/controlplane_backup_restore/00_index>`


.. note::

   After running the ffwd-upgrade prepare and until successful completion
   of the ffwd-upgrade converge operation, stack updates to the deployment
   Heat stack are expected to fail. That is, operations such as scaling to
   add a new node or to apply any new TripleO configuration via Heat stack
   update **must not** be performed on a Heat stack that has been prepared
   for upgrade with the 'prepare' command. Only consider doing so after
   running the converge step. See the queens-upgrade-dev-docs_ for more.

Run **overcloud ffwd-upgrade prepare**. This command expects the full set
of environment files that were passed into the deploy command, as well as the
roles_data.yaml file used to deploy the overcloud you are about to upgrade. The
environment file should point to the file that was output by the image
prepare command you ran to get the latest container image references.

.. note::

   It is especially important to remember that you **must** include all
   environment files that were used to deploy the overcloud that you are about
   to upgrade.

.. code-block:: bash

   openstack overcloud ffwd-upgrade prepare --templates \
     -e /home/stack/containers-default-parameters.yaml \
     <OPTIONS> \
     -e init-repo.yaml
     -e cli_opts_params.yaml
     -r /path/to/roles_data.yaml


In place of the `<OPTIONS>` token should go all parameters that you used with
previous `openstack overcloud deploy` command.

This will begin an update on the overcloud Heat stack but without
applying any of the TripleO configuration, as explained above. Once this
`ffwd-upgrade prepare` operation has successfully completed the heat stack will
be in the UPDATE_COMPLETE state. At that point you can use `config download` to
download and inspect the configuration ansible playbooks that will be used
to deliver the upgrade in the next step:

.. code-block:: bash

   openstack overcloud config download --config-dir SOMEDIR
   # playbooks will be downloaded to SOMEDIR directory

openstack overcloud ffwd-upgrade run
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This will execute the ffwd-upgrade initial steps in all nodes.

.. code-block:: bash

   openstack overcloud ffwd-upgrade run --yes

After this step, the upgrade commands can be executed in all nodes.

openstack overcloud upgrade run
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This will run the ansible playbooks to deliver the upgrade configuration.
By default, 3 playbooks are executed: the upgrade_steps_playbook, then the
deploy_steps_playbook and finally the post_upgrade_steps_playbook. These
playbooks are invoked on those overcloud nodes specified by the ``--limit``
parameter.

.. code-block:: bash

   openstack overcloud upgrade run --limit Controller


.. note::

   *Optionally* you can specify ``--playbook`` to manually step through the upgrade
   playbooks: You need to run all three in this order and as specified below
   (no path) for a full upgrade to Queens.


.. code-block:: bash

   openstack overcloud upgrade run --limit Controller --playbook upgrade_steps_playbook.yaml
   openstack overcloud upgrade run --limit Controller --playbook deploy_steps_playbook.yaml
   openstack overcloud upgrade run --limit Controller --playbook post_upgrade_steps_playbook.yaml

After all three playbooks have been executed without error on all nodes of
the controller role the controlplane will have been fully upgraded to Queens.
At a minimum an operator should check the health of the pacemaker cluster

.. admonition:: Stable Branch
   :class: stable

   The ``--limit`` was introduced in the Stein release. In previous versions,
   use ``--nodes`` or ``--roles`` paremeters.

For control plane nodes, you are expected to upgrade all nodes within a role at
the same time: pass a role name to ``--limit``. For non-control-plane nodes,
you often want to specify a single node or a list of nodes to ``--limit``.

The controller nodes need to be the first upgraded, following by the compute
and storage ones.

.. code-block:: bash

   [root@overcloud-controller-0 ~]# pcs status | grep -C 10 -i "error\|fail\|unmanaged"

The operator may also want to confirm that openstack and related service
containers are all in a good state and using the image references passed
during upgrade prepare with the ``--container-registry-file`` parameter.

.. code-block:: bash

   [root@overcloud-controller-0 ~]# docker ps -a

.. warning::

   When the upgrade has been applied on the Controllers, but not on the other
   nodes, it is important to don't execute any operation on the overcloud. The
   nova, neutron.. commands will be up at this point but users are not advised
   to use them, until all the steps of Fast Forward Upgrade have been
   completed, or it may drive unexpected results.

For non controlplane nodes, such as Compute or ObjectStorage, you can use
``--limit overcloud-compute-0`` to upgrade particular nodes, or even
"compute0,compute1,compute3" for multiple nodes. Note these are again
upgraded in parallel. Also note that you can pass roles names to upgrade all
nodes in a role at the same time is preferred.

.. code-block:: bash

   openstack overcloud upgrade run --limit overcloud-compute-0

Use of ``--limit`` allows the operator to upgrade some subset, perhaps just
one, compute or other non controlplane node and verify that the upgrade is
successful. One may even migrate workloads onto the newly upgraded node and
confirm there are no problems, before deciding to proceed with upgrading the
remaining nodes that are still on Newton.

Again you can optionally step through the upgrade playbooks if you prefer. Be
sure to run upgrade_steps_playbook.yaml then deploy_steps_playbook.yaml and
finally post_upgrade_steps_playbook.yaml in that order.

For re-run, you can specify ``--skip-tags validation`` to skip those step 0
ansible tasks that check if services are running, in case you can't or
don't want to start them all.

.. code-block:: bash

   openstack overcloud upgrade run --limit Controller --skip-tags validation

openstack overcloud ceph-upgrade run
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This step is only necessary if Ceph was deployed in the Overcloud. It triggers
an upgrade of the Ceph cluster which will be performed without taking down
the cluster.

   .. note::

      It is especially important to remember that you **must** include all
      environment files that were used to deploy the overcloud that you are about
      to upgrade.

   .. code-block:: bash

      openstack overcloud ceph-upgrade run --templates \
        --container-registry-file /home/stack/containers-default-parameters.yaml \
        <OPTIONS> -r /path/to/roles_data.yaml

In place of the `<OPTIONS>` token should go all parameters that you used with
previous `openstack overcloud deploy` command.

At the end of the process, Ceph will be upgraded from Jewel to Luminous so
there will be new containers for the `ceph-mgr` service running on the
controlplane node.

openstack overcloud ffwd-upgrade converge
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Finally, run the converge heat stack update. This will re-apply all Queens
configuration across all nodes and unset all variables that were used during
the upgrade. Until you have successfully completed this step, heat stack
updates against the overcloud stack are expected to fail. You can read more
about why this is the case in the queens-upgrade-dev-docs_.

.. note::

   It is especially important to remember that you **must** include all
   environment files that were used to deploy the overcloud that you are about
   to upgrade converge, including the list of Queens container image references
   and the roles_data.yaml roles and services definition. You should omit
   any repo switch commands and ensure that none of the environment files
   you are about to use is specifying a value for UpgradeInitCommand.

.. note::

   The Queens container image references that were passed into the
   `openstack overcloud ffwd-upgrade prepare`_ with the
   ``--container-registry-file`` parameter **must** be included as an
   environment file, with the -e option to the openstack overcloud
   ffwd-upgrade run command, together with all other environment files
   for your deployment.

.. code-block:: bash

   openstack overcloud ffwd-upgrade converge --templates
     -e /home/stack/containers-default-parameters.yaml \
     -e cli_opts_params.yaml \
     <OPTIONS> -r /path/to/roles_data.yaml


In place of the `<OPTIONS>` token should go all parameters that you used with
previous `openstack overcloud deploy` command.

The Heat stack will be in the **UPDATE_IN_PROGRESS** state for the duration of
the openstack overcloud upgrade converge. Once converge has completed
successfully the Heat stack should also be in the **UPDATE_COMPLETE** state.

Annex: Template changes needed from Newton to Queens
----------------------------------------------------
In order to reuse the Newton templates when the cloud has been upgraded to
Queens, several changes are needed. Those changes need to be done before
starting Fast Forward Upgrade on the overcloud.

Following there is a list of all the changes needed:


1. Remove those deprecated services from your custom roles_data.yaml file:

* OS::TripleO::Services::Core
* OS::TripleO::Services::GlanceRegistry
* OS::TripleO::Services::VipHosts


2. Add the following new services to your custom roles_data.yaml file:

* OS::TripleO::Services::MySQLClient
* OS::TripleO::Services::NovaPlacement
* OS::TripleO::Services::PankoApi
* OS::TripleO::Services::Sshd
* OS::TripleO::Services::CertmongerUser
* OS::TripleO::Services::Docker
* OS::TripleO::Services::MySQLClient
* OS::TripleO::Services::ContainersLogrotateCrond
* OS::TripleO::Services::Securetty
* OS::TripleO::Services::Tuned
* OS::TripleO::Services::Clustercheck (just required on roles that also uses
  OS::TripleO::Services::MySQL)
* OS::TripleO::Services::Iscsid (to configure iscsid on Controller, Compute
  and BlockStorage roles)
* OS::TripleO::Services::NovaMigrationTarget (to configure migration on
  Compute roles)


3. Update any additional parts of the overcloud that might require these new
   services such as:

* Custom ServiceNetMap parameter - ensure to include the latest
  ServiceNetMap for the new services. You can locate in
  network/service_net_map.j2.yaml file
* External Load Balancer - if using an external load balancer, include
  these new services as a part of the external load balancer configuration


4. A new feature for composable networks was introduced on Pike. If using a
   custom roles_data file, edit the file to add the composable networks to each
   role. For example, for Controller nodes:

   ::

     - name: Controller
       networks:
       - External
       - InternalApi
       - Storage
       - StorageMgmt
       - Tenant

   Check the default networks on roles_data.yaml for further examples of syntax.


5. The following parameters are deprecated and have been replaced with
   role-specific parameters:

* from controllerExtraConfig to ControllerExtraConfig
* from OvercloudControlFlavor to OvercloudControllerFlavor
* from controllerImage to ControllerImage
* from NovaImage to ComputeImage
* from NovaComputeExtraConfig to ComputeExtraConfig
* from NovaComputeServerMetadata to ComputeServerMetadata
* from NovaComputeSchedulerHints to ComputeSchedulerHints
* from NovaComputeIPs to ComputeIPs
* from SwiftStorageServerMetadata to ObjectStorageServerMetadata
* from SwiftStorageIPs to ObjectStorageIPs
* from SwiftStorageImage to ObjectStorageImage
* from OvercloudSwiftStorageFlavor to OvercloudObjectStorageFlavor


6. Some composable services include new parameters that configure Puppet
   hieradata. If you used hieradata to configure these parameters in the past,
   the overcloud update might report a Duplicate declaration error.
   If this situation, use the composable service parameter.
   For example, instead of the following:

   ::

     parameter_defaults:
       controllerExtraConfig:
         heat::config::heat_config:
           DEFAULT/num_engine_workers:
             value: 1

   Use the following:

   ::

     parameter_defaults:
       HeatWorkers: 1


7. In your resource_registry, check that you are using the containerized
   services from the docker/services subdirectory of your core Heat template
   collection. For example:

   ::

     resource_registry:
       OS::TripleO::Services::CephMon: ../docker/services/ceph-ansible/ceph-mon.yaml
       OS::TripleO::Services::CephOSD: ../docker/services/ceph-ansible/ceph-osd.yaml
       OS::TripleO::Services::CephClient: ../docker/services/ceph-ansible/ceph-client.yaml


8. When upgrading to Queens, if Ceph has been deployed in the Overcloud, then
   use the `ceph-ansible.yaml` environment file **instead of**
   `storage-environment.yaml`. Make sure to move any customization into
   `ceph-ansible.yaml` (or a copy of ceph-ansible.yaml)

   .. code-block:: bash

      openstack overcloud deploy --templates \
        -e <full environment> \
        -e /usr/share/openstack-tripleo-heat-templates/environments/docker.yaml \
        -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
        -e /usr/share/openstack-tripleo-heat-templates/environments/major-upgrade-composable-steps-docker.yaml \
        -e overcloud-repos.yaml

   Customizations for the Ceph deployment previously passed as hieradata via
   \*ExtraConfig should be removed as they are ignored, specifically the
   deployment will stop if ``ceph::profile::params::osds`` is found to
   ensure the devices list has been migrated to the format expected by
   ceph-ansible. It is possible to use the ``CephAnsibleExtraConfig`` and
   `CephAnsibleDisksConfig`` parameters to pass arbitrary variables to
   ceph-ansible, like ``devices`` and ``dedicated_devices``.  See the
   :doc:`TripleO Ceph config guide <../install/advanced_deployment/ceph_config>`

   The other parameters (for example ``CinderRbdPoolName``,
   ``CephClientUserName``, ...) will behave as they used to with puppet-ceph
   with the only exception of ``CephPools``. This can be used to create
   additional pools in the Ceph cluster but the two tools expect the list
   to be in a different format. Specifically while puppet-ceph expected it
   in this format::

     {
       "mypool": {
         "size": 1,
         "pg_num": 32,
         "pgp_num": 32
        }
     }

   with ceph-ansible that would become::

     [{"name": "mypool", "pg_num": 32, "rule_name": ""}]

9. If using custom nic-configs, the format has changed, and it is using an
   script to generate the entries now. So you will need to convert your old
   syntax from:

   ::

    resources:
      OsNetConfigImpl:
        properties:
          config:
            os_net_config:
              network_config:
                - type: interface
                  name: nic1
                  mtu: 1350
                  use_dhcp: false
                  addresses:
                  - ip_netmask:
                    list_join:
                      - /
                      - - {get_param: ControlPlaneIp}
                      - {get_param: ControlPlaneSubnetCidr}
                    routes:
                  - ip_netmask: 169.254.169.254/32
                    next_hop: {get_param: EC2MetadataIp}
                - type: ovs_bridge
                  name: br-ex
                  dns_servers: {get_param: DnsServers}
                  use_dhcp: false
                  addresses:
                  - ip_netmask: {get_param: ExternalIpSubnet}
                  routes:
                  - ip_netmask: 0.0.0.0/0
                    next_hop: {get_param: ExternalInterfaceDefaultRoute}
                  members:
                    - type: interface
                      name: nic2
                      mtu: 1350
                      primary: true
                - type: interface
                  name: nic3
                  mtu: 1350
                  use_dhcp: false
                  addresses:
                  - ip_netmask: {get_param: InternalApiIpSubnet}
                - type: interface
                  name: nic4
                  mtu: 1350
                  use_dhcp: false
                  addresses:
                  - ip_netmask: {get_param: StorageIpSubnet}
                - type: interface
                  name: nic5
                  mtu: 1350
                  use_dhcp: false
                  addresses:
                  - ip_netmask: {get_param: StorageMgmtIpSubnet}
                - type: ovs_bridge
                  name: br-tenant
                  dns_servers: {get_param: DnsServers}
                  use_dhcp: false
                  addresses:
                  - ip_netmask: {get_param: TenantIpSubnet}
                  members:
                  - type: interface
                    name: nic6
                    mtu: 1350
                    primary: true
            group: os-apply-config
          type: OS::Heat::StructuredConfig


   To

   ::

    resources:
      OsNetConfigImpl:
        type: OS::Heat::SoftwareConfig
        properties:
          group: script
          config:
            str_replace:
              template:
                get_file: ../../../../../network/scripts/run-os-net-config.sh
            params:
              $network_config:
                network_config:
                  - type: interface
                    name: nic1
                    mtu: 1350
                    use_dhcp: false
                    addresses:
                    - ip_netmask:
                        list_join:
                        - /
                        - - {get_param: ControlPlaneIp}
                        - {get_param: ControlPlaneSubnetCidr}
                    routes:
                    - ip_netmask: 169.254.169.254/32
                      next_hop: {get_param: EC2MetadataIp}
                  - type: ovs_bridge
                    name: br-ex
                    dns_servers: {get_param: DnsServers}
                    use_dhcp: false
                    addresses:
                    - ip_netmask: {get_param: ExternalIpSubnet}
                    routes:
                    - ip_netmask: 0.0.0.0/0
                      next_hop: {get_param: ExternalInterfaceDefaultRoute}
                    members:
                    - type: interface
                      name: nic2
                      mtu: 1350
                      primary: true
                  - type: interface
                    name: nic3
                    mtu: 1350
                    use_dhcp: false
                    addresses:
                    - ip_netmask: {get_param: InternalApiIpSubnet}
                  - type: interface
                    name: nic4
                    mtu: 1350
                    use_dhcp: false
                    addresses:
                    - ip_netmask: {get_param: StorageIpSubnet}
                  - type: interface
                    name: nic5
                    mtu: 1350
                    use_dhcp: false
                    addresses:
                    - ip_netmask: {get_param: StorageMgmtIpSubnet}
                  - type: ovs_bridge
                    name: br-tenant
                    dns_servers: {get_param: DnsServers}
                    use_dhcp: false
                    addresses:
                    - ip_netmask: {get_param: TenantIpSubnet}
                    members:
                    - type: interface
                      name: nic6
                      mtu: 1350
                      primary: true


10. If using a modified version of the core Heat template collection from
    Newton, you need to re-apply your customizations to a copy of the Queens
    version. To do this, use a git version control system or similar toolings
    to compare.


Annex: NFV template changes needed from Newton to Queens
--------------------------------------------------------
Following there is a list of general changes needed into NFV context:

1. Fixed VIP addresses for overcloud networks use new parameters as syntax:

   ::

      parameter_defaults:
        ...
        # Predictable VIPs
        ControlFixedIPs: [{'ip_address':'192.168.201.101'}]
        InternalApiVirtualFixedIPs: [{'ip_address':'172.16.0.9'}]
        PublicVirtualFixedIPs: [{'ip_address':'10.1.1.9'}]
        StorageVirtualFixedIPs: [{'ip_address':'172.18.0.9'}]
        StorageMgmtVirtualFixedIPs: [{'ip_address':'172.19.0.9'}]
        RedisVirtualFixedIPs: [{'ip_address':'172.16.0.8'}]


For DPDK environments:

1. Modify HostCpuList and NeutronDpdkCoreList to match your configuration.
   Ensure that you use only double quotation marks in the yaml file for these
   parameters:

   ::

      HostCpusList: "0,16,8,24"
      NeutronDpdkCoreList: "1,17,9,25"

2. Modify NeutronDpdkSocketMemory to match your configuration. Ensure that you
   use only double quotation marks in the yaml file for this parameter:

   ::

      NeutronDpdkSocketMemory: "2048,2048"

3. Modify NeutronVhostuserSocketDir as follows:

   ::

      NeutronVhostuserSocketDir: "/var/lib/vhost_sockets"

4. Modify VhostuserSocketGroup as follows, mapping to the right compute role:

   ::

     parameter_defaults:
       <name_of_your_compute_role>Parameters:
         VhostuserSocketGroup: "hugetlbfs"

5. In the parameter_defaults section, add a network deployment parameter to run
   os-net-config during the upgrade process to associate OVS PCI address with
   DPDK ports:

   ::

     parameter_defaults:
       ComputeNetworkDeploymentActions: ['CREATE', 'UPDATE']

   The parameter name must match the name of the role you use to deploy DPDK.
   In this example, the role name is Compute so the parameter name is
   ComputeNetworkDeploymentActions.

6. In the resource_registry section, override the
   ComputeNeutronOvsDpdk service to the neutron-ovs-dpdk-agent docker service:

   ::

     resource_registry:
       OS::TripleO::Services::ComputeNeutronOvsDpdk: ../docker/services/neutron-ovs-dpdk-agent.yaml

   And remove the previous entry:

   ::

     resource_registry:
       OS::TripleO::Services::ComputeNeutronOvsAgent: ../puppet/services/neutron-ovs-dpdk-agent.yaml

   Please notice the naming change between `ComputeNeutronOvsAgent` and
   `ComputeNeutronOvsDpdk`.


For SR-IOV environments:

1. In the resource registry section, override the NeutronSriovAgent service
   to the neutron-sriov-agent docker service:

   ::

     resource_registry:
       OS::TripleO::Services::NeutronSriovAgent: ../docker/services/neutron-sriov-agent.yaml

   And remove the previous entry:

   ::

     resource_registry:
       OS::TripleO::Services::NeutronSriovAgent: ../puppet/services/neutron-sriov-agent.yaml
