.. _ephemeral_heat:

Ephemeral Heat
==============

Introduction
------------

Ephemeral Heat is a means to install the overcloud by using an ephemeral Heat
process instead of a system installed Heat process. This change is possible
beginning in the Wallaby release.

In a typical undercloud, Heat is installed on the undercloud and processes are
run in podman containers for heat-api and heat-engine. When using ephemeral
Heat, there is no longer a requirement that Heat is installed on the
undercloud, instead these processes are started on demand by the deployment,
update, and upgrade commands.

This model has been in use within TripleO already for both the undercloud and
:ref:`standalone <standalone>` installation methods, which start an on demand
all in one heat-all process in order to perform only the installation. Using
ephemeral Heat in this way allows for re-use of the Heat templates from
tripleo-heat-templates without having to require an already fully installed
undercloud.

Description
-----------

Ephemeral Heat is enabled by passing the ``--heat-type`` argument to
``openstack overcloud deploy``. The ephemeral process can also be launched
outside of a deployment with the ``openstack tripleo launch heat`` command. The
latter command also takes a ``--heat-type`` argument to enable selecting the
type of Heat process to use.

Heat types
__________

The ``--heat-type`` argument allows for the following options described below.

installed
    Use the system Heat installation. This is the historical TripleO usage of
    Heat with Heat fully installed on the undercloud. This is the default
    value, and requires a fully installed undercloud.

native
    Use an ephemeral ``heat-all`` process. The process will be started natively
    on the system executing tripleoclient commands by way of an OS (operating
    system) fork.

container
    A podman container will be started on the executing system that runs a
    single ``heat-all`` process.

pod
    A podman pod will be started on the executing system that runs containers
    for ``heat-api`` and ``heat-engine``.

In all cases, the process(es) are terminated at the end of the deployment.

.. note::

    The native and container methods are limited in scale due to being a single
    Heat process. Deploying more than 3 nodes or 2 roles will significantly
    impact the deployment time with these methods as Heat has only a single
    worker thread.

    Using the installed or pod methods enable scaling node and role counts as
    is typically required.

Using
-----

The following example shows using ``--heat-type`` to enable ephemeral Heat::

    openstack overcloud deploy \
      --stack overcloud \
      --work-dir ~/overcloud-deploy/overcloud \
      --heat-type <pod|container|native> \
      <other cli arguments>

With ephemeral Heat enabled, several additional deployment artifacts are
generated related to the management of the Heat process(es). These artifacts
are generated under the working directory of the deployment in a
``heat-launcher`` subdirectory. The working directory can be overridden with
the ``--work-dir`` argument.

Using the above example, the Heat artifact directory would be located at
``~/overcloud-deploy/overcloud/heat-launcher``. An example of the directory
contents is shown below::

   [centos@ephemeral-heat ~]$ ls -l ~/overcloud-deploy/overcloud/heat-launcher/
   total 41864
   -rw-rw-r--. 1 centos centos      650 Mar 24 18:39 api-paste.ini
   -rw-rw-r--. 1 centos centos     1054 Mar 24 18:39 heat.conf
   -rw-rw-r--. 1 centos centos 42852118 Mar 24 18:31 heat-db-dump.sql
   -rw-rw-r--. 1 centos centos     2704 Mar 24 18:39 heat-pod.yaml
   drwxrwxr-x. 2 centos centos       49 Mar 24 16:02 log
   -rw-rw-r--. 1 centos centos     1589 Mar 24 18:39 token_file.json

The directory contains the necessary files to inspect and debug the Heat
process(es), and if necessary reproduce the deployment.

.. note::

    The consolidated log file for the Heat process is the ``log`` file in the
    ``heat-launcher`` directory.

Launching Ephemeral Heat
________________________

Outside of a deployment, the ephemeral Heat process can also be started with the
``openstack tripleo launch heat`` command. This can be used to interactively
use the ephemeral Heat process or to debug a previous deployment.

When combined with ``--heat-dir`` and ``--restore-db``, the command can be used
to restore the Heat process and database from a previous deployment::

    openstack tripleo launch heat \
      --heat-type pod \
      --heat-dir ~/overcloud-deploy/overcloud/heat-launcher \
      --restore-db

The command will exit after launching the Heat process, and the Heat process
will continue to run in the background.

Interacting with ephemeral Heat
...............................

With the ephemeral Heat process launched and running, ``openstackclient`` can be
used to interact with the Heat API. The following shell environment
configuration must set up access to the Heat API::

    unset OS_CLOUD
    unset OS_PROJECT_NAME
    unset OS_PROJECT_DOMAIN_NAME
    unset OS_USER_DOMAIN_NAME
    OS_AUTH_TYPE=none
    OS_ENDPOINT=http://127.0.0.1:8006/v1/admin

You can also use the ``OS_CLOUD`` environment to set up the same::

    export OS_CLOUD=heat

Once the environment is configured, ``openstackclient`` work as expected
against the Heat API::

    [centos@ephemeral-heat ~]$ openstack stack list
    +--------------------------------------+------------+---------+-----------------+----------------------+--------------+
    | ID                                   | Stack Name | Project | Stack Status    | Creation Time        | Updated Time |
    +--------------------------------------+------------+---------+-----------------+----------------------+--------------+
    | 761e2a54-c6f9-4e0f-abe6-c8e0ad51a76c | overcloud  | admin   | CREATE_COMPLETE | 2021-03-22T20:48:37Z | None         |
    +--------------------------------------+------------+---------+-----------------+----------------------+--------------+

Killing ephemeral Heat
......................

To stop the ephemeral Heat process previously started with ``openstack tripleo
launch heat``, use the ``--kill`` argument::

    openstack tripleo launch heat \
      --heat-type pod \
      --heat-dir ~/overcloud-deploy/overcloud/heat-launcher \
      --kill

Limitations
-----------
Ephemeral Heat currently only supports new deployments. Update and Upgrade
support for deployments that previously used the system installed Heat will be
coming.
