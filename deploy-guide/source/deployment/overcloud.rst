Containers based Overcloud Deployment
======================================

This documentation explains how to deploy a fully containerized overcloud
utilizing Podman which is the default since the Stein release.

The requirements for a containerized overcloud are the same as for any other
overcloud deployment. The real difference is in where the overcloud services
will be deployed (containers vs base OS).

Architecture
------------

The container-based overcloud architecture is not very different from the
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

Preparing overcloud images
..........................

As part of the undercloud install, an image registry is configured on port
`8787`.  This is used to increase reliability of overcloud image pulls, and
minimise overall network transfers. The undercloud registry will be populated
with images required by the overcloud deploy by generating the following
`containers-prepare-parameter.yaml` file and using that for the prepare call::

  openstack tripleo container image prepare default \
    --local-push-destination \
    --output-env-file containers-prepare-parameter.yaml

.. note:: The file `containers-prepare-parameter.yaml` may have already been
          created during :ref:`install_undercloud`. It is
          encouraged to share the same `containers-prepare-parameter.yaml` file
          for undercloud install and overcloud deploy.

See :ref:`prepare-environment-containers` for details on using
`containers-prepare-parameter.yaml` to control what can be done
with image preparation during overcloud deployment.

.. _overcloud-prepare-container-images:

Deploying the containerized Overcloud
-------------------------------------

A containerized overcloud deployment follows all the steps described in the
baremetal :ref:`deploy-the-overcloud` documentation with the exception that it
requires an extra environment file to be added to the ``openstack overcloud
deploy`` command::

  -e ~/containers-prepare-parameter.yaml

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

..  _TripleO Quickstart: https://docs.openstack.org/tripleo-quickstart/
