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

    +------------+-------------------+
    | Stack Name | Deployment Status |
    +------------+-------------------+
    | overcloud  |   DEPLOY_SUCCESS  |
    +------------+-------------------+

A different stack name can be specified with ``--stack``::

    [stack@undercloud ]$ openstack overcloud status --stack my-deployment

    +---------------+-------------------+
    | Stack Name    | Deployment Status |
    +-----------+-----------------------+
    | my-deployment |   DEPLOY_SUCCESS  |
    +---------------+-------------------+

The deployment status is stored in the YAML file, generated at
``$HOME/overcloud-deploy/<stack>/<stack>-deployment_status.yaml`` in
the undercloud node.
