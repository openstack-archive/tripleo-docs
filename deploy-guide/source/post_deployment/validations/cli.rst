CLI support for validations
===========================

The following section describes the options when running or listing the existing
validations.

Running validations
^^^^^^^^^^^^^^^^^^^

Validations can be executed by groups or individually.  The current CLI
implementation allows to run them using the following CLI options:

.. code-block:: bash

  $ openstack tripleo validator run [options]

``--plan, --stack``: This option allows to execute the validations overriding the
default plan name.  The default value is set to ``overcloud``.  To override this
options use for example:

.. code-block:: bash

  $ openstack tripleo validator run --plan mycloud

``--validation``: This options allows to execute a set of specific
validations. Specify them as <validation_id>[,<validation_id>,...] which means a
comma separated list. The default value for this option is [].

For example you can run this as:

.. code-block:: bash

  $ openstack tripleo validator run --validation check-ftype,512e

.. _running_validation_group:

Running validation groups
-------------------------

``--group``: This option allows to run specific group validations, if more than
one group is required, then separate the group names with commas. The default
value for this option is ['pre-deployment'].

Run this option for example like:

.. code-block:: bash

  $ openstack tripleo validator run --group pre-upgrade,prep

``--extra-vars``: This option allows to add a dictionary of extra variables to a
run of a group or specific validations.

.. code-block:: bash

  $ openstack tripleo validator run \
                --extra-vars '{"min_undercloud_ram_gb": 24, "min_undercloud_cpu_count": 8}' \
                --validation undercloud-ram,undercloud-cpu

``--extra-vars-file``: This
option allows to add a valid ``JSON`` or ``YAML``
file containing extra variables to a run of a group or specific validations.

.. code-block:: bash

  $ openstack tripleo validator run \
                --extra-vars-file /home/stack/envvars.json \
                --validation undercloud-ram,undercloud-cpu

``--workers, -w``: This option will configure the maximum of threads that can be
used to execute the given validation.

.. code-block:: bash

  $ openstack tripleo validator run \
                --extra-vars-file /home/stack/envvars.json \
                --validation undercloud-ram,undercloud-cpu \
                --workers 3

Getting the list of the Groups of validations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To get the list of all groups used by tripleo-validations and get their
description, the user can type the following command:

.. code-block:: bash

   $ openstack tripleo validator group info

``--format, -f``: This option allows to change the default output for listing
the validations.  The options are csv, value, json, yaml or table.

.. code-block:: bash

  $ openstack tripleo validator group info --format json

Getting a list of validations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Validations can be listed by groups and depending which validations will be
listed, the output might be configured as a table, json or yaml. The user can
list the validations using the following command:

.. code-block:: bash

  $ openstack tripleo validator list [options]

``--group``: This option allows to list specific group validations, if more than
one group is required, then separate the group names with commas.

.. code-block:: bash

  $ openstack tripleo validator list --group prep,pre-introspection

``--format, -f``: This option allows to change the default output for listing
the validations.  The options are csv, value, json, yaml or table.

.. code-block:: bash

  $ openstack tripleo validator list --format json

Getting detailed information about a validation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To get a full description of a validation, the user can run the following
command:

.. code-block:: bash

  $ openstack tripleo validator show dns

Getting the parameters list for a validation or a group of validations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To get all the available ``Ansible`` variables for one or more validations:

``--validation``: This options allows to execute a set of specific
validations. Specify them as <validation_id>[,<validation_id>,...] which means a
comma separated list. The default value for this option is [].

.. code-block:: bash

  openstack tripleo validator show parameter --validation undercloud-ram,undercloud-cpu
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

``--group``: This option allows to list specific group validations, if more than
one group is required, then separate the group names with commas.

.. code-block:: bash

  openstack tripleo validator show parameter --group prep
  {
      "512e": {
          "parameters": {}
      },
      "service-status": {
          "parameters": {}
      },
      "tls-everywhere-prep": {
          "parameters": {}
      },
      "undercloud-cpu": {
          "parameters": {
              "min_undercloud_cpu_count": 8
          }
      },
      "undercloud-disk-space": {
          "parameters": {
              "volumes": [
                  {
                      "min_size": 10,
                      "mount": "/var/lib/docker"
                  },
                  {
                      "min_size": 3,
                      "mount": "/var/lib/config-data"
                  },
                  {
                      "min_size": 3,
                      "mount": "/var/log"
                  },
                  {
                      "min_size": 5,
                      "mount": "/usr"
                  },
                  {
                      "min_size": 20,
                      "mount": "/var"
                  },
                  {
                      "min_size": 25,
                      "mount": "/"
                  }
              ]
          }
      },
      "undercloud-ram": {
          "parameters": {
              "min_undercloud_ram_gb": 24
          }
      },
      "undercloud-selinux-mode": {
          "parameters": {}
      }
  }

``--download``: This option allows to generate a valid ``JSON`` or
``YAML`` file containing the available ``Ansible`` variables for the validations.

To generate a ``JSON`` or ``YAML`` file containing for the variables of the
``undercloud-ram`` and ``undercloud-cpu`` validations:

.. code-block:: bash

  openstack tripleo validator show parameter \
                                   --download [json|yaml] /home/stack/envvars \
                                   --validation undercloud-ram,undercloud-cpu

To generate a ``JSON`` or ``YAML`` file containing for the variables of the
validations belonging to the ``prep`` and ``pre-introspection`` groups:

.. code-block:: bash

  openstack tripleo validator show parameter \
                                   --download [json|yaml] /home/stack/envvars \
                                   --group prep,pre-introspection

``--format, -f``: This option allows to change the default output for listing
the validations parameters.  The options are json or yaml.
