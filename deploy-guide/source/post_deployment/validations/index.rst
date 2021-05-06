Validations guide
=================

Since the Newton release, TripleO ships with extensible checks for
verifying the Undercloud configuration, hardware setup, and the
Overcloud deployment to find common issues early.

Since Stein, it is possible to run the validations using the TripleO CLI.

Validations are used to efficiently and reliably verify various facts about
the cloud on the level of individual nodes and hosts.

Validations are non-intrusive by design, and recommended when performing large
scale changes to the cloud, for example upgrades, or to aid in the diagnosis
of various issues. Detailed docs for both the CLI and the API are provided
by the Validations Framework project.

* tripleo-validations: https://docs.openstack.org/tripleo-validations/latest/
* validations-common: https://docs.openstack.org/validations-common/latest/
* validations-libs: https://docs.openstack.org/validations-libs/latest/

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

.. note::
  In case of the most validations, a failure does not mean that
  you'll be unable to deploy or run OpenStack. But it can indicate
  potential issues with long-term or production setups. If you're
  running an environment for developing or testing TripleO, it's okay
  that some validations fail. In a production setup, they should not.

The list of all existing validations and the specific documentation
for the project can be found on the `tripleo-validations documentation page`_.

With implementation specifics described in docs for the `validations-libs`_,
and `validations-common`_.

The following sections describe the different ways of running and listing the
currently installed validations.

.. toctree::
  :maxdepth: 2
  :includehidden:

  cli
  ansible
  in-flight

.. _tripleo-validations documentation page: https://docs.openstack.org/tripleo-validations/latest/
.. _validations-libs: https://docs.openstack.org/validations-libs/latest/
.. _validations-common: https://docs.openstack.org/validations-common/latest/