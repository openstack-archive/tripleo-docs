Deploying TLS-everywhere
========================

Setting up *TLS-everywhere* primarily consists of a few additional steps you
need to take on the undercloud and FreeIPA server. These steps consist of
installing additional packages and enrolling the undercloud host as a FreeIPA
client.

The OpenStack release you are deploying affects which tools you can use to
deploy *TLS-everywhere*. For deployments using Queens through Stein you must
use Novajoin. For deployments using Train or Ussuri, you can use either
Novajoin or tripleo-ipa. For deployments using Victoria or newer releases you
must use tripleo-ipa. Deployments :ref:`deployed_server` must also use
tripleo-ipa. We recommend using tripleo-ipa whenever possible. Let's walk
through each step using both tripleo-ipa and Novajoin.

You can find a primer on the various TLS deployment strategies and components
in the :doc:`tls-introduction` documentation.

TLS-everywhere with tripleo-ipa
-------------------------------

.. note::

    This deployment strategy is only supported on Train and newer releases. If
    you're deploying a version older than Train, you'll need to use Novajoin to
    accomplish *TLS-everywhere*, which is documented below.

Do the following steps before deploying your undercloud.

Configure DNS
~~~~~~~~~~~~~

*TLS-everywhere* deployments use FreeIPA as the DNS server. You need to set the
proper search domain and nameserver on the undercloud. To do this, you need to
know the deployment domain, the domain of the FreeIPA server, and the FreeIPA
server's IP address. For example, if the deployment domain is `example.com` and
the FreeIPA server domain is `bigcorp.com`, you should set the following in
`/etc/resolv.conf`::

    search example.com bigcorp.com
    nameserver $FREEIPA_IP_ADDRESS

This step ensures the undercloud can resolve newly added hosts and services
after TripleO enrolls them as FreeIPA clients. You only need to add both search
domains if they're different. If the FreeIPA server is using the same domain as
the deployment you only need to specify the deployment domain.

Configure FreeIPA
~~~~~~~~~~~~~~~~~

.. note::
    This section assumes you have permissions to make writeable changes to your
    FreeIPA server. If you don't have those permissions or direct access to the
    FreeIPA server, you'll need to contact your FreeIPA administrator and have
    them perform the following steps either using ansible scripts or manually.

Before you configure the undercloud, you need to ensure FreeIPA is configured
with the correct principal and privileges. This allows the undercloud to add
new hosts, services, and DNS records in FreeIPA during the overcloud
installation.

The undercloud will enroll itself as a FreeIPA client and download a keytab to
use for authentication during the installation process. To do this, it needs a
one-time password (OTP) from FreeIPA that you configure in ``undercloud.conf``.

You can generate the OTP manually if you have the correct permissions to add
hosts, modify permissions, update roles, and create principals in FreeIPA. You
need to perform these actions from an existing FreeIPA client. Note, the
FreeIPA server itself is enrolled as a client.

You can find a set of `playbooks
<https://opendev.org/x/tripleo-ipa/src/branch/master/tripleo_ipa/playbooks#user-content-tls-e-ipa-server-configuration-roles>`_
in tripleo-ipa that automate creating permissions, hosts, and principals for
the undercloud. These playbooks expect the ``IPA_PRINCIPAL``, which is a user
in FreeIPA, to have the necessary permissions to perform the tasks in each
playbook (e.g., ``ipa privilege-add-permission``, ``ipa host-add``, etc). They
also expect you to generate a kerberos token before executing each playbook.

Create a FreeIPA role
^^^^^^^^^^^^^^^^^^^^^

First, you need to create a new FreeIPA role with the appropriate permissions
for managing hosts, principals, services, and DNS entries::

    $ kinit
    $ export IPA_PASSWORD=$IPA_PASSWORD
    $ export IPA_PRINCIPAL=$IPA_USER
    $ export UNDERCLOUD_FQDN=undercloud.example.com
    $ ansible-playbook /usr/share/ansible/tripleo-playbooks/ipa-server-create-role.yaml

Register the undercloud
^^^^^^^^^^^^^^^^^^^^^^^

Next, you need to register the undercloud as a FreeIPA client and generate a
OTP that the undercloud will use for enrollment, which is necessary before it
can manage entities in FreeIPA::

    $ export IPA_PASSWORD=$IPA_PASSWORD
    $ export IPA_PRINCIPAL=$IPA_USER
    $ export UNDERCLOUD_FQDN=undercloud.example.com
    $ ansible-playbook /usr/share/ansible/tripleo-playbooks/ipa-server-register-undercloud.yaml

If successful, the ansible output will contain an OTP. Save this OTP because
you will need it when you configure the undercloud.

Create a principal
^^^^^^^^^^^^^^^^^^

Finally, create a FreeIPA principal and grant it the necessary permissions to
manage hosts, services, and DNS entries in FreeIPA::

    $ export IPA_PASSWORD=$IPA_PASSWORD
    $ export IPA_PRINCIPAL=$IPA_USER
    $ export UNDERCLOUD_FQDN=undercloud.example.com
    $ ansible-playbook /usr/share/ansible/tripleo-playbooks/ipa-server-create-principal.yaml

