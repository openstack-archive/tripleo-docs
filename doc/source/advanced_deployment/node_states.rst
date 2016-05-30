Bare Metal Node States
======================

This document provides a brief explanation of the bare metal node states that
TripleO uses or might use. Please refer to `the Ironic documentation
<http://docs.openstack.org/developer/ironic/>`_ for more details.

enroll
------

With recent versions of the bare metal API (starting with 1.11), nodes begin
their life in a state called ``enroll``. Nodes in this state are not available
for deployment, nor for most of other actions. Ironic does not touch such nodes
in any way.

Starting with the Newton release, TripleO node registration command allows
to import nodes in this state instead of the default ``available``.
To do so pass the option ``--initial-state=enroll``::

    openstack baremetal import --initial-state=enroll instackenv.json

Then move the nodes to manageable_ state and eventually to available_.

manageable
----------

To make nodes alive an operator uses ``manage`` provisioning action to move
nodes to ``manageable`` state. During this transition the power and management
credentials (IPMI, SSH, etc) are validated to ensure that nodes in
``manageable`` state are actually manageable by Ironic. This state is still not
available for deployment.  With nodes in this state an operator can execute
various pre-deployment actions, such as introspection, RAID configuration, etc.
So to sum it up, nodes in ``manageable`` state are being configured before
exposing them into the cloud.

Nodes get into ``manageable`` state automatically. The ``manage`` action
can be used to bring nodes already moved to available_ state back to
``manageable`` for configuration::

    ironic node-set-provision-state <NAME OR UUID> manage


available
---------

The last step before the deployment is to make nodes ``available`` using the
``provide`` provisioning action. Such nodes are exposed to nova, and can be
deployed to at any moment. No long-running configuration actions should be run
in this state.

.. note::
   The TripleO introspection command ``openstack baremetal introspection bulk
   start`` moves ``available`` nodes to manageable_ state automatically
   before and moves them back after a successful introspection. However, nodes
   which failed introspection stay in ``manageable`` state and must be
   reintrospected or made ``available`` manually::

    ironic node-set-provision-state <NAME OR UUID> provide
