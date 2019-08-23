|project| Introduction
========================

|project| is an OpenStack Deployment & Management tool.


**Architecture**

With |project|, you start by creating an **undercloud** (an actual operator
facing deployment cloud) that will contain the necessary OpenStack components to
deploy and manage an **overcloud** (an actual tenant facing workload cloud). The
overcloud is the deployed solution and can represent a cloud for any purpose
(e.g. production, staging, test, etc). The operator can choose any of available
Overcloud Roles (controller, compute, etc.) they want to deploy to the environment.

Go to :doc:`architecture` to learn more.

|

**Components**

|project| is composed of set of official OpenStack components accompanied by
few other open source plugins which increase |project|'s capabilities.

Go to :doc:`components` to learn more.


**Deployment Guide**

See additional information about how to deploy TripleO in the `Deploy Guide <tripleo_deploy_guide_>`_.

.. _tripleo_deploy_guide: https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/

.. toctree::
   :hidden:

   architecture
   components
