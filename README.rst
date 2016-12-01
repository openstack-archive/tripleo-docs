========================
Team and repository tags
========================

.. image:: http://governance.openstack.org/badges/tripleo-docs.svg
    :target: http://governance.openstack.org/reference/tags/index.html

.. Change things from this point on

TripleO Documentation
=====================

This is the documentation source for the TripleO project. You can read
the generated documentation at `TripleO
Docs <http://docs.openstack.org/developer/tripleo-docs/>`__.

You can find out more about TripleO at the `TripleO
Wiki <https://wiki.openstack.org/wiki/TripleO>`__.

Getting Started
---------------

Documentation for the TripleO project is hosted on the OpenStack Gerrit
site. You can view all open and resolved issues in the
``openstack/tripleo-docs`` project at `TripleO
Reviews <https://review.openstack.org/#/q/project:openstack/tripleo-docs>`__.

General information about contributing to the OpenStack documentation
available at `OpenStack Documentation Contributor
Guide <http://docs.openstack.org/contributor-guide/>`__

Quick Start
-----------

The following is a quick set of instructions to get you up and running
by building the TripleO documentation locally. The first step is to get
your Python environment configured. Information on configuring is
available at `Python Project
Guide <http://docs.openstack.org/project-team-guide/project-setup/python.html>`__

Next you can generate the documentation using the following command. Be
sure to run all the commands from within the recently checked out
repository.

::

    tox -edocs

Now you have the documentation generated for the various available
formats from the local source. The resulting documentation will be
available within the ``doc/build/`` directory.
