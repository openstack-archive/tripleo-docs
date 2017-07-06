Configuring Cinder with a Custom Unmanaged Backend
==================================================

This guide assumes that your undercloud is already installed and ready to
deploy an overcloud.

Adding a custom backend to Cinder
---------------------------------

It is possible to provide the config settings to add an arbitrary and
unmanaged backend to Cinder at deployment time via Heat environment files.

Each backend is represented in `cinder.conf` with a ``stanza`` and a
reference to it from the `enabled_backends` key. The keys valid in the
backend ``stanza`` are dependent on the actual backend driver and
unknown to Cinder.

For example, to provision in Cinder two additional backends one could
create a Heat environment file with the following contents::

  parameter_defaults:
    ExtraConfig:
      cinder::config::cinder_config:
          netapp1/volume_driver:
                  value: cinder.volume.drivers.netapp.common.NetAppDriver
          netapp1/netapp_storage_family:
                  value: ontap_7mode
          netapp1/netapp_storage_protocol:
                  value: iscsi
          netapp1/netapp_server_hostname:
                  value: 1.1.1.1
          netapp1/netapp_server_port:
                  value: 80
          netapp1/netapp_login:
                  value: root
          netapp1/netapp_password:
                  value: 123456
          netapp1/volume_backend_name:
                  value: netapp_1
          netapp2/volume_driver:
                  value: cinder.volume.drivers.netapp.common.NetAppDriver
          netapp2/netapp_storage_family:
                  value: ontap_7mode
          netapp2/netapp_storage_protocol:
                  value: iscsi
          netapp2/netapp_server_hostname:
                  value: 2.2.2.2
          netapp2/netapp_server_port:
                  value: 80
          netapp2/netapp_login:
                  value: root
          netapp2/netapp_password:
                  value: 123456
          netapp2/volume_backend_name:
                  value: netapp_2
      cinder_user_enabled_backends: ['netapp1','netapp2']

This will not interfere with the deployment of the other backends managed by
TripleO, like Ceph or NFS and will just add these two to the list of the
backends enabled in Cinder.

Remember to add such an environment file to the deploy commandline::

    openstack overcloud deploy [other overcloud deploy options] -e ~/my-backends.yaml

.. note::

    The :doc:`extra_config` doc has more details on the usage of the different
    ExtraConfig interfaces.
