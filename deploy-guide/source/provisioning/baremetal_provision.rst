.. _baremetal_provision:

Provisioning Baremetal Before Overcloud Deploy
==============================================

Baremetal provisioning is a feature which interacts directly with the
Bare Metal service to provision baremetal before the overcloud is deployed.
This adds a new provision step before the overcloud deploy, and the output of
the provision is a valid :doc:`../features/deployed_server` configuration.

In the Wallaby release the baremetal provisining was extended to also manage
the neutron API resources for :doc:`../features/network_isolation` and
:doc:`../features/custom_networks`, and apply network configuration on the
provisioned nodes using os-net-config.

Undercloud Components For Baremetal Provisioning
------------------------------------------------

A new YAML file format is introduced to describe the baremetal required for
the deployment, and the new command ``openstack overcloud node provision``
will consume this YAML and make the specified changes. The provision command
interacts with the following undercloud components:

* A baremetal provisioning workflow which consumes the YAML and runs to
  completion

* The `metalsmith`_ tool which deploys nodes and associates ports. This tool is
  responsible for presenting a unified view of provisioned baremetal while
  interacting with:

  * The Ironic baremetal node API for deploying nodes

  * The Ironic baremetal allocation API which allocates nodes based on the YAML
    provisioning criteria

  * The Neutron API for managing ports associated with the node's NICs


In a future release this will become the default way to deploy baremetal, as
the Nova compute service and the Glance image service will be switched off on
the undercloud.

Baremetal Provision Configuration
---------------------------------

A declarative YAML format specifies what roles will be deployed and the
desired baremetal nodes to assign to those roles. Defaults can be relied on
so that the simplest configuration is to specify the roles, and the count of
baremetal nodes to provision for each role

.. code-block:: yaml

  - name: Controller
    count: 3
  - name: Compute
    count: 100

Often it is desirable to assign specific nodes to specific roles, and this is
done with the ``instances`` property

.. code-block:: yaml

  - name: Controller
    count: 3
    instances:
    - hostname: overcloud-controller-0
      name: node00
    - hostname: overcloud-controller-1
      name: node01
    - hostname: overcloud-controller-2
      name: node02
  - name: Compute
    count: 100
    instances:
    - hostname: overcloud-novacompute-0
      name: node04

Here the instance ``name`` refers to the logical name of the node, and the
``hostname`` refers to the generated hostname which is derived from the
overcloud stack name, the role, and an incrementing index. In the above
example, all of the Controller servers are on predictable nodes, as well as
one of the Compute servers. The other 99 Compute servers are on nodes
allocated from the pool of available nodes.

The properties in the ``instances`` entries can also be set in the
``defaults`` section so that they do not need to be repeated in every entry.
For example, the following are equivalent

.. code-block:: yaml

  - name: Controller
    count: 3
    instances:
    - hostname: overcloud-controller-0
      name: node00
      image:
        href: overcloud-full-custom
    - hostname: overcloud-controller-1
      name: node01
      image:
        href: overcloud-full-custom
    - hostname: overcloud-controller-2
      name: node02
      image:
        href: overcloud-full-custom

  - name: Controller
    count: 3
    defaults:
      image:
        href: overcloud-full-custom
    instances:
    - hostname: overcloud-controller-0
      name: node00
    - hostname: overcloud-controller-1
      name: node01
    - hostname: overcloud-controller-2
      name: node02

When using :doc:`../features/network_isolation`,
:doc:`../features/custom_networks` or a combination of the two the **networks**
and **network_configuration** must either be set in the ``defaults`` for the
role or for each specific node (instance). The following example extends the
first simple configuration example adding typical TripleO network isolation by
setting defaults for each role

.. code-block:: yaml

  - name: Controller
    count: 3
    defaults:
      networks:
      - network: ctlplane
        vif: true
      - network: external
        subnet: external_subnet
      - network: internalapi
        subnet: internal_api_subnet01
      - network: storage
        subnet: storage_subnet01
      - network: storagemgmt
        subnet: storage_mgmt_subnet01
      - network: tenant
        subnet: tenant_subnet01
      network_config:
        template: /home/stack/nic-config/controller.j2
        default_route_network:
        - external
  - name: Compute
    count: 100
    defaults:
      networks:
      - network: ctlplane
        vif: true
      - network: internalapi
        subnet: internal_api_subnet02
      - network: tenant
        subnet: tenant_subnet02
      - network: storage
        subnet: storage_subnet02
      network_config:
        template: /home/stack/nic-config/compute.j2


