Multiple Overclouds from a Single Undercloud
============================================

TripleO can be used to deploy multiple Overclouds from a single Undercloud
node.

In this scenario, a single Undercloud deploys and manages multiple Overclouds
as unique Heat stacks, with no stack resources shared between them. This can
be useful for environments where having a 1:1 ratio of Underclouds and
Overclouds creates an unmanageable amount of overhead, such as edge cloud
deployments.

Requirements
------------

All Overclouds must be deployed in the same tenant (admin) on the Undercloud.
If using Ironic for baremetal provisioning, all Overclouds must be on the same
provisioning network.


Undercloud Deployment
---------------------

Deploy the Undercloud :doc:`as usual <../deployment/install_undercloud>`.

First Overcloud
---------------

The first Overcloud can be deployed as usual using the :doc:`cli <../deployment/install_overcloud>`.

Deploying Additional Overclouds
-------------------------------

Additional Overclouds can be deployed by specifying a new stack name and any
necessary parameters in a new deployment plan. Networks for additional
overclouds must be defined as :doc:`custom networks <./custom_networks>`
with ``name_lower`` and ``service_net_map_replace`` directives for each
overcloud to have unique networks in the resulting stack.

If your first cloud was named ``overcloud`` and had the following
``network_data.yaml``::

    cat overcloud/network_data.yaml
     - name: InternalApi
       name_lower: internal_api_cloud_1
       service_net_map_replace: internal_api
       vip: true
       vlan: 201
       ip_subnet: '172.17.0.0/24'
       allocation_pools: [{'start': '172.17.0.4', 'end': '172.17.0.250'}]

You would create a new ``network_data.yaml`` with unique ``name_lower`` values
and VLANs for each network, making sure to specify ``service_net_map_replace``::

    cat overcloud-two/network_data.yaml
     - name: InternalApi
       name_lower: internal_api_cloud_2
       service_net_map_replace: internal_api
       vip: true
       vlan: 301
       ip_subnet: '172.21.0.0/24'
       allocation_pools: [{'start': '172.21.0.4', 'end': '172.21.0.250'}]

Then deploy the second Overcloud as::

    openstack overcloud deploy --templates ~/overcloud-two/templates/ \
     --stack overcloud-two \
     -n ~/overcloud-two/network_data.yaml


Managing Heat Templates
-----------------------

If the Heat templates will be customized for any of the deployed clouds
(undercloud, or any overclouds) they should be copied from
/usr/share/openstack-tripleo-heat-templates to a new location before being
modified. Then the location would be specified to the deploy command using
the --templates flag.

The templates could be managed using separate directories for each deployed
cloud::

    ~stack/undercloud-templates
    ~stack/overcloud-templates
    ~stack/overcloud-two-templates

Or by creating a repository in a version control system for the templates
and making a branch for each deployment. For example, using git::

    ~stack/tripleo-heat-templates $ git branch
    * master
      undercloud
      overcloud
      overcloud-two

To deploy to a specific cloud, ensure you are using the correct branch first::

    cd ~stack/tripleo-heat-templates ;\
    git checkout overcloud-two ;\
    openstack overcloud deploy --templates ~stack/tripleo-heat-templates --stack overcloud-two -e $ENV_FILES

Using Pre-Provisioned Nodes
---------------------------

Deploying multiple overclouds with the Ironic baremetal installer currently
requires a shared provisioning network. If this is not possible, you may use
the :ref:`Deployed Servers <deployed_server>` method with routed networks. Ensure that the values
in the ``HostnameMap`` match the stack name being used for each Overcloud.

For example:
``hostnamemap.yaml`` for stack ``overcloud``::

  parameter_defaults:
    HostnameMap:
      overcloud-controller-0: controller-00-rack01
      overcloud-controller-1: controller-01-rack02
      overcloud-controller-2: controller-02-rack03
      overcloud-novacompute-0: compute-00-rack01
      overcloud-novacompute-1: compute-01-rack01
      overcloud-novacompute-2: compute-02-rack01


``hostnamemap.yaml`` for stack ``overcloud-two``::

  parameter_defaults:
    HostnameMap:
      overcloud-two-controller-0: controller-00-rack01
      overcloud-two-controller-1: controller-01-rack02
      overcloud-two-controller-2: controller-02-rack03
      overcloud-two-novacompute-0: compute-00-rack01
      overcloud-two-novacompute-1: compute-01-rack01
      overcloud-two-novacompute-2: compute-02-rack01
