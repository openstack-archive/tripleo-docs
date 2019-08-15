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

|project| Architecture
----------------------

.. toctree::
  :maxdepth: 3
  :includehidden:

  install/introduction/architecture.rst

|project| Components
----------------------

.. toctree::
  :maxdepth: 2
  :includehidden:

  install/introduction/components.rst

Tripleo CI Guide
----------------

.. toctree::
  :maxdepth: 3
  :includehidden:

  ci/index

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

.. admonition:: |oldest_version_name|
   :class: |oldest_version_name_lower|

   Step that should only be run when installing from the |oldest_version_name|
   stable branch.

.. admonition:: |before_oldest_version_name|

   Step that should only be run when installing from the
   |before_oldest_version_name| stable branch.

.. admonition:: |before_latest_version_name|
   :class: |before_latest_version_name_lower|

   Step that should only be run when installing from the
   |before_latest_version_name| stable branch.

.. admonition:: |latest_version_name|
   :class: |latest_version_name_lower|

   Step that should only be run when installing from the |latest_version_name|
   stable branch.

.. admonition:: Validations
   :class: validations

   Steps that will run the pre and post-deployment validations

.. admonition:: Optional Feature
   :class: optional

   Step that is optional.  A deployment can be done without these steps, but they
   may provide useful additional functionality.

Any such steps should *not* be run if the target environment does not match
the section marking.
