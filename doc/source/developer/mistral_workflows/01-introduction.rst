Introduction and prerequisites
------------------------------

.. include:: ../../links.rst

Let's assume you have a TripleO development environment healthy and working
properly. All the commands and customization we are going to run will run in
the Undercloud, as usual logged in as the stack user and having sourced the
stackrc file.

Then let's proceed by cloning the repositories we are going to work within a
temporary folder:

::


    mkdir dev-docs
    cd dev-docs
    git clone https://github.com/openstack/python-tripleoclient
    git clone https://github.com/openstack/tripleo-common
    git clone https://github.com/openstack/instack-undercloud

- **python-tripleoclient:** Will define the OpenStack CLI commands.
- **tripleo-common:** Will have the Mistral logic.
- **instack-undercloud:** Allows to update and create mistral
  environments to store configuration details
  needed when executing Mistral workflows.
