.. _ssl:

Deploying with SSL
==================

TripleO supports deploying with SSL on the public OpenStack endpoints as well
as deploying SSL in the internal network for most services.

This document will focus on deployments using network isolation.  For more
details on deploying that way, see
:doc:`../advanced_deployment/network_isolation`

Undercloud SSL
--------------

To enable SSL with an automatically generated certificate, you must set
the ``generate_service_certificate`` option in ``undercloud.conf`` to
``True``. This will generate a certificate in ``/etc/pki/tls/certs`` with
a file name that follows the following pattern::

    undercloud-[undercloud_public_vip].pem

This will be a PEM file in a format that HAProxy can understand (see the
HAProxy documentation for more information on this).

This option for auto-generating certificates uses Certmonger to request
and keep track of the certificate. So you will see a certificate with the
ID of ``undercloud-haproxy-public-cert`` in certmonger (you can check this
by using the ``sudo getcert list`` command). Note that this also implies
that certmonger will manage the certificate's lifecycle, so when it needs
renewing, certmonger will do that for you.

The default is to use Certmonger's ``local`` CA. So using this option has
the side-effect of extracting Certmonger's local CA to a PEM file that is
located in the following path::

    /etc/pki/ca-trust/source/anchors/cm-local-ca.pem

This certificate will then be added to the trusted CA chain, since this is
needed to be able to use the undercloud's endpoints with that certificate.

However, it is possible to not use certmonger's ``local`` CA. For
instance, one can use FreeIPA as the CA by setting the option
``certificate_generation_ca`` in ``undercloud.conf`` to have 'IPA' as the
value. This requires the undercloud host to be enrolled as a FreeIPA
client, and to define a ``haproxy/<undercloud FQDN>@<KERBEROS DOMAIN>``
service in FreeIPA. We also need to set the option ``service_principal``
to the relevant value in ``undercloud.conf``. Finally, we need to set the
public endpoints to use FQDNs instead of IP addresses, which will also
then use an FQDN for the certificate.

To enable an FQDN for the certificate we set the ``undercloud_public_vip``
to the desired hostname in ``undercloud.conf``. This will in turn also set
the keystone endpoints to relevant values.

Note that the ``generate_service_certificate`` option doesn't take into
account the ``undercloud_service_certificate`` option and will have
precedence over it.

To enable SSL on the undercloud with a pre-created certificate, you must
set the ``undercloud_service_certificate`` option in ``undercloud.conf``
to an appropriate certificate file.  Important:
The certificate file's Common Name *must* be set to the value of
``undercloud_public_vip`` in undercloud.conf.

If you do not have a trusted CA signed certificate file, you can alternatively
generate a self-signed certificate file using the following command::

    openssl genrsa -out privkey.pem 2048

The next command will prompt for some identification details.  Most of these don't
matter, but make sure the ``Common Name`` entered matches the value of
``undercloud_public_vip`` in undercloud.conf::

    openssl req -new -x509 -key privkey.pem -out cacert.pem -days 365

Combine the two files into one for HAProxy to use.  The order of the
files in this command matters, so do not change it::

    cat cacert.pem privkey.pem > undercloud.pem

Move the file to a more appropriate location and set the SELinux context::

    sudo mkdir /etc/pki/instack-certs
    sudo cp undercloud.pem /etc/pki/instack-certs
    sudo semanage fcontext -a -t etc_t "/etc/pki/instack-certs(/.*)?"
    sudo restorecon -R /etc/pki/instack-certs

``undercloud_service_certificate`` should then be set to
``/etc/pki/instack-certs/undercloud.pem``.

Add the self-signed CA certificate to the undercloud system's trusted
certificate store::

   sudo cp cacert.pem /etc/pki/ca-trust/source/anchors/
   sudo update-ca-trust extract

Overcloud SSL
-------------

Certificate and Public VIP Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The public VIP of the deployed overcloud needs to be predictable in order for
the SSL certificate to be configured properly.  There are two options for
configuring the certificate:

#. The certificate's Common Name can be set to the IP of the public
   VIP.  In this case, the Common Name must match *exactly*.  If the public
   VIP is ``10.0.0.1``, the certificate's Common Name must also be ``10.0.0.1``.
   Wild cards will not work.

