TLS everywhere for the overcloud
--------------------------------

It is possible to deploy most of the services to use TLS for communications in
the internal network as well. This, however, needs several more certificates
than the public approach, with the number being dependant on the number of
nodes in your deployment. This complicates certificate and key management to
the extent where it's not sustainable to have the deployer inject all the
certificates and keys needed and then have to handle all their lifecycles.
Then, we have to take into account that a certificate revocation might be
needed at some point. So, from both the maintenance and security standpoints
this is not sustainable.

For the aforementioned reasons, we decided to rely on `certmonger`_ to get the
certificates from an actual CA. Certmonger will do the certificate requests and
do the certificate renewals when it's needed, thus reducing the maintenance
burden.

FreeIPA has been chosen as the default CA. Certmonger already has a plugin for
it, and it has the added value that, besides being able to automatically
provide the certificates we need, we can also keep track of the nodes and have
an identity for them.

.. note:: The default CA can be overriden via the **CertmongerCA** parameter.
          However, the CA has to be something that certmonger understands, so
          there are adjustments to be done. For more information on how to
          change it you can consult the `certmonger`_ documentation

Communicating with the CA (FreeIPA) requires the nodes to have proper
credentials, and these credentials also need to be transported into the
overcloud nodes in a secure manner. To address this, we use a
`Nova vendordata plugin`_ called `novajoin`_ whose purpose is to detect the
nodes that are created by nova, register or join them in FreeIPA and provide an
OTP that the node can subsequently use to enroll to FreeIPA. The node
subsequently enrolls by loading the vendordata-provided JSON via the
config-drive, which ends up executing a cloud-init script to do this. With the
node enrolled, certificates can be requested securely. Novajoin can also
receive extra entries from nova metadata to create extra principals that the
services will need. These create service principals for services such as httpd,
mysql and haproxy, and are used to requests the certificates for the specific
service users with the correct SubjectAltNames.

The following are instructions assuming the default CA, which is FreeIPA.

CA installation
~~~~~~~~~~~~~~~

As mentioned before, the default CA is FreeIPA. Due to port conflicts between
FreeIPA and some OpenStack services, it's not trivial to install it in the
undercloud. On the other hand, often folks already have a FreeIPA server
installed on-premise.

To have a minimal FreeIPA installation with the required functionalities for TLS
everywhere in TripleO, you can run the following command::

    ipa-server-install -U -r `hostname -d|tr "[a-z]" "[A-Z]"` \
                   -p $DirectoryManagerPassword -a $AdminPassword \
                   --hostname `hostname -f` \
                   --ip-address=$FreeIPAIP \
                   --setup-dns --auto-forwarders --auto-reverse

There are several things to note from the aforementioned command:

* This command assumes that your kerberos realm is the same as your host's
  domain name. This is a fairly normal setup, but note that it's possible to
  use different values.

* The $DirectoryManagerPassword and $AdminPassword don't need to be the same.
  However, please remember $AdminPassword since it's what you will use in a
  subsequent command to set up the undercloud.

* $FreeIPAIP needs to be an IP address that's accessible from both the
  undercloud and the overcloud nodes.

* We enable DNS because FreeIPA will serve as the nameserver for both the
  undercloud and overcloud nodes. This simplifies the installation as the nodes
  can autodiscover FreeIPA's capabilities (for enrollment and for the CA)
  through DNS.

* Assuming the FreeIPA node has working DNS, setting the ``--auto-forwarders``
  flag adds the node's configured nameservers and forwards DNS queries to them
  if needed. Note that you can also specify manually the forwarders for the DNS
  setup through the ``--forwarder`` option.

For more information on FreeIPA and its capabilities, please consult the
`FreeIPA documentation`_

.. note:: You could also try out the `IdM container`_ as an alternative to the
          FreeIPA installation. Just make sure to enable DNS and follow the
          considerations listed above.

CA setup
~~~~~~~~

The undercloud needs to be enrolled to FreeIPA, and we need to create some
extra privileges/permissions to be used by the novajoin services. Assuming
there's an already existing FreeIPA installation, we can use a script that
comes with the python-novajoin package::

    sudo /usr/libexec/novajoin-ipa-setup \
        --principal admin \
        --password < freeipa admin password > \
        --server < freeipa server hostname > \
        --realm < overcloud cloud domain in upper case > \
        --domain < overcloud cloud domain > \
        --hostname < undercloud hostname > \
        --precreate

This command will give us a One-Time Password (OTP) that we can then use
for the undercloud enrollment. We can also specify the command to output
the OTP into a file by using the ``--otp-file`` option.

.. note:: This can be run from either the undercloud node itself or the FreeIPA
          node. Just note that the example provided is using the FreeIPA admin
          credentials. This can be done using another principal if it has the
          approprite permissions.

Undercloud setup
~~~~~~~~~~~~~~~~

Now that we have an OTP we can either deploy or update the undercloud. The
following settings in **undercloud.conf** will get the undercloud to enroll
to FreeIPA and deploy novajoin::

    enable_novajoin = True
    ipa_otp = < OTP provided by the novajoin-ipa-setup script >

The undercloud fully-qualified hostname should also be set in
**undercloud.conf**, since this is the host that will be used to enroll
to FreeIPA. It should match the one provided in the novajoin-ipa setup
script. We can set it like this::

    undercloud_hostname = < undercloud FQDN >

