Composable services tutorial
============================

.. include:: ../../links.rst

This guide will be a walkthrough related to how to add new services to a TripleO
deployment through additions to the tripleo-heat-templates and puppet-tripleo
repositories, using part of the architecture defined in the `composable services architecture`_.

.. note::

  No puppet manifests may be defined in the `THT repository`_, they
  should go to the `puppet-tripleo repository`_ instead.

.. toctree::
   :maxdepth: 2

   introduction
   changes-tht
   changes-puppet-tripleo
   design-patterns
   tls_for_services
   summary
