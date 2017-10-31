Containers based Undercloud Deployment
======================================

This documentation explains how to deploy a fully containerized undercloud on
Docker. This feature is supported starting with Pike.

While this is not currently used to deploy the overcloud, it is a great
development tool as it uses the same templates and infrastructure as the
overcloud.  This lets you do faster iterations for development.

The requirements for a containerized undercloud are the same as for any other
undercloud deployment. The real difference is in where the undercloud services
will be deployed (containers vs base OS).

Architecture
------------

The docker based undercloud architecture is not very different from the
baremetal/VM based one. The services deployed in the traditional baremetal
undercloud are also deployed in the docker based one.

One obvious difference between these two types of deployments is that the
openstack services are deployed as containers in a container runtime rather than
directly on the host operating system. This reduces the required packages in
the host to the bare minimum for running the container runtime and managing the
base network layer.


Manual undercloud deployment
----------------------------

This section explains how to deploy a containerized undercloud manually. For an
automated undercloud deployment.  A Quickstart version also exists and is
documented here https://docs.openstack.org/developer/tripleo-quickstart/

Preparing the environment
~~~~~~~~~~~~~~~~~~~~~~~~~

Prepare a host (either baremetal or VM) following the normal undercloud
provisioning steps (see :doc:`../installation/installing`) and stop right before
the undercloud install command. This should leave you with an updated base
operating system with no openstack packages installed but required repositories
configured.

Make sure these packages are installed before proceeding with the undercloud
installation:

* python-tripleoclient >= Pike
* python-openstackclient >= Pike
* openstack-heat-agents >= Pike
* docker >= 1.12.5
* openvswitch (minimum version supported by neutron)

See also
`Docker installation documentation <https://docs.docker.com/engine/installation/>`_.

Start the required host services::

    $ sudo systemctl start openvswitch
    $ sudo systemctl start docker

Verify that your docker environment is up and that your user can use sudo::

    $ sudo docker info

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

Preparing container images
--------------------------

Images for undercloud services should be prepared with the
``openstack overcloud container image prepare`` command. The process is very
similar to the containerized overcloud case, see
:ref:`prepare-environment-containers`. The simplified command looks like::

    $ openstack overcloud container image prepare \
      --output-env-file $HOME/docker_registry.yaml

Deploying the undercloud
~~~~~~~~~~~~~~~~~~~~~~~~

The following command will install an undercloud with ironic, mistral and zaqar
(substitute $THT_ROOT with the right path, which is normally
`/usr/share/openstack-tripleo-heat-templates`, and $YOUR_SERVER_IP with your
server's private IP)::

    $ sudo openstack undercloud deploy \
      --templates=$THT_ROOT \
      --local-ip=$YOUR_SERVER_IP \
      --keep-running \
      -e $THT_ROOT/environments/services-docker/ironic.yaml \
      -e $THT_ROOT/environments/services-docker/mistral.yaml \
      -e $THT_ROOT/environments/services-docker/zaqar.yaml \
      -e $THT_ROOT/environments/docker.yaml \
      -e $THT_ROOT/environments/mongodb-nojournal.yaml \
      -e $HOME/custom.yaml \
      -e $HOME/docker_registry.yaml


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

To stop and remove all running containers::

    $ sudo docker ps -qa --filter label=managed_by=docker-cmd | xargs sudo docker rm -f

To remove the existing named volumes (bear in mind this will remove your
database files too)::

    $ sudo docker volume rm $(sudo docker volume ls -q)

Configuration files are generated and overwritten on every run. However, you can
also remove them by running::

    $ sudo rm -Rf /var/lib/docker-puppet
    $ sudo rm -Rf /var/lib/config-data
    $ sudo rm -Rf /var/lib/kolla


How does the undercloud deploy work?
------------------------------------

The `undercloud deploy` command as written in the `Deploying the undercloud`_
section will run all the OpenStack services in a container runtime (docker)
unless the default settings are overwritten.

This command requires 2 services to be running at all times. The first one is a
basic keystone service, which is currently mocked by `tripleoclient` itself, the
second one is `heat-all` which executes the templates and installs the services.
The latter can be run on baremetal or in a container (tripleoclient will run it
in a container by default).

Checkout the :doc:`architecture` for more detailed info on how
TripleO builds, creates and runs containers.
