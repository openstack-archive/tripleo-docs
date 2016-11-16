Configuring High Availability
=============================

|project| supports high availability of the controller services using
Pacemaker. To enable this feature, you need at least three controller
nodes, enable Pacemaker as the resource manager and specify an NTP
server.

Creating an environment file and adding the following arguments to your
`openstack overcloud deploy` command will do the trick::

  $ cat ~/environment.yaml
  parameter_defaults:
    ControllerCount: 3

  -e environment.yaml -e /usr/share/openstack-tripleo-heat-templates/environments/puppet-pacemaker.yaml --ntp-server pool.ntp.org
