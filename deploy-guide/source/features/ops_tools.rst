Deploying Operational Tools
===========================

TripleO comes with an optional suite of tools designed to help operators
maintain an OpenStack environment. The tools perform the following functions:

- Availability Monitoring
- Centralized Logging
- Performance Monitoring

This document will go through the presentation and installation of these tools.

Architecture
------------

#. Operational Tool Server:

   - Monitoring Relay/proxy (RabbitMQ_)
   - Monitoring Controller/Server (Sensu_)
   - Data Store (Redis_)
   - API/Presentation Layer (Uchiwa_)
   - Log relay/transformer (Fluentd_)
   - Data store (Elastic_)
   - API/Presentation Layer (Kibana_)
   - Performance receptor (Collectd_)
   - Aggregator/Relay (Graphite_)
   - An API/Presentation Layer (Grafana_)

#. Undercloud:

   - There is no operational tools installed by default on the undercloud

#. Overcloud:

   - Monitoring Agent (Sensu_)
   - Log Collection Agent (Fluentd_)
   - Performance Collector Agent (Collectd_)

.. _RabbitMQ: https://www.rabbitmq.com
.. _Sensu: http://sensuapp.org
.. _Redis: https://redis.io
.. _Uchiwa: https://uchiwa.io
.. _Fluentd: http://www.fluentd.org
.. _Elastic: https://www.elastic.co
.. _Kibana: https://www.elastic.co/products/kibana
.. _Collectd: https://collectd.org
.. _Graphite: https://graphiteapp.org
.. _Grafana: https://grafana.com

Deploying the Operational Tool Server
-------------------------------------

There is an ansible project called opstools-ansible (OpsTools_) on github that helps to install the Operator Server, further documentation of the operational tool server instalation can be founded at (OpsToolsDoc_).

.. _OpsTools: https://github.com/centos-opstools/opstools-ansible
.. _OpsToolsDoc: https://github.com/centos-opstools/opstools-doc

Deploying the Undercloud
------------------------

As there is nothing to install on the undercloud nothing needs to be done.

Before deploying the Overcloud
------------------------------

.. note::

    The :doc:`../deployment/template_deploy` document has a more detailed explanation of the
    following steps.


1. Install client packages on overcloud-full image:

   - Mount the image and create a chroot::

      temp_dir=$(mktemp -d)
      sudo tripleo-mount-image -a /path/to/overcloud-full.qcow2 -m $temp_dir
      sudo mount -o bind /dev $temp_dir/dev/
      sudo cp /etc/resolv.conf $temp_dir/etc/resolv.conf
      sudo chroot $temp_dir /bin/bash

   - Install the packages inside the chroot::

      dnf install -y centos-release-opstools
      dnf install -y sensu fluentd collectd
      exit

   - Unmount the image::

      sudo rm $temp_dir/etc/resolv.conf
      sudo umount $temp_dir/dev
      sudo tripleo-unmount-image -m $temp_dir

   - Upload new image to undercloud image registry::

      openstack overcloud image upload --update-existing

2. Operational tools configuration files:

   The files have some documentation about the parameters that need to be configured

   - Availability Monitoring::

        /usr/share/openstack-tripleo-heat-templates/environments/monitoring-environment.yaml

   - Centralized Logging::

        /usr/share/openstack-tripleo-heat-templates/environments/logging-environment.yaml

   - Performance Monitoring::

        /usr/share/openstack-tripleo-heat-templates/environments/collectd-environment.yaml

3. Configure the environment

   The easiest way to configure our environment will be to create a parameter file, let's called parameters.yaml with all the parameters defined.

   - Availability Monitoring::

         MonitoringRabbitHost: server_ip          # Server were the rabbitmq was installed
         MonitoringRabbitPort: 5672               # Rabbitmq port
         MonitoringRabbitUserName: sensu_user     # the rabbitmq user to be used by sensu
         MonitoringRabbitPassword: sensu_password # The password of the sensu user
         MonitoringRabbitUseSSL: false            # Set to false
         MonitoringRabbitVhost: "/sensu_vhost"    # The virtual host of the rabbitmq

   - Centralized Logging::

         LoggingServers:        # The servers
           - host: server_ip    # The ip of the server
             port: 24224        # Port to send the logs [ 24224 plain & 24284 SSL ]
         LoggingUsesSSL: false  # Plain or SSL connections
                                # If LoggingUsesSSL is set to  false the following lines can
                                # be deleted
         LoggingSharedKey: secret           # The key
         LoggingSSLCertificate: |           # The content of the SSL Certificate
           -----BEGIN CERTIFICATE-----
           ...contens of server.pem here...
           -----END CERTIFICATE-----

   - Performance Monitoring::

         CollectdServer: collectd0.example.com   # Collectd server, where the data is going to be sent
         CollectdServerPort: 25826               # Collectd port
         # CollectdSecurityLevel: None           # Security by default None the other values are
                                                 #   Encrypt & Sign, but the two following parameters
                                                 #   need to be set too
         # CollectdUsername: user                # User to connect to the server
         # CollectdPassword: password            # Password to connect to the server

                                                 # Collectd, by default, comes with several plugins
                                                 #  extra plugins can added on this parameter
         CollectdExtraPlugins:
           - disk                                # disk plugin
           - df                                  # df   plugin
         ExtraConfig:                            # If the plugins need to be set, this is the location
           collectd::plugin::disk::disks:
             - "/^[vhs]d[a-f][0-9]?$/"
           collectd::plugin::df::mountpoints:
             - "/"
           collectd::plugin::df::ignoreselected: false


4. Continue following the TripleO instructions for deploying an overcloud::

        openstack overcloud deploy --templates \
           [-e /usr/share/openstack-tripleo-heat-templates/environments/monitoring-environment.yaml] \
           [-e  /usr/share/openstack-tripleo-heat-templates/environments/logging-environment.yaml] \
           [-e /usr/share/openstack-tripleo-heat-templates/environments/collectd-environment.yaml] \
           -e parameters.yaml


5. Wait for the completion of the overcloud deployment process.
