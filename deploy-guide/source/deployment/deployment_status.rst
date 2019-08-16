Deployment Status
^^^^^^^^^^^^^^^^^
Since Heat is no longer the source of authority on the status of the overcloud
deployment, a new tripleoclient command is available to show the overcloud
deployment status::

    openstack overcloud status

The output will report the status of the deployment, taking into consideration
the result of all the steps to do the full deployment. The following is an
example of the output::

    [stack@undercloud ]$ openstack overcloud status

    +-----------+---------------------+---------------------+-------------------+
    | Plan Name |       Created       |       Updated       | Deployment Status |
    +-----------+---------------------+---------------------+-------------------+
    | overcloud | 2018-05-03 21:24:50 | 2018-05-03 21:27:59 |   DEPLOY_SUCCESS  |
    +-----------+---------------------+---------------------+-------------------+

A different plan name can be specified with ``--plan``::

    [stack@undercloud ]$ openstack overcloud status --plan my-deployment

    +---------------+---------------------+---------------------+-------------------+
    | Plan Name     |       Created       |       Updated       | Deployment Status |
    +-----------+-------------------------+---------------------+-------------------+
    | my-deployment | 2018-05-03 21:24:50 | 2018-05-03 21:27:59 |   DEPLOY_SUCCESS  |
    +---------------+---------------------+---------------------+-------------------+

Deployment failures can also be shown with a new command::

    [stack@undercloud ]$ openstack overcloud failures --plan my-deployment

.. note::

    Heat CLI commands such as ``openstack stack failures list`` can still be used
    to show stack failures, however since Heat no longer applies software
    configuration, it will no longer show any errors related to configuration.

Setting the status
__________________
The status of the deployment will be automatically set by the API used by the
Mistral workflows. However, in some cases, it may be required to manually set
the status to reflect what has been done manually outside of the API. The
following commands can be used to manually set the status.

Set the status to ``DEPLOY_SUCCESS``::

      openstack workflow execution create tripleo.deployment.v1.set_deployment_status_success

Set the status to ``DEPLOYING``::

      openstack workflow execution create tripleo.deployment.v1.set_deployment_status_deploying

Set the status to ``DEPLOY_FAILED``::

      openstack workflow execution create tripleo.deployment.v1.set_deployment_status_failed

The default plan name of overcloud will be used in the above commands. It can
be overridden with any of the above commands if needed::


      openstack workflow execution create tripleo.deployment.v1.set_deployment_status_success '{"plan":"my-cloud-name"}'