Configure the Undercloud
~~~~~~~~~~~~~~~~~~~~~~~~

.. warning::
    This section only provides guidance for configuring *TLS-everywhere*. You
    need to make sure your undercloud configuration is complete before starting
    the undercloud installation process.

Set the following variables in `undercloud.conf`::

    ipa_otp = $OTP
    overcloud_domain_name = example.com
    undercloud_nameservers = $FREEIPA_IP_ADDRESS

Your undercloud configuration is ready to be deployed and has the necessary
changes to allow you to deploy *TLS-everywhere* for the overcloud.

Undercloud Install
~~~~~~~~~~~~~~~~~~

After you've had an opportunity to verify all undercloud configuration options,
including the options listed above, start the undercloud installation process::

    $ openstack undercloud install

Undercloud Verification
~~~~~~~~~~~~~~~~~~~~~~~

You should verify that the undercloud was enrolled properly by listing the
hosts in FreeIPA::

    $ sudo kinit
    $ sudo ipa host-find

You should also confirm that ``/etc/novajoin/krb5.keytab`` exists on the
undercloud. The ``novajoin`` directory name is purely for legacy naming
reasons. The keytab is placed in this directory regardless of using novajoin
to enroll the undercloud as a FreeIPA client.

You can proceed with the :ref:`Overcloud TLS-everywhere` if the undercloud
installation was successful.

TLS-everywhere with Novajoin
----------------------------

.. warning:: This deployment strategy is only supported up to the Train release. We
    recommend using tripleo-ipa to accomplish *TLS-everywhere* in newer
    releases. Steps for using tripleo-ipa are documented above.  This deployment
    strategy has been removed in Victoria.

Do the following steps before deploying your undercloud.

Configure DNS
~~~~~~~~~~~~~

*TLS-everywhere* deployments use FreeIPA as the DNS server. You need to set the
proper search domain and nameserver on the undercloud. To do this, you need to
know the deployment domain, the domain of the FreeIPA server, and the FreeIPA
server's IP address. For example, if the deployment domain is `example.com` and
the FreeIPA server domain is `bigcorp.com`, you should set the following in
`/etc/resolv.conf`::

    search example.com bigcorp.com
    nameserver $FREEIPA_IP_ADDRESS

This step ensures the undercloud can resolve newly added hosts and services
after TripleO enrolls them as FreeIPA clients. You only need to add both search
domains if they're different. If the FreeIPA server is using the same domain as
the deployment you only need to specify the deployment domain.

Add Undercloud as a FreeIPA host
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Next, you need to add the undercloud as a host in FreeIPA. This will generate a
one-time password that TripleO uses to enroll the undercloud as a FreeIPA
client, giving the undercloud the permissions it needs to add new hosts,
services, and DNS records. You can use the following command-line utility to
add the undercloud as a FreeIPA host::

    novajoin-ipa-setup \
    --principal $IPA_USER \
    --password $IPA_PASSWORD \
    --server ipa.bigcorp.com \
    --realm BIGCORP.COM \
    --domain example.com \
    --hostname undercloud.example.com \
    --precreate

If successful, the command will return a one-time password. Save this password
because you will need it later to configure the undercloud.

Configure the Undercloud
~~~~~~~~~~~~~~~~~~~~~~~~

.. warning::
    This section only provides guidance for configuring *TLS-everywhere*. You
    need to make sure your undercloud configuration is complete before starting
    the undercloud installation process.

Set the following variables in `undercloud.conf`::

    enable_novajoin = True
    ipa_otp = $IPA_OTP
    overcloud_domain_name = example.com

Your undercloud configuration is ready to be deployed and has the necessary
changes to allow you to deploy *TLS-everywhere* for the overcloud.

Undercloud Install
~~~~~~~~~~~~~~~~~~

After you've had an opportunity to verify all undercloud configuration options,
including the options listed above, start the undercloud installation process::

    $ openstack undercloud install

Undercloud Verification
~~~~~~~~~~~~~~~~~~~~~~~

You should verify that the undercloud was enrolled properly by listing the
hosts in FreeIPA::

    $ sudo kinit
    $ sudo ipa host-find

You should also confirm that ``/etc/novajoin/krb5.keytab`` exists on the
undercloud and that the ``novajoin`` and ``novajoin-notifier`` services are
running.

You can proceed with the :ref:`Overcloud TLS-everywhere` if the undercloud
installation was successful.

.. _Overcloud TLS-everywhere:

Configuring the Overcloud
-------------------------

*TLS-everywhere* requires you to set extra parameters and templates before you
deploy, or update, your overcloud. These changes consist of settings domain
information and including additional heat templates in your deploy command.
Let's walk through each step individually.

Set Parameters
~~~~~~~~~~~~~~

Next, you need to set parameters so that TripleO knows where to find your
FreeIPA server and configures DNS. You need to set these variables so that
TripleO adds DNS records that map to the correct hosts. Let's continue assuming
we have a file called ``tls-parameters.yaml`` and it contains the following
parameter_defaults section::

    parameter_defaults:
      DnsSearchDomains: ["example.com"]
      DnsServers: ["192.168.1.13"]
      CloudDomain: example.com
      CloudName: overcloud.example.com
      CloudNameInternal: overcloud.internalapi.example.com
      CloudNameStorage: overcloud.storage.example.com
      CloudNameStorageManagement: overcloud.storagemgmt.example.com
      CloudNameCtlplane: overcloud.ctlplane.example.com

