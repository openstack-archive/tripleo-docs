Deployment Status
-----------------
Since Heat is no longer the source of authority on the status of the overcloud
deployment, a new tripleoclient command is available to show the overcloud
deployment status::

    openstack overcloud plan deployment status

The output will report the status of the deployment, taking into consideration
the result of all the steps to do the full deployment. The following is an
example of the sample output::

    (undercloud) [stack@undercloud ]$ openstack overcloud plan deployment status


    +-----------+---------------------+---------------------+-------------------+
    | Plan Name |       Created       |       Updated       | Deployment Status |
    +-----------+---------------------+---------------------+-------------------+
    | overcloud | 2018-05-03 21:24:50 | 2018-05-03 21:27:59 |   DEPLOY_SUCCESS  |
    +-----------+---------------------+---------------------+-------------------+

.. note::

    Heat CLI commands such as ``openstack stack failures list`` can still be used
    to show stack failures, however since Heat no longer applies software
    configuration, it will no longer show any errors related to configuration.