Role Properties
^^^^^^^^^^^^^^^

Each role entry supports the following properties:

* ``name``: Mandatory role name

* ``hostname_format``: Override the default hostname format for this role. The
  default format uses the lower case role name, so for the ``Controller`` role the
  default format is ``%stackname%-controller-%index%``. Only the ``Compute`` role
  doesn't follow the role name rule, the ``Compute`` default format is
  ``%stackname%-novacompute-%index%``

* ``count``: Number of nodes to provision for this role, defaults to 1

* ``defaults``: A dict of default values for ``instances`` entry properties. An
  ``instances`` entry property will override a default specified here See
  :ref:`instance-defaults-properties` for supported properties

* ``instances``: A list of dict for specifying attributes for specific nodes.
  See :ref:`instance-defaults-properties` for supported properties. The length
  of this list must not be greater than ``count``

* ``ansible_playbooks``: A list of dict for Ansible playbooks and Ansible vars,
  the playbooks are run against the role instances after node provisioning,
  prior to the node network configuration. See
  :ref:`ansible-playbook-properties` for more details and examples.

.. _instance-defaults-properties:

Instance and Defaults Properties
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

These properties serve three purposes:

* Setting selection criteria when allocating nodes from the pool of available nodes

* Setting attributes on the baremetal node being deployed

* Setting network configuration properties for the deployed nodes

Each ``instances`` entry and the ``defaults`` dict support the following properties:

* ``capabilities``: Selection criteria to match the node's capabilities

* ``config_drive``: Add data and first-boot commands to the config-drive passed
  to the node. See :ref:`config-drive`

* ``hostname``: If this complies with the ``hostname_format`` pattern then
  other properties will apply to the node allocated to this hostname.
  Otherwise, this allows a custom hostname to be specified for this node.
  (Cannot be specified in ``defaults``)

* ``image``: Image details to deploy with. See :ref:`image-properties`

* ``managed``: Boolean to determine whether the instance is actually
  provisioned with metalsmith, or should be treated as preprovisioned.

* ``name``: The name of a node to deploy this instance on (Cannot be specified
  in ``defaults``)

* ``nics``: (**DEPRECATED:** Replaced by ``networks`` in Wallaby) List of
  dicts representing requested NICs. See :ref:`nics-properties`

* ``networks``: List of dicts representing instance networks. See
  :ref:`networks-properties`

* ``network_config``: Network configuration details. See
  :ref:`network-config-properties`

* ``profile``: Selection criteria to use :doc:`./profile_matching`

* ``provisioned``: Boolean to determine whether this node is provisioned or
  unprovisioned. Defaults to ``true``, ``false`` is used to unprovision a node.
  See :ref:`scaling-down`

* ``resource_class``: Selection criteria to match the node's resource class,
  defaults to ``baremetal``. See :doc:`./profile_matching`

* ``root_size_gb``: Size of the root partition in GiB, defaults to 49

* ``swap_size_mb``: Size of the swap partition in MiB, if needed

* ``traits``: A list of traits as selection criteria to match the node's ``traits``

.. _image-properties:

Image Properties
________________

* ``href``: Glance image reference or URL of the root partition or whole disk
  image. URL schemes supported are ``file://``, ``http://``, and ``https://``.
  If the value is not a valid URL, it is assumed to be a Glance image reference

* ``checksum``: When the ``href`` is a URL, the ``MD5`` checksum of the root
  partition or whole disk image

* ``kernel``: Glance image reference or URL of the kernel image (partition images only)

* ``ramdisk``: Glance image reference or URL of the ramdisk image (partition images only)


.. _networks-properties:

Networks Properties
___________________

The ``instances`` ``networks`` property supports a list of dicts, one dict per
network.

* ``network``: Neutron network to create the port for this network:

* ``fixed_ip``: Specific IP address to use for this network

* ``network``: Neutron network to create the port for this network

* ``subnet``: Neutron subnet to create the port for this network

* ``port``: Existing Neutron port to use instead of creating one

