Configuring High Availability
=============================

|project| supports high availability of the controller services using
Pacemaker. To enable this feature, you need at least three controller
nodes, enable Pacemaker as the resource manager and specify an NTP
server.

Adding the following arguments to your `openstack overcloud deploy`
command will do the trick::

  --control-scale 3 -e /usr/share/openstack-tripleo-heat-templates/environments/puppet-pacemaker.yaml --ntp-server pool.ntp.org