It is useful to have FreeIPA set as the DNS server since this will
automatically: discover the FreeIPA server hostname, set up the Kerberos
realm/domain automatically, and it will set the DNS entries of the
overcloud nodes once they're deployed. We can set it in **undercloud.conf**
with the following setting::

    undercloud_nameservers = < FreeIPA IP >

.. note:: This takes a comma-separated list, so we can set another nameserver
          with this configuration option.

The undercloud's neutron must also use the appropriate domain that it will
advertise to the overcloud nodes. Assuming we're using *example.com* as the
domain for the overcloud nodes. We must set the following::

    overcloud_domain_name = example.com

.. note:: The value for ``overcloud_domain_name`` in **undercloud.conf** must
          match the value for ``CloudDomain`` that we'll set for the overcloud
          deployment in the following section.

With these settings, do the following command to set the desired configurations
and enable novajoin::

    openstack undercloud install

Overcloud deployment
~~~~~~~~~~~~~~~~~~~~

The TLS-everywhere setup only works with FQDNs so we need to set the
appropriate entries for the overcloud endpoints as well as setting an
appropriate domain for the nodes that matches the one we set for FreeIPA.
We can do this by overriding some parameters via ``parameter_defaults``.
Assuming that the domain for our cloud is *example.com* We'll set the
following in a file we'll call **cloud-names.yaml** which we'll include
in our overcloud deploy command::

    parameter_defaults:
      CloudDomain: example.com
      CloudName: overcloud.example.com
      CloudNameInternal: overcloud.internalapi.example.com
      CloudNameStorage: overcloud.storage.example.com
      CloudNameStorageManagement: overcloud.storagemgmt.example.com
      CloudNameCtlplane: overcloud.ctlplane.example.com

.. note:: The value for ``CloudDomain`` must match the value for
          ``overcloud_domain_name`` that was configured in **undercloud.conf**

As with our undercloud, we also want the overcloud nodes' name server to point
to FreeIPA. We can do this by setting the ``DnsServers`` parameter via
parameter_defaults. You can create an environment file for it, however, since
you probably are deploying with network isolation, you can already set this
parameter in the **network-environment.yaml** file that's referenced in
:doc:`../advanced_deployment/network_isolation`. So that setting would look
like this::

    parameter_defaults:
      ...
      DnsServers: ["< FreeIPA IP >"]
      ...

Remembering that optionally we can set other nameservers with this parameter.

You'll also need to add set the DNS server for the ctlplane network to point to
FreeIPA as described in :ref:`basic-deployment-cli-configure-namserver`.

To tell the overcloud deployment to deploy the keystone endpoints (and
references) using DNS names instead of IPs, we need to add the following
environment to our overcloud deployment::

    ~/ssl-heat-templates/environments/tls-everywhere-endpoints-dns.yaml

Finally, to enable TLS in the internal network, we need to use the following
environment::

    ~/ssl-heat-templates/environments/enable-internal-tls.yaml

This will set the appropriate resources that enable the certificate requests
via certmonger and create the appropriate service principals for kerberos
(which are used by FreeIPA).

.. note:: As part of the enrollment, FreeIPA is set as a trusted CA, so we
   don't need to do any extra steps for this.

.. note:: When using Ceph, you need to tell it to use FQDNs instead of
          shortnames. One can do this by passing the following in a heat
          environment file::

              parameter_defaults:
                CephAnsibleExtraConfig:
                  mon_use_fqdn: true

Classic public TLS and certmonger-based internal TLS
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**enable-internal-tls.yaml** will be used for the internal network
endpoints. One can still use the **enable-tls.yaml** environment for the
public endpoints if a specific certificate for the public endpoints is needed.

The arguments for a deployment using injected certificates for the public
endpoints, and certmonger-provided certificates for the internal endpoints
look like the following::

    openstack overcloud deploy \
        ...
        -e ~/ssl-heat-templates/environments/tls-everywhere-endpoints-dns.yaml \
        -e ~/ssl-heat-templates/environments/enable-tls.yaml \
        -e ~/ssl-heat-templates/environments/enable-internal-tls.yaml \
        -e ~/cloud-names.yaml

Certmonger-based public and Internal TLS
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

It is also possible to get all your certificates from a CA. For this you
need to include the
**environments/services/haproxy-public-tls-certmonger.yaml** environment
file.

To do a deployment with both public and internal endpoints using
certificates provided by certmonger, we would need to issue a command similar
to the following::

    openstack overcloud deploy \
        ...
        -e ~/ssl-heat-templates/environments/tls-everywhere-endpoints-dns.yaml \
        -e ~/ssl-heat-templates/environments/services/haproxy-public-tls-certmonger.yaml \
        -e ~/ssl-heat-templates/environments/enable-internal-tls.yaml \
        -e ~/cloud-names.yaml

.. References

.. _certmonger: https://pagure.io/certmonger
.. _Nova vendordata plugin: https://docs.openstack.org/developer/nova/vendordata.html
.. _novajoin: https://github.com/openstack/novajoin
.. _FreeIPA documentation: https://www.freeipa.org/page/Documentation
.. _IdM container: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/using_containerized_identity_management_services/