* ``vif``: When ``true`` the network is attached as VIF (virtual-interface) by
  metalsmith/ironic. When ``false`` the baremetal provisioning workflow creates
  the Neutron API resource, but no VIF attachment happens in metalsmith/ironic.
  (Typically only the provisioning network (``ctlplane``) has this set to
  ``true``.)

By default there is one network representing

.. code-block:: yaml

  - network: ctlplane
    vif: true

Other valid network entries would be

.. code-block:: yaml

  - network: ctlplane
    fixed_ip: 192.168.24.8
    vif: true
  - port: overcloud-controller-0-ctlplane
  - network: internal_api
    subnet: internal_api_subnet01

.. _network-config-properties:

Network Config Properties
_________________________

The ``network_config`` property contains os-net-config related properties.

* ``template``: The ansible j2 nic config template to use when
  applying node network configuration. (default:
  ``templates/net_config_bridge.j2``)

* ``physical_bridge_name``:  Name of the OVS bridge to create for accessing
  external networks. (default: ``br-ex``)

* ``public_interface_name``: Which interface to add to the public bridge
  (default: ``nic1``)

* ``network_config_update``: Whether to apply network configuration changes,
  on update or not. Boolean value. (default: ``false``)

* ``net_config_data_lookup``: Per node and/or per node group os-net-config nic
  mapping config.

* ``default_route_network``: The network to use for the default route (default:
  ``ctlplane``)

* ``networks_skip_config``: List of networks that should be skipped when
  configuring node networking

* ``dns_search_domains``: A list of DNS search domains to be added (in order)
  to resolv.conf.

* ``bond_interface_ovs_options``: The ovs_options or bonding_options string for
  the bond interface. Set things like lacp=active and/or bond_mode=balance-slb
  for OVS bonds or like mode=4 for Linux bonds using this option.

.. _nics-properties:

Nics Properties
_______________

The ``instances`` ``nics`` property supports a list of dicts, one dict per NIC.

* ``fixed_ip``: Specific IP address to use for this NIC

* ``network``: Neutron network to create the port for this NIC

* ``subnet``: Neutron subnet to create the port for this NIC

* ``port``: Existing Neutron port to use instead of creating one

By default there is one NIC representing

.. code-block:: yaml

  - network: ctlplane

Other valid NIC entries would be

.. code-block:: yaml

  - subnet: ctlplane-subnet
    fixed_ip: 192.168.24.8
  - port: overcloud-controller-0-ctlplane

.. _config-drive:

Config Drive
____________

The ``instances`` ``config_drive`` property supports two sub-properties:

* ``cloud_config``: Dict of cloud-init `cloud-config`_ data for tasks to run on
  node boot. A task specified in an ``instances`` ``cloud_config`` will
  overwrite a task with the same name in in ``defaults`` ``cloud_config``.

* ``meta_data``: Extra metadata to include with the config-drive cloud-init
  metadata. This will be added to the generated metadata ``public_keys``,
  ``uuid``, ``name``, ``hostname``, and ``instance-type`` which is set to
  the role name. Cloud-init makes this metadata available as `instance-data`_.
  A key specified in an ``instances`` ``meta_data`` entry will overwrite the
  same key in ``defaults`` ``meta_data``.

Below are some examples of what can be done with ``config_drive``.

Run arbitrary scripts on first boot:

.. code-block:: yaml

  config_drive:
    cloud_config:
      bootcmd:
        # temporary workaround to allow steering in ConnectX-3 devices
        - echo "options mlx4_core log_num_mgm_entry_size=-1" >> /etc/modprobe.d/mlx4.conf
        - /sbin/dracut --force

Enable and configure ntp:

.. code-block:: yaml

  config_drive:
    cloud_config:
      enabled: true
      ntp_client: chrony  # Uses cloud-init default chrony configuration

Allow root ssh login (for development environments only):

.. code-block:: yaml

  config_drive:
    cloud_config:
      ssh_pwauth: true
      disable_root: false
      chpasswd:
        list: |-
          root:sekrit password
        expire: False

Use values from custom metadata:

.. code-block:: yaml

  config_drive:
    meta_data:
      foo: bar
    cloud_config:
      runcmd:
        - echo The value of foo is `jq .foo < /run/cloud-init/instance-data.json`


.. _ansible-playbook-properties:

Ansible Playbooks
-----------------

The role ``ansible_playbooks`` takes a list of playbook definitions, supporting
the ``playbook`` and ``extra_vars`` sub-properties.

