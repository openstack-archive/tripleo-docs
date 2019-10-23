.. _deployed_server:

Using Already Deployed Servers
==============================

TripleO can be used with servers that have already been deployed and
provisioned with a running operating system.

In this deployment scenario, Nova and Ironic from the Undercloud are not used
to do any server deployment, installation, or power management. An external to
TripleO and already existing provisioning tool is expected to have already
installed an operating system on the servers that are intended to be used as
nodes in the Overcloud.

.. note::
   It's an all or nothing approach when using already deployed servers. Mixing
   using deployed servers with servers provisioned with Nova and Ironic is not
   currently possible.

Benefits to using this feature include not requiring a dedicated provisioning
network, and being able to use a custom partitioning scheme on the already
deployed servers.

Deployed Server Requirements
----------------------------

Networking
^^^^^^^^^^

Network interfaces
__________________

It's recommended that each server have a dedicated management NIC with
externally configured connectivity so that the servers are reachable outside of
any networking configuration done by the OpenStack deployment.

A separate interface, or set of interfaces should then be used for the
OpenStack deployment itself, configured in the typical fashion with a set of
NIC config templates during the Overcloud deployment. See
:doc:`../features/network_isolation` for more information on configuring networking.

.. note::

  When configuring network isolation be sure that the configuration does not
  result in a loss of network connectivity from the deployed servers to the
  undercloud. The interface(s) that are being used for this connectivity should
  be excluded from the NIC config templates so that the configuration does not
  unintentionally drop all networking access to the deployed servers.


Undercloud
__________

Neutron in the Undercloud is not used for providing DHCP services for the
Overcloud nodes, hence a dedicated provisioning network with L2 connectivity is
not a requirement in this scenario. Neutron is however still used for IPAM for
the purposes of assigning IP addresses to the port resources created by
tripleo-heat-templates.

Network L3 connectivity is still a requirement between the Undercloud and
Overcloud nodes. The Overcloud nodes will communicate over HTTP(s) to poll the
Undercloud for software configuration to be applied by their local agents.

The polling process requires L3 routable network connectivity from the deployed
servers to the Undercloud OpenStack API's.

If the ctlplane is a routable network from the deployed servers, then the
deployed servers can connect directly to the IP address specified by
``local_ip`` from ``undercloud.conf``. Alternatively, they could connect to the
virtual IP address (VIP) specified by ``undercloud_public_host``, if VIP's are
in use.

In the scenario where the ctlplane is **not** routable from the deployed
servers, then ``undercloud_public_host`` in ``undercloud.conf`` must be set to
a hostname that resolves to a routable IP address for the deployed servers. SSL
also must be configured on the Undercloud so that HAProxy is bound to that
configured hostname. Specify either ``undercloud_service_certifcate`` or
``generate_service_certificate`` to enable SSL during the Undercloud
installation. See :doc:`../features/ssl` for more information on configuring SSL.

Additionally, when the ctlplane is not routable from the deployed
servers, Heat on the Undercloud must be configured to use the public
endpoints for OpenStack service communication during the polling process
and be configured to use Swift temp URL's for signaling. Add the
following hiera data to a new or existing hiera file::

    heat_clients_endpoint_type: public
    heat::engine::default_deployment_signal_transport: TEMP_URL_SIGNAL

Specify the path to the hiera file with the ``hieradata_override``
configuration in ``undercloud.conf``::

    hieradata_override = /path/to/custom/hiera/file.yaml

Overcloud
_________

Configure the deployed servers that will be used as nodes in the Overcloud with
L3 connectivity to the Undercloud as needed. The configuration could be done
via static or DHCP IP assignment.

Further networking configuration of Overcloud nodes is the same as in a typical
TripleO deployment, except for:

* Initial configuration of L3 connectivity to the Undercloud
* No requirement for dedicating a separate L2 network for provisioning

Testing Connectivity
____________________

On each Overcloud node run the following commands that test connectivity to the
Undercloud's IP address where OpenStack services are bound. Use either
``local_ip`` or ``undercloud_public_host`` in the following examples.

Test basic connectivity to the Undercloud::

  ping <undercloud local_ip>

Test HTTP/HTTPS connectivity to Heat API on the Undercloud::

  curl <undercloud local_ip>:8000

Sample output::

  {"versions": [{"status": "CURRENT", "id": "v1.0", "links": [{"href": "http://10.12.53.41:8000/v1/", "rel": "self"}]}]}

Test HTTP/HTTPS connectivity to Swift on the Undercloud The html output shown
here is expected! While it indicates no resource was found, it demonstrates
successful connectivity to the HTTP service::

  curl <undercloud local_ip>:8080

Sample output::

  <html><h1>Not Found</h1><p>The resource could not be found.</p></html>

The output from the above curl commands demonstrates successful connectivity to
the web services bound at the Undercloud's ``local_ip`` IP address. It's
important to verify this connectivity prior to starting the deployment,
otherwise the deployment may be unsuccessful and difficult to debug.

Package repositories
^^^^^^^^^^^^^^^^^^^^

The servers will need to already have the appropriately enabled yum repositories
as packages will be installed on the servers during the Overcloud deployment.
The enabling of repositories on the Overcloud nodes is the same as it is for
other areas of TripleO, such as Undercloud installation. See
:doc:`../repositories` for the detailed steps on how to
enable the standard repositories for TripleO.

Initial Package Installation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Once the repositories have been enabled on the deployed servers, the initial
packages for the Heat agent need to be installed. Run the following command on
each server intending to be used as part of the Overcloud::

    sudo yum install python-heat-agent*

Certificate Authority Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If SSL is enabled on the Undercloud endpoints, the deployed servers need to be
configured to trust the Certificate Authority (CA) that signed the SSL
certificates.

On a default Undercloud install with SSL where the CA is automatically
generated, the CA file will be at
``/etc/pki/ca-trust/source/anchors/cm-local-ca.pem``.  Copy this CA file to the
``/etc/pki/ca-trust/source/anchors/`` directory on each deployed server. Then
run the following command on each server to update the CA trust::

  sudo update-ca-trust extract

Deploying the Overcloud
-----------------------

Deployment Command
^^^^^^^^^^^^^^^^^^

The functionality of using already deployed servers is enabled by passing
additional Heat environment files to the ``openstack overcloud deploy``
command.::

    openstack overcloud deploy \
      <other cli arguments> \
      --disable-validations \
      -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-environment.yaml

The ``--disable-validations`` option disables the basic Nova, Ironic, and
Glance related validations executed by python-tripleoclient. These validations
are not necessary since those services will not be used to deploy the
Overcloud.

The ``deployed-server.yaml`` environment takes advantage of the template
composition nature of Heat and tripleo-heat-templates to substitute
``OS::Heat::DeployedServer`` resources in place of ``OS::Nova::Server``.

.. note::

   Previously a custom roles file was needed when using deployed-server. The
   custom roles file was located in the templates directory at
   deployed-server/deployed-server-roles-data.yaml. The custom roles file
   addressed setting ``disable_constraints: true`` on each of the roles. This
   is no longer required starting in the train release.

.. note::

   Previously, environment files were used to enable bootstrap tasks on the
   deployed servers. These files were
   environments/deployed-server-bootstrap-environment-centos.yaml and
   environments/deployed-server-bootstrap-environment-rhel.yaml. Starting in
   the train release, these environment files are no longer required and they
   have been removed from tripleo-heat-templates.

Optionally include an environment to set the ``DeploymentSwiftDataMap``
paramter called ``deployment-swift-data-map.yaml``::

    # Append to deploy command
    -e deployment-swift-data-map.yaml \

This environment sets the Swift container and object names for the deployment
metadata for each deployed server. This environment file must be written
entirely by the user. Example contents are as follows::

    parameter_defaults:
      DeploymentSwiftDataMap:
        overcloud-controller-0:
          container: overcloud-controller
          object: 0
        overcloud-controller-1:
          container: overcloud-controller
          object: 1
        overcloud-controller-2:
          container: overcloud-controller
          object: 2
        overcloud-novacompute-0:
          container: overcloud-compute
          object: 0

The ``DeploymentSwiftDataMap`` parameter's value is a dict. The keys are the
Heat assigned names for each server resource. The values are another dict of
the Swift container and object name Heat should use for storing the deployment
data for that server resource. These values should match the container and
object names as described in the
:ref:`pre-configuring-metadata-agent-configuration` section.

