Bare Metal Node States
======================

This document provides a brief explanation of the bare metal node states that
TripleO uses or might use. Please refer to `the Ironic documentation
<https://docs.openstack.org/ironic/>`_ for more details.

enroll
------

In a typical Ironic workflow nodes begin their life in a state called ``enroll``.
Nodes in this state are not available for deployment, nor for most of other
actions. Ironic does not touch such nodes in any way.

In the TripleO workflow the nodes start their life in the ``manageable`` state
and only see the ``enroll`` state if their power management fails to validate::

        openstack overcloud import instackenv.json

Nodes can optionally be introspected in this step by passing the --provide flag
which will progress them through through the manageable_ state and eventually to
the available_ state ready for deployment.

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

The ``manage`` action
can be used to bring nodes from enroll_ to ``manageable`` or nodes already
moved to available_ state back to ``manageable`` for configuration::

    openstack baremetal node manage <NAME OR UUID>

available
---------

The last step before the deployment is to make nodes ``available`` using the
``provide`` provisioning action. Such nodes are exposed to nova, and can be
deployed to at any moment. No long-running configuration actions should be run
in this state.

.. note::
   Nodes which failed introspection stay in ``manageable`` state and must be
   reintrospected or made ``available`` manually::

    openstack baremetal node provide <NAME OR UUID>
