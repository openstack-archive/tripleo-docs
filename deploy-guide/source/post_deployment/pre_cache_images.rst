.. _precache_image:

Pre-caching images on Compute Nodes
===================================

Fetching an image from Glance is often the most time consuming step when
booting an instance. This can be particularly significant in cases where the
data must traverse a high latency or limited bandwidth network, for example
with :doc:`../features/distributed_compute_node`.

An ansible playbook is available to ensure images are already cached in cases
where instance creation times must be minimized.

.. admonition:: Ussuri
   :class: ussuri

   Since Ussuri Nova also provides an API to pre-cache images on Compute nodes.
   See the `Nova Image pre-caching documentation <https://docs.openstack.org/nova/ussuri/admin/image-caching.html#image-pre-caching>`_.

.. note:: The Nova Image Cache is not used when using Ceph RBD for Glance images and Nova ephemeral disk. See `Nova Image Caching documentation <https://docs.openstack.org/nova/ussuri/admin/image-caching.html>`_.

Image Cache Cleanup
-------------------

The nova-compute service remains responsible for cleaning up old unused images
on a compute node.
A periodic job examines each of the images that are not currently used by an
instance on the host.
If an image is older than the configured maximum age it will be removed.

When an image is pre-cached the modification time is set to the current
time. This effectively sets the image age back to 0.
Therefore the pre-caching task should be repeated on an interval that is less
than the maximum image age to ensure images remain cached.

Configuring the maximum image age
---------------------------------

The default maximum image age is 86400 seconds (24 hours).
This can be increased for all computes by setting `NovaImageCacheTTL` in the
deployment parameters::

    [stack@undercloud-0 ~]$ cat nova-cache-environment.yaml
    parameter_defaults:
      # Set the max image age to 30 days
      NovaImageCacheTTL: 2592000

Alternatively `NovaImageCacheTTL` can be set for individual compute roles::

    [stack@undercloud-0 ~]$ cat nova-cache-environment.yaml
    parameter_defaults:
      # Set the max image age to 30 days for the ComputeSite1 role
      ComputeSite1Parameters:
         NovaImageCacheTTL: 2592000
      # Set the max image age to 7 days for the ComputeSite2 role
      ComputeSite2Parameters:
         NovaImageCacheTTL: 604800
      # Any other Compute roles default to 86400

.. _cache_all_computes:

Pre-caching a list of images on all Compute nodes
-------------------------------------------------

Get an ansible inventory for the stack name (default `overcloud`):

.. admonition:: Wallaby
   :class: wallaby

   ``tripleo-ansible-inventory`` is deprecated as of Wallaby.

   .. code-block:: bash

      [stack@undercloud-0 ~]$ mkdir -p inventories
      [stack@undercloud-0 ~]$ . stackrc
      (undercloud) [stack@undercloud-0 ~]$ tripleo-ansible-inventory \
        --plan overcloud --static-yaml-inventory inventories/inventory.yaml

.. code-block:: bash

   [stack@undercloud-0 ~]$ find ~/overcloud-deploy/*/config-download \
     -name tripleo-ansible-inventory.yaml |\
     while read f; do cp $f inventories/$(basename $(dirname $f)).yaml; done

Determine the list of image IDs to pre-cache::

    [stack@undercloud-0 ~]$ . overcloudrc
    (overcloud) [stack@undercloud-0 ~]$ openstack image list
    +--------------------------------------+---------+--------+
    | ID                                   | Name    | Status |
    +--------------------------------------+---------+--------+
    | 07bc2424-753b-4f65-9da5-5a99d8383fe6 | image_0 | active |
    | d5187afa-c821-4f22-aa4b-4e76382bef86 | image_1 | active |
    +--------------------------------------+---------+--------+

Add the image ids to an argument file for the ansible playbook::

    (overcloud) [stack@undercloud-0 ~]$ cat <<EOF > nova_cache_args.yml
    tripleo_nova_image_cache_images:
      - id: 07bc2424-753b-4f65-9da5-5a99d8383fe6
      - id: d5187afa-c821-4f22-aa4b-4e76382bef86
    EOF

Source the overcloud rc file to provide the necessary credentials for image download::

    [stack@undercloud-0 ~]$ . overcloudrc

Run the `tripleo_nova_image_cache` playbook::

    (overcloud) [stack@undercloud-0 ~]$ ansible-playbook -i inventories --extra-vars "@nova_cache_args.yml" /usr/share/ansible/tripleo-playbooks/tripleo_nova_image_cache.yml

    PLAY [TripleO Nova image cache management] ***************************************************************************************************************************************************************************************************

    TASK [tripleo-nova-image-cache : Cache image 07bc2424-753b-4f65-9da5-5a99d8383fe6] ***********************************************************************************************************************************************************
    changed: [compute-0]
    changed: [compute-1]

    TASK [tripleo-nova-image-cache : Cache image d5187afa-c821-4f22-aa4b-4e76382bef86] ***********************************************************************************************************************************************************
    changed: [compute-0]
    changed: [compute-1]

.. note:: If the image already exists in cache then no change is reported however the image modification time is updated

.. warning:: The ansible `forks` config option (default=5) will affect the number of concurrent image downloads. Consider the load on the image service if adjusting this.

Multi-stacks inventory
----------------------

When a multi-stack deployment is used, such as in
:doc:`../features/distributed_compute_node` and
:doc:`../features/deploy_cellv2`, a merged inventory allows images to be cached
on all compute nodes with a single playbook run.

For each deployed stack, its ansible inventory is generated in
``overcloud-deploy/<stack>/config-download/tripleo-ansible-inventory.yaml``.
Collect all inventories under the ``inventories`` directory:

.. admonition:: Wallaby
   :class: wallaby

   ``tripleo-ansible-inventory`` is deprecated as of Wallaby. A multi-stack
   inventory can be created by specifying a comma separated list of stacks:

   .. code-block:: bash

      [stack@undercloud-0 ~]$ mkdir -p inventories
      [stack@undercloud-0 ~]$ . stackrc
      (undercloud) [stack@undercloud-0 ~]$ tripleo-ansible-inventory \
        --plan overcloud,site1,site2 \
        --static-yaml-inventory inventories/multiinventory.yaml

.. code-block:: bash

   [stack@undercloud-0 ~]$ mkdir -p inventories
   [stack@undercloud-0 ~]$ find ~/overcloud-deploy/*/config-download \
     -name tripleo-ansible-inventory.yaml |\
     while read f; do cp $f inventories/$(basename $(dirname $f)).yaml; done

