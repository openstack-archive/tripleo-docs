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

Deployed Server Requirements
----------------------------

Networking
^^^^^^^^^^

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
Thus, the polling process requires that the Undercloud services are bound to an
IP address that is on a L3 routed network that is accessible to the Overcloud
nodes. This is the IP address that is configured via ``local_ip`` in the
``undercloud.conf`` file used during Undercloud installation.

On each Overcloud node run the following commands that test connectivity to the
Undercloud's IP address where OpenStack services are bound.

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

Overcloud
_________

Other than not having to be connected to a dedicated L2 network for
provisioning by the Undercloud, networking requirements for deployed servers
that will be used as nodes in the Overcloud remains unchanged.

Depending on what networking configuration and if network isolation is
in use determines the requirements around Overcloud network requirements, the
same as in a typical TripleO deployment where already deployed servers are not
used.

Package repositories
^^^^^^^^^^^^^^^^^^^^

The servers will need to already have the appropriately enabled yum repositories
as packages will be installed on the servers during the Overcloud deployment.
The enabling of repositories on the Overcloud nodes is the same as it is for
other areas of TripleO, such as Undercloud installation. See
:doc:`../basic_deployment/repositories` for the detailed steps on how to
enable the standard repositories for TripleO.

Initial Package Installation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Once the repositories have been enabled on the deployed servers, the initial
packages for the Heat agent need to be installed. Run the following command on
each server intending to be used as part of the Overcloud::

    sudo yum -y install python-heat-agent*

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
      -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-environment.yaml \
      -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-bootstrap-centos.yaml \
      -r /usr/share/openstack-tripleo-heat-templates/deployed-server/deployed-server-roles-data.yaml

The ``--disable-validations`` option disables the basic Nova, Ironic, and
Glance related validations executed by python-tripleoclient. These validations
are not necessary since those services will not be used to deploy the
Overcloud.

The ``deployed-server.yaml`` environment takes advantage of the template
composition nature of Heat and tripleo-heat-templates to substitute
``OS::Heat::DeployedServer`` resources in place of ``OS::Nova::Server``.

The ``deployed-server-bootstrap-centos.yaml`` environment triggers execution of
a bootstrap script on the deployed servers to install further needed packages
and make other configurations necessary for Overcloud deployment.

The custom roles file, ``deployed-server-roles-data.yaml`` contains the custom
roles used during the deployment. Further customization of the roles data is
possible when using deployed servers. When doing so, be sure to include the
``disable_constraints`` key on each defined role as seen in
``deployed-server-roles-data.yaml``. This key disables the Heat defined
constraints in the generated role templates. These constraints validate
resources such as Nova flavors and Glance images, resources that are not needed
when using deployed servers. An example role using ``disable_constraints``
looks like::

    - name: ControllerDeployedServer
      disable_constraints: True
      CountDefault: 1
      ServicesDefault:
        - OS::TripleO::Services::CACerts
        - OS::TripleO::Services::CephMon
        - OS::TripleO::Services::CephExternal
        - OS::TripleO::Services::CephRgw
        ... <additional services>


Configuring Deployed Servers to poll Heat
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

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
___________________________________

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
______________________________________

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

Each ``$<role-name>_hosts`` variable is a space separated list of IP addresses
that are the IP addresses of the deployed servers for the roles. For example,
in the above command, 192.168.25.1 is the IP of Controller 0, 192.168.25.2 is
the IP of Controller 1, etc.

The script will take care of querying Heat for each request metadata url,
configure the url in the agent configuration file on each deployed server, and
restart the agent service.

Once the script executes successfully, the deployed servers will start polling
Heat for software deployments and the stack will continue creating.
