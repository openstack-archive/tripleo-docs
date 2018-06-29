Configuring Messaging RPC and Notifications
===========================================

TripleO can configure oslo.messaging RPC and Notification services and
deploy the corresponding messaging backends for the undercloud and
overcloud. The roles OsloMessagingRPC and OsloMessagingNotify have been
added in place of the RabbitMQ Server. Having independent roles for RPC
and Notify allows for the separation of messaging backends as well as
the deployment of different messaging backend intermediaries that are
supported by oslo.messaging drivers::

  +----------------+-----------+-----------+-----+--------+-----------+
  | Oslo.Messaging | Transport |  Backend  | RPC | Notify | Messaging |
  |     Driver     | Protocol  |  Server   |     |        |   Type    |
  +================+===========+===========+=====+========+===========+
  |     rabbit     | AMQP V0.9 | rabbitmq  | yes |   yes  |   queue   |
  +----------------+-----------+-----------+-----+--------+-----------+
  |      amqp      | AMQP V1.0 | qdrouterd | yes |        |   direct  |
  +----------------+-----------+-----------+-----+--------+-----------+
  |     kafka      |   kafka   |  kafka    |     |   yes  |   queue   |
  | (experimental) |   binary  |           |     |        |  (stream) |
  +----------------+-----------+-----------+-----+--------+-----------+

Standard Deployment of RabbitMQ Server Backend
----------------------------------------------

A single RabbitMQ backend (e.g. server or cluster) is the default
deployment for TripleO. This messaging backend provides the services
for both RPC and Notification communications through its integration
with the oslo.messaging rabbit driver.

The example `standard messaging`_ environment file depicts the
resource association for this defacto deployment configuration::

  # *******************************************************************
  # This file was created automatically by the sample environment
  # generator. Developers should use `tox -e genconfig` to update it.
  # Users are recommended to make changes to a copy of the file instead
  # of the original, if any customizations are needed.
  # *******************************************************************
  # title: Share single rabbitmq backend for rpc and notify messaging backend
  # description: |
  #   Include this environment to enable a shared rabbitmq backend for
  #   oslo.messaging rpc and notification services
    parameter_defaults:
    # The network port for messaging backend
    # Type: number
    RpcPort: 5672

  resource_registry:
    OS::TripleO::Services::OsloMessagingNotify: ../../docker/services/messaging/notify-rabbitmq-shared.yaml
    OS::TripleO::Services::OsloMessagingRpc: ../../docker/services/messaging/rpc-rabbitmq.yaml

The `rpc-rabbitmq.yaml`_ instantiates the rabbitmq server backend
while `notify-rabbitmq-shared.yaml`_ sets up the notification
transport configuration to use the same shared rabbitmq server.

Deployment of Separate RPC and Notify Messaging Backends
--------------------------------------------------------

Separate messaging backends can be deployed for RPC and Notification
communications. For this TripleO deployment, the apache dispatch
router (qdrouterd) can be deployed for the RPC messaging backend using
the oslo.messaging AMQP 1.0 driver.

The example `hybrid messaging`_ environment file can be used for an
overcloud deployment::

  # *******************************************************************
  # This file was created automatically by the sample environment
  # generator. Developers should use `tox -e genconfig` to update it.
  # Users are recommended to make changes to a copy of the file instead
  # of the original, if any customizations are needed.
  # *******************************************************************
  # title: Hybrid qdrouterd for rpc and rabbitmq for notify messaging backend
  # description: |
  #   Include this environment to enable hybrid messaging backends for
  #   oslo.messaging rpc and notification services
  parameter_defaults:
    # The network port for messaging Notify backend
    # Type: number
    NotifyPort: 5672

    # The network port for messaging backend
    # Type: number
    RpcPort: 31459

  resource_registry:
    OS::TripleO::Services::OsloMessagingNotify: ../../docker/services/messaging/notify-rabbitmq.yaml
    OS::TripleO::Services::OsloMessagingRpc: ../../docker/services/messaging/rpc-qdrouterd.yaml

The above will instantiate qdrouterd server(s) and configure them for
use as the RPC transport and will also instantiate the rabbitmq backend
and configure it for use as the Notification transport. It should
be noted that the RPC and Notify ports must be distinct to prevent the
qdrouterd and rabbitmq servers from simultaneously using the amqp
standard port (5672).

Add the following arguments to your `openstack overcloud deploy`
command to deploy with separate messaging backends::

  openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/messaging/rpc-qdrouterd-notify-rabbitmq-hybrid.yaml

.. _`standard messaging`: https://github.com/openstack/tripleo-heat-templates/blob/master/environments/messaging/rpc-rabbitmq-notify-rabbitmq-shared.yaml
.. _`rpc-rabbitmq.yaml`: https://github.com/openstack/tripleo-heat-templates/blob/master/docker/services/messaging/rpc-rabbitmq.yaml
.. _`notify-rabbitmq-shared.yaml`: https://github.com/openstack/tripleo-heat-templates/blob/master/docker/services/messaging/notify-rabbitmq-shared.yaml
.. _`hybrid messaging`: https://github.com/openstack/tripleo-heat-templates/blob/master/environments/messaging/rpc-qdrouterd-notify-rabbitmq-hybrid.yaml
