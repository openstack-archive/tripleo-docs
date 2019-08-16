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


Once the deployment is completed, the overcloud should show a stonith device for each compute node and one for each controller node and a GuestNode for every compute node. The expected behavior is that if a compute node dies, it will be fenced and the VMs that were running on it will be evacuated (i.e. restarted) on another compute node.

In case it is necessary to limit which VMs are to be resuscitated on another compute node it is possible to tag with ``evacuable`` either the image::

   openstack image set --tag evacuable 0c305437-89eb-48bc-9997-e4e4ea77e449

the flavor::

   nova flavor-key bb31d84a-72b3-4425-90f7-25fea81e012f set evacuable=true

or the VM::

   nova server-tag-add 89b70b07-8199-46f4-9b2d-849e5cdda3c2 evacuable

At the moment this last method should be avoided because of a significant reason: setting the tag on a single VM means that just *that* instance will be evacuated, tagging no VM implies that *all* the servers on the compute node will resuscitate. In a partial tagging situation, if a compute node runs only untagged VMs, the cluster will evacuate all of them, ignoring the overall tag status.
