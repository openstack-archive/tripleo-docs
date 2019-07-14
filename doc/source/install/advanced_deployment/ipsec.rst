.. _ipsec:

Deploying with IPSec
====================

As of the Queens release, it is possible to encrypt communications within the
internal network by setting up IPSec tunnels configured by TripleO.

There are several options that TripleO provides deployers whose requirements call
for encrypting everything in the network. For example, TLS Everywhere has been
supported since the Pike release. This method requires the deployer
to procure a CA server on a separate node. FreeIPA is recommended for this.

However, there are cases where a deployers authorized CA does not have an
interface that can automatically request certificates. Furthermore, it may
not be possible to add another node to the network for various other reasons.
For these cases, IPSec is a viable, alternative solution.

.. note:: For more information on TLS Everywhere, please see
          :doc:`tls_everywhere`.

IPSec thus, provides an alternative to TLS Everywhere. With IPSec the encryption
happens on the IP layer, and not over TCP (as happens in TLS). As a result, the
services will communicate with each other over standard 'http', and not
actually know that the underlying traffic is being encrypted. This means that
the services do not require any extra configuration.

Solution Overview
-----------------

The current IPSec solution relies on `Libreswan`_, which is already available
in RHEL and CentOS, and is driven and configured via Ansible.

There are two types of tunnels configured in the overcloud:

* **node-to-node tunnels**: These tunnels are a simple 1-to-1 tunnel between the ip
  addresses of two nodes on the same network. This results in a tunnel to each node
  in the overcloud for each network that the node is connected to.

* **Virtual IP tunnels**: These are tunnels from each Virtual IP address and
  each node that can contact to them. The node hosting the VIP will open a tunnel
  for any host in the specific network that can properly authenticate. This
  makes the configuration simpler, allows for easier scaling, and assists
  deployers to securely communicate with the Virtual IP from hosts
  or services that are not necessarily managed by TripleO.

Authentication is currently done via a Pre-Shared Key (PSK) which all the nodes
share. However, future iterations will add more authentication methods to the
deployment.

Currently, the default encryption method is AES using GCM with a block size of
128 bits. Changing this default will be talked about in a further section.

To handle the moving of a Virtual IP from one node to another (VIP failover),
we also deploy a pacemaker resource agent per VIP. This resource agent is in
charge of creating the tunnel when the VIP is set in a certain node, and
removing the tunnel when it moves to another node.

.. note:: One important thing to note is that we set tunnels for every network
          except the control plane network. The reason for this is that in our
          testing, setting up tunnels for this network cuts of the
          communication between the overcloud nodes and the undercloud. We thus
          rely on the fact that Ansible uses SSH to communicate with the
          overcloud nodes, thus, still giving the deployment secure
          communications.

Deployment
----------

.. note:: Please note that the IPSec deployment depends on Ansible being used
          for the overcloud deployment. For more information on this, please
          see :doc:`ansible_config_download`

.. note:: Also note that the IPSec deployment assumes that you're using network
          isolation. For more information on this, please see
          :doc:`network_isolation`

To enable IPSec tunnels for the overcloud, you need to use the following
environment file::

    /usr/share/openstack-tripleo-heat-templates/environments/ipsec.yaml

With this, your deployment command will be similar to this::

    openstack overcloud deploy \
        ...
        -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
        -e /home/stack/templates/network-environment.yaml \
        -e /usr/share/openstack-tripleo-heat-templates/environments/ipsec.yaml

.. note:: For the Queens release, you need to specify the config-download
          related parameters yourself::

              openstack overcloud deploy \
                  ...
                  -e /usr/share/openstack-tripleo-heat-templates/environments/config-download-environment.yaml \
                  --config-download \
                  ...

To change the default encryption algorithm, you can use an environment file
that looks as follows::

    parameter_defaults:
      IpsecVars:
        ipsec_algorithm: 'aes_gcm256-null'

The ``IpsecVars`` option is able to change any parameter in the tripleo-ipsec
ansible role.

.. note:: For more information on the algorithms that Libreswan suppports,
          please check the `Libreswan documentation`_

.. note:: For more information on the available parameters, check the README
          file in the `tripleo-ipsec repository`_.


Verification
------------

To verify that the IPSec tunnels were setup correctly after the overcloud
deployment is done, you'll need to do several things:

* Log into each node

* In each node, check the output of ``ipsec status`` with sudo or root
  privileges. This will show you the status of all the tunnels that are set up
  in the node.

  - The line starting with "Total IPsec connections" should show
    that there are active connections.
  - The Security Associations should be all authenticated::

        000 IKE SAs: total(23), half-open(0), open(0), authenticated(23), anonymous(0)
        000 IPsec SAs: total(37), authenticated(37), anonymous(0)

    Note that this number will vary depending on the number of networks and
    nodes you have.

* The configuration files generated can be found in the ``/etc/ipsec.d``
  directory.

  - They conveniently all start with the prefix **overcloud-** and
    you could list them with the following command::

        ls /etc/ipsec.d/overcloud-*.conf

  - The PSKs can be found with the following command::

        ls /etc/ipsec.d/overcloud-*.secrets

  - You can find the connection names from the ``*.conf`` files.

  - To view the status of a certain connection, you can use the aforementioned
    ``ipsec status`` command, and filter the result, searching for the specific
    connection name. For instance, in the node that's hosting the Internal API
    VIP, you can view the status of the tunnels for that VIP with the following
    command::

        ipsec status | grep overcloud-internal_api-vip-tunnel

* To view the status of the resource agents, you can use ``pcs status``.

  - The IPSEC-related agents will have a name with the **tripleo-ipsec**
    prefix.

  - Note that the resource agents for the tunnels are collocated with the IP
    resource agents. This is enforced through a collocation rule in pacemaker.
    You can verify this by running the ``pcs constraint`` command.

.. note:: To get further explanations for understanding the output of the
          ``ipsec status`` command, you can read the `Libreswan wiki entry on
          the subject`_.

.. References

.. _Libreswan: https://libreswan.org/
.. _Libreswan documentation: https://libreswan.org/man/ipsec.conf.5.html
.. _Libreswan wiki entry on the subject: https://libreswan.org/wiki/How_to_read_status_output
.. _tripleo-ipsec repository: https://github.com/openstack/tripleo-ipsec/blob/master/README.md