* ``playbook``: The path (relative to the roles definition YAML file) to the
  ansible playbook.

* ``extra_vars``: Extra Ansible variables to set when running the playbook.

.. note:: Playbooks only run if '--network-config' is enabled.

Run arbitrary playbooks:

.. code-block:: yaml

  ansible_playbooks:
    - playbook: a_playbook.yaml
    - playbook: b_playbook.yaml

Run arbitrary playbooks with extra variables defined for one of the playbooks:

.. code-block:: yaml

  ansible_playbooks:
    - playbook: a_playbook.yaml
      extra_vars:
        param1: value1
        param2: value2
    - playbook: b_playbook.yaml

Grow volumes playbook
^^^^^^^^^^^^^^^^^^^^^

After custom playbooks are run, an in-built playbook is run to grow the LVM
volumes of any node deployed with the whole-disk overcloud image
`overcloud-hardened-uefi-full.qcow2`. The implicit `ansible_playbooks` would be:

.. code-block:: yaml

  ansible_playbooks:
    - playbook: /usr/share/ansible/tripleo-playbooks/cli-overcloud-node-growvols.yaml
      extra_vars:
        growvols_args: >
          /=8GB
          /tmp=1GB
          /var/log=10GB
          /var/log/audit=2GB
          /home=1GB
          /var=100%

Each LVM volume is grown by the amount specified until the disk is 100%
allocated, and any remaining space is given to the `/` volume.  In some cases it
may be necessary to specify different `growvols_args`. For example the
`ObjectStorage` role deploys swift storage which stores state in `/srv`, so this
volume needs the remaining space instead of `/var`. The playbook can be
explicitly written to override the default `growvols_args` value, for example:

.. code-block:: yaml

  ansible_playbooks:
    - playbook: /usr/share/ansible/tripleo-playbooks/cli-overcloud-node-growvols.yaml
      extra_vars:
        growvols_args: >
          /=8GB
          /tmp=1GB
          /var/log=10GB
          /var/log/audit=2GB
          /home=1GB
          /var=1GB
          /srv=100%

Set kernel arguments playbook
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Features such as DPDK require that kernel arguments are set and the node is
rebooted before the network configuration is run. A playbook is provided to
allow this. Here it is run with the default variables set:

.. code-block:: yaml

  ansible_playbooks:
    - playbook: /usr/share/ansible/tripleo-playbooks/cli-overcloud-node-kernelargs.yaml
      extra_vars:
        kernel_args: ''
        reboot_wait_timeout: 900
        defer_reboot: false
        tuned_profile: 'throughput-performance'
        tuned_isolated_cores: ''

Here is an example for a specific DPDK deployment:

.. code-block:: yaml

  ansible_playbooks:
    - playbook: /usr/share/ansible/tripleo-playbooks/cli-overcloud-node-kernelargs.yaml
      extra_vars:
        kernel_args: 'default_hugepagesz=1GB hugepagesz=1G hugepages=64 intel_iommu=on iommu=pt'
        tuned_isolated_cores: '1-11,13-23'
        tuned_profile: 'cpu-partitioning'

.. _deploying-the-overcloud:

Deploying the Overcloud
-----------------------

This example assumes that the baremetal provision configuration file has the
filename ``~/overcloud_baremetal_deploy.yaml`` and the resulting deployed
server environment file is ``~/overcloud-baremetal-deployed.yaml``. It also
assumes overcloud networks are pre-deployed using the ``openstack overcloud
network provision`` command and the deployed networks environment file is
``~/overcloud-networks-deployed.yaml``.

The baremetal nodes are provisioned with the following command::

  openstack overcloud node provision \
    --stack overcloud \
    --network-config \
    --output ~/overcloud-baremetal-deployed.yaml \
    ~/overcloud_baremetal_deploy.yaml

.. note:: Removing the ``--network-config`` argument will disable the management
          of non-VIF networks and post node provisioning network configuration
          with os-net-config via ansible.

The overcloud can then be deployed using the output from the provision command::

  openstack overcloud deploy \
    -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-environment.yaml \
    -e ~/overcloud-networks-deployed.yaml \
    -e ~/templates/vips-deployed-environment.yaml \
    -e ~/overcloud-baremetal-deployed.yaml \
    --deployed-server \
    --disable-validations \ # optional, see note below
    # other CLI arguments

