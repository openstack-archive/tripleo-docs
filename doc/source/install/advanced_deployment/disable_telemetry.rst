Disable Telemetry
=================

This guide assumes that your undercloud is already installed and ready to
deploy an overcloud without Telemetry services.

Deploy your overcloud without Telemetry services
------------------------------------------------

If you don't need or don't want Telemetry services (Ceilometer, Gnocchi,
Panko and Aodh), you can disable the services by adding this environment
file when deploying the overcloud::

    openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/disable-telemetry.yaml

Disabling Notifications
~~~~~~~~~~~~~~~~~~~~~~~

When Telemetry is disabled, OpenStack Notifications will be disabled as well, and
the driver will be set to 'noop' for all OpenStack services.
If you would like to restore notifications, you would need to set NotificationDriver to
'messagingv2' in your environment.

.. Warning::

   NotificationDriver parameter can only support 'noop' and 'messagingv2' for now.
   Also note that 'messaging' driver is obsolete and isn't supported by TripleO.
