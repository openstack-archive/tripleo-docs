Containers based Undercloud Deployment
======================================

The requirements for a containerized undercloud are the same as for any other
undercloud deployment. The real difference is in where the undercloud services
will be deployed (containers vs base OS).

The undercloud architecture based on Moby_ (also Podman_ as of Stein) containers
is not very different from the baremetal/VM based one. The services deployed in
the traditional baremetal undercloud are also deployed in the containers based
one.

.. _Moby: https://mobyproject.org/
.. _Podman: https://podman.io/

One obvious difference between these two types of deployments is that the
openstack services are deployed as containers in a container runtime rather than
directly on the host operating system. This reduces the required packages in
the host to the bare minimum for running the container runtime and managing the
base network layer.

.. note:: Check the :doc:`install_undercloud` and :doc:`../post_deployment/upgrade/undercloud`
          sections for deploying and upgrading a containerized undercloud.

.. note:: Check the :ref:`debug-containers` section for more tips and tricks for
          debugging containers.

.. note:: Check our "Deep Dive" video_ which explain the architecture backgrounds and changes
          as well as some demos and Q/A.

.. _video: https://www.youtube.com/watch?v=lv233gPynwk
