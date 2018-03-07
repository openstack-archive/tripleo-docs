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

A tag needs to be specified which is unique to the images being deployed.  This
makes it possible to later update the overcloud to newer image versions. It
also makes it easier to determine what images you are running in the overcloud.
This unique tag can be discovered by running the command ``openstack overcloud
container image tag discover`` with a known stable tag such as ``latest``. The
following command will return the tag from the RDO docker registry using the
stable tag ``current-tripleo-rdo``::

    openstack overcloud container image tag discover \
      --image docker.io/tripleomaster/centos-binary-base:current-tripleo-rdo \
      --tag-from-label rdo_version

.. note:: The tag is actually a Delorean hash. You can find out the versions
          of packages by using this tag.
          For example, `ac82ea9271a4ae3860528eaf8a813da7209e62a6_28eeb6c7` tag,
          is in fact using this `Delorean repository`_.

The option ``--image
docker.io/tripleomaster/centos-binary-base:current-tripleo-rdo``
will typically be replaced with a value specific to the environment. You may
wish to use stable tag ``tripleo-passed-ci`` for a more stable set of
containers.

It is necessary to generate a heat environment file which specifies the
container image parameters. These parameters will deploy the overcloud with
images from a specific repository with the discovered ``<tag>``.

The ``openstack overcloud container image prepare`` command is an easy
way to generate these parameters. The following command will generate
a heat environment file `~/docker_registry.yaml` to deploy an overcloud
with container images from RDO docker registry::

    openstack overcloud container image prepare \
      --namespace docker.io/tripleomaster \
      --tag <tag> \
      --output-env-file ~/docker_registry.yaml

The option ``--namespace docker.io/tripleomaster``
will typically be replaced with a value specific to the
environment. Run with ``--help`` to see the other options available for
controlling what is generated.

It is possible to limit the output to only the images that are going to be used
in the deployment by specifying the heat environment files with the
``--environment-file`` option and the roles file with the ``--roles-file``
option.

Populate local docker registry
..............................

Serving container images from a local registry is optional, but it can make
overcloud deployment faster and more reliable. For development purposes an
insecure docker registry is already setup to listen on port 8787 as part of the
undercloud install.

To copy the images from one registry to another, the above `prepare` command is
modified to also generate the `overcloud_containers.yaml` file. This describes
the source and destination image locations consumed by the `upload` command.

To copy the pre-built images coming from the `rdoproject` registry to
the local repository, the following commands are run.  The first sets
up the ``overcloud_containers.yaml`` configuration file containing the
pull and push destinations::

    openstack overcloud container image prepare \
      --namespace docker.io/tripleomaster \
      --tag <tag> \
      --push-destination 192.168.24.1:8787 \
      --output-env-file ~/docker_registry.yaml \
      --output-images-file overcloud_containers.yaml

.. admonition:: Stable Branch
  :class: stable

  If you wish to deploy a stable version, you will need to pull down the
  correct containers for the version being deployed.

  .. admonition:: Pike
     :class: pike

     Use the Pike containers::

         openstack overcloud container image prepare \
          --namespace docker.io/tripleopike \
          --tag <tag> \
          --push-destination 192.168.24.1:8787 \
          --output-env-file ~/docker_registry.yaml \
          --output-images-file overcloud_containers.yaml

  .. admonition:: Queens
     :class: queens

     Use the Queens containers::

         openstack overcloud container image prepare \
          --namespace docker.io/tripleoqueens \
          --tag <tag> \
          --push-destination 192.168.24.1:8787 \
          --output-env-file ~/docker_registry.yaml \
          --output-images-file overcloud_containers.yaml

Then upload the images to the local registry using the generated file::

    openstack overcloud container image upload --config-file overcloud_containers.yaml

.. note::
   If this command fails with the following error::

      Error while fetching server API version: ('Connection aborted.', error(13, 'Permission denied'))

   You may need to run ``newgrp docker``. This is because the undercloud install
   adds the current user to the docker group, but that change will not
   automatically take effect in the current session.

Or :ref:`build and push the images <build_container_images>` yourself.  This is
useful if you wish to customize the containers or modify an existing one.

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
..  _Delorean repository: https://trunk.rdoproject.org/centos7-master/ac/82/ac82ea9271a4ae3860528eaf8a813da7209e62a6_28eeb6c7/