When all inventory files are stored in a single directory, ansible merges it.
The playbook can then be run once as in :ref:`cache_all_computes` to pre-cache on all compute nodes.

.. _scp_distribution:

Pre-caching on one node and distributing to remaining nodes
-----------------------------------------------------------

In the case of a :doc:`../features/distributed_compute_node` it may be desirable to transfer an image to a single compute node at a remote site and then redistribute it from that node to the remaining compute nodes.
The SSH/SCP configuration that exists between the compute nodes to support cold migration/resize is reused for this purpose.

.. warning:: SSH/SCP is inefficient over high latency networks. The method should only be used when the compute nodes targeted by the playbook are all within the same site. To ensure this is the case set tripleo_nova_image_cache_plan to the stack name of the site. Multiple runs of ansible-playbook are then required, targeting a different site each time.

To enable this simply set `tripleo_nova_image_cache_use_proxy: true` in the arguments file.
The image is distributed from the first compute node by default. To use a specific compute node also set `tripleo_nova_image_cache_proxy_hostname`.

For example::

    (central) [stack@undercloud-0 ~]$ cat <<EOF > dcn1_nova_cache_args.yml
    tripleo_nova_image_cache_use_proxy: true
    tripleo_nova_image_cache_proxy_hostname: dcn1-compute-1
    tripleo_nova_image_cache_images:
      - id: 07bc2424-753b-4f65-9da5-5a99d8383fe6
    tripleo_nova_image_cache_plan: dcn1
    EOF

    (central) [stack@undercloud-0 ~]$ ansible-playbook -i inventories --extra-vars "@dcn1_nova_cache_args.yml" /usr/share/ansible/tripleo-playbooks/tripleo_nova_image_cache.yml

    PLAY [TripleO Nova image cache management] ***************************************************************************************************************************************************************************************************

    TASK [tripleo-nova-image-cache : Show proxy host] ********************************************************************************************************************************************************************************************
    ok: [dcn-compute-0] => {
        "msg": "Proxy host is dcn-compute-1"
    }

    TASK [tripleo-nova-image-cache : Cache image 07bc2424-753b-4f65-9da5-5a99d8383fe6] ***********************************************************************************************************************************************************
    skipping: [dcn1-compute-0]
    changed: [dcn1-compute-1]

    TASK [tripleo-nova-image-cache : Cache image (via proxy) 07bc2424-753b-4f65-9da5-5a99d8383fe6] ***********************************************************************************************************************************************
    skipping: [dcn1-compute-1]
    changed: [dcn1-compute-0]

    (central) [stack@undercloud-0 ~]$ cat <<EOF > dcn2_nova_cache_args.yml
    tripleo_nova_image_cache_use_proxy: true
    tripleo_nova_image_cache_images:
      - id: 07bc2424-753b-4f65-9da5-5a99d8383fe6
    tripleo_nova_image_cache_plan: dcn2
    EOF

    (central) [stack@undercloud-0 ~]$ ansible-playbook -i inventories --extra-vars "@dcn2_nova_cache_args.yml" /usr/share/ansible/tripleo-playbooks/tripleo_nova_image_cache.yml

    PLAY [TripleO Nova image cache management] ***************************************************************************************************************************************************************************************************
    ...
    ...

.. warning:: The ansible `forks` config option (default=5) will affect the number of concurrent SCP transfers. Consider the load on the proxy compute node if adjusting this.