deployed-server with config-download
____________________________________
When using :doc:`config-download <../deployment/ansible_config_download>` with
``deployed-server`` (pre-provisioned nodes), a ``HostnameMap`` parameter must
be provided. Create an environment file to define the parameter, and assign the
node hostnames in the parameter value. The following example shows a sample
value::

  parameter_defaults:
    HostnameMap:
      overcloud-controller-0: controller-00-rack01
      overcloud-controller-1: controller-01-rack02
      overcloud-controller-2: controller-02-rack03
      overcloud-novacompute-0: compute-00-rack01
      overcloud-novacompute-1: compute-01-rack01
      overcloud-novacompute-2: compute-02-rack01

Write the contents to an environment file such as ``hostnamemap.yaml``, and
pass it the environment as part of the deployment command.



Network Configuration
_____________________

The default network interface configuration mappings for the deployed-server
roles are::

  OS::TripleO::ControllerDeployedServer::Net::SoftwareConfig: net-config-static-bridge.yaml
  OS::TripleO::ComputeDeployedServer::Net::SoftwareConfig: net-config-static.yaml
  OS::TripleO::BlockStorageDeployedServer::Net::SoftwareConfig: net-config-static.yaml
  OS::TripleO::ObjectStorageDeployedServer::Net::SoftwareConfig: net-config-static.yaml
  OS::TripleO::CephStorageDeployedServer::Net::SoftwareConfig: net-config-static.yaml

The default NIC configs use static IP assignment instead of the default of
DHCP. This is due to there being no requirement of L2 connectivity between the
undercloud and overcloud.  However, the NIC config templates can be overridden
to use whatever configuration is desired (including DHCP).

As is the case when not using deployed-servers, the following parameters need
to also be specified::

    parameter_defaults:
      NeutronPublicInterface: eth1
      ControlPlaneDefaultRoute: 192.168.24.1
      EC2MetadataIp: 192.168.24.1

``ControlPlaneDefaultRoute`` and ``EC2MetadataIp`` are not necessarily
meaningful parameters depending on the network architecture in use with
deployed servers. However, they still must be specified as they are required
parameters for the template interface.

The ``DeployedServerPortMap`` parameter can be used to assign fixed IP's
from either the ctlplane network or the IP address range for the
overcloud.

If the deployed servers were preconfigured with IP addresses from the ctlplane
network for the initial undercloud connectivity, then the same IP addresses can
be reused during the overcloud deployment. Add the following to a new
environment file and specify the environment file as part of the deployment
command::

    resource_registry:
      OS::TripleO::DeployedServer::ControlPlanePort: ../deployed-server/deployed-neutron-port.yaml
    parameter_defaults:
      DeployedServerPortMap:
        controller0-ctlplane:
          fixed_ips:
            - ip_address: 192.168.24.9
          subnets:
            - cidr: 192.168.24.0/24
          network:
            tags:
              - 192.168.24.0/24
        compute0-ctlplane:
          fixed_ips:
            - ip_address: 192.168.24.8
          subnets:
            - cidr: 192.168.24..0/24
          network:
            tags:
              - 192.168.24.0/24

The value of the DeployedServerPortMap variable is a map. The keys correspond
to the ``<short hostname>-ctlplane`` of the deployed servers. Specify the ip
addresses and subnet CIDR to be assigned under ``fixed_ips``.

In the case where the ctlplane is not routable from the deployed
servers, you can use ``DeployedServerPortMap`` to assign an IP address
from any CIDR::

    resource_registry:
      OS::TripleO::DeployedServer::ControlPlanePort: /usr/share/openstack-tripleo-heat-templates/deployed-server/deployed-neutron-port.yaml
      OS::TripleO::Network::Ports::ControlPlaneVipPort: /usr/share/openstack-tripleo-heat-templates/deployed-server/deployed-neutron-port.yaml
      OS::TripleO::Network::Ports::RedisVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/noop.yaml

    parameter_defaults:
      NeutronPublicInterface: eth1
      EC2MetadataIp: 192.168.100.1
      ControlPlaneDefaultRoute: 192.168.100.1

      DeployedServerPortMap:
        control_virtual_ip:
          fixed_ips:
            - ip_address: 192.168.100.1
          subnets:
            - cidr: 192.168.100.0/24
          network:
            tags:
              - 192.168.100.0/24
        controller0-ctlplane:
          fixed_ips:
            - ip_address: 192.168.100.2
          subnets:
            - cidr: 192.168.100.0/24
          network:
            tags:
              - 192.168.100.0/24
        compute0-ctlplane:
          fixed_ips:
            - ip_address: 192.168.100.3
          subnets:
            - cidr: 192.168.100.0/24
          network:
            tags:
              - 192.168.100.0/24

