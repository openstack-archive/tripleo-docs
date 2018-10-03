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

Deploy the Undercloud :doc:`as usual <../installation/installation>`.

First Overcloud
---------------

The first Overcloud can be deployed as usual using the :doc:`cli <../basic_deployment/basic_deployment_cli>` or :doc:`UI <../basic_deployment/basic_deployment_ui>`.

Deploying Additional Overclouds
-------------------------------

Additional Overclouds can be deployed by specifying a new stack name and any
parameters that are unique to the new cloud in a new deployment plan.

If your first cloud was named ``overcloud`` and had the following
``network-environment.yaml``::

    grep VlanID overcloud/network-environment.yaml
      InternalApiNetworkVlanID: 201
      StorageNetworkVlanID: 202
      StorageMgmtNetworkVlanID: 203
      TenantNetworkVlanID: 204
      ExternalNetworkVlanID: 100

And your second Overcloud has a different set of VLANs, you would create a new
``network-environment.yaml`` file with the appropriate values::

    grep VlanID overcloud-two/network-environment.yaml
      InternalApiNetworkVlanID: 301
      StorageNetworkVlanID: 302
      StorageMgmtNetworkVlanID: 303
      TenantNetworkVlanID: 304
      ExternalNetworkVlanID: 100

And deploy the second Overcloud as::

    openstack overcloud deploy --templates ~/overcloud-two/templates/ \
     --stack overcloud-two \
     -e ~/overcloud-two/network-environment.yaml


Managing Heat Templates
-----------------------

If the Heat templates will be customized for any of the deployed clouds
(undercloud, or any overclouds) they should be copied from
/usr/share/openstack-tripleo-heat-templates to a new location before being
modified. Then the location would be specified to the deploy command using
the --templates flag.

The templates could be managed using seperate directories for each deployed
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
