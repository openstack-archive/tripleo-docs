Welcome to |project| documentation
====================================

.. include:: index-introduction.rst

Contents:

.. toctree::
   :maxdepth: 2

   Introduction <introduction/introduction>
   Environment Setup <environments/environments>
   Undercloud Installation <installation/installation>
   Basic Deployment (CLI) <basic_deployment/basic_deployment_cli>
   Basic Deployment (GUI) <basic_deployment/basic_deployment_gui>
   Advanced Deployment <advanced_deployment/advanced_deployment>
   Post Deployment <post_deployment/post_deployment>
   Troubleshooting <troubleshooting/troubleshooting>
   How to Contribute <contributions/contributions>


Documentation Conventions
=========================

Some steps in the following instructions only apply to certain environments,
such as deployments to real baremetal and deployments using RHEL. These
steps are marked as follows:

.. admonition:: RHEL
   :class: rhel

   Step that should only be run when using RHEL

.. admonition:: RHEL Portal Registration
   :class: portal

   Step that should only be run when using RHEL Portal Registration

.. admonition:: RHEL Satellite Registration
   :class: satellite

   Step that should only be run when using RHEL Satellite Registration

.. admonition:: CentOS
   :class: centos

   Step that should only be run when using CentOS

.. admonition:: Baremetal
   :class: baremetal

   Step that should only be run when deploying to baremetal

.. admonition:: Virtual
   :class: virtual

   Step that should only be run when deploying to virtual machines

.. admonition:: Ceph
   :class: ceph

   Step that should only be run when deploying Ceph for use by the Overcloud

.. admonition:: Source
   :class: source

   Step that should only be run when choosing to use some components directly
   from their git source code repositories instead of packages.

Any such steps should *not* be run if the target environment does not match
the section marking.
