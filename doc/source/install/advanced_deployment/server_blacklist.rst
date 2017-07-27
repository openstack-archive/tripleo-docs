Blacklisting Servers from Heat Deployments
==========================================
Servers can be excluded from getting any updated Heat deployments by adding
them to a blacklist parameter called ``DeploymentServerBlacklist``.


Setting the blacklist
---------------------
The ``DeploymentServerBlacklist`` parameter is a list of Heat server names.

Write a new environment file, or add the parameter value to an existing
custom environment file and pass the file to the deployment command::

  parameter_defaults:
    DeploymentServerBlacklist:
      - overcloud-compute-0
      - overcloud-compute-1
      - overcloud-compute-2

.. note::
  The server names in the parameter value are the names according to Heat, not
  the actual server hostnames.

Any servers in the list will be blacklisted by Heat from getting any updated
triggered deployments from Heat. After the stack operation completes, any
blacklisted servers will be unchanged. The blacklisted servers also could have
been powered off, or had their ``os-collect-config`` agents stopped during the
stack operation.

The blacklist can be used during scale out operations or for isolating changes
to certain servers only.

.. warning::
  Blacklisting servers should be done with caution, and only when the operator
  understands that the requested change can be applied with a blacklist in
  effect.

  It would be possible to blacklist servers in ways to create a hung stack in
  Heat, or a misconfigured overcloud. For example, cluster configuration
  changes that would need to be applied to all members of a pacemaker cluster
  would not support blacklisting certain cluster members since it
  could result is a misconfigured cluster.

Clearing the blacklist
----------------------
When clearing the blacklist for subsequent stack operations, an empty parameter
value must be sent with the deploy command. It is not sufficient to simply omit
the parameter since Heat will use the previously saved value.

Send an empty list value to force Heat to clear the blacklist::

  parameter_defaults:
    DeploymentServerBlacklist: []