In the above example, notice how ``RedisVipPort`` is mapped to
``network/ports/noop.yaml``. This mapping is due to the fact that the
Redis VIP IP address comes from the ctlplane by default. The
``EC2MetadataIp`` and ``ControlPlaneDefaultRoute`` parameters are set
to the value of the control virtual IP address. These parameters are
required to be set by the sample NIC configs, and must be set to a
pingable IP address in order to pass the validations performed during
deployment. Alternatively, the NIC configs could be further customized
to not require these parameters.

When using network isolation, refer to the documentation on using fixed
IP addresses for further information at :ref:`predictable_ips`.

Configuring Deployed Servers to poll Heat
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _pre-configuring-metadata-agent-configuration:

Pre-configuring metadata agent configuration
____________________________________________

Beginning with the Pike release of TripleO, the deployed server's agents can be
configured for polling of the Heat deployment data independently of creating the
overcloud stack.

This is accomplished using the ``DeploymentSwiftDataMap`` parameter as shown in
the previous section. Once the Swift container and object names are chosen for
each deployed server, create Swift temporary url's that correspond to each
container/object and configure the temporary url in the agent configuration for
the respective deployed server.

For this example, the following ``DeploymentSwiftDataMap`` parameter value is
assumed to be::

    parameter_defaults:
      DeploymentSwiftDataMap:
        overcloud-controller-0:
          container: overcloud-controller
          object: 0
        overcloud-controller-1:
          container: overcloud-controller
          object: 1
        overcloud-controller-2:
          container: overcloud-controller
          object: 2
        overcloud-novacompute-0:
          container: overcloud-compute
          object: 0

Start by showing the Swift account and temporary URL key::

    swift stat

Sample output looks like::

    Account: AUTH_aa7784aae1ae41c38e6e01fd76caaa7c
    Containers: 5
    Objects: 706
    Bytes: 3521748
    Containers in policy "policy-0": 5
    Objects in policy "policy-0": 706
    Bytes in policy "policy-0": 3521748
    Meta Temp-Url-Key: 25ad317c25bb89c62f5730f3b8cf8fca
    X-Account-Project-Domain-Id: default
    X-Openstack-Request-Id: txedaadba016cd474dac37f-00595ea5af
    X-Timestamp: 1499288311.20888
    X-Trans-Id: txedaadba016cd474dac37f-00595ea5af
    Content-Type: text/plain; charset=utf-8
    Accept-Ranges: bytes

Record the value of ``Account`` and ``Meta Temp-Url-Key`` from the output from
the above command.

If ``Meta Temp-Url-Key`` is not set, it can be set by running the following
command (choose a unique value for the actual key value)::

    swift post -m "Temp-URL-Key:b3968d0207b54ece87cccc06515a89d4"

Create temporary URL's for each Swift object specified in
``DeploymentSwiftDataMap``::

    swift tempurl GET 600000000 /v1/AUTH_aa7784aae1ae41c38e6e01fd76caaa7c/overcloud-controller/0 25ad317c25bb89c62f5730f3b8cf8fca
    swift tempurl GET 600000000 /v1/AUTH_aa7784aae1ae41c38e6e01fd76caaa7c/overcloud-controller/1 25ad317c25bb89c62f5730f3b8cf8fca
    swift tempurl GET 600000000 /v1/AUTH_aa7784aae1ae41c38e6e01fd76caaa7c/overcloud-controller/2 25ad317c25bb89c62f5730f3b8cf8fca
    swift tempurl GET 600000000 /v1/AUTH_aa7784aae1ae41c38e6e01fd76caaa7c/overcloud-compute/0 25ad317c25bb89c62f5730f3b8cf8fca

See ``swift tempurl --help`` for a detailed explanation of each argument.

The above commands output URL paths which need to be joined with the Swift
public api endpoint to construct the full metadata URL. In a default TripleO
deployment, this value is ``http://192.168.24.1:8080`` but is likely different
for any real deployment.

Joining the output from one of the above commands with the Swift public
endpoint results in a URL that looks like::

    http://192.168.24.1:8080/v1/AUTH_aa7784aae1ae41c38e6e01fd76caaa7c/overcloud-controller/0?temp_url_sig=92de8e4c66b77c54630dede8150b3ebcd46a1fca&temp_url_expires=700000000

