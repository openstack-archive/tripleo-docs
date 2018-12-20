.. _update_network_configuration_post_deploymenet:

Updating network configuration on the Overcloud after a deployment
==================================================================

By default, subsequent change(s) made to network configuration templates
(bonding options, mtu, bond type, etc) are not applied on existing nodes when
the overcloud stack is updated.

.. Warning:: Network configuration updates are disabled by default to avoid
             issues that may arise from network reconfiguration.

             Network configuration updates should only be enabled when needed.

To push an updated network configuration add ``UPDATE`` to list of actions set
in the ``NetworkDeploymentActions`` parameter. (The default is ``['CREATE']``,
to enable network configuration on stack update it must be changed to:
``['CREATE','UPDATE']``.)

* Enable update of the network configuration for all roles by adding the
  following to ``parameter_defaults`` in an environment file::

    parameter_defaults:
      NetworkDeploymentActions: ['CREATE','UPDATE']

* Limit the network configuration update to nodes of a specific role by using a
  role-specific parameter, i.e: ``{role.name}NetworkDeploymentActions``. For
  example to update the network configuration on the nodes in the Compute role,
  add the following to ``parameter_defaults`` in an environment file::

    parameter_defaults:
      ComputeNetworkDeploymentActions: ['CREATE','UPDATE']
