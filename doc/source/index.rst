Welcome to |project| documentation
====================================

TripleO is a project aimed at installing, upgrading and operating OpenStack
clouds using OpenStack's own cloud facilities as the foundation - building on
Nova, Ironic, Neutron and Heat to automate cloud management at datacenter
scale

Contributor Guide
-----------------

.. toctree::
  :maxdepth: 3
  :includehidden:

  contributor/index
  developer/index

Install Guide
-------------

.. toctree::
  :maxdepth: 3
  :includehidden:

  install/index

Upgrades/Updates/FFWD-Upgrade
-----------------------------

.. toctree::
  :maxdepth: 3
  :includehidden:

  upgrade/index

Documentation Conventions
=========================

Some steps in the following instructions only apply to certain environments,
such as deployments to real baremetal and deployments using Red Hat Enterprise
Linux (RHEL). These steps are marked as follows:

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

.. admonition:: Stable Branch
   :class: stable

   Step that should only be run when choosing to use components from their
   stable branches rather than using packages/source based on current master.

.. admonition:: Ocata
   :class: ocata

   Step that should only be run when installing from the Ocata stable branch.

.. admonition:: Pike
   :class: pike

   Step that should only be run when installing from the Pike stable branch.

.. admonition:: Queens
   :class: queens

   Step that should only be run when installing from the Queens stable branch.

.. admonition:: Rocky
   :class: rocky

   Step that should only be run when installing from the Rocky stable branch.

.. admonition:: Validations
   :class: validations

   Steps that will run the pre and post-deployment validations

.. admonition:: Optional Feature
   :class: optional

   Step that is optional.  A deployment can be done without these steps, but they
   may provide useful additional functionality.

Any such steps should *not* be run if the target environment does not match
the section marking.
