Validations
===========

Since the Newton release, TripleO ships with extensible checks for
verifying the undercloud configuration, hardware setup, and the
deployment to find common issues early.

The TripleO GUI runs the validations automatically. While work is
underway for having them in the command line workflow, too, they need
to be run by calling a Mistral workflow currently.

The validations are assigned into various groups that indicate when in
the deployment workflow are they expected to run:

* **prep** validations check the hardware configuration of the
  undercloud node and should be run before ``openstack undercloud
  install``.

* **pre-introspection** should be run before we introspect nodes using
  Ironic Inspector.

* **pre-deployment** validations should be executed before ``openstack
  overcloud deploy``

* **post-deployment** should be run after the overcloud deployment has
  finished.

Note that for most of these validations, a failure does not mean that
you'll be unable to deploy or run OpenStack. But it can indicate
potential issues with long-term or production setups. If you're
running an environment for developing or testing TripleO, it's okay
that some validations fail. In a production setup they should not.

The list of all existing validations and their groups is on the
`tripleo-validations documentation page`_.



Running a single validation
---------------------------

If you want to run one validation in particular (because you're trying
things out or want to see whether you fixed the failure), you can run
it like so::

   $ source stackrc
   $ openstack workflow execution create tripleo.validations.v1.run_validation '{"validation_name": "undercloud-ram"}'

This will run the "undercloud-ram.yaml" validation. All validations
are stored in the
``/usr/share/openstack-tripleo-validations/validations/`` directory,
so to get the name of the validation (``undercloud-ram`` here) find
the right one there and use its filename without an extension.

The command will return an execution ID, which you can query for the
results. To find out whether the validation finished, run::

  $ openstack workflow execution show ID

Note the ``State`` value: it can be either ``RUNNING`` or ``SUCCESS``.
Note that success here only means that the validation finished, not
that it succeeded. To find that out, we need to read its output::

  $ mistral execution-get-output ID | jq .stdout -r

.. note:: There is an ``openstack workflow execution show output``
          command which should do the same thing, but it currently
          doesn't work in all environments supported by |project|.

This will return the hosts the validation ran against, any warnings
and error messages it encountered along the way as well as an overall
summary.


.. _running_validation_group:

Running a group of validations
------------------------------

The deployment documentation highlights places where you can run a
specific group of validations. Here's how to run the
``pre-deployment`` validations::

  $ source stackrc
  $ openstack workflow execution create tripleo.validations.v1.run_groups '{"group_names": ["pre-deployment"]}'

Unfortunately, getting the results of all these validations is more
complicated than in the case a single one. You need to see the tasks
the workflow above generated and see their results one by one: ::

  $ openstack task execution list

Look for tasks with the ``run_validation`` name. Then take the ID of
each of those and run::

  $ mistral task-get-result ID | jq .stdout -r

.. note:: There is a ``task execution show result`` command that
          should do the same thing, but it's not working on all
          platforms supported by |project|.

Since this can be tedious and hard to script, you can get the list of
validations belonging to a group instead and run them one-by-one using
the method above::

  $ openstack action execution run tripleo.validations.list_validations '{"groups": ["pre-deployment"]}' | jq ".result[] | .id"

.. _tripleo-validations documentation page: http://docs.openstack.org/developer/tripleo-validations/readme.html#existing-validations
