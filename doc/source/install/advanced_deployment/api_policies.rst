Configuring API access policies
===============================

Each OpenStack service, has its own role-based access policies.
They determine which user can access which resources in which way,
and are defined in the service’s policy.json file.

.. Warning::

   While editing policy.json is supported, modifying the policy can
   have unexpected side effects and is not encouraged.

|project| supports custom API access policies through parameters in
TripleO Heat Templates.
To enable this feature, you need to use some parameters to enable
the custom policies on the services you want.

Creating an environment file and adding the following arguments to your
``openstack overcloud deploy`` command will do the trick::

  $ cat ~/nova-policies.yaml
  parameter_defaults:
    NovaApiPolicies: { nova-context_is_admin: { key: 'compute:get_all', value: '' } }

  -e nova-policies.yaml

In this example, we allow anyone to list Nova instances, which is very insecure but
can be done with this feature.
