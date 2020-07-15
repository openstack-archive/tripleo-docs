Validations guide
=================

Since the Newton release, TripleO ships with extensible checks for
verifying the Undercloud configuration, hardware setup, and the
Overcloud deployment to find common issues early.

The TripleO UI runs the validations automatically. Since
Stein, it is possible to run the validations using the TripleO
CLI.

.. note:: The TripleO UI is marked for deprecation beginning with
  OpenStack Stein.

The validations are assigned into various groups that indicate when in
the deployment workflow they are expected to run:

* **no-op** validations will run a no-op operation to verify that
  the workflow is working as it supposed to, it will run in both
  the Undercloud and Overcloud nodes.

* **openshift-on-openstack** validations will check that the
  environment meets the requirements to be able to deploy OpenShift
  on OpenStack.

* **prep** validations check the hardware configuration of the
  Undercloud node and should be run before ``openstack undercloud
  install``. Running prep validations should not rely on Mistral
  because it might not be installed yet.

* **pre-introspection** should be run before we introspect nodes using
  Ironic Inspector.

* **pre-deployment** validations should be executed before ``openstack
  overcloud deploy``

* **post-deployment** should be run after the Overcloud deployment has
  finished.

* **pre-upgrade** try to validate your OpenStack deployment before you upgrade it.

* **post-upgrade** try to validate your OpenStack deployment after you upgrade it.

Note that for most of these validations, a failure does not mean that
you'll be unable to deploy or run OpenStack. But it can indicate
potential issues with long-term or production setups. If you're
running an environment for developing or testing TripleO, it's okay
that some validations fail. In a production setup, they should not.

The list of all existing validations and the specific documentation
for the project is described on the `tripleo-validations documentation page`_.

The following sections describe the different ways of running and listing the
currently installed validations.

.. toctree::
  :maxdepth: 2
  :includehidden:

  cli
  mistral
  ansible
  in-flight

.. _tripleo-validations documentation page: https://docs.openstack.org/tripleo-validations/latest/readme.html
