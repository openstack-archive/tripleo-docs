Deploying with RHSM
===================

Summary
-------

Starting in the Queens release, it is possible to use Ansible to apply the
RHSM (Red Hat Subscription Management) configuration.

Instead of the pre_deploy rhel-registration script, the new RHSM service will
allow our operators to:

#. deploy advanced RHSM configurations, where each role can have their own
   repositories for example.

#. use config-download mechanism so operators can run the playbooks at anytime
   after the deployment, in case RHSM parameters have changed.


Using RHSM
----------
To enable deployment with Ansible and config-download pass the additional arg
to the deployment command::

    openstack overcloud deploy \
        <other cli args> \
        -e /usr/share/openstack-tripleo-heat-templates/environments/config-download-environment.yaml \
        --config-download
        -e ~/rhsm.yaml

The ``rhsm.yaml`` environment enables mapping the OS::TripleO::Services::Rhsm to
the extraconfig service::

    resource_registry:
      OS::TripleO::Services::Rhsm: /usr/share/openstack-tripleo-heat-templates/extraconfig/services/rhsm.yaml
    parameter_defaults:
      RhsmVars:
        rhsm_repos:
          - rhel-7-server-rpms
          - rhel-7-server-extras-rpms
          - rhel-ha-for-rhel-7-server-rpms
          - rhel-7-server-openstack-13-rpms
          - rhel-7-server-rhceph-3-mon-rpms
          - rhel-7-server-rhceph-3-tools-rpms
        rhsm_activation_key: 'secrete-key'

In some advanced use cases, you might want to configure RHSM for a specific role::

    parameter_defaults:
      ComputeHCIParameters:
        RhsmVars:
          rhsm_repos:
            - rhel-7-server-rpms
            - rhel-7-server-extras-rpms
            - rhel-ha-for-rhel-7-server-rpms
            - rhel-7-server-openstack-13-rpms
            - rhel-7-server-rhceph-3-osd-rpms
            - rhel-7-server-rhceph-3-mon-rpms
            - rhel-7-server-rhceph-3-tools-rpms
          rhsm_activation_key: 'anothersecrete-key'

In that case, all nodes deployed with ComputeHCI will be configured with these RHSM parameters.

Scale-down the Overcloud
------------------------
The automatic unsubscription isn't currently supported and before scaling down the Overcloud,
the operator will have to run this playbook against the host(s) that will be removed.
Example when we want to remove 2 compute nodes::

    - hosts:
         - overcloud-compute47
         - overcloud-compute72
      vars:
        rhsm_username: bob.smith@acme.com
        rhsm_password: my_secret
        rhsm_state: absent
      roles:
         - openstack.redhat-subscription

The playbook needs to be executed prior to the actual scale-down.

Transition from previous method
-------------------------------

The previous method ran a script called rhel-registration during
pre_deploy step, which is located in the ``extraconfig/pre_deploy/rhel-registration``
folder. While the script is still working, you can perform a
migration to the new service by replacing the parameters used in
rhel-registration with RhsmVars and switching the resource_registry
from::

    resource_registry:
      OS::TripleO::NodeExtraConfig: rhel-registration.yaml

To::

    resource_registry:
      OS::TripleO::Services::Rhsm: /usr/share/openstack-tripleo-heat-templates/extraconfig/services/rhsm.yaml

The following table shows a migration path from the old
rhe-registration parameters to the new RhsmVars:

+------------------------------+------------------------------+
| rhel-registration script     | rhsm with Ansible (RhsmVars) |
+==============================+==============================+
| rhel_reg_activation_key      | rhsm_activation_key          |
+------------------------------+------------------------------+
| rhel_reg_auto_attach         | rhsm_autosubscribe           |
+------------------------------+------------------------------+
| rhel_reg_sat_url             | rhsm_satellite_url           |
+------------------------------+------------------------------+
| rhel_reg_org                 | rhsm_org_id                  |
+------------------------------+------------------------------+
| rhel_reg_password            | rhsm_password                |
+------------------------------+------------------------------+
| rhel_reg_repos               | rhsm_repos                   |
+------------------------------+------------------------------+
| rhel_reg_pool_id             | rhsm_pool_ids                |
+------------------------------+------------------------------+
| rhel_reg_user                | rhsm_username                |
+------------------------------+------------------------------+
| rhel_reg_method              | rhsm_method                  |
+------------------------------+------------------------------+
| rhel_reg_http_proxy_host     | rhsm_rhsm_proxy_hostname     |
+------------------------------+------------------------------+
| rhel_reg_http_proxy_port     | rhsm_rhsm_proxy_port         |
+------------------------------+------------------------------+
| rhel_reg_http_proxy_username | rhsm_rhsm_proxy_user         |
+------------------------------+------------------------------+
| rhel_reg_http_proxy_password | rhsm_rhsm_proxy_password     |
+------------------------------+------------------------------+


More about the Ansible role
---------------------------

TripleO is using the Ansible role_ for Red Hat Subscription.

.. _role: https://github.com/openstack/ansible-role-redhat-subscription

You can find all available parameters in this repository.
