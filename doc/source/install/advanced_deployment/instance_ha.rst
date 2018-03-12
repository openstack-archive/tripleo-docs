Configuring Instance High Availability
======================================

|project|, starting with the Queens release, supports a form of instance
high availability when the overcloud is deployed in a specific way.

In order to activate instance high-availability (also called ``IHA``)
the following steps are needed:

1. Add the following environment file to your overcloud deployment command. Make sure you are deploying an HA overcloud::

   -e /usr/share/openstack-tripleo-heat-templates/environments/compute-instanceha.yaml

2. Instead of using the ``Compute`` role use the ``ComputeInstanceHA`` role for your compute plane. The ``ComputeInstanceHA`` role has the following additional services when compared to the ``Compute`` role::

   - OS::TripleO::Services::ComputeInstanceHA
   - OS::TripleO::Services::PacemakerRemote

3. Make sure that fencing is configured for the whole overcloud (controllers and computes). You can do so by adding an environment file to your deployment command that contains the necessary fencing information. For example::

    parameter_defaults:
      EnableFencing: true
      FencingConfig:
        devices:
        - agent: fence_ipmilan
          host_mac: 00:ec:ad:cb:3c:c7
          params:
            login: admin
            ipaddr: 192.168.24.1
            ipport: 6230
            passwd: password
            lanplus: 1
        - agent: fence_ipmilan
          host_mac: 00:ec:ad:cb:3c:cb
          params:
            login: admin
            ipaddr: 192.168.24.1
            ipport: 6231
            passwd: password
            lanplus: 1
        - agent: fence_ipmilan
          host_mac: 00:ec:ad:cb:3c:cf
          params:
            login: admin
            ipaddr: 192.168.24.1
            ipport: 6232
            passwd: password
            lanplus: 1
        - agent: fence_ipmilan
          host_mac: 00:ec:ad:cb:3c:d3
          params:
            login: admin
            ipaddr: 192.168.24.1
            ipport: 6233
            passwd: password
            lanplus: 1
        - agent: fence_ipmilan
          host_mac: 00:ec:ad:cb:3c:d7
          params:
            login: admin
            ipaddr: 192.168.24.1
            ipport: 6234
            passwd: password
            lanplus: 1


Once the deployment is completed, the overcloud should show a stonith device for each compute node and one for each controller node and a GuestNode for every compute node. The expected behaviour is that if a compute node dies, it will be fenced and the VMs that were running on it will be evacuated (i.e. restarted) on another compute node.

In case it is necessary to limit which VMs are to be resuscitated on another compute node, it is possible to either tag the image, the flavor or the VM with the ``evacuable`` tag. For example::

  openstack image set --tag evacuable 0c305437-89eb-48bc-9997-e4e4ea77e449
