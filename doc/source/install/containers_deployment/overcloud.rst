Containers based Overcloud Deployment
======================================

.. Warning::

   The TripleO containers support is still under heavy development. Things
   documented here may change during the Pike cycle.

This documentation explains how to deploy a fully containerized overcloud on
Docker. This feature is supported starting with Pike.

The requirements for a containerized overcloud are the same as for any other
overcloud deployment. The real difference is in where the overcloud services
will be deployed (containers vs base OS).

Architecture
------------

The docker-based overcloud architecture is not very different from the
baremetal/VM based one. The services deployed in the traditional baremetal
overcloud are also deployed in the docker-based one.

One obvious difference between these 2 types of deployments is that the
openstack services are deployed as containers in a container runtime rather than
in the host operating system. This reduces the required packages in the host to
the bare minimum for running the container runtime and managing the base network
layer.


Manual overcloud deployment
----------------------------

This section explains how to deploy a containerized overcloud manually. For an
automated overcloud deployment, please follow the steps in the
`Using TripleO Quickstart`_ section below.

Preparing the environment
~~~~~~~~~~~~~~~~~~~~~~~~~

To prepare your environment, you must follow all the steps described in the
:ref:`basic-deployment-cli` documentation. Stop right at the
:ref:`deploy-the-overcloud` section.

Populate local docker registry
..............................

It's useful to run a local docker registry on the undercloud to speed up the
overcloud deployment. A docker registry is normally already setup to listen on
port 8787 as part of the undercloud install.

To use the pre-built images coming from the `tripleoupstream` registry on the
dockerhub, use the following command::

    openstack overcloud container image upload --config-file /usr/share/openstack-tripleo-common/container-images/overcloud_containers.yaml

Or use `kolla-build` to build the images yourself::

    kolla-build --base centos --type binary --namespace tripleoupstream --registry 192.168.24.1:8787 --tag latest --template-override /usr/share/tripleo-common/container-images/tripleo_kolla_template_overrides.j2 --push

Finally, point the heat templates to your local registry, for example in
a `$HOME/docker_registry.yaml` file::

    parameter_defaults:
      DockerNamespace: 192.168.24.1:8787/tripleoupstream
      DockerNamespaceIsRegistry: true

Deploying the containerized Overcloud
-------------------------------------

A containerized overcloud deployment follows all the steps described in the
baremetal :ref:`deploy-the-overcloud` documentation with the exception that it
requires an extra environment file to be added to the `openstack overcloud
deploy` command::

  -e /usr/share/openstack-tripleo-heat-templates/environments/docker.yaml

If deploying with highly available controller nodes, include the
following extra environment file in addition to the above and in place
of the `environments/puppet-pacemaker.yaml` file::

  -e /usr/share/openstack-tripleo-heat-templates/environments/docker-ha.yaml

In case of a local docker registry, also add the path to the override file::

  -e $HOME/docker_registry.yaml


Using TripleO Quickstart
------------------------

.. note:: Please refer to the `TripleO Quickstart`_ docs for more info about
          quickstart, the minimum requirements, the setup process and the
          available plugins.


The command below will deploy a containerized overcloud on top of a baremetal undercloud::

    bash quickstart.sh --config=~/.quickstart/config/general_config/containers_minimal.yml $VIRTHOST

..  _TripleO Quickstart: https://docs.openstack.org/developer/tripleo-quickstart/
