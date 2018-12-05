Information for New Developers
==============================

The intention of this document is to give new developers some information
regarding how to get started with TripleO as well as some best practices that
the TripleO community has settled on.

In general TripleO is a very complex chunk of software.  It uses numerous
technologies to implement an OpenStack installer.  The premise of TripleO was
to use the OpenStack platform itself as the installer and API for user
interfaces.  As such the first step to installing TripleO is to create what is
called an `undercloud`.  Today this is typically done using an undercloud image
which contains all the software necessary to stand up an all-in-one style
OpenStack installation.

Once the `undercloud` is deployed, we use a combination of Mistral and a set of
Heat templates, found in the `tripleo-heat-templates` repository, to drive the
deployment of an overcloud.  Ironic is used to provision hardware and boot an
operating system either on baremetal (for real deployments) or on VMs (for
development).  As of the Pike release we now deploy almost all services in
containers on the overcloud.  By the end of Queens all services should be
containerized and the overcloud image should be reduced to a basic operating
system image.

We are also working on unifying the architecture of the overcloud and
undercloud by moving to a containerized undercloud.  This allows you to do
development on the undercloud using the same Heat templates as are used to
deploy the overcloud.  This greatly accelerates development iteration cycle
speed.

Repositories that are part of TripleO
-------------------------------------

* `tripleo-common <https://git.openstack.org/cgit/openstack/tripleo-common/>`_:
  This is intended to be for TripleO libraries of common code including Mistral
  workflows and actions.  Unfortunately it has become a bit overrun with
  unrelated bits.  Work is ongoing to clean this up and split this into
  separate repositories.

* `tripleo-heat-templates <https://git.openstack.org/cgit/openstack/tripleo-heat-templates>`_:
  This contains all the Heat templates necessary to deploy the overcloud (and
  hopefully soon the undercloud as well).

* `python-tripleoclient <https://git.openstack.org/cgit/openstack/python-tripleoclient>`_:
  The CLI for deploying TripleO.  This contains some logic but remember that we
  want to call Mistral actions from here where needed so that the logic can be
  shared with the UI.

* `tripleo-docs <https://git.openstack.org/cgit/openstack/tripleo-docs>`_:
  Where these docs are kept.

* `tripleo-image-elements <https://git.openstack.org/cgit/openstack/tripleo-image-elements>`_:
  Image elements (snippets of puppet that prepare specific parts of the
  image) for building the undercloud and overcloud disk images.

* `tripleo-puppet-elements <https://git.openstack.org/cgit/openstack/tripleo-puppet-elements>`_:
  Puppet elements used to configure and deploy the overcloud.  These
  used during installation to set up the services.

* `puppet-tripleo <https://git.openstack.org/cgit/openstack/puppet-tripleo>`_:
  Puppet is used to configure the services in TripleO.  This repository
  contains various puppet modules for doing this.

* `tripleo-quickstart <https://git.openstack.org/cgit/openstack/tripleo-quickstart>`_:
  Quickstart is an Ansible driven deployment for TripleO used in CI.  Most
  developers also use this to stand up instances for development as well.

* `tripleo-quickstart-extras <https://git.openstack.org/cgit/openstack/tripleo-quickstart-extras>`_:
  Extended functionality for tripleo-quickstart allowing for end-to-end
  deployment and testing.

* `tripleo-ui <https://git.openstack.org/cgit/openstack/tripleo-ui>`_:
  The web based graphical user interface for deploying TripleO.

* `paunch <https://git.openstack.org/cgit/openstack/paunch>`_:
  This is a library that is used to deploy containers.  It is called from the
  Heat templates during installation to deploy our containerized services.

* `kolla <https://git.openstack.org/cgit/openstack/kolla>`_:
  We use the containers built by the Kolla project for services in TripleO.
  Any new containers or additions to existing containers should be submitted
  here.

* `diskimage-builder <https://git.openstack.org/cgit/openstack/diskimage-builder>`_:
  Disk image builder is used to build our base images for the TripleO
  deployment.

Definition of Done
------------------

This is basically a check list of things that you want to think about when
implementing a new feature.  Especially important is considering how the user
interface functions along side the command line interface.

- Ensure that any new feature is provided through Rest API in form of Mistral
  actions, workflow or by directly accessing an OpenStack service API.
- Ensure that GUI and CLI can both operate this feature through this API.
- Start your feature work by creating Mistral action or Mistral workflow -
  defining inputs and outputs. This is necessary so that the CI and UI have
  feature parity and both use the same API calls to implement a given feature.
- Ensure that the continuous integration (CI) is in place and passing, adding
  coverage to tests if required.  See
  http://specs.openstack.org/openstack/tripleo-specs/specs/policy/adding-ci-jobs.html
  for more information.
- Ensure there are unit tests where possible.
- Maintain backwards compatibility with our existing template interfaces from
  tripleo-heat-templates.
- New features should be reviewed by cores who have knowledge in that area of
  the codebase.
- One should consider logging and support implications. If you have new logs,
  would they be available via sosreport.
- Error messages are easy to understand and work their way back to the user
  (stack traces are not sufficient).
- Documentation should be updated if necessary. New features need a
  tripleo-docs patch.
- If any new dependencies are used for your feature, be sure they are properly
  packaged and available in RDO. You can ask on #rdo (on freenode server) for
  help with this.
- If a Mistral workflow changes between releases, make version notes so that
  users know how they might have to update their workflow call. Make sure it's
  backwards-compatible, or have at least one cycle deprecation period before
  removing the old way of doing things.


Using the Containerized Undercloud for Development.
---------------------------------------------------

The containerized undercloud can be used for development purposes.
This reuses the existing TripleO Heat Templates, allowing you to do the
development using this framework instead of a complete overcloud.
This is very useful if you are developing Heat templates or containerized
services.

Please see the following guide on how to set up a containerized undercloud:
https://docs.openstack.org/tripleo-docs/latest/install/containers_deployment/undercloud.html
