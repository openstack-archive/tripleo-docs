Information for New Developers
==============================

The intention of this document is to give new developers some information
regarding how to get started with TripleO as well as some best practices that
the TripleO community has settled on.

In general TripleO is a very complex chunk of software. It uses numerous
technologies to implement an OpenStack installer. The premise of TripleO was
to use the OpenStack platform itself as the installer and API for user
interfaces.  As such the first step to installing TripleO is to create what is
called an `undercloud`. We use almost similar architecture for both
`undercloud` and `overcloud` that leverages same set of Heat templates found
in `tripleo-heat-templates` repository, with a few minor differences. The
`undercloud` services are deployed in containers and can be managed by the
same tool chain used for `overcloud`.

Once the `undercloud` is deployed, we use a combination of Ansible playbooks
and a set of Heat templates, to drive the deployment of an overcloud. Ironic
is used to provision hardware and boot an operating system either on baremetal
(for real deployments) or on VMs (for development).  All services are deployed
in containers on the overcloud like undercloud.

Repositories that are part of TripleO
-------------------------------------

* `tripleo-common <https://opendev.org/openstack/tripleo-common/>`_:
  This is intended to be for TripleO libraries of common code.
  Unfortunately it has become a bit overrun with unrelated bits. Work
  is ongoing to clean this up and split this into separate repositories.

* `tripleo-ansible <https://opendev.org/openstack/tripleo-ansible/>`_:
  Contains Ansible playbooks, roles, plugins, modules, filters for use with
  TripleO deployments.

* `tripleo-heat-templates <https://opendev.org/openstack/tripleo-heat-templates>`_:
  This contains all the Heat templates necessary to deploy the overcloud (and
  hopefully soon the undercloud as well).

* `python-tripleoclient <https://opendev.org/openstack/python-tripleoclient>`_:
  The CLI for deploying TripleO.  This contains some logic but remember that we
  want to call Mistral actions from here where needed so that the logic can be
  shared with the UI.

* `tripleo-docs <https://opendev.org/openstack/tripleo-docs>`_:
  Where these docs are kept.

* `tripleo-image-elements <https://opendev.org/openstack/tripleo-image-elements>`_:
  Image elements (snippets of puppet that prepare specific parts of the
  image) for building the undercloud and overcloud disk images.

* `tripleo-puppet-elements <https://opendev.org/openstack/tripleo-puppet-elements>`_:
  Puppet elements used to configure and deploy the overcloud.  These
  used during installation to set up the services.

* `puppet-tripleo <https://opendev.org/openstack/puppet-tripleo>`_:
  Puppet is used to configure the services in TripleO.  This repository
  contains various puppet modules for doing this.

* `tripleo-quickstart <https://opendev.org/openstack/tripleo-quickstart>`_:
  Quickstart is an Ansible driven deployment for TripleO used in CI.  Most
  developers also use this to stand up instances for development as well.

* `tripleo-quickstart-extras <https://opendev.org/openstack/tripleo-quickstart-extras>`_:
  Extended functionality for tripleo-quickstart allowing for end-to-end
  deployment and testing.

* `tripleo-ui <https://opendev.org/openstack/tripleo-ui>`_:
  The web based graphical user interface for deploying TripleO.

* `kolla <https://opendev.org/openstack/kolla>`_:
  We use the containers built by the Kolla project for services in TripleO.
  Any new containers or additions to existing containers should be submitted
  here.

* `diskimage-builder <https://opendev.org/openstack/diskimage-builder>`_:
  Disk image builder is used to build our base images for the TripleO
  deployment.

Definition of Done
------------------

This is basically a check list of things that you want to think about when
implementing a new feature.

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


Using TripleO Standalone for Development
----------------------------------------

The Standalone container based deployment can be used for development purposes.
This reuses the existing TripleO Heat Templates, allowing you to do the
development using this framework instead of a complete overcloud.
This is very useful if you are developing Heat templates or containerized
services.

Please see `Standalone Deployment Guide <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/standalone.html>`_
on how to set up a Standalone OpenStack node.

