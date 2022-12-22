TLS support for services
========================

.. _public-tls-dev:

Public TLS
----------

If you're adding a REST service to TripleO, chances are that you'll need your
service to be terminated by HAProxy. Unfortunately, adding your service to
HAProxy needs extra changes to existing modules. Fortunately, it's not that
hard to do.

You can add your service to be terminated by HAproxy by modifying the
`manifests/haproxy.pp`_ file.

First off, we need a flag to tell the HAProxy module to write the frontend for
your service in the HAProxy configuration file if your service is deployed. For
this, we will add a parameter for the manifest. If you have followed the
walk-through, you may have noticed that the `tripleo-heat-templates`_ yaml
template requires you to set a name for your service in the ``role_data``
output::

    ...
    outputs:
      role_data:
        description: Description of your service
        value:
          service_name: my_service
    ...

The overcloud stack generated from the tripleo-heat-templates will use this
name and automatically generate several hieradata entries that are quite
useful. One of this entries is a global flag that can tell if your service is
enabled at all or not. So we'll use this flag and fetch it from hiera to set
the parameter we need in haproxy.pp::

    ...
    $keystone_admin              = hiera('keystone_enabled', false),
    $keystone_public             = hiera('keystone_enabled', false),
    $neutron                     = hiera('neutron_api_enabled', false),
    $cinder                      = hiera('cinder_api_enabled', false),
    $glance_api                  = hiera('glance_api_enabled', false),
    ...
    $my_service                  = hiera('my_service_enabled', false),
    ...

Note that the name of the hiera key matches the following format
"<service name>_enabled" and defaults to ``false``.

Next, you need to add a parameter that tells HAProxy which network your service
is listening on::

    ...
    $barbican_network            = hiera('barbican_api_network', false),
    $ceilometer_network          = hiera('ceilometer_api_network', undef),
    $cinder_network              = hiera('cinder_api_network', undef),
    $glance_api_network          = hiera('glance_api_network', undef),
    $heat_api_network            = hiera('heat_api_network', undef),
    ...
    $my_service_network          = hiera('my_service_network', undef),
    ...

Tripleo-heat-templates will also autogenerate this key for you. However for it
to do this, you need to specify the network for your service in the templates.
The file where this needs to be set is `network/service_net_map.j2.yaml`_, and
you'll be looking for a parameter called ``ServiceNetMapDefaults``. It will
look like this::

    # Note that the key in this map must match the service_name
    # see the description above about conversion from CamelCase to
    # snake_case - the names must still match when converted
    ServiceNetMapDefaults:
      default:
        # Note the values in this map are replaced by *NetName
        # to allow for sane defaults when the network names are
        # overridden.
        ...
        NeutronTenantNetwork: tenant
        CeilometerApiNetwork: internal_api
        BarbicanApiNetwork: internal_api
        CinderApiNetwork: internal_api
        GlanceApiNetwork: storage
        ...
        MyServiceNetwork: <some network>

Now, having added this, you'll have access to the aforementioned hiera key and
several others.

Note that the network is used by HAProxy to terminate TLS for your service.
This is used when Internal TLS is enabled and you'll learn more about it in the
:ref:`internal-tls-dev` section.

Then, you need to add the ports that HAProxy will listen on. There is a list
with the defaults which is called ``default_service_ports``, and you need to
add your service here::

    $default_service_ports = {
      ...
      neutron_api_port => 9696,
      neutron_api_ssl_port => 13696,
      nova_api_port => 8774,
      nova_api_ssl_port => 13774,
      nova_placement_port => 8778,
      nova_placement_ssl_port => 13778,
      nova_metadata_port => 8775,
      nova_novnc_port => 6080,
      nova_novnc_ssl_port => 13080,
      ...
      my_service_port => 5123,
      my_service_ssl_port => 13123,
      ...
    }

You are specifying two ports here, one that is the standard port, and another
one that is used for SSL in the public VIP/host. This was done initially to
address deployments without network isolation. In these cases, deploying TLS
would effectively take over the other interfaces, so HAProxy would be listening
with TLS everywhere accidentally if only using one port, and further
configuration for the services would need to happen to address this. However,
this is not really an issue in network isolated deployments, since they would
be using different IP addresses. So this extra port might not be needed in the
future if network isolation becomes the standard mode of deploying.

.. note:: The SSL port is not needed if your service is only internal and
   doesn't listen on the public VIP.

