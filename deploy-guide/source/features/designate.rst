Deploying DNSaaS (Designate)
============================

Because some aspects of a Designate deployment are specific to the environment
in which it is deployed, there is some additional configuration required
beyond just including an environment file.  The following instructions will
explain this configuration.

First, make a copy of the ``designate-config.yaml`` environment.

.. note:: For HA deployments, there is a separate ``designate-config-ha.yaml``
          file that should be used instead.

::

    cp /usr/share/openstack-tripleo-heat-templates/environments/designate-config.yaml .

This file contains a sample pool configuration which must be edited to match
the intended environment.  Each section has comments that explain how to
configure it.

.. TODO(bnemec): Include these notes in the sample environments, or figure
                 out how to pull these values from the Heat stack and populate
                 the file automatically.

* ``ns_records``: There should be one of these for each node running designate,
  and they should point at the public IP of the node.
* ``nameservers``: There should be one of these for each node running BIND.
  The ``host`` value should be the public IP of the node.
* ``targets``: There should be one of these for each node running BIND.  Each
  target has the following attributes which need to be configured:

  * ``masters``: There should be one of these for each node running
    designate-mdns.  The ``host`` value should be the public IP of the node.
  * ``options``: This specifies where the target BIND instance will be
    listening.  ``host`` should be the public IP of the node, and
    ``rndc_host`` should be the internal_api IP of the node.

Because this configuration requires the node IPs to be known ahead of time, it
is necessary to use predictable IPs.  Full details on configuring those can be
found at :doc:`../provisioning/node_placement`.

Only the external (public) and internal_api networks need to be predictable
for Designate.  The following is an example of the addresses that need to be
set::

    parameter_defaults:
      ControllerIPs:
        external:
        - 10.0.0.51
        - 10.0.0.52
        - 10.0.0.53
        internal_api:
        - 172.17.0.251
        - 172.17.0.252
        - 172.17.0.253

Include ``enable-designate.yaml``, ``ips-from-pool.yaml``, and either
``designate-config.yaml`` or ``designate-config-ha.yaml`` in the deploy
command::

    openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/enable-designate.yaml -e ips-from-pool.yaml -e designate-config.yaml [...]
