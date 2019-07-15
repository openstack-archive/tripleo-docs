CLI support for validations
===========================

The following section describes the options
when running or listing the existing validations.

.. note:: Validations can be performed either by calling Mistral or by calling
  ``ansible-playbook``. By default, the latter is used. However, listing
  validations are supported currently only by calling Mistral.

Running validations
^^^^^^^^^^^^^^^^^^^

Validations can be executed by groups or individually.  The current CLI
implementation allows to run them using the following CLI options:

.. code-block:: bash

  openstack tripleo validator run [options]

``--plan``: This option allows to execute the validations overriding the default
plan name.  The default value is set to ``overcloud``.  To override this options
use for example:

.. code-block:: bash

  openstack tripleo validator run --plan mycloud

``--validation-name``: This options allows to execute a set of specific
validations. Specify them as <validation_id>[,<validation_id>,...] which means a
comma separated list. The default value for this option is [].

For example you can run this as:

.. code-block:: bash

  openstack tripleo validator run --validation-name check-ftype,512e

``--group``: This option allows to run specific group validations, if more than
one group is required, then separate the group names with commas. The default
value for this option is ['pre-deployment'].

Run this option for example like:

.. code-block:: bash

  openstack tripleo validator run --group pre-upgrade,prep

``--extra-vars``: This option allows to add a dictionary of extra variables to a
run of a group or specific validations.

.. code-block:: bash

  openstack tripleo validator run \
                --extra-vars '{"min_undercloud_ram_gb": 24, "min_undercloud_cpu_count": 8}' \
                --validation-name undercloud-ram,undercloud-cpu

``--extra-vars-file``: This option allows to add a valid ``JSON`` or ``YAML``
file containg extra variables to a run of a group or specific validations.

.. code-block:: bash

  openstack tripleo validator run \
                --extra-vars-file /home/stack/envvars.json \
                --validation-name undercloud-ram,undercloud-cpu

``--use-mistral``: This options allows to execute either groups or a set of
specific validations by calling Mistral instead of using ``ansible-playbook``,
which is the default.

Listing validations
^^^^^^^^^^^^^^^^^^^

Validations can be listed by groups and depending which validations will be
listed, the output might be configured as a table, json or yaml.  The user can
list the validations using the following command:

.. code-block:: bash

  openstack tripleo validator list [options]

``--output``: This option allows to change the default output for listing the
validations.  The options are json, yaml or table.  Run this option for example
like:

.. code-block:: bash

  openstack tripleo validator list --output json

``--parameters``: This option allows to get only the available ``Ansible``
variables for the validations

.. code-block:: bash

  openstack tripleo validator list --parameters --validation-name undercloud-ram,undercloud-cpu
  Waiting for messages on queue 'tripleo' with no timeout.
  {
      "undercloud-cpu": {
          "parameters": {
              "min_undercloud_cpu_count": 8
          }
      },
      "undercloud-ram": {
          "parameters": {
              "min_undercloud_ram_gb": 24
          }
      }
  }

``--create-vars-file``: This option allows to generate a valid ``JSON`` or
``YAML`` file containing the available ``Ansible`` variables for the validations.

To generate a ``JSON`` or ``YAML`` file containing for the variables of the
``undercloud-ram`` and ``undercloud-cpu`` validations:

.. code-block:: bash

  openstack tripleo validator list --parameters \
                                   --create-vars-file [json|yaml] /home/stack/envvars \
                                   --validation-name undercloud-ram,undercloud-cpu

To generate a ``JSON`` or ``YAML`` file containing for the variables of the
validations belonging to the ``prep`` and ``pre-introspection`` groups:

.. code-block:: bash

  openstack tripleo validator list --parameters \
                                   --create-vars-file [json|yaml] /home/stack/envvars \
                                   --group prep,pre-introspection


``--group``: This option allows to filter and list specific group validations,
if more than one group is required to be listed, separate the group names with
commas. By default all group validations will be listed.

.. code-block:: bash

  openstack tripleo validator list --group pre-upgrade,prep