.. note:: These ports can be overwritten by using the ``$service_ports``
   parameter from this manifest. Once could pass it via hieradata through the
   ``ExtraConfig`` tripleo-heat-templates parameter, and setting something
   like this as the value::

        tripleo::haproxy::service_ports:
          my_service_ssl_port: 5123
          my_service_2_ssl_port: 5124

   Please consider that this will overwrite any entry from the list of
   defaults, so you have to be careful to update all the relevant entries in
   tripleo-heat-templates if you want to change port (be it SSL port or
   non-SSL port).

Finally, you need to add the actual endpoint to HAproxy which will configure
the listen directive (or frontend and backend) in the haproxy configuration.
For this, we have a helper class called ``::tripleo::haproxy::endpoint`` that
sets the relevant bits for you. All we need to do is pass in all the
information that class needs. And we need to make sure that this only happens
if the service is enabled, so we'll enclose it with the flag we mentioned
above. So here's a code snippet that demonstrates what you need to add::

    if $my_service {
      ::tripleo::haproxy::endpoint { 'my_service':
        public_virtual_ip => $public_virtual_ip,
        internal_ip       => hiera('my_service_vip', $controller_virtual_ip),
        service_port      => $ports[my_service_port],
        ip_addresses      => hiera('my_service_node_ips', $controller_hosts_real),
        server_names      => hiera('my_service_node_names', $controller_hosts_names_real),
        mode              => 'http',
        listen_options    => {
            'http-request' => [
              'set-header X-Forwarded-Proto https if { ssl_fc }',
              'set-header X-Forwarded-Proto http if !{ ssl_fc }'],
        },
        public_ssl_port   => $ports[my_service_ssl_port],
        service_network   => $my_service_network,
      }
    }

* The ``public_virtual_ip`` variable contains the public IP address that's used
  for your cloud, and it's the one that people will usually have access to
  externally.

* The hiera keys ``my_service_node_ips``, ``my_service_vip``,
  ``my_service_node_names`` are automatically generated by
  tripleo-heat-templates. These are other keys that you'll get access to once
  you add the network for your service in ``ServiceNetMapDefaults``.

* ``my_service_vip`` is, as mentioned, automatically generated, and will point
  HAProxy to the non-public VIP where other services will be able to access
  your service. This will usually be the Internal API network, but it depends
  on your use-case.

* ``my_service_node_ips`` is, as mentioned, automatically generated, and will
  tell HAProxy which nodes are hosting your service, so it will point to them.
  The address depends on the network your service is listening on.

* ``my_service_node_names`` is, as mentioned, automatically generated, and will
  be the names that HAProxy will use for the nodes. These are the FQDNs of the
  nodes that are hosting your service.

* This example is an HTTP service, so note that we set the mode to ``http``,
  and that we set the option for HAProxy to detect if TLS was used for the
  request, and set an appropriate value for the ``X-Forwarded-Proto`` HTTP
  header if that's the case. Not all services can read this HTTP header, so
  this depends on your service. For more information on the available options
  and the mode, consult the `haproxy documentation`_.

.. note:: If your service is only internal and doesn't listen on the public
   VIP, you don't need all of the parameters listed above, and you would
   instead do something like this::

       if $my_service {
         ::tripleo::haproxy::endpoint { 'my_service':
           internal_ip     => hiera('my_service_vip', $controller_virtual_ip),
           service_port    => $ports[my_service_port],
           ip_addresses    => hiera('my_service_node_ips', $controller_hosts_real),
           server_names    => hiera('my_service_node_names', $controller_hosts_names_real),
           service_network => $my_service_network,
         }
       }

   The most relevant bits are that we omitted the SSL port and the
   ``public_virtual_ip``, since these won't be used.


Having added this to the manifest, you should be covered for both getting your
service to be proxied by HAProxy, and letting it to TLS in the public interface
for you.

.. _internal-tls-dev:

Internal TLS
------------

How it works
~~~~~~~~~~~~

If you haven't read the section `TLS Everywhere <tls_everywhere_deploy_guide_>`_
it is highly recommended you read that first before continuing.

As mentioned, the default CA is FreeIPA, which issues the certificates that the
nodes request, and they do the requests via certmonger.

FreeIPA needs to have the nodes registered in its database and those nodes need
to be enrolled in order to authenticate to the CA. This is already being
handled for us, so there's nothing you need to do for your service on this
side.

