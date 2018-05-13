|project| Components
======================

.. contents::
   :depth: 2
   :backlinks: none

This section contains a list of components that |project| uses. The components
are organized in categories, and include a basic description, useful links, and
contribution information.

..
    [Example Category Name]
    -----------------------

    [Example Component Name]
    ^^^^^^^^^^^^^^^^^^^^^^^^
    This is short description what the project is about and how |project| uses
    this project. Three sentences max.

    **How to contribute**

    * Instructions to prepare development environment. Should be mostly pointing to
      upstream docs. If upstream docs doesn't exist, please, create one. Add tips
      how to test the feature in |project| + other useful information.


    **Useful links**

    * Upstream Project:  `link <#>`_
    * Bugs: `link <#>`_
    * Blueprints:  `link <#>`_


Shared Libraries
----------------
diskimage-builder
^^^^^^^^^^^^^^^^^

diskimage-builder is an image building tool. It is used by
``openstack overcloud image build``.

**How to contribute**

See the diskimage-builder `README.rst
<https://git.openstack.org/cgit/openstack/diskimage-builder/tree/README.rst>`_
for a further explanation of the tooling. Submit your changes via
OpenStack Gerrit (see `OpenStack Developer's Guide
<http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Upstream Project Documentation: http://docs.openstack.org/developer/diskimage-builder/
* Bugs: https://bugs.launchpad.net/diskimage-builder
* Git repository: https://git.openstack.org/cgit/openstack/diskimage-builder/


dib-utils
^^^^^^^^^

dib-utils contains tools that are used by diskimage-builder.

**How to contribute**

Submit your changes via OpenStack Gerrit (see `OpenStack Developer's Guide
<http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Bugs: https://bugs.launchpad.net/diskimage-builder
* Git repository: https://git.openstack.org/cgit/openstack/dib-utils/


os-\*-config
^^^^^^^^^^^^

The os-\*-config projects are a suite of tools used to configure instances
deployed via TripleO. They include:

* os-collect-config
* os-refresh-config
* os-apply-config
* os-net-config

**How to contribute**

Each tool uses `tox <https://tox.readthedocs.org/en/latest/>`_ to manage the
development environment. Submit your changes via OpenStack Gerrit (see
`OpenStack Developer's Guide
<http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Bugs:

  * os-collect-config: https://bugs.launchpad.net/os-collect-config
  * os-refresh-config: https://bugs.launchpad.net/os-refresh-config
  * os-apply-config: https://bugs.launchpad.net/os-apply-config
  * os-net-config: https://bugs.launchpad.net/os-net-config

* Git repositories:

  * os-collect-config: https://git.openstack.org/cgit/openstack/os-collect-config
  * os-refresh-config https://git.openstack.org/cgit/openstack/os-refresh-config
  * os-apply-config https://git.openstack.org/cgit/openstack/os-apply-config
  * os-net-config https://git.openstack.org/cgit/openstack/os-net-config

tripleo-image-elements
^^^^^^^^^^^^^^^^^^^^^^

tripleo-image-elements is a repository of diskimage-builder style elements used
for installing various software components.

**How to contribute**

Submit your changes via OpenStack Gerrit (see
`OpenStack Developer's Guide
<http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Git repository: https://git.openstack.org/cgit/openstack/tripleo-image-elements


Installer
---------

instack
^^^^^^^
instack executes diskimage-builder style elements on the current system. This
enables a current running system to have an element applied in the same way
that diskimage-builder applies the element to an image build.

instack, in its current form, should be considered low level tooling. It is
meant to be used by higher level scripting that understands what elements and
hook scripts need execution. Using instack requires a rather in depth knowledge
of the elements within diskimage-builder and tripleo-image-elements.

**How to contribute**

Submit your changes via OpenStack Gerrit (see
`OpenStack Developer's Guide
<http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Git repository: https://git.openstack.org/cgit/openstack/instack
* Bugs: https://launchpad.net/tripleo

instack-undercloud
^^^^^^^^^^^^^^^^^^
instack-undercloud is a TripleO style undercloud installer based around
instack.

**How to contribute**

Submit your changes via OpenStack Gerrit (see
`OpenStack Developer's Guide
<http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Git repository: https://git.openstack.org/cgit/openstack/instack-undercloud
* Bugs: https://launchpad.net/tripleo

tripleo-incubator
^^^^^^^^^^^^^^^^^
tripleo-incubator contains various scripts to aid in deploying a TripleO cloud.

**How to contribute**

Submit your changes via OpenStack Gerrit (see
`OpenStack Developer's Guide
<http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Documentation: http://docs.openstack.org/developer/tripleo-incubator/index.html
* Git repository: https://git.openstack.org/cgit/openstack/tripleo-incubator
* Bugs: https://launchpad.net/tripleo


Node Management
---------------
ironic
^^^^^^

Ironic project is responsible for provisioning and managing bare metal
instances.

For testing purposes Ironic can also be used for provisioning and managing
virtual machines which act as bare metal nodes via special driver ``pxe_ssh``.

**How to contribute**

Ironic uses `tox <https://tox.readthedocs.org/en/latest/>`_ to manage the
development environment, see the `Developer Quick-Start
<http://docs.openstack.org/developer/ironic/dev/dev-quickstart.html>`_,
`Ironic Developer Guidelines
<https://wiki.openstack.org/wiki/Ironic/Developer_guidelines>`_
and `OpenStack Developer's Guide`_ for details.

**Useful links**

* Upstream Project: http://docs.openstack.org/developer/ironic/index.html
* Bugs: https://bugs.launchpad.net/ironic
* Blueprints: https://blueprints.launchpad.net/ironic

  * `Specs process <https://wiki.openstack.org/wiki/Ironic/Specs_Process>`_
    should be followed for suggesting new features.
  * Approved Specs: http://specs.openstack.org/openstack/ironic-specs/


ironic inspector (former ironic-discoverd)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Ironic Inspector project is responsible for inspection of hardware properties
for newly enrolled nodes (see also ironic_).

**How to contribute**

Ironic Inspector uses `tox <https://tox.readthedocs.org/en/latest/>`_ to manage
the development environment, see `upstream documentation
<https://github.com/openstack/ironic-inspector/blob/master/CONTRIBUTING.rst>`_
for details.

**Useful links**

* Upstream Project: https://github.com/openstack/ironic-inspector
* PyPI: https://pypi.org/project/ironic-inspector
* Bugs: https://bugs.launchpad.net/ironic-inspector
* Blueprints: https://blueprints.launchpad.net/ironic-inspector

VirtualBMC
^^^^^^^^^^

A helper command to translate IPMI calls into libvirt calls. Used for testing
bare metal provisioning on virtual environments.

**How to contribute**

VirtualBMC uses `tox <https://tox.readthedocs.org/en/latest/>`_ to manage the
development environment in a similar way to Ironic.

**Useful links**

* Source: https://git.openstack.org/cgit/openstack/virtualbmc
* Bugs: https://bugs.launchpad.net/virtualbmc


Deployment & Orchestration
--------------------------
heat
^^^^

Heat is OpenStack's orchestration tool. It reads YAML files describing
the OpenStack deployment's resources (machines, their configurations
etc.) and gets those resources into the desired state, often by
talking to other components (e.g. Nova).

**How to contribute**

* Use `devstack with Heat
  <http://docs.openstack.org/developer/heat/getting_started/on_devstack.html>`_
  to set up a development environment. Submit your changes via
  OpenStack Gerrit (see `OpenStack Developer's Guide
  <http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Upstream Project: https://wiki.openstack.org/wiki/Heat
* Bugs: https://bugs.launchpad.net/heat
* Blueprints: https://blueprints.launchpad.net/heat

heat-templates
^^^^^^^^^^^^^^

The heat-templates repository contains additional image elements for
producing disk images ready to be configured by Puppet via Heat.

**How to contribute**

* Use `devtest with Puppet
  <http://docs.openstack.org/developer/tripleo-incubator/puppet.html>`_
  to set up a development environment. Submit your changes via
  OpenStack Gerrit (see `OpenStack Developer's Guide
  <http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Upstream Project: https://git.openstack.org/cgit/openstack/heat-templates
* Bugs: https://bugs.launchpad.net/heat-templates
* Blueprints: https://blueprints.launchpad.net/heat-templates

tripleo-heat-templates
^^^^^^^^^^^^^^^^^^^^^^

The tripleo-heat-templates describe the OpenStack deployment in Heat
Orchestration Template YAML files and Puppet manifests. The templates
are deployed via Heat.

**How to contribute**

* Use `devtest with Puppet
  <http://docs.openstack.org/developer/tripleo-incubator/puppet.html>`_
  to set up a development environment. Submit your changes via
  OpenStack Gerrit (see `OpenStack Developer's Guide
  <http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Upstream Project: https://git.openstack.org/cgit/openstack/tripleo-heat-templates
* Bugs: https://bugs.launchpad.net/tripleo
* Blueprints: https://blueprints.launchpad.net/tripleo

nova
^^^^

nova provides a cloud computing fabric controller.

**How to contribute**

* Read the
  `Development Quickstart <http://docs.openstack.org/developer/nova/development.environment.html>`_
  to set up a development environment. Submit your changes via OpenStack
  Gerrit (see 
  `OpenStack Developer's Guide <http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Git repository: https://git.openstack.org/cgit/openstack/nova
* Bugs: https://bugs.launchpad.net/nova
* Blueprints: https://blueprints.launchpad.net/nova

puppet-\*
^^^^^^^^^

The OpenStack Puppet modules are used to configure the OpenStack
deployment (write configuration, start services etc.). They are used
via the tripleo-heat-templates.

**How to contribute**

* Use `devtest with Puppet
  <http://docs.openstack.org/developer/tripleo-incubator/puppet.html>`_
  to set up a development environment. Submit your changes via
  OpenStack Gerrit (see `OpenStack Developer's Guide
  <http://docs.openstack.org/infra/manual/developers.html>`_).

**Useful links**

* Upstream Project: https://wiki.openstack.org/wiki/Puppet


tripleo-puppet-elements
^^^^^^^^^^^^^^^^^^^^^^^

The tripleo-puppet-elements describe the contents of disk images which
|project| uses to deploy OpenStack. It's the same kind of elements
as in tripleo-image-elements, but tripleo-puppet-elements are specific
for Puppet-enabled images.

**How to contribute**

* Use `devtest with Puppet
  <http://docs.openstack.org/developer/tripleo-incubator/puppet.html>`_
  to set up a development environment. Submit your changes via
  OpenStack Gerrit (see `OpenStack Developer's Guide`_).

**Useful links**

* Upstream Project: https://git.openstack.org/cgit/openstack/tripleo-puppet-elements
* Bugs: https://bugs.launchpad.net/tripleo
* Blueprints: https://blueprints.launchpad.net/tripleo


User Interfaces
---------------

python-openstackclient
^^^^^^^^^^^^^^^^^^^^^^
The python-openstackclient is an upstream CLI tool which can manage multiple
openstack services. It wraps openstack clients like glance, nova, etc. and maps
them under intuitive names like openstack image, compute, etc.

The main value is that all services can be controlled by a single (openstack)
command with consistent syntax and behaviour.

**How to contribute**

* python-openstackclient uses `tox <https://tox.readthedocs.org/en/latest/>`_
  to manage the development environment, see the
  `python-openstackclient documentation
  <https://github.com/openstack/python-openstackclient/blob/master/README.rst>`_
  for details. Submit your changes via OpenStack Gerrit
  (see `OpenStack Developer's Guide`_).

**Useful links**

* Upstream Project: http://git.openstack.org/cgit/openstack/python-openstackclient
* Bugs: https://bugs.launchpad.net/python-openstackclient
* Blueprints: https://blueprints.launchpad.net/python-openstackclient
* Human interface guide: http://docs.openstack.org/developer/python-openstackclient/humaninterfaceguide.html

python-tripleoclient
^^^^^^^^^^^^^^^^^^^^
The python-tripleoclient is a CLI tool embedded into python-openstackclient. It
provides functions related to instack installation and initial configuration
like node introspection, overcloud image building and uploading, etc.

**How to contribute**

* python-tripleoclient uses `tox <https://tox.readthedocs.org/en/latest/>`_
  to manage the development environment, see the
  `python-tripleoclient documentation
  <http://git.openstack.org/cgit/openstack/python-tripleoclient/tree/CONTRIBUTING.rst>`_
  for details. Submit your changes via
  `Gerrit <https://review.openstack.org/#/q/project:openstack/python-tripleoclient,n,z>`_.

**Useful links**

* Project: https://git.openstack.org/cgit/openstack/python-tripleoclient

..
    <GLOBAL_LINKS>

.. _OpenStack Developer's Guide: http://docs.openstack.org/infra/manual/developers.html

tripleo-ui
^^^^^^^^^^

TripleO UI is the web interface for TripleO.

**How to contribute**

* See the `documentation <http://git.openstack.org/cgit/openstack/tripleo-ui/tree/README.md>`_
  for details.


**Useful links**

* Bugs: https://bugs.launchpad.net/tripleo-ui
* Blueprints: https://blueprints.launchpad.net/tripleo-ui

tripleo-validations
-------------------

Pre and post-deployment validations for the deployment workflow.

**Useful links**

* Upstream Project: http://git.openstack.org/cgit/openstack/tripleo-validations/
* Bugs: https://bugs.launchpad.net/tripleo/+bugs?field.tag=validations
* Documentation for individual validations: http://docs.openstack.org/developer/tripleo-validations/readme.html#existing-validations

.. note:: When reporting an issue, make sure you add the
          ``validations`` tag.

Deprecated
----------
Tuskar
^^^^^^

The Tuskar project was responsible for planning the deployments and
generating the corresponding Heat templates. This is no longer
necessary as Heat supports this composability out of the box.

The source code is available below, but please note that it should not
be used for new deployments.

https://github.com/openstack/tuskar