Once each URL is obtained, configure the agent on each deployed server with its
respective metadata URL (e.g., use the metadata URL for controller 0 on the
deployed server intended to be controller 0, etc). Create the following file
(and ``local-data`` directory if necessary). Both should be root owned::

    mkdir -p /var/lib/os-collect-config/local-data
    /var/lib/os-collect-config/local-data/deployed-server.json

Example file contents::

    {
      "os-collect-config": {
        "collectors": ["request", "local"],
        "request": {
          "metadata_url": "http://192.168.24.1:8080/v1/AUTH_aa7784aae1ae41c38e6e01fd76caaa7c/overcloud-controller/0?temp_url_sig=92de8e4c66b77c54630dede8150b3ebcd46a1fca&temp_url_expires=700000000"
        }
      }
    }

The deployed server's agent is now configured.


Reading metadata configuration from Heat
________________________________________

If not using ``DeploymentSwiftDataMap``, the metadata configuration will have
to be read directly from Heat once the stack starts to create.

Upon executing the deployment command, Heat will begin creating the
``overcloud`` stack. The stack events are shown in the terminal as the stack
operation is in progress.

The resources corresponding to the deployed server will enter
CREATE_IN_PROGRESS. At this point, the Heat stack will not continue as it is
waiting for signals from the servers. The agents on the deployed servers need
to be configured to poll Heat for their configuration.

This point in the Heat events output will look similar to::

    2017-01-14 13:25:13Z [overcloud.Compute.0.NovaCompute]: CREATE_IN_PROGRESS  state changed
    2017-01-14 13:25:14Z [overcloud.Controller.0.Controller]: CREATE_IN_PROGRESS  state changed
    2017-01-14 13:25:14Z [overcloud.Controller.1.Controller]: CREATE_IN_PROGRESS  state changed
    2017-01-14 13:25:15Z [overcloud.Controller.2.Controller]: CREATE_IN_PROGRESS  state changed

The example output above is from a deployment with 3 controllers and 1 compute.
As seen, these resources have entered the CREATE_IN_PROGRESS state.

To configure the agents on the deployed servers, the request metadata url needs
to be read from Heat resource metadata on the individual resources, and
configured in the ``/etc/os-collect-config.conf`` configuration file on the
corresponding deployed servers.