.. note::
    If you are using deployed servers, you must also specify the following
    parameters::

        IdMInstallClientPackages: True

    This option is required to install packages needed to enroll overcloud
    hosts as FreeIPA clients. Deployments using Novajoin do not require this
    option since the necessary packages are built into the overcloud images. If
    you do not specify this argument, you need to ensure dependencies for
    ansible-freeipa are present on the overcloud servers before deploying the
    overcloud.

The ``DnsServers`` value above assumes we have FreeIPA available at
192.168.1.13.

It's important to note that you will need to update the `DnsSearchDomains` to
include the domain of the IPA server if it's different than the `CloudDomain`.
For example, if your `CloudDomain` is `example.com` and your IPA server is
located at `ipa.bigcorp.com`, then you need to include `bigcorp.com` as an
additional search domain::

    DnsSearchDomains: ["example.com", "bigcorp.com"]

Composable Services
~~~~~~~~~~~~~~~~~~~

In addition to the parameters above, you might need to update the
``resource_registry`` in ``tls-parameters.yaml`` to include a composable
service. There are two composable services, one for Novajoin and the other is
for tripleo-ipa. TripleO uses the Novajoin composable service for deploying
*TLS-everywhere* by default. If you need or want to use tripleo-ipa, you'll
need to update the registry to use a different composable service. Both options
are described below.

Novajoin Composable Service
~~~~~~~~~~~~~~~~~~~~~~~~~~~

This was the default option until Ussuri.  As of Victoria, this option has
been removed, and deployers upgrading to Victoria will be migrated to tripleo-ipa.

For reference, the Novajoin based composable service is located at
/usr/share/openstack-tripleo-heat-templates/deployment/ipa/ipaclient-baremetal-ansible.yaml

tripleo-ipa Composable Service
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you're deploying *TLS-everywhere* with tripleo-ipa prior to Victoria, you need to
override the default Novajoin composable service. Add the following composable service to
the ``resource_registry`` in ``tls-parameters.yaml``::

    resource_registry:
      OS::TripleO::Services::IpaClient: /usr/share/openstack-tripleo-heat-templates/deployment/ipa/ipaservices-baremetal-ansible.yaml

As of Victoria, this is the only method for deploying *TLS-everywhere*.

Specify Templates
~~~~~~~~~~~~~~~~~

At this point, you should have all the settings configured for a successful
*TLS-everywhere* deployment. The only remaining step is to include the
following templates in your overcloud deploy command::

    $ openstack overcloud deploy \
    -e /usr/share/openstack-tripleo-heat-templates/environments/ssl/tls-everywhere-endpoints-dns.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/services/haproxy-public-tls-certmonger.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/ssl/enable-internal-tls.yaml \
    -e tls-parameters.yaml

Remember, ``tls-parameters.yaml`` is the file containing the parameters above.

Overcloud Verification
----------------------

After the overcloud is deployed, you can confirm each endpoint is using HTTPS
by querying keystone's endpoints::

    $ openstack --os-cloud overcloud endpoint list

Deleting Overclouds
-------------------

.. note::
    This functionality is only invoked when you use the ``openstack overcloud
    delete`` command using Train or newer releases. The overcloud is
    technically a heat stack, but using ``openstack stack delete`` will not
    clean up FreeIPA.

.. note::
    This section is only applicable to deployments using tripleo-ipa. Novajoin
    cleans up FreeIPA after consuming notifications about instance deletion.

The python-tripleoclient CLI cleans up hosts, services, and DNS records in
FreeIPA when you delete an overcloud::

    $ openstack overcloud delete overcloud

You can verify the hosts, services, DNS records were removed by querying
FreeIPA::

    $ kinit
    $ ipa host-find
    $ ipa service-find
    $ ipa dnsrecord-find example.com.

The undercloud host, service, and DNS records are untouched when deleting
overclouds. Overcloud hosts, services, and DNS records are re-added to FreeIPA
during subsequent deployments.

If you don't want to clean up FreeIPA when you delete your overcloud, you can
use the ``openstack overcloud delete --skip-ipa-cleanup`` parameter. This
option leaves all overcloud hosts, services, and DNS records in FreeIPA. You
might find this useful if your FreeIPA server is unreachable or if you plan to
clean up FreeIPA later.

To clean up FreeIPA manually, you need the Ansible inventory file that
describes your deployment. If you don't have it handy, you can generate one
from the undercloud using::

    $ source stackrc
    $ tripleo-ansible-inventory --static-yaml-inventory generated-inventory.yaml

The utility will generate an inventory file and store it as
``generated-inventory.yaml``. You can invoke the playbook that cleans up
FreeIPA using::

    $ ansible-playbook -i generated-inventory.yaml /usr/share/ansible/tripleo-playbooks/cli-cleanup-ipa.yml
