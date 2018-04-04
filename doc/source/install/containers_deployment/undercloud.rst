Containers based Undercloud Deployment
======================================

The requirements for a containerized undercloud are the same as for any other
undercloud deployment. The real difference is in where the undercloud services
will be deployed (containers vs base OS).

The docker based undercloud architecture is not very different from the
baremetal/VM based one. The services deployed in the traditional baremetal
undercloud are also deployed in the docker based one.

One obvious difference between these two types of deployments is that the
openstack services are deployed as containers in a container runtime rather than
directly on the host operating system. This reduces the required packages in
the host to the bare minimum for running the container runtime and managing the
base network layer.

.. note:: Check the :doc:`../installation/installing` and :doc:`../installation/updating`
          sections for deploying and upgrading a containerized undercloud.

.. note:: Check the :ref:`debug-containers` section for more tips and tricks for
          debugging containers.
