Containers based Overcloud Deployment
======================================

This documentation explains how to deploy a fully containerized overcloud on
Docker. This feature is now the default in Queens.

The requirements for a containerized overcloud are the same as for any other
overcloud deployment. The real difference is in where the overcloud services
will be deployed (containers vs base OS).

Architecture
------------

The docker-based overcloud architecture is not very different from the
baremetal/VM based one. The services deployed in the traditional baremetal
overcloud are also deployed in the docker-based one.

One obvious difference between these two types of deployments is that the
Openstack services are deployed as containers in a container runtime rather
than directly on the host operating system. This reduces the required packages
in the host to the bare minimum for running the container runtime and managing
the base network layer.


Manual overcloud deployment
----------------------------

This section explains how to deploy a containerized overcloud manually. For an
automated overcloud deployment, please follow the steps in the
`Using TripleO Quickstart`_ section below.

.. _prepare-environment-containers:

Preparing the environment
~~~~~~~~~~~~~~~~~~~~~~~~~

To prepare your environment, you must follow all the steps described in the
:ref:`basic-deployment-cli` documentation. Stop right at the
:ref:`deploy-the-overcloud` section.

It is necessary to generate a heat environment file which specifies the
container image parameters. These parameters will deploy the overcloud with
images from a specific repository with specific tags.

The ``openstack overcloud container image prepare`` command is an easy
way to generate these parameters. The following command will generate
a heat environment file `~/docker_registry.yaml` to deploy an overcloud
with container images from RDO docker registry::

    openstack overcloud container image prepare \
      --namespace trunk.registry.rdoproject.org/master \
      --tag tripleo-ci-testing \
      --output-env-file ~/docker_registry.yaml

The options ``--namespace trunk.registry.rdoproject.org/master`` and ``--tag
tripleo-ci-testing`` will typically be replaced with values specific to the
environment. You may wish to use ``tripleo-passed-ci`` for a more stable set of
containers.  Run with ``--help`` to see the other options available for
controlling what is generated.

For production deployments (or for testing upgrades and rollbacks) stable tags
like `passed-ci` should never be used, instead explicit versioned tags are
required to specify the exact images which will be deployed.

Populate local docker registry
..............................

Serving container images from a local registry is optional, but it can make
overcloud deployment faster and more reliable. For development purposes an
insecure docker registry is already setup to listen on port 8787 as part of the
undercloud install.

To copy the images from one registry to another, the `prepare` command is run
to generate the `overcloud_containers.yaml` file. This describes the source and
destination image locations consumed by the `upload` command.

To copy the pre-built images coming from the `rdoproject` registry to
the local repository, the following commands are run.  The first sets
up the ``overcloud_containers.yaml`` configuration file containing the
pull and push diestinations::

    openstack overcloud container image prepare \
      --namespace trunk.registry.rdoproject.org/master \
      --tag tripleo-ci-testing \
      --push-destination 192.168.24.1:8787 \
      --output-images-file overcloud_containers.yaml

It is possible to limit the output to only the images that are going to be used
in the deployment by specifying the heat environment files with the
``--environment-file`` option and the roles file with the ``--roles-file``
option.

Then upload the images to the local registry using the generated file::

    openstack overcloud container image upload --config-file overcloud_containers.yaml

Or :ref:`build and push the images <build_container_images>` yourself.  This is
useful if you wish to customize the containers or modify an existing one.

The command ``openstack overcloud container image prepare`` then needs to be
called again to generate the `~/docker_registry.yaml` file that specifies the
containers available in the local registry::

    openstack overcloud container image prepare \
      --namespace 192.168.24.1:8787/master \
      --tag tripleo-ci-testing \
      --output-env-file ~/docker_registry.yaml


Deploying the containerized Overcloud
-------------------------------------

A containerized overcloud deployment follows all the steps described in the
baremetal :ref:`deploy-the-overcloud` documentation with the exception that it
requires extra environment files to be added to the ``openstack overcloud
deploy`` command::

  -e /usr/share/openstack-tripleo-heat-templates/environments/docker.yaml
  -e ~/docker_registry.yaml

If deploying with highly available controller nodes, include the
following extra environment file in addition to the above and in place
of the `environments/puppet-pacemaker.yaml` file::

  -e /usr/share/openstack-tripleo-heat-templates/environments/docker-ha.yaml

Using TripleO Quickstart
------------------------

.. note:: Please refer to the `TripleO Quickstart`_ docs for more info about
          quickstart, the minimum requirements, the setup process and the
          available plugins.


The command below will deploy a containerized overcloud on top of a baremetal undercloud::

    bash quickstart.sh --config=~/.quickstart/config/general_config/containers_minimal.yml $VIRTHOST

..  _TripleO Quickstart: https://docs.openstack.org/developer/tripleo-quickstart/
