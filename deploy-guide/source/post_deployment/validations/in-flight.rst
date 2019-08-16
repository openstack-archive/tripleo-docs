In-flight validations
=====================

The What
--------
In-flight validations are launched during the deploy, usually at the beginning
of a step, in order to ensure a service deployed at previous step does actually
work.

The Why
-------
Being able to validate services early also ensures we get early failures. For
instance, if "service-one" is deployed at step 2 and never used until step 4,
we won't notice its failed state before step 4.

Adding a validation at the beginning of step 3 would prevent this issue, by
failing early with a human readable message.

The How
-------
The in-flight validations can be added directly in `the service template`_,
either at the end of the step we want to check, or at the beginning of the
next step.

Since steps are launched with ansible, order does matter.

Tagging
_______
In order to ensure we can actually deactivate validations, we have to tag
validation related tasks with, at least, two tags::

  - opendev-validation
  - opendev-validation-SERVICE_NAME

Plain ansible task
__________________
The following example will ensure rabbitmq service is running after its
deployment:

.. code-block:: YAML

  deploy_steps_tasks:
    # rabbitmq container is supposed to be started during step 1
    # so we want to ensure it's running during step 2
    - name: validate rabbitmq state
      when: step|int == 2
      tags:
        - opendev-validation
        - opendev-validation-rabbitmq
      wait_for_connection:
        host: {get_param: [ServiceNetMap, RabbitmqNetwork]}
        port: 5672
        delay: 10

Validation Framework import
___________________________
We can also include already existing validations from the
`Validation Framework`_ roles. This can be archived like this:

.. code-block:: YAML

  deploy_steps_tasks:
    - name: some validation
      when: step|int == 2
      tags:
        - opendev-validation
        - opendev-validation-rabbitmq
      include_role:
        role: rabbitmq-limits
      # We can pass vars to included role, in this example
      # we override the default min_fd_limit value:
      vars:
        min_fd_limit: 32768

You can find the definition of the ``rabbitmq-limits`` role `here`_.

Use existing health checks
__________________________
We can also go for a simple thing, and use the existing service health check:

.. code-block:: YAML

  deploy_steps_tasks:
    # rabbitmq container is supposed to be started during step 1
    # so we want to ensure it's running during step 2
    - name: validate rabbitmq state
      when: step|int == 2
      tags:
        - opendev-validation
        - opendev-validation-rabbitmq
      command: >
        podman exec rabbitmq /openstack/healthcheck

.. _the service template: https://github.com/openstack/tripleo-heat-templates/tree/master/deployment
.. _Validation Framework: https://docs.openstack.org/tripleo-validations/latest/readme.html
.. _here: https://github.com/openstack/tripleo-validations/tree/master/roles/rabbitmq-limits
