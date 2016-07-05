Deploying Operational Tools
===========================

TripleO comes with an optional suite of tools designed to help operators
maintain an OpenStack environment. The tools perform the following functions:

- Availability Monitoring
- Centralized Logging

This document will go through the presentation and installation of these tools.

Architecture
------------

#. Undercloud:

   - Monitoring Relay/proxy (RabbitMQ_)
   - Monitoring Controller/Server (Sensu_)
   - API/Presentation Layer (Uchiwa_)
   - Log relay/transformer (Fluentd_)
   - Data store (Elastic_)
   - API/Presentation Layer (Kibana_)

#. Overcloud:

   - Monitoring Agent (Sensu_)
   - Log Collection Agent (Fluentd_)

.. _RabbitMQ: https://www.rabbitmq.com
.. _Sensu: http://sensuapp.org
.. _Uchiwa: https://uchiwa.io
.. _Fluentd: http://www.fluentd.org
.. _Elastic: https://www.elastic.co
.. _Kibana: https://www.elastic.co/products/kibana

Deploying the Undercloud
------------------------

#. Edit ``undercloud.conf`` before the deployment:

   - set ``enable_monitoring`` to true if you want to enable Availability
     Monitoring.
     You can also specify a password for ``undercloud_sensu_password``,
     otherwise one will be automatically generated.
   - set ``enable_logging`` to true if you want to enable Centralized Logging.
     By default, the indexes will be closed after 7 days and deleted after
     10 days by using Curator_. You can change that with the 2 parameters:
     ``elastic_close_index_days`` and ``elastic_delete_index_days``.

#. That's it, you're ready to deploy your undercloud!

.. _Curator: https://www.elastic.co/guide/en/elasticsearch/client/curator/current/index.html

Deploying the Overcloud
-----------------------

.. note::

    The :doc:`template_deploy` document has a more detailed explanation of the
    following steps.

#. As the ``stack`` user, copy the Operational tools configuration files to your
   home directory:

   - Availability Monitoring::

      cp /usr/share/openstack-tripleo-heat-templates/environments/monitoring-sensu-config.yaml ~

   - Centralized Logging::

      cp /usr/share/openstack-tripleo-heat-templates/environments/logging-fluentd-config.yaml ~

#. Edit the parameters in the Operational tools configuration files to fit your
   requirements.

#. Continue following the TripleO instructions for deploying an overcloud.
   Before entering the command to deploy the overcloud, add the environment
   file that you just configured as an argument::

    openstack overcloud deploy --templates -e ~/monitoring-sensu-config.yaml -e ~/logging-fluentd-config.yaml

#. Wait for the completion of the overcloud deployment process.

Accessing to the Dashboards
---------------------------

#. Availability Monitoring: you can reach the Sensu dashboard (Uchiwa) on this
   URL::

    http://<UNDERCLOUD_IP_ADDRESS>:3000

#. Centralized Logging: you can reach the Elastic dashboard (Kibana) on this
   URL::

    http://<UNDERCLOUD_IP_ADDRESS>:8300/index.html#/dashboard/file/logstash.json
