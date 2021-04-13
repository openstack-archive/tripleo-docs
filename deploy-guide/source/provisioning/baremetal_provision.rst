.. _baremetal_provision:

Provisioning Baremetal Before Overcloud Deploy
==============================================

Baremetal provisioning is a feature which interacts directly with the
Bare Metal service to provision baremetal before the overcloud is deployed.
This adds a new provision step before the overcloud deploy, and the output of
the provision is a valid :doc:`../features/deployed_server` configuration.

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

.. _instance-defaults-properties:

Instance and Defaults Properties
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

These properties serve two purposes:

* Setting selection criteria when allocating nodes from the pool of available nodes

* Setting attributes on the baremetal node being deployed

Each ``instances`` entry and the ``defaults`` dict support the following properties:

* ``capabilities``: Selection criteria to match the node's capabilities

* ``config_drive``: Add data and first-boot commands to the config-drive passed
  to the node. See :ref:`config-drive`

* ``hostname``: If this complies with the ``hostname_format`` pattern then
  other properties will apply to the node allocated to this hostname.
  Otherwise, this allows a custom hostname to be specified for this node.
  (Cannot be specified in ``defaults``)

* ``image``: Image details to deploy with. See :ref:`image-properties`

* ``name``: The name of a node to deploy this instance on (Cannot be specified
  in ``defaults``)

* ``nics``: List of dicts representing requested NICs. See :ref:`nics-properties`

* ``profile``: Selection criteria to use :doc:`./profile_matching`

* ``provisioned``: Boolean to determine whether this node is provisioned or
  unprovisioned. Defaults to ``true``, ``false`` is used to unprovision a node.
  See :ref:`scaling-down`

* ``resource_class``: Selection criteria to match the node's resource class,
  defaults to ``baremetal``

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
        list: |
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

.. _deploying-the-overcloud:

Deploying the Overcloud
-----------------------

This example assumes that the baremetal provision configuration file has the
filename ``~/overcloud_baremetal_deploy.yaml`` and the resulting deployed
server environment file is ``~/overcloud-baremetal-deployed.yaml``

The baremetal nodes are provisioned with the following command::

  openstack overcloud node provision \
    --stack overcloud \
    --output ~/overcloud-baremetal-deployed.yaml \
    ~/overcloud_baremetal_deploy.yaml

The overcloud can then be deployed using the output from the provision command::

  openstack overcloud deploy \
    -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-environment.yaml \
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

For example the following would remove ``overcloud-controller-1``

.. code-block:: yaml

  - name: Controller
    count: 2
    instances:
    - hostname: overcloud-controller-0
      name: node00
    - hostname: overcloud-controller-1
      name: node01
      # Removed from cluster due to disk failure
      provisioned: false
    - hostname: overcloud-controller-2
      name: node02

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

  - name: Controller
    count: 3
    instances:
    - hostname: overcloud-controller-0
      name: node00
    - hostname: overcloud-controller-1
      name: node01
      # Removed from cluster due to disk failure
      provisioned: false
    - hostname: overcloud-controller-2
      name: node02
    - hostname: overcloud-controller-3
      name: node11

Unprovisioning All Nodes
^^^^^^^^^^^^^^^^^^^^^^^^

After ``openstack overcloud delete`` is called, all of the baremetal nodes
can be unprovisioned without needing to edit
``~/overcloud_baremetal_deploy.yaml`` by running the unprovision command with
the ``--all`` argument::

  openstack overcloud node unprovision --all \
    --stack overcloud \
    ~/overcloud_baremetal_deploy.yaml

.. _metalsmith: https://docs.openstack.org/metalsmith/

.. _cloud-config: https://cloudinit.readthedocs.io/en/latest/topics/examples.html

.. _instance-data: https://cloudinit.readthedocs.io/en/latest/topics/instancedata.html