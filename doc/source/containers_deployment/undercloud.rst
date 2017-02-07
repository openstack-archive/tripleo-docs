Containers based Undercloud Deployment
======================================

This documentation explains how to deploy a fully containerized undercloud on
Docker. This feature is supported starting with Pike.

The requirements for a containerized undercloud are the same as for any other
undercloud deployment. The real difference is in where the undercloud services
will be deployed (containers vs base OS).

Architecture
------------

The docker based undercloud architecture is not very different from the
baremetal/VM based one. The services deployed in the traditional baremetal
undercloud are also deployed in the docker based one.

One obvious difference between these 2 types of deployments is that the
openstack services are deployed as containers in a container runtime rather than
in the host operating system. This reduces the required packages in the host to
the bare minimum for running the container runtime and managing the base network
layer.


Manual undercloud deployment
----------------------------

This section explains how to deploy a containerized undercloud manually. For an
automated undercloud deployment, please follow the steps in the
`Using TripleO Quickstart`_ section below.

Preparing the environment
~~~~~~~~~~~~~~~~~~~~~~~~~

Prepare a host (either baremetal or VM) following the normal undercloud
provisioning steps and stop right before the undercloud install command. This
should leave you with an updated base operating system with no openstack
packages installed.

Make sure these packages are installed before proceeding with the undercloud
installation:

* python-tripleoclient >= Pike
* python-openstackclient >= Pike
* python-heat-agent-hiera >= Pike
* python-heat-agent-apply-config >= Pike
* python-heat-agent-puppet >= Pike
* python-heat-agent-docker-cmd >= Pike
* docker >= 1.12.5
* openvswitch (minimum version supported by neutron)

Verify that your docker environment is up and that your user can use sudo::

    $ sudo docker ps -a


.. note:: Check the :ref:`debug-containers` section for more tips and tricks for
          debugging containers.

The above should show no containers running. If it shows running containers,
please refer to the `Cleaning up`_ section below.

Configuring the undercloud
~~~~~~~~~~~~~~~~~~~~~~~~~~

The containers based undercloud uses the same heat templates that are used for
an overcloud deployment. Therefore, controlling the undercloud configuration can
be done by modifying the existing environment files or creating new ones.

A basic `custom.yaml` file would look like this::

    parameter_defaults:
      UndercloudNameserver: 8.8.8.8
      NeutronServicePlugins: ""

The above configuration file overwrites the default nameserver and sets the
service plugins for neutron. If your undercloud node has a single nic and you
don't want to create a new one, it's possible to set the network configuration
to noop by changing the `custom.yaml` file to (substitute $THT_ROOT with the
right path)::

    resource_registry:
      OS::TripleO::Undercloud::Net::SoftwareConfig: $THT_ROOT/net-config-noop.yaml

    parameter_defaults:
      UndercloudNameserver: 8.8.8.8
      NeutronServicePlugins: ""


Deploying the undercloud
~~~~~~~~~~~~~~~~~~~~~~~~

The following command will install an undercloud with ironic, mistral and zaqar
(substitute $THT_ROOT with the right path)::

    $ sudo openstack undercloud deploy \
      --templates=$THT_ROOT \
      --local-ip=$YOUR_SERVER_IP \
      --keep-running \
      -e $THT_ROOT/environments/services/ironic.yaml \
      -e $THT_ROOT/environments/services/mistral.yaml \
      -e $THT_ROOT/environments/services/zaqar.yaml \
      -e $THT_ROOT/environments/docker.yaml \
      -e /home/stack/custom.yaml


The `keep-running` flag will keep the `openstack undercloud deploy` process
running on failures, which allows for debugging the current execution. A minimal
`stackrc` file will be required to query both, the keystone and the heat, APIs::

    export OS_NO_CACHE=True
    export OS_CLOUDNAME=overcloud
    export OS_AUTH_URL=http://127.0.0.1:35358
    export NOVA_VERSION=1.1
    export COMPUTE_API_VERSION=1.1
    export OS_USERNAME=foo
    export OS_PROJECT_NAME=foo
    export OS_PASSWORD=bar

Cleaning up
~~~~~~~~~~~

The following commands will help cleaning up your undercloud environment to
start the deployment from scratch:

To stop and remove all running containers (this will remove non-openstack
containers too)::

    $ sudo docker stop $(sudo docker ps -a -q)
    $ sudo docker rm $(sudo docker ps -a -q)

To remove the existing volumes (bear in mind this will remove your database
files too)::

    $ sudo docker volume rm $(sudo docker volume ls -q)

Configuration files are generated and overwritten on every run. However, you can
also remove them by running::

    $ sudo rm -Rf /var/lib/docker-puppet
    $ sudo rm -Rf /var/lib/config-data
    $ sudo rm -Rf /var/lib/kolla


Using TripleO Quickstart
------------------------

TBW


How does the undercloud deploy work?
------------------------------------

The `undercloud deploy` command as written in the `Deploying the undercloud`_
section will run all the OpenStack services in a container runtime (docker)
unless the default settings are overwritten.

This command requires 2 services to be running at all times. The first one is a
basic keystone service, which is currently mocked by `tripleclient` itself, the
second one is `heat-all` which executes the templates and installs the services.
The latter can be run on baremetal or in a container (tripleoclient will run it
in a container by default).

Checkout the `TripleO Containers Architecture`_ for more detailed info on how
TripleO builds, creates and runs containers.