In order to issue certificates, FreeIPA also needs to have registered a
Kerberos principal for the service (or service principal). This way it knows
what service is using what certificate. The service principal will look
something like this::

    <service name>/<host>.<domain>

We assume that the domain matches the kerberos realm, so specifying it is
redundant.

Fortunately, one doesn't need to do much but fill in some boilerplate code in
tripleo-heat-templates to get this service principal. And this will be covered
in subsequent sections.

So, with this one can finally request certificates for the service and use
them.

.. _internal-tls-for-your-service:

Enabling internal TLS for your service
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Aside from the actual certificate request, if your service is a RESTful
service, getting TLS to work with the current solution requires usually two
fronts:

* To get your service to actually serve with TLS.

* To tell HAProxy to try to access your service using TLS.

This can be different for other types of services. For instance, at the time of
writing this, RabbitMQ isn't proxied by HAProxy, so there wasn't a need to
configure anything in HAProxy. Another example is MariaDB: Even though it is
proxied by HAProxy, TLS is handled on the MariaDB side and HAProxy doesn't do
TLS termination, so there was no need to configure HAProxy.

Also, for services in general, there are two options for the Subject
Alternative Name (SAN) for the certificate:

1) It should be a hostname that points to a specific interface in the node.

2) It should be a hostname that points to a VIP (or a Virtual IP Address).