#. The overcloud endpoints can be configured to point at
   a DNS name.  In this case, the certificate's Common Name must be valid
   for the FQDN of the overcloud endpoints.  Wild cards should work fine.
   Note that this option also requires pre-configuration of the specified
   DNS server with the appropriate FQDN and public VIP.

In either case, the public VIP must be explicitly specified as part of the
deployment configuration.  This can be done by passing an environment file
like the following::

    parameter_defaults:
        PublicVirtualFixedIPs: [{'ip_address':'10.0.0.1'}]

.. note:: If network isolation is not in use, the ControlFixedIPs parameter
          should be set instead.

The selected IP should fall in the specified allocation range for the public
network.

Certificate Details
~~~~~~~~~~~~~~~~~~~

.. This admonition is intentionally left class-less because it is only used
   on the SSL page.
.. admonition:: Self-Signed SSL

   It is not recommended that the self-signed certificate is trusted; So for
   this purpose, having a self-signed CA certificate is a better choice. In
   this case we will trust the self-signed CA certificate, and not the leaf
   certificate that will be used for the public VIP; This leaf certificate,
   however, will be signed by the self-signed CA.

   For the self-signed case, just the predictable public VIP method will
   be documented, as DNS configuration is outside the scope of this document.

   Generate a private key::

       openssl genrsa -out overcloud-ca-privkey.pem 2048

   Generate a self-signed CA certificate.  This command will prompt for some
   identifying information.  Most of the fields don't matter, and the CN should
   not be the same as the one we'll give the leaf certificate. You can choose a
   CN for this such as "TripleO CA"::

       openssl req -new -x509 -key overcloud-ca-privkey.pem \
            -out overcloud-cacert.pem -days 365

   Add the self-signed CA certificate to the undercloud's trusted certificate
   store.  Adding this file to the overcloud nodes will be discussed later::

       sudo cp overcloud-cacert.pem /etc/pki/ca-trust/source/anchors/
       sudo update-ca-trust extract

   Generate the leaf certificate request and key that will be used for the
   public VIP. Again, Most of the fields don't matter, but this is where the
   Common Name must be set to the fixed IP in the external network allocation
   pool::

       openssl req -newkey rsa:2048 -days 365 \
            -nodes -keyout server-key.pem -out server-req.pem

   Process the server RSA key::

       openssl rsa -in server-key.pem -out server-key.pem

   Sign the leaf certificate with the CA certificate and generate the
   certificate::

       openssl x509 -req -in server-req.pem -days 365 \
             -CA overcloud-cacert.pem -CAkey overcloud-ca-privkey.pem \
             -set_serial 01 -out server-cert.pem

   The following is a list of which files generated in the previous steps
   map to which parameters in the SSL environment files::

       overcloud-cacert.pem: SSLRootCertificate
       server-key.pem: SSLKey
       server-cert.pem: SSLCertificate

The contents of the private key and certificate files must be provided
to Heat as part of the deployment command.  To do this, there is a sample
environment file in tripleo-heat-templates with fields for the file contents.

It is generally recommended that the original copy of tripleo-heat-templates
in ``/usr/share/openstack-tripleo-heat-templates`` not be altered, since it
could be overwritten by a package update at any time.  Instead, make a copy
of the templates::

    cp -r /usr/share/openstack-tripleo-heat-templates ~/ssl-heat-templates

Then edit the enable-tls.yaml environment file.  If using the location from the
previous command, the correct file would be in
``~/ssl-heat-templates/environments/ssl/enable-tls.yaml``.  Insert the contents of
the private key and certificate files in their respective locations.

.. admonition:: Stable Branch
   :class: stable

   In the Pike release the SSL environment files in the top-level environments
   directory were deprecated and moved to the ``ssl`` subdirectory as
   shown in the example paths.  For Ocata and older the paths will still need
   to refer to the top-level environments.  The filenames are all the same, but
   the ``ssl`` directory must be removed from the path.

.. note:: The certificate and key will be multi-line values, and all of the lines
          must be indented to the same level.

An abbreviated version of how the file should look::

    parameter_defaults:
        SSLCertificate: |
          -----BEGIN CERTIFICATE-----
          MIIDgzCCAmugAwIBAgIJAKk46qw6ncJaMA0GCSqGSIb3DQEBCwUAMFgxCzAJBgNV
          [snip]
          sFW3S2roS4X0Af/kSSD8mlBBTFTCMBAj6rtLBKLaQbIxEpIzrgvp
          -----END CERTIFICATE-----
    [rest of file snipped]

