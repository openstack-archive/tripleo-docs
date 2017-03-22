TLS support for services
========================

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

.. References

.. _tripleo-heat-templates: https://github.com/openstack/tripleo-heat-templates
.. _manifests/haproxy.pp: https://github.com/openstack/puppet-tripleo/blob/master/manifests/haproxy.pp
.. _network/service_net_map.j2.yaml: https://github.com/openstack/tripleo-heat-templates/blob/master/network/service_net_map.j2.yaml
.. _haproxy documentation: http://www.haproxy.org/