The usual case for a RESTful service will be the first option. HAProxy will do
TLS termination, listening on the cloud's VIPs, and will then forward the
request to your service trying to access it via the node's internal network
interface (not the VIP). So for this case (#1), your service should be serving
a TLS certificate with the nodes' interface as the SAN. RabbitMQ has a similar
situation even if it's not proxied by HAProxy. Services try to access the
RabbitMQ cluster through the individual nodes, so each broker server has a
certificate with the node's hostname for a specific network interface as the
SAN. On the other hand, MariaDB follows the SAN pattern #2. It's terminated by
HAProxy, so the services access it through a VIP. However, MariaDB handles TLS
by itself, so it ultimately serves certificates with the hostname pointing to a
VIP interface as the SAN. This way, the hostname validation works as expected.

If you're not sure how to go forward with your service, consult the TripleO
team.

.. _services-over-httpd-internal-tls:

Services that run over httpd
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Good news! Certificates are already requested for you and there is a hash where
you can fetch the path to the certificates and use them for your service.

In `puppet-tripleo`_ you need to go to the manifest that deploys the API for
your service. Here, you will add the following parameters to the class::

    class tripleo::profile::base::my_service::api (
      ...
      $my_service_network  = hiera('my_service_network', undef),
      $certificates_specs  = hiera('apache_certificates_specs', {}),
      $enable_internal_tls = hiera('enable_internal_tls', false),
      ...
    ) {

* ``my_service_network`` is a hiera key that's already generated by
  tripleo-heat-templates and it references the name of the network your service
  is listening on. This was referenced in the :ref:`public-tls-dev` section.
  Where it mentioned the addition of your service's network to the
  ``ServiceNetMapDefaults`` parameter. So, if this was done, you'll get this
  key autogenerated.

* ``apache_certificates_specs`` is a hash containing the specifications for all
  the certificates requested for services running over httpd. These are
  network-dependant, which is why we needed the network name. Note that this
  also contains the paths where the keys are located in the filesystem.

* ``enable_internal_tls`` is a flag that tells TripleO if TLS for the internal
  network is enabled. We should base the usage of the certificates for your
  service on this.

In order to get the certificate and key for your application you can use the
following boilerplate code::

    if $enable_internal_tls {
      if !$my_service_network {
        fail('my_service_network is not set in the hieradata.')
      }
      $tls_certfile = $certificates_specs["httpd-${my_service_network}"]['service_certificate']
      $tls_keyfile = $certificates_specs["httpd-${my_service_network}"]['service_key']
    } else {
      $tls_certfile = undef
      $tls_keyfile = undef
    }

If internal TLS is not enabled, we set the variables for the certificate and
key to ``undef``, this way TLS won't be enabled. If it's enabled, we get the
certificate and key from the hash.

Now, having done this, we can pass in the variables to the class that deploys
your service over httpd::

    class { '::my_service::wsgi::apache':
      ssl_cert => $tls_certfile,
      ssl_key  => $tls_keyfile,
    }

Now, in `tripleo-heat-templates`_, hopefully the template for your service's
API already uses the base profile for apache services. To verify this, you need
to look in the ``resources`` section of your template for something like this::

    ApacheServiceBase:
      type: ./apache.yaml
      properties:
        ServiceNetMap: {get_param: ServiceNetMap}
        EndpointMap: {get_param: EndpointMap}

Note that this is of type ./apache.yaml which is the template that contains the
common configurations for httpd based services.

You will also need to make sure that the ssl hieradata is set correctly. You
will find it usually like this::

    my_service::wsgi::apache::ssl: {get_param: EnableInternalTLS}

Where, EnableInternalTLS should be defined in the ``parameters`` section of the
template.

Finally, you also need to add the ``metadata_settings`` to the output of the
template. This section will be in the same level as ``config_settings`` and
``step_config``, and will contain the following::

    metadata_settings:
      get_attr: [ApacheServiceBase, role_data, metadata_settings]

Note that it merely outputs the metadata_settings section that the apache base
stack already outputs. This will give the appropriate parameters to a hook that
sets the nova metadata, which in turn will be taken by the *novajoin* service
generate the service principals for httpd for the host.

See the `TLS Everywhere Deploy Guide <tls_everywhere_deploy_guide_>`_

.. _tls_everywhere_deploy_guide: https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/features/ssl.html#tls-everywhere-for-the-overcloud
.. _configuring-haproxy-internal-tls:

Configuring HAProxy to use TLS for your service
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now that your service will be serving with TLS enabled, we go back to the
`manifests/haproxy.pp`_ file. You already have added the HAProxy endpoint
resource for your service, so for this, you need to add now the option to tell
it to use TLS to communicate with the server backend nodes. This is done by
adding this::

    if $my_service {
      ::tripleo::haproxy::endpoint { 'my_service':
        ...
        member_options    => union($haproxy_member_options, $internal_tls_member_options),
      }
    }

This adds the TLS options to the default member options we use in TripleO for
HAProxy. It will tell HAProxy to require TLS for your service if internal TLS
is enabled; if it's not enabled, then it won't use TLS.

This was all the extra configuration you needed to do for HAProxy.

Internal TLS for services that don't run over httpd
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If your service supports being run with TLS enabled, and it's not
python/eventlet-based (see :ref:`internal-tls-via-proxy`). This section is for
you.

In `tripleo-heat-templates`_ we'll need to specify the specs for doing the
certificate request, and we'll need to get the appropriate information to
generate a service principal. To make this optional, you should add the
following to your service's base template::

    parameters:
      ...
      EnableInternalTLS:
        type: boolean
        default: false

    conditions:

      internal_tls_enabled: {equals: [{get_param: EnableInternalTLS}, true]}
      ...
    ...

* ``EnableInternalTLS`` is a parameter that's passed via ``parameter_defaults``
  which tells the templates that we want to use TLS in the internal network.

* ``internal_tls_enabled`` is a condition that we'll furtherly use to add the
  relevant bits to the output.

The next thing to do is to add the certificate specs, the relevant hieradata
and the required metadata to the output. In the ``roles_data`` output, lets
modify the ``config_settings`` to add what we need::

      config_settings:
        map_merge:
          -
            # The regular hieradata for your service goes here.
            ...
          -
            if:
            - internal_tls_enabled
            - generate_service_certificates: true
              my_service_certificate_specs:
                service_certificate: '/etc/pki/tls/certs/my_service.crt'
                service_key: '/etc/pki/tls/private/my_service.key'
                hostname:
                  str_replace:
                    template: "%{hiera('fqdn_NETWORK')}"
                    params:
                      NETWORK: {get_param: [ServiceNetMap, MyServiceNetwork]}
                principal:
                  str_replace:
                    template: "my_service/%{hiera('fqdn_NETWORK')}"
                    params:
                      NETWORK: {get_param: [ServiceNetMap, MyServiceNetwork]}
            - {}
      ...
      metadata_settings:
        if:
          - internal_tls_enabled
          -
            - service: my_service
              network: {get_param: [ServiceNetMap, MyServiceNetwork]}
              type: node
          - null

* The conditional mentioned above is used in the ``config_settings``. So, if
  ``internal_tls_enabled`` evaluates to ``true``, the hieradata necessary to
  enable TLS in the internal network for your service will be added. Else, we
  output ``{}``, which won't affect the ``map_merge`` and won't add anything
  to the regular hieradata for your service.

* For this case, we are only requesting one certificate for the service.

* The service will be terminated by HAProxy in a conventional way, which means
  that the SAN will be case #1 as described in
  :ref:`internal-tls-for-your-service`. So the SAN will point to the specific
  node's network interface, and not the VIP.

* The ``ServiceNetMap`` contains the references to the networks every service
  is listening on, and the key to get the network is the name of your service
  but using camelCase instead of underscores. This value is the name of the
  network and if used under the ``config_settings`` section, it will be
  replaced by the actual IP. Else, it will just be the network name.

* tripleo-heat-templates automatically generates hieradata that contains the
  different network-dependant hostnames. They keys are in the following
  format::

      fqdn_<network name>

* The ``my_service_certificate_specs`` key will contain the specifications for
  the certificate we'll request. They need to follow some conventions:

  * ``service_certificate`` will specify the path to the certificate file. It
    should be an absolute path.

  * ``service_key`` will specify the path to the private key file that will be
    used for the certificate. It should be an absolute path.

  * ``hostname`` is the name that will be used both in the Common Name (CN) and
    the Subject Alternative Name (SAN) of the certificate. We can get this
    value by using the hiera key described above. So we first get the name of
    the network the service is listening on from the ``ServiceNetMap`` and we
    then use ``str_replace`` to place that in a hiera call in the appropriate
    format.

  * ``principal`` is the service principal that will be the one used for the
    certificate request. We can get this in a similar manner as we got the
    hostname, and prepending an identifying name for your service. The format
    will be as follows::

        < service identifier >/< network-based hostname >

  * These are the names used by convention, and will eventually be passed to
    the ``certmonger_certificate`` resource from `puppet-certmonger`_.

* The ``metadata_settings`` section will pass some information to a metadata
  hook that will create the service principal before the certificate request is
  done. The format as follows:

  * ``service``: This contains the service identifier to be used in the
    kerberos service principal. It should match the identifier you put in the
    ``principal`` section of the certificate specs.

  * ``network``: Tells the hook what network to use for the service. This will
    be used for the hook and novajoin to use an appropriate hostname for the
    kerberos principal.

  * ``type``: Will tell the hook what type of case is this service. The
    available options are ``node`` and ``vip``. These are the cases mentioned
    in the :ref:`internal-tls-for-your-service` for the SANs.

  Note that this is a list, which can be useful if we'll be creating several
  service principals (which is not the case for our example). Also, if
  ``internal_tls_enabled`` evaluates to ``false``, we then output ``null``.

* Remember to set any relevant flags or parameters that your service might
  need as hieradata in ``config_settings``. These might be things that
  explicitly enable TLS such as flags or paths. But these details depend on the
  puppet module that deploys your service.

.. note:: **VIP-based hostname case**

   If your service requires the certificate to contain a VIP-based hostname, as
   is the case for MariaDB. It would instead look like the following::

      metadata_settings:
        if:
          - internal_tls_enabled
          -
            - service: my_service
              network: {get_param: [ServiceNetMap, MyServiceNetwork]}
              type: vip
          - null

   * One can get the hostname for the VIP in a similar fashion as we got the
     hostname for the node. The VIP hostnames are also network based, and one
     can get them from a hiera key as well. It has the following format::

        cloud_name_< network name >

   * The ``type`` in the ``metadata_settings`` entry is ``vip``.

In `puppet-tripleo`_ We'll create a class that does the actual certificate
request and add it to the resource that gets the certificates for all the
services.

Lets create a class to do the request::

    class tripleo::certmonger::my_service (
      $hostname,
      $service_certificate,
      $service_key,
      $certmonger_ca = hiera('certmonger_ca', 'local'),
      $principal     = undef,
    ) {
      include ::my_service::params

      $postsave_cmd  = "systemctl restart ${::my_service::params::service_name}"
      certmonger_certificate { 'my_service' :
        ensure       => 'present',
        certfile     => $service_certificate,
        keyfile      => $service_key,
        hostname     => $hostname,
        dnsname      => $hostname,
        principal    => $principal,
        postsave_cmd => $postsave_cmd,
        ca           => $certmonger_ca,
        wait         => true,
        require      => Class['::certmonger'],
      }

      file { $service_certificate :
        owner   => $::my_service::params::user,
        group   => $::my_service::params::group,
        require => Certmonger_certificate['my_service'],
      }
      file { $service_key :
        owner   => $::my_service::params::user,
        group   => $::my_service::params::group,
        require => Certmonger_certificate['my_service'],
      }

      File[$service_certificate] ~> Service<| title == $::my_service::params::service_name |>
      File[$service_key] ~> Service<| title == $::my_service::params::service_name |>
    }

* You'll note that the parameters mostly match the certificate specs that we
  created before in tripleo-heat-templates.

* By convention, we'll add this class in the **manifests/certmonger** folder.

* ``certmonger_ca`` is a value that comes from tripleo-heat-templates and tells
  certmonger which CA to use.

* If it's available, by convention, many puppet modules contain a manifest
  called *params*. This usually contains the name and group that the service
  runs with, as well as the name of the service in a specific distribution.
  So we include this.

* We do then the actual certificate request by using the
  ``certmonger_certificate`` provider and passing all the relevant data for the
  request.

  * The post-save command which is specified via the ``postsave_cmd`` is a
    command that will be ran after the certificate is saved. This is useful for
    when certmonger has to resubmit the request to get an updated certificate,
    since this way we can reload or restart the service so it can serve the new
    certificate.

* Using the ``file`` resource from puppet, we set the appropriate user and
  group for the certificate and keys. Fortunately, certmonger has sane defaults
  for the file modes, so we didn't set those here.

Having this class, we now need to add to the `certmonger_user`_ resource. This
resource is in charge of making all the certificate requests and should be
available on all roles (or at least it should be added). You would add the
certificate specs as a parameter to this class::

    class tripleo::profile::base::certmonger_user (
      ...
      $my_service_certificate_specs = hiera('my_service_certificate_specs', {}),
      ...
    ) {

And finally, we call the class that does the request::

  ...
  unless empty($my_service_certificate_specs) {
    ensure_resource('class', 'tripleo::certmonger::my_service', $my_service_certificate_specs)
  }
  ...

.. note::
   It is also possible to do several requests for your service. See the
   `certmonger_user`_ source code for examples.

Finally, you can do the same steps described in
`configuring-haproxy-internal-tls`_ to make HAProxy connect to your service
using TLS.

.. _internal-tls-via-proxy:

Internal TLS via a TLS-proxy
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you have a RESTful service that runs over python (most likely using
eventlet) or if your service requires a TLS proxy in order to have TLS in the
internal network, there are extra steps to be done.

For python-based services, due to performance issues with eventlet, the best
thing you can do is try to move your service to run over httpd, and let it
handle crypto instead. Then you'll be able to follow the instructions from the
:ref:`services-over-httpd-internal-tls` section above. If for any reason this
can't be done at the moment, we could still use httpd to service as a TLS proxy
in the node. It would then listen on the service's port and forward all the
requests to the service, which would then be listening on localhost.

In `puppet-tripleo`_ you need to go to the manifest that deploys the API for
your service, and add the following parameters::

    class tripleo::profile::base::my_service::api (
      ...
      $certificates_specs  = hiera('apache_certificates_specs', {}),
      $enable_internal_tls = hiera('enable_internal_tls', false),
      $my_service_network  = hiera('my_service_api_network', undef),
      $tls_proxy_bind_ip   = undef,
      $tls_proxy_fqdn      = undef,
      $tls_proxy_port      = 5123,
      ...
    ) {
    ...

* ``certificates_specs``, ``enable_internal_tls`` and ``my_service_network``
  have already been mentioned in the :ref:`services-over-httpd-internal-tls`
  section.

* ``tls_proxy_bind_ip``, ``tls_proxy_fqdn`` and ``tls_proxy_port`` are
  parameters that will be used by the httpd-based TLS proxy. They will tell it
  where what IP to listen on, the FQDN (which will be used as the servername)
  and the port it will use. Usually the port will match your service's port.
  These values are expected to be set from tripleo-heat-templates.

Next comes the code for the actual proxy::

    ...
    if $enable_internal_tls {
      if !$my_service_network {
        fail('my_service_network is not set in the hieradata.')
      }
      $tls_certfile = $certificates_specs["httpd-${my_service_network}"]['service_certificate']
      $tls_keyfile = $certificates_specs["httpd-${my_service_network}"]['service_key']

      ::tripleo::tls_proxy { 'my_service_proxy':
        servername => $tls_proxy_fqdn,
        ip         => $tls_proxy_bind_ip,
        port       => $tls_proxy_port,
        tls_cert   => $tls_certfile,
        tls_key    => $tls_keyfile,
        notify     => Class['::my_service::api'],
      }
    }
    ...

* The ``::tripleo::tls_proxy`` is the resource that will configure the TLS
  proxy for your service. As you can see, it receives the certificates that
  come from the ``certificates_specs`` which contain the specification
  for the certificates, including the paths for the keys.

* The notify is added here since we want the proxy to be set before the
  service.

In `tripleo-heat-templates`_, you should modify your service's template and add
the following::

    parameters:
    ...
      EnableInternalTLS:
        type: boolean
        default: false
    ...
    conditions:
      ...
      use_tls_proxy: {equals : [{get_param: EnableInternalTLS}, true]}
    ...
    resources:
    ...
      TLSProxyBase:
        type: OS::TripleO::Services::TLSProxyBase
        properties:
          ServiceNetMap: {get_param: ServiceNetMap}
          EndpointMap: {get_param: EndpointMap}
          EnableInternalTLS: {get_param: EnableInternalTLS}


* ``EnableInternalTLS`` is a parameter that's passed via ``parameter_defaults``
  which tells the templates that we want to use TLS in the internal network.

* ``use_tls_proxy`` is a condition that we'll use to modify the behaviour of
  the template depending on whether TLS in the internal network is enabled or
  not.

* ``TLSProxyBase`` will make the default values from the proxy's template
  available to where our service is deployed. We should make sure that we
  combine our service's hieradata with the hieradata coming from that resource
  by doing a ``map_merge`` with the ``config_settings``::

      ...
      config_settings:
        map_merge:
          - get_attr: [TLSProxyBase, role_data, config_settings]
          - # Here goes our service's metadata
            ...

So, with this, we can tell the service to bind on localhost instead of the
default interface depending if TLS in the internal network is enabled or not.
Lets now set the hieradata that the puppet module needs in our service's
hieradata, which is in the ``config_settings`` section::

    tripleo::profile::base::my_service::api::tls_proxy_bind_ip:
      get_param: [ServiceNetMap, MyServiceNetwork]
    tripleo::profile::base::my_service::api::tls_proxy_fqdn:
      str_replace:
        template:
          "%{hiera('fqdn_$NETWORK')}"
        params:
          $NETWORK: {get_param: [ServiceNetMap, MyServiceNetwork]}
    tripleo::profile::base::my_service::api::tls_proxy_port:
      get_param: [EndpointMap, NeutronInternal, port]
    my_service::bind_host:
      if:
      - use_tls_proxy
      - 'localhost'
      - {get_param: [ServiceNetMap, MyServiceNetwork]}

* The ``ServiceNetMap`` contains the references to the networks every service
  is listening on, and the key to get the network is the name of your service
  but using camelCase instead of underscores. This value will be automatically
  replaced by the actual IP.

* tripleo-heat-templates generates automatically hieradata that contains the
  different network-dependant hostnames. They keys are in the following
  format::

      fqdn_<network name>

  So, to get it, we get the network name from the ``ServiceNetMap``, and do a
  ``str_replace`` in heat that will use that network name and add it to a hiera
  call that will then gets us the FQDN we need.

* The port we can easily get from the ``EndpointMap``.

* The conditional uses the actual IP if there's no TLS in the internal network
  enabled and localhost if it is.

Finally, we add the ``metadata_settings`` section to make sure we get a
kerberos service principal::

    metadata_settings:
      get_attr: [TLSProxyBase, role_data, metadata_settings]

.. References

.. _certmonger_user: https://github.com/openstack/puppet-tripleo/blob/master/manifests/profile/base/certmonger_user.pp
.. _haproxy documentation: http://www.haproxy.org/
.. _manifests/haproxy.pp: https://github.com/openstack/puppet-tripleo/blob/master/manifests/haproxy.pp
.. _network/service_net_map.j2.yaml: https://github.com/openstack/tripleo-heat-templates/blob/master/network/service_net_map.j2.yaml
.. _puppet-certmonger: https://github.com/earsdown/puppet-certmonger
.. _puppet-tripleo: https://github.com/openstack/puppet-tripleo
.. _tripleo-heat-templates: https://github.com/openstack/tripleo-heat-templates
