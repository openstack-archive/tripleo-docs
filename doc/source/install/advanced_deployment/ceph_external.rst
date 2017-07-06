Use an external Ceph cluster with the Overcloud
===============================================

|project| supports use of an external Ceph cluster for certain services deployed
in the Overcloud.

This happens by enabling a particular environment file when deploying the
Overcloud, specifically `environments/puppet-ceph-external.yaml`.

Some of the parameters in the above environment file can be overridden::

  parameter_defaults:
    # Enable use of RBD backend in nova-compute
    NovaEnableRbdBackend: true
    # Enable use of RBD backend in cinder-volume
    CinderEnableRbdBackend: true
    # Backend to use for cinder-backup
    CinderBackupBackend: ceph
    # Backend to use for glance
    GlanceBackend: rbd
    # Backend to use for gnocchi-metricsd
    GnocchiBackend: rbd
    # Name of the Ceph pool hosting Nova ephemeral images
    NovaRbdPoolName: vms
    # Name of the Ceph pool hosting Cinder volumes
    CinderRbdPoolName: volumes
    # Name of the Ceph pool hosting Cinder backups
    CinderBackupRbdPoolName: backups
    # Name of the Ceph pool hosting Glance images
    GlanceRbdPoolName: images
    # Name of the Ceph pool hosting Gnocchi metrics
    GnocchiRbdPoolName: metrics
    # Name of the user to authenticate with the external Ceph cluster
    CephClientUserName: openstack

The pools and the CephX user **must** be created on the external Ceph cluster
before deploying the Overcloud. TripleO expects a single user, configured via
CephClientUserName, to have the capabilities to use all the OpenStack pools;
the user could be created with a command like this::

  ceph auth add client.openstack mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rwx pool=images, allow rwx pool=backups, allow rwx pool=metrics'

In addition to the above customizations, the deployer **needs** to provide
at least three required parameters related to the external Ceph cluster::

  parameter_defaults:
    # The cluster FSID
    CephClusterFSID: '4b5c8c0a-ff60-454b-a1b4-9747aa737d19'
    # The CephX user auth key
    CephClientKey: 'AQDLOh1VgEp6FRAAFzT7Zw+Y9V6JJExQAsRnRQ=='
    # The list of Ceph monitors
    CephExternalMonHost: '172.16.1.7, 172.16.1.8, 172.16.1.9'

Last but not least, from the Newton release TripleO will install Ceph Jewel. If
the external Ceph cluster uses the Hammer release instead, you should also
pass the following parameters to enable backward compatibility features::

  parameter_defaults:
    ExtraConfig:
      ceph::profile::params::rbd_default_features: '1'

Finally add the above environment files to the deploy commandline::

  openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/puppet-ceph-external.yaml -e ~/my-additional-ceph-settings.yaml
