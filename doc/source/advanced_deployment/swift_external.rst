Use an external Swift Proxy with the Overcloud
===============================================

|project| supports use of an external Swift proxy already available to the
operator, that may need to configure at deploy time.

This happens by enabling a particular environment file when deploying the
Overcloud, specifically `environments/swift-external.yaml`.

In the environment file above user must adjust the parameters to fit
its setup by creating a custom environment file (i.e.
*~/my-swift-settings.yaml*)::

  parameter_defaults:
     ExternalPublicUrl: 'http://swiftproxy:9024/v1/%(tenant_id)s'
     ExternalInternalUrl: 'http://swiftproxy:9024/v1/%(tenant_id)s'
     ExternalAdminUrl: 'http://swiftproxy:9024/v1/%(tenant_id)s'
     ExternalSwiftUserTenant: 'service'


The user can create an environment file with the required settings
and add the files above to the deploy commandline::

  openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/swift-external.yaml -e ~/my-swift-settings.yaml

Once the deploy has succeeded, user has to complete the
configuration on the external swift proxy, configuring it to use the
keystone authentication provider. This environment files creates also
a service user called *swift* that can be used for this purpose.

