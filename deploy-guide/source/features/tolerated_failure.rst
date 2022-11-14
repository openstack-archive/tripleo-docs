Tolerate deployment failures
============================

When proceeding to large scale deployments, it happens very often to have
infrastructure problems such as network outages, wrong configurations applied
on hardware, hard drive issues, etc.

It is unpleasant to deploy hundred of nodes and only have a few of them which
failed. On most of large-scale use-cases, deployers would not care about
these nodes, as long as the cloud can already be used with the successfully
deployed servers.

For that purpose, it is possible in |project|  to specify a percentage value,
per role, that will tell how much failures we tolerate.

Example: We deploy 50 compute nodes with the role "Compute". If I set the
following environment, my deployment will go until the end even if up to 5
nodes fail to deploy::

  parameter_defaults:
    ComputeMaxFailPercentage: 10

At the end of the deployment, a report will be printed and if nodes failed to
deploy, it'll be shown like this::

  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  ~~~~~~~~~~~~~~~~~~~~~~~~~~ State Information ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  ~~~~~~~~~ Number of nodes which did not deploy successfully: 3 ~~~~~~~~~~~~~~
   This or these node(s) failed to deploy: compute3, compute24, compute29
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If one or multiple node(s) failed to deploy, the tripleoclient return code
won't be 0 and an error will be printed with a Python trace. Very often the
problem can be read from the Ansible logs by searching for the nodes which
didn't deploy successfully.

If you want to target all the compute nodes in our deployment and you have more
than one role to deploy computes, then you'll probably want to allocate one
value per role and distribute it based on your expectations and needs.

.. Warning::

   For now, this only works for the execution of the deployment steps
   from config-download playbooks. Minor updates, major upgrades, fast forward
   upgrades and baremetal provisioning operations aren't supported yet, but
   will certainly be in the future.