Manual configuration of Heat agents
"""""""""""""""""""""""""""""""""""

These steps can be used to manually configure the Heat agents
(``os-collect-config``) on the deployed servers.

Query Heat for the request metadata url by first listing the nested
``deployed-server`` resources::

    openstack stack resource list -n 5 overcloud | grep deployed-server

Example output::

    | deployed-server | 895c08b8-f6f4-4564-b344-586603e7e970 | OS::Heat::DeployedServer | CREATE_COMPLETE    | 2017-01-14T13:25:12Z | overcloud-Controller-pgeu4nxsuq6r-1-v4slfaduprak-Controller-ltxdxz2fin3d |
    | deployed-server | 87cd8d81-8bbe-4c0b-9bd9-f5bcd1343265 | OS::Heat::DeployedServer | CREATE_COMPLETE    | 2017-01-14T13:25:15Z | overcloud-Controller-pgeu4nxsuq6r-0-5uin56wp3ign-Controller-5wkislg4kiv5 |
    | deployed-server | 3d387f61-dc6d-41f7-b3b8-5c9a0ab0ed7b | OS::Heat::DeployedServer | CREATE_COMPLETE    | 2017-01-14T13:25:16Z | overcloud-Controller-pgeu4nxsuq6r-2-m6tgzatgnqrb-Controller-yczqaulovrla |
    | deployed-server | cc230478-287e-4591-a905-bbfca6c89742 | OS::Heat::DeployedServer | CREATE_COMPLETE    | 2017-01-14T13:25:13Z | overcloud-Compute-vllmnqf5d77h-0-kfm2xsdmtmr6-NovaCompute-67djxtyrwi6z |

Show the resource metadata for one of the resources. The last column in the
above output is a nested stack name and is used in the command below. The
command shows the resource metadata for the first controller (Controller.0)::

    openstack stack resource metadata overcloud-Controller-pgeu4nxsuq6r-0-5uin56wp3ign-Controller-5wkislg4kiv5 deployed-server

The above command outputs a significant amount of JSON output representing the
resource metadata. To see just the request metadata_url, the command can be
piped to jq to show just the needed url::

    openstack stack resource metadata overcloud-Controller-pgeu4nxsuq6r-0-5uin56wp3ign-Controller-5wkislg4kiv5 deployed-server | jq -r '.["os-collect-config"].request.metadata_url'

Example output::

    http://10.12.53.41:8080/v1/AUTH_cf85adf63bc04912854473ff2b08b5a2/ov-ntroller-5wkislg4kiv5-deployed-server-yc4lx2d43dmb/244744c2-4af1-4626-92c6-94b2f78e3791?temp_url_sig=6d33b16ee2ae166a306633f04376ee54f0451ae4&temp_url_expires=2147483586

Using the above url, configure ``/etc/os-collect-config.conf`` on the deployed
server that is intended to be used as Controller 0. The full configuration
would be::

    [DEFAULT]
    collectors=request
    command=os-refresh-config
    polling_interval=30

    [request]
    metadata_url=http://10.12.53.41:8080/v1/AUTH_cf85adf63bc04912854473ff2b08b5a2/ov-ntroller-5wkislg4kiv5-deployed-server-yc4lx2d43dmb/244744c2-4af1-4626-92c6-94b2f78e3791?temp_url_sig=6d33b16ee2ae166a306633f04376ee54f0451ae4&temp_url_expires=2147483586

Once the configuration has been updated on the deployed server for Controller
0, restart the os-collect-config service::

    sudo systemctl restart os-collect-config

Repeat the configuration for the other nodes in the Overcloud, by querying Heat
for the request metadata url, and updating the os-collect-config configuration
on the respective deployed servers.

Once all the agents have been properly configured, they will begin polling for
the software deployments to apply locally from Heat, and the Heat stack will
continue creating. If the deployment is successful, the Heat stack will
eventually go to the ``CREATE_COMPLETE`` state.

Automatic configuration of Heat agents
""""""""""""""""""""""""""""""""""""""

A script is included with ``tripleo-heat-templates`` that can be used to do
automatic configuration of the Heat agent on the deployed servers instead of
relying on the above manual process.

The script requires that the environment variables needed to authenticate with
the Undercloud's keystone have been set in the current shell. These environment
variables can be set by sourcing the Undercloud's ``stackrc`` file.

The script also requires that the user executing the script can ssh as the same
user to each deployed server, and that the remote user account has password-less
sudo access.

The following shows an example of running the script::

    export OVERCLOUD_ROLES="ControllerDeployedServer ComputeDeployedServer"
    export ControllerDeployedServer_hosts="192.168.25.1 192.168.25.2 192.168.25.3"
    export ComputeDeployedServer_hosts="192.168.25.4"
    tripleo-heat-templates/deployed-server/scripts/get-occ-config.sh

As shown above, the script is further configured by the ``$OVERCLOUD_ROLES``
environment variable, and corresponding ``$<role-name>_hosts`` variables.

``$OVERCLOUD_ROLES`` is a space separated list of the role names used for the
Overcloud deployment. These role names correspond to the name of the roles from
the roles data file used during the deployment.

Each ``$<role-name>_hosts`` variable is a space separated **ordered** list of
IP addresses that are the IP addresses of the deployed servers for the roles.
For example, in the above command, 192.168.25.1 is the IP of Controller 0,
192.168.25.2 is the IP of Controller 1, etc.

.. Note:: The IP addresses for the hosts in the ``$<role-name>_hosts`` variable
          must be **ordered** to avoid Heat agent configuration mismatch.

          Start with the address for the node with the lowest node-index and
          count from there. For example, when deployed server IP addresses are:

          * overcloud-controller-0: 192.168.25.10
          * overcloud-controller-1: 192.168.25.11
          * overcloud-controller-2: 192.168.25.12
          * overcloud-compute-0: 192.168.25.20
          * overcloud-compute-1: 192.168.25.21

          The variables must be set as follows.
          (**The order of entries is critical!**)

          For Controllers::

            #                               controller-0  controller-1  controller-2
            ControllerDeployedServer_hosts="192.168.25.10 192.168.25.11 192.168.25.12"

          For Computes::

            #                            compute-0     compute-1
            ComputeDeployedServer_hosts="192.168.25.20 192.168.25.21"

The script will take care of querying Heat for each request metadata url,
configure the url in the agent configuration file on each deployed server, and
restart the agent service.

Once the script executes successfully, the deployed servers will start polling
Heat for software deployments and the stack will continue creating.


Scaling the Overcloud
---------------------

Scaling Up
^^^^^^^^^^
When scaling up the Overcloud, the heat agents on the new servers being added
to the deployment need to be configured to correspond to the new nodes being
added.

For example, when scaling out compute nodes, the steps to be completed by the
user are as follows:

#. Prepare the new deployed server(s) as shown in `Deployed Server
   Requirements`_.
#. Start the scale out command. See :doc:`../post_deployment/scale_roles` for reference.
#. Once Heat has created the new resources for the new deployed server(s),
   query Heat for the request metadata url for the new nodes, and configure the
   remote agents as shown in `Manual configuration of Heat agents`_. The manual
   configuration of the agents should be used when scaling up because the
   automated script method will reconfigure all nodes, not just the new nodes
   being added to the deployment.

Scaling Down
^^^^^^^^^^^^
When scaling down the Overcloud, follow the scale down instructions as normal
as shown in :doc:`../post_deployment/delete_nodes`, however use the following
command to get the uuid values to pass to `openstack overcloud node delete`
instead of using `nova list`::

    openstack stack resource list overcloud -n5 --filter type=OS::TripleO::<RoleName>Server

Replace `<RoleName>` in the above command with the actual name of the role that
you are scaling down. The `stack_name` column in the command output can be used
to identify the uuid associated with each node. The `stack_name` will include
the integer value of the index of the node in the Heat resource group. For
example, in the following sample output::

    $ openstack stack resource list overcloud -n5 --filter type=OS::TripleO::ComputeDeployedServerServer
    +-----------------------+--------------------------------------+------------------------------------------+-----------------+----------------------+-------------------------------------------------------------+
    | resource_name         | physical_resource_id                 | resource_type                            | resource_status | updated_time         | stack_name                                                  |
    +-----------------------+--------------------------------------+------------------------------------------+-----------------+----------------------+-------------------------------------------------------------+
    | ComputeDeployedServer | 66b1487c-51ee-4fd0-8d8d-26e9383207f5 | OS::TripleO::ComputeDeployedServerServer | CREATE_COMPLETE | 2017-10-31T23:45:18Z | overcloud-ComputeDeployedServer-myztzg7pn54d-0-pixawichjjl3 |
    | ComputeDeployedServer | 01cf59d7-c543-4f50-95df-6562fd2ed7fb | OS::TripleO::ComputeDeployedServerServer | CREATE_COMPLETE | 2017-10-31T23:45:18Z | overcloud-ComputeDeployedServer-myztzg7pn54d-1-ooCahg1vaequ |
    | ComputeDeployedServer | 278af32c-c3a4-427e-96d2-3cda7e706c50 | OS::TripleO::ComputeDeployedServerServer | CREATE_COMPLETE | 2017-10-31T23:45:18Z | overcloud-ComputeDeployedServer-myztzg7pn54d-2-xooM5jai2ees |
    +-----------------------+--------------------------------------+------------------------------------------+-----------------+----------------------+-------------------------------------------------------------+

The index 0, 1, or 2 can be seen in the `stack_name` column. These indices
correspond to the order of the nodes in the Heat resource group. Pass the
corresponding uuid value from the `physical_resource_id` column to `openstack
overcloud node delete` command.

The physical deployed servers that have been removed from the deployment need
to be powered off. In a deployment not using deployed servers, this would
typically be done with Ironic. When using deployed servers, it must be done
manually, or by whatever existing power management solution is already in
place. If the nodes are not powered down, they will continue to be operational
and could be part of the deployment, since there are no steps to unconfigure,
uninstall software, or stop services on nodes when scaling down.

Once the nodes are powered down and all needed data has been saved from the
nodes, it is recommended that they be reprovisioned back to a base operating
system configuration so that they do not unintentionally join the deployment in
the future if they are powered back on.

.. note::

  Do not attempt to reuse nodes that were previously removed from the
  deployment without first reprovisioning them using whatever provisioning tool
  is in place.

Deleting the Overcloud
----------------------

When deleting the Overcloud, the Overcloud nodes need to be manually powered
off, otherwise, the cloud will still be active and accepting any user requests.

After archiving important data (log files, saved configurations, database
files), that needs to be saved from the deployment, it is recommended to
reprovision the nodes to a clean base operating system. The reprovision will
ensure that they do not start serving user requests, or interfere with future
deployments in the case where they are powered back on in the future.

.. note::

  As with scaling down, do not attempt to reuse nodes that were previously part
  of a now deleted deployment in a new deployment without first reprovisioning
  them using whatever provisioning tool is in place.
