Introspecting a Single Node
===========================

In addition to bulk introspection, you can also introspect nodes one by one.
When doing so, you must take care to set the correct node states manually.
Use ``baremetal node show UUID`` command to figure out whether nodes
are in ``manageable`` or ``available`` state. For all nodes in ``available``
state, start with putting a node to ``manageable`` state (see
:doc:`node_states` for details)::

    baremetal node manage <UUID>

Then you can run introspection::

    baremetal introspection start UUID

This command won't poll for the introspection result, use the following command
to check the current introspection state::

    baremetal introspection status UUID

Repeat it for every node until you see ``True`` in the ``finished`` field.
The ``error`` field will contain an error message if introspection failed,
or ``None`` if introspection succeeded for this node.

Do not forget to make nodes available for deployment afterwards::

    baremetal node provide <UUID>
