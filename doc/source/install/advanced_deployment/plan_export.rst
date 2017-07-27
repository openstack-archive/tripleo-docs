Exporting Deployment Plans
==========================

Exporting a deployment plan enables you to quickly retrieve the contents of an
existing deployment plan. A deployment plan consists of the heat templates and
the environment files used to deploy an overcloud, as well as the
`plan-environment.yaml`_ file which holds the plan metadata. Exporting a plan
can be useful if you want to use an existing plan as a starting point for
further customizations (instead of starting from scratch with a fresh copy of
`tripleo-heat-templates`_).

Exporting a plan using the CLI
------------------------------

To export a plan using the CLI, use the following command::

    $ openstack overcloud plan export <plan_name>

E.g::

    $ openstack overcloud plan export overcloud

will export the default plan called ``overcloud``. By default, a tarball named
``overcloud.tar.gz`` containing the plan files will be created in the current
directory. If you would like to use a custom file name, you can specify it
using the ``--output-file`` option.

Exporting a plan using the UI
-----------------------------

To export a plan using the UI, navigate to the ``Plans`` page using the
``All Plans`` link from the ``Plans`` tab. Then open the kebab menu of the
plan you want to export and click the ``Export`` link. This will trigger the
plan export workflow and after the plan export completes you will be presented
with a link to download the plan tarball.



.. _`plan-environment.yaml`: https://github.com/openstack/tripleo-heat-templates/blob/master/plan-environment.yaml
.. _`tripleo-heat-templates`: https://github.com/openstack/tripleo-heat-templates