``SSLKey`` should look similar, except with the value of the private key.

``SSLIntermediateCertificate`` can be set in the same way if the certificate
signer uses an intermediate certificate.  Note that the ``|`` character must
be added as in the other values to indicate that this is a multi-line value.

When using a self-signed certificate or a signer whose certificate is
not in the default trust store on the overcloud image it will be necessary
to inject the certificate as part of the deploy process.  This can be done
with the environment file ``~/ssl-heat-templates/environments/ssl/inject-trust-anchor.yaml``.
Insert the contents of the signer's root CA certificate in the appropriate
location, in a similar fashion to what was done for the certificate and key
above.

.. admonition:: Self-Signed SSL
   :class: selfsigned

   Injecting the root CA certificate is required for self-signed SSL.  The
   correct value to use is the contents of the ``overcloud-cacert.pem`` file.

DNS Endpoint Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~

When deploying with DNS endpoint addresses, two additional parameters must be
passed in a Heat environment file.  These are ``CloudName`` and ``DnsServers``.
To do so, create a new file named something like ``cloudname.yaml``::

    parameter_defaults:
        CloudName: my-overcloud.my-domain.com
        DnsServers: 10.0.0.100

Replace the values with ones appropriate for the target environment.  Note that
the configured DNS server(s) must have an entry for the configured ``CloudName``
that matches the public VIP.

In addition, when a DNS endpoint is being used, make sure to pass the
``tls-endpoints-public-dns.yaml`` environment to your deploy command.  See the examples
below.

Deploying an SSL Environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``enable-tls.yaml`` file must always be passed to use SSL on the public
endpoints.  Depending on the specific configuration, additional files will
also be needed.  Examples of the necessary parameters for different scenarios
follow.

IP-based certificate::

    -e ~/ssl-heat-templates/environments/ssl/enable-tls.yaml -e ~/ssl-heat-templates/environments/ssl/tls-endpoints-public-ip.yaml

Self-signed IP-based certificate::

    -e ~/ssl-heat-templates/environments/ssl/enable-tls.yaml -e ~/ssl-heat-templates/environments/ssl/tls-endpoints-public-ip.yaml -e ~/ssl-heat-templates/environments/ssl/inject-trust-anchor.yaml

DNS-based certificate::

    -e ~/ssl-heat-templates/environments/ssl/enable-tls.yaml -e ~/ssl-heat-templates/environments/ssl/tls-endpoints-public-dns.yaml -e ~/cloudname.yaml

Self-signed DNS-based certificate::

    -e ~/ssl-heat-templates/environments/ssl/enable-tls.yaml -e ~/ssl-heat-templates/environments/ssl/tls-endpoints-public-dns.yaml -e ~/cloudname.yaml -e ~/ssl-heat-templates/environments/ssl/inject-trust-anchor.yaml

.. note:: It is also possible to get the public certificate from a CA. See
          :doc:`../advanced_deployment/tls_everywhere`

Getting the overcloud to trust CAs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As mentioned above, it is possible to get the overcloud to trust a CA by using
the ``~/ssl-heat-templates/environments/ssl/inject-trust-anchor.yaml`` environment
and adding the necessary details there. However, that environment has the
restriction that it will only allow you to inject one CA. However, the
file ``~/ssl-heat-templates/environments/ssl/inject-trust-anchor-hiera.yaml`` is an
alternative that actually supports as many CA certificates as you need.

.. note:: This is only available since Newton. Older versions of TripleO don't
          support this.

This file is a template of how you should fill the ``CAMap`` parameter which is
passed via parameter defaults. It looks like this::

    CAMap:
      first-ca-name:
        content: |
          The content of the CA cert goes here
      second-ca-name:
        content: |
          The content of the CA cert goes here

where ``first-ca-name`` and ``second-ca-name`` will generate the files
``first-ca-name.pem`` and ``second-ca-name.pem`` respectively. These files will
be stored in the ``/etc/pki/ca-trust/source/anchors/`` directory in each node
of the overcloud and will be added to the trusted certificate chain of each of
the nodes. You must be careful that the content is a block string in yaml and
is in PEM format.

.. include:: ./tls_everywhere.rst
