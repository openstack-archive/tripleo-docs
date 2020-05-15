.. _tls-introduction:

TLS Introduction
================

Depending on your deployment's security requirements, you might be required to
encrypt network traffic. TripleO helps you accomplish this by supporting
various TLS deployment options. Let's start by understanding the different ways
we can deploy TLS.

The first option is to only encrypt traffic between clients and public
endpoints. This approach results in fewer certificates to manage, and we refer
to it as *public TLS*. Public endpoints, in this sense, are endpoints only
exposed to end-users. Traffic between internal endpoints is not encrypted.

The second option leverages TLS for all endpoints in the entire deployment,
including the overcloud, undercloud, and any systems that natively support TLS.
We typically refer to this approach as *TLS-everywhere* because we use TLS
everywhere we can, encrypting as much network traffic as possible. Certificate
management automation is critical with this approach because the number of
certificates scales linearly with the number of services in your deployment.
TripleO uses several components to help ease the burden of managing
certificates. This option is desirable for deployments susceptible to industry
regulation or those who have a higher security risk. Healthcare,
telecommunications, and the public sector are but a few industries that make
extensive use of *TLS-everywhere*. You can think of *public TLS* as a subset of
what *TLS-everywhere* offers.

TripleO uses the following components to implement *public TLS* and
*TLS-everywhere*.

Certmonger
----------

`Certmonger`_ is a daemon that helps simplify certificate management between
endpoints and certificate authorities (CAs). You can use it to generate key
pairs and certificate signing requests (CSRs). It can self-sign CSRs or send
them to external CAs for signing. Certmonger also tracks the expiration of each
certificate it manages. When a certificate is about to expire, Certmonger
requests a new certificate, updates it accordingly, and may restart a service.
This automation keeps the node enrolled as a client of the certificate
authority so that you don’t have to update hundreds, or thousands, of
certificates manually. Certmonger runs on each node that provides an endpoint
in your deployment.

.. _Certmonger: https://pagure.io/certmonger

FreeIPA
-------

`FreeIPA`_ is a multi-purpose system that includes a certificate authority
(DogTag Certificate System), LDAP (389 Directory Server), MIT Kerberos, NTP
server, and DNS. TripleO uses all of these subsystems to implement TLS across
OpenStack. For example, if you use FreeIPA in your deployment, you can sign
CSRs with DogTag, as opposed to self-signing CSRs with certmonger locally.

FreeIPA runs on a supplemental node in your deployment, and it is kept separate
from other infrastructure.

.. _FreeIPA: https://www.freeipa.org/page/Main_Page

Installing FreeIPA
~~~~~~~~~~~~~~~~~~

Similar to setting up the undercloud node, you need to set the hostname
properly for the FreeIPA server. For this example, let's assume we're using
``example.com`` as the domain name for the deployment.::

    sudo hostnamectl set-hostname ipa.example.come
    sudo hostnamectl set-hostname --transient ipa.example.com

Collect and install the FreeIPA packages::

    sudo yum install -y ipa-server ipa-server-dns

Configure FreeIPA::

    sudo ipa-server-install --realm EXAMPLE.COM /
    --ds-password $DIRECTORY_MANAGER_PASSWORD /
    --admin-password $ADMIN_PASSWORD /
    --hostname ipa.example.com /
    --setup-dns /
    --auto-forwarders /
    --auto-reverse /
    --unattended

By default, FreeIPA does not public it's Certificate Revocation List (CRL)
on startup.  As the CRL is retrieved when the overcloud nodes retrieve
certificates from FreeIPA, we should configure it to do so and restart
FreeIPA.::

    sed -i -e \
    's/ca.crl.MasterCRL.publishOnStart=.*/ca.crl.MasterCRL.publishOnStart=true/' \
    /etc/pki/pki-tomcat/ca/CS.cfg
    systemctl restart ipa

Finally, if your IPA server is not at 4.8.5 or higher, you will need to add an
ACL to allow for the proper generation of certificates with a IP SAN.::

    cat << EOF | ldapmodify -x -D "cn=Directory Manager" -w $DIRECTORY_MANAGER_PASSWORD
    dn: cn=dns,dc=example,dc=com
    changetype: modify
    add: aci
    aci: (targetattr = "aaaarecord || arecord || cnamerecord || idnsname || objectclass || ptrrecord")(targetfilter = "(&(objectclass=idnsrecord)(|(aaaarecord=*)(arecord=*)(cnamerecord=*)(ptrrecord=*)(idnsZoneActive=TRUE)))")(version 3.0; acl "Allow hosts to read DNS A/AAA/CNAME/PTR records"; allow (read,search,compare) userdn = "ldap:///fqdn=*,cn=computers,cn=accounts,dc=example,dc=com";)
    EOF

Please refer to ``ipa-server-install --help`` for specifics on each argument or
reference the `FreeIPA documentation`_. The directions above are only a guide.
You may need to adjust certain values and configuration options to use FreeIPA,
depending on your requirements.

.. _FreeIPA documentation: https://www.freeipa.org/page/Documentation

Novajoin
--------

`Novajoin`_ is a vendor data service that extends nova's config drive
functionality and you use it when you want to deploy *TLS-everywhere*. When the
undercloud creates new nodes for the overcloud, novajoin creates a host entry
in FreeIPA to enable the overcloud node to enroll as a FreeIPA client.

If you want to use novajoin, you must have nova deployed in your undercloud.
Novajoin isn't supported for deployments :doc:`deployed_server`.

Novajoin was introduced in the Queens release and is supported through Train.
The `tripleo-ipa`_ project, described below, effectively replaced novajoin in
the Train release.

.. _Novajoin: https://opendev.org/x/novajoin

tripleo-ipa
-----------

`tripleo-ipa`_ is a collection of Ansible roles used to integrate FreeIPA into
TripleO deployments and you use it when you want to deploy *TLS-everywhere*.
These playbooks support deployments using nova and ironic in the undercloud as
well as :doc:`deployed_server`. This project was introduced in Train and
effectively replaces the novajoin metadata service.

We recommend using tripleo-ipa for all *TLS-everywhere* deployments as of the
Train release. In a future release, we will update TripleO to only support
tripleo-ipa as the default method for configuring and deploying
*TLS-everywhere*.

.. _tripleo-ipa: https://opendev.org/x/tripleo-ipa