.. note::
    The validation which is part of `openstack overcloud node
    provision` may fail with the default overcloud image unless the
    Ironic node has more than 4 GB of RAM. For example, a VBMC node
    provisioned with 4096 MB of memory failed because the image size
    plus the reserved RAM size were not large enough (Image size: 4340
    MiB, Memory size: 3907 MiB).

Viewing Provisioned Node Details
--------------------------------

The commands ``openstack baremetal node list`` and ``openstack baremetal node
show`` continue to show the details of all nodes, however there are some new
commands which show a further view of the provisioned nodes.

The `metalsmith`_ tool provides a unified view of provisioned nodes, along with
allocations and neutron ports. This is similar to what Nova provides when it
is managing baremetal nodes using the Ironic driver. To list all nodes
managed by metalsmith, run::

  metalsmith list

The baremetal allocation API keeps an association of nodes to hostnames,
which can be seen by running::

  openstack baremetal allocation list

The allocation record UUID will be the same as the Instance UUID for the node
which is allocated. The hostname can be seen in the allocation record, but it
can also be seen in the ``openstack baremetal node show`` property
``instance_info``, ``display_name``.


Scaling the Overcloud
---------------------

Scaling Up
^^^^^^^^^^

To scale up an existing overcloud, edit ``~/overcloud_baremetal_deploy.yaml``
to increment the ``count`` in the roles to be scaled up (and add any desired
``instances`` entries) then repeat the :ref:`deploying-the-overcloud` steps.

.. _scaling-down:

Scaling Down
^^^^^^^^^^^^

Scaling down is done with the ``openstack overcloud node delete`` command but
the nodes to delete are not passed as command arguments.

To scale down an existing overcloud edit
``~/overcloud_baremetal_deploy.yaml`` to decrement the ``count`` in the roles
to be scaled down, and also ensure there is an ``instances`` entry for each
node being unprovisioned which contains the following:

* The ``name`` of the baremetal node to remove from the overcloud

* The ``hostname`` which is assigned to that node

* A ``provisioned: false`` property

* A YAML comment explaining the reason for making the node unprovisioned (optional)

For example the following would remove ``overcloud-compute-1``

.. code-block:: yaml

  - name: Compute
    count: 1
    instances:
    - hostname: overcloud-compute-0
      name: node10
      # Removed from deployment due to disk failure
      provisioned: false
    - hostname: overcloud-compute-1
      name: node11

Then the delete command will be called with ``--baremetal-deployment``
instead of passing node arguments::

  openstack overcloud node delete \
  --stack overcloud \
  --baremetal-deployment ~/overcloud_baremetal_deploy.yaml

Before any node is deleted, a list of nodes to delete is displayed
with a confirmation prompt.

What to do when scaling back up depends on the situation. If the scale-down
was to temporarily remove baremetal which is later restored, then the
scale-up can increment the ``count`` and set ``provisioned: true`` on nodes
which were previously ``provisioned: false``. If that baremetal node is not
going to be re-used in that role then the ``provisioned: false`` can remain
indefinitely and the scale-up can specify a new ``instances`` entry, for
example

.. code-block:: yaml

  - name: Compute
    count: 2
    instances:
    - hostname: overcloud-compute-0
      name: node10
      # Removed from deployment due to disk failure
      provisioned: false
    - hostname: overcloud-compute-1
      name: node11
    - hostname: overcloud-compute-2
      name: node12


Unprovisioning All Nodes
^^^^^^^^^^^^^^^^^^^^^^^^

After ``openstack overcloud delete`` is called, all of the baremetal nodes
can be unprovisioned without needing to edit
``~/overcloud_baremetal_deploy.yaml`` by running the unprovision command with
the ``--all`` argument::

  openstack overcloud node unprovision --all \
    --stack overcloud \
    --network-ports \
    ~/overcloud_baremetal_deploy.yaml

.. note:: Removing the ``--network-ports`` argument will disable the management
          of non-VIF networks, non-VIF ports will _not_ be deleted in that
          case.

.. _metalsmith: https://docs.openstack.org/metalsmith/

.. _cloud-config: https://cloudinit.readthedocs.io/en/latest/topics/examples.html

.. _instance-data: https://cloudinit.readthedocs.io/en/latest/topics/instancedata.html
