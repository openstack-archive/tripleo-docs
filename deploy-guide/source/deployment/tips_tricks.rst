Tips and Tricks for containerizing services
===========================================

This document contains a list of tips and tricks that are useful when
containerizing an OpenStack service.

Important Notes
---------------

Podman
------

Prior to Stein, containerized OpenStack deployments used Docker.

Starting with the Stein release, Docker is no longer part of OpenStack,
and Podman has taken its place.  The notes here are regarding Stein and later.

Monitoring containers
---------------------

It's often useful to monitor the running containers and see what has been
executed and what not. The puppet containers are created and removed
automatically unless they fail. For all the other containers, it's enough to
monitor the output of the command below::

    $ watch -n 0.5 sudo podman ps -a --filter label=managed_by=tripleo_ansible

.. admonition:: Train
   :class: stable

   ::

    $ watch -n 0.5 sudo podman ps -a --filter label=managed_by=paunch

.. _debug-containers:

Viewing container logs
----------------------

You can view the output of the main process running in a container by running::

    $ sudo podman logs $CONTAINER_ID_OR_NAME

Since the Stein release, standard out and standard error from containers are
captured in `/var/log/containers/stdouts`.

We export traditional logs from containers into the `/var/log/containers`
directory on the host, where you can look at them.

systemd and podman
------------------

Throughout this document you'll find references to direct podman commands
for things like restarting services.  These are valid and supported methods,
but it's worth noting that services are tied into the systemd management
system, which is often the preferred way to operate.

Restarting nova_scheduler for example::

    $ sudo systemctl restart tripleo_nova_scheduler

Stopping a container with systemd::

    $ sudo systemctl stop tripleo_nova_scheduler


.. _toggle_debug:

Toggle debug
------------

For services that support `reloading their configuration at runtime`_::

    $ sudo podman exec -u root nova_scheduler crudini --set /etc/nova/nova.conf DEFAULT debug true
    $ sudo podman kill -s SIGHUP nova_scheduler

.. _reloading their configuration at runtime: https://storyboard.openstack.org/#!/story/2001545

Restart the container to turn back the configuration to normal::

    $ sudo podman restart nova_scheduler

Otherwise, if the service does not yet support reloading its configuration, it
is necessary to change the configuration on the host filesystem and restart the
container::

    $ sudo crudini --set /var/lib/config-data/puppet-generated/nova/etc/nova/nova.conf DEFAULT debug true
    $ sudo podman restart nova_scheduler

Apply the inverse change to restore the default log verbosity::

    $ sudo crudini --set /var/lib/config-data/puppet-generated/nova/etc/nova/nova.conf DEFAULT debug false
    $ sudo podman restart nova_scheduler

Debugging container failures
----------------------------

The following commands are useful for debugging containers.

* **inspect**: This command allows for inspecting the container's structure and
  metadata. It provides info about the bind mounts on the container, the
  container's labels, the container's command, etc::

    $ sudo podman inspect $CONTAINER_ID_OR_NAME

* **top**: Viewing processes running within a container is trivial with Podman::

    $ sudo podman top $CONTAINER_ID_OR_NAME

* **exec**: Running commands on or attaching to a running container is extremely
  useful to get a better understanding of what's happening in the container.
  It's possible to do so by running the following command::

    $ sudo podman exec -ti $CONTAINER_ID_OR_NAME /bin/bash

  Replace the `/bin/bash` above with other commands to run oneshot commands. For
  example::

    $ sudo podman exec -ti mysql mysql -u root -p $PASSWORD

  The above will start a mysql shell on the mysql container.

* **export** When the container fails, it's basically impossible to know what
  happened. It's possible to get the logs from docker but those will contain
  things that were printed on the stdout by the entrypoint. Exporting the
  filesystem structure from the container will allow for checking other logs
  files that may not be in the mounted volumes::

    $ sudo podman export $CONTAINER_ID_OR_NAME -o $CONTAINER_ID_OR_NAME.tar

Debugging with tripleo_container_manage Ansible role
----------------------------------------------------

The debugging manual for tripleo_container_manage is documented in the role_
directly.

.. _role: https://docs.openstack.org/tripleo-ansible/latest/roles/role-tripleo_container_manage.html#debug

Debugging with Paunch
---------------------

.. note:: During Ussuri cycle, Paunch has been replaced by the
   tripleo_container_manage Ansible role. Therefore, the following block
   is deprecated in favor of the new role which contains a Debug manual.

The ``paunch debug`` command allows you to perform specific actions on a given
container.  This can be used to:

* Run a container with a specific configuration.
* Dump the configuration of a given container in either json or yaml.
* Output the docker command line used to start the container.
* Run a container with any configuration additions you wish such that you can
  run it with a shell as any user etc.

The configuration options you will likely be interested in include:

::

  --file <file>         YAML or JSON file containing configuration data
  --action <name>       Action can be one of: "dump-json", "dump-yaml",
                        "print-cmd", or "run"
  --container <name>    Name of the container you wish to manipulate
  --interactive         Run container in interactive mode - modifies config
                        and execution of container
  --shell               Similar to interactive but drops you into a shell
  --user <name>         Start container as the specified user
  --overrides <name>    JSON configuration information used to override
                        default config values
  --default-runtime     Default runtime for containers. Can be docker or
                        podman.

``file`` is the name of the configuration file to use
containing the configuration for the container you wish to use.
TripleO creates configuration files for starting containers in
``/var/lib/tripleo-config/container-startup-config``.  If you look in this directory
you will see a number of files corresponding with the steps in
TripleO heat templates.  Most of the time, you will likely want to use
``/var/lib/tripleo-config/container-startup-config/step_4``
as it contains most of the final startup configurations for the running
containers.

``shell``, ``user`` and ``interactive`` are available as shortcuts that
modify the configuration to easily allow you to run an interactive session
in a given container.

To make sure you get the right container you can use the ``paunch list``
command to see what containers are running and which config id they
are using.  This config id corresponds to which file you will find the
container configuration in.

TripleO uses ``managed_by`` and ``config_id`` labels to help identify the
containers it is managing. These can be checked by inspecting the labels section
like so:

::

  # podman inspect nova_api | jq '.[0].Config.Labels | "managed_by=\(.managed_by) config_id=\(.config_id)"'
  "managed_by=tripleo-Controller config_id=tripleo_step4"

Note that if you wish to replace a currently running container you will
want to ``sudo podman rm -f`` the running container before starting a new one.

Here is an example of using ``paunch debug`` to start a root shell inside the
heat api container:

::

  # paunch debug --file /var/lib/tripleo-config/container-startup-config/step_4  --managed-by=tripleo-Controller --config-id=tripleo_step4 --interactive --shell --user root --container nova_api --action run

This will drop you into an interactive session inside the heat api container,
starting /bin/bash running as root.

To see how this container is started by TripleO:

::

  # paunch debug --file /var/lib/tripleo-config/container-startup-config/step_4 --managed-by=tripleo-Controller --config-id=tripleo_step4 --container nova_api --action print-cmd

  podman run --name nova_api-1jpm5kyv --label config_id=tripleo_step4 --label container_name=nova_api --label managed_by=tripleo-Controller --label config_data={"environment": {"KOLLA_CONFIG_STRATEGY": "COPY_ALWAYS", "TRIPLEO_CONFIG_HASH": "5cbcd2d39667626874f547214d3980ec"}, "healthcheck": {"test": "/openstack/healthcheck"}, "image": "undercloud-0.ctlplane.redhat.local:8787/rh-osbs/rhosp16-openstack-nova-api:16.1_20210726.1", "net": "host", "privileged": false, "restart": "always", "start_order": 2, "user": "root", "volumes": ["/etc/hosts:/etc/hosts:ro", "/etc/localtime:/etc/localtime:ro", "/etc/pki/ca-trust/extracted:/etc/pki/ca-trust/extracted:ro", "/etc/pki/ca-trust/source/anchors:/etc/pki/ca-trust/source/anchors:ro", "/etc/pki/tls/certs/ca-bundle.crt:/etc/pki/tls/certs/ca-bundle.crt:ro", "/etc/pki/tls/certs/ca-bundle.trust.crt:/etc/pki/tls/certs/ca-bundle.trust.crt:ro", "/etc/pki/tls/cert.pem:/etc/pki/tls/cert.pem:ro", "/dev/log:/dev/log", "/etc/puppet:/etc/puppet:ro", "/var/log/containers/nova:/var/log/nova:z", "/var/log/containers/httpd/nova-api:/var/log/httpd:z", "/var/lib/kolla/config_files/nova_api.json:/var/lib/kolla/config_files/config.json:ro", "/var/lib/config-data/puppet-generated/nova:/var/lib/kolla/config_files/src:ro"]} --conmon-pidfile=/var/run/nova_api-1jpm5kyv.pid --detach=true --env=KOLLA_CONFIG_STRATEGY=COPY_ALWAYS --env=TRIPLEO_CONFIG_HASH=5cbcd2d39667626874f547214d3980ec --net=host --privileged=false --user=root --volume=/etc/hosts:/etc/hosts:ro --volume=/etc/localtime:/etc/localtime:ro --volume=/etc/pki/ca-trust/extracted:/etc/pki/ca-trust/extracted:ro --volume=/etc/pki/ca-trust/source/anchors:/etc/pki/ca-trust/source/anchors:ro --volume=/etc/pki/tls/certs/ca-bundle.crt:/etc/pki/tls/certs/ca-bundle.crt:ro --volume=/etc/pki/tls/certs/ca-bundle.trust.crt:/etc/pki/tls/certs/ca-bundle.trust.crt:ro --volume=/etc/pki/tls/cert.pem:/etc/pki/tls/cert.pem:ro --volume=/dev/log:/dev/log --volume=/etc/puppet:/etc/puppet:ro --volume=/var/log/containers/nova:/var/log/nova:z --volume=/var/log/containers/httpd/nova-api:/var/log/httpd:z --volume=/var/lib/kolla/config_files/nova_api.json:/var/lib/kolla/config_files/config.json:ro --volume=/var/lib/config-data/puppet-generated/nova:/var/lib/kolla/config_files/src:ro undercloud-0.ctlplane.redhat.local:8787/rh-osbs/rhosp16-openstack-nova-api:16.1_20210726.1

You can also dump the configuration of a container to a file so you can
edit it and rerun it with different a different configuration:

::

  # paunch debug --file /var/lib/tripleo-config/container-startup-config/step_4 --container nova_api --action dump-json > nova_api.json

You can then use ``nova_api.json`` as your ``--file`` argument after
editing it to your liking.

To add configuration elements on the command line you can use the
``overrides`` option.  In this example I'm adding a health check to
the container:

::

  # paunch debug --file nova_api.json --overrides '{"health-cmd": "/usr/bin/curl -f http://localhost:8004/v1/", "health-interval": "30s"}' --container nova_api --managed-by=tripleo-Controller --config-id=tripleo_step4 --action run
    f47949a7cb205083a3adaa1530fcdd4ed7dcfa9b9afb4639468357b36786ecf0

Debugging container-puppet.py
-----------------------------

The :ref:`container-puppet.py` script manages the config file generation and
puppet tasks for each service.  This also exists in the `common` directory
of tripleo-heat-templates.  When writing these tasks, it's useful to be
able to run them manually instead of running them as part of the entire
stack. To do so, one can run the script as shown below::

  CONFIG=/path/to/task.json /path/to/container-puppet.py

.. note:: Prior to the Train cycle, container-puppet.py was called
   docker-puppet.py which was located in the `docker` directory.

The json file must follow the following form::

    [
        {
            "config_image": ...,
            "config_volume": ...,
            "puppet_tags": ...,
            "step_config": ...
        }
    ]


Using a more realistic example. Given a `puppet_config` section like this::

      puppet_config:
        config_volume: glance_api
        puppet_tags: glance_api_config,glance_api_paste_ini,glance_swift_config,glance_cache_config
        step_config: {get_attr: [GlanceApiPuppetBase, role_data, step_config]}
        config_image: {get_param: DockerGlanceApiConfigImage}


Would generated a json file called `/var/lib/container-puppet/container-puppet-tasks2.json` that looks like::

    [
        {
            "config_image": "tripleomaster/centos-binary-glance-api:latest",
            "config_volume": "glance_api",
            "puppet_tags": "glance_api_config,glance_api_paste_ini,glance_swift_config,glance_cache_config",
            "step_config": "include ::tripleo::profile::base::glance::api\n"
        }
    ]


Setting the path to the above json file as the `CONFIG` environment
variable passed to `container-puppet.py` will create a container using
the `centos-binary-glance-api:latest` image and it and run puppet on a
catalog restricted to the given puppet `puppet_tags`.

As mentioned above, it's possible to create custom json files and call
`container-puppet.py` manually, which makes developing and debugging puppet
steps easier.

`container-puppet.py` also supports the environment variable `SHOW_DIFF`,
which causes it to print out a docker diff of the container before and
after the configuration step has occurred.

By default `container-puppet.py` runs things in parallel.  This can make
it hard to see the debug output of a given container so there is a
`PROCESS_COUNT` variable that lets you override this.  A typical debug
run for container-puppet might look like::

    SHOW_DIFF=True PROCESS_COUNT=1 CONFIG=glance_api.json ./container-puppet.py

Testing a code fix in a container
---------------------------------
Let's assume that we need to test a code patch or an updated package in a
container. We will look at a few steps that can be taken to test a fix
in a container on an existing deployment.

For example let's update packages for the mariadb container::

    (undercloud) [stack@undercloud ~]$ sudo podman images | grep mariadb
    192.168.24.1:8787/tripleomaster/centos-binary-mariadb    latest     035a8237c376    2 weeks ago    723.5 MB

So container image `035a8237c376` is the one we need to base our work on. Since
container images are supposed to be immutable we will base our work off of
`035a8237c376` and create a new one::

    mkdir -p galera-workaround
    cat > galera-workaround/Dockerfile <<EOF
    FROM 192.168.24.1:8787/tripleomaster/centos-binary-mariadb:latest
    USER root
    RUN yum-config-manager --add-repo http://people.redhat.com/mbaldess/rpms/container-repo/pacemaker-bundle.repo && yum clean all && rm -rf /var/cache/yum
    RUN yum update -y pacemaker pacemaker-remote pcs libqb resource-agents && yum clean all && rm -rf /var/cache/yum
    USER mysql
    EOF

To determine which user is the default one being used in a container you can run  `docker run -it 035a8237c376 whoami`.
Then we build the new image and tag it with `:workaround1`::

    docker build --rm -t 192.168.24.1:8787/tripleomaster/centos-binary-mariadb:workaround1 ~/galera-workaround

Then we push it in our docker registry on the undercloud::

    docker push 192.168.24.1:8787/tripleomaster/centos-binary-mariadb:workaround1

At this stage we can either point THT to use
`192.168.24.1:8787/tripleomaster/centos-binary-mariadb:workaround1` as the
container image by tweaking the necessary environment files and we redeploy the overcloud.
If we only want to test a tweaked image, the following steps can be used:
First, determine if the containers are managed by pacemaker (those will typically have a `:pcmklatest` tag) or by paunch.
For the paunch-managed containers see `Debugging with Paunch`_.
For the pacemaker-managed containers you can (best done on your staging env, as it might be an invasive operation) do the following::

    1. `pcs cluster cib cib.xml`
    2. Edit the cib.xml with the changes around the bundle you are tweaking
    3. `pcs cluster cib-push --config cib.xml`


Testing in CI
-------------

When new service containers are added, be sure to update the image names in
`container-images` in the tripleo-common repo. These service
images are pulled in and available in the local docker registry that the
containers ci job uses.

Packages versions in containers
-------------------------------

With the container CI jobs, it can be challenging to find which version of OpenStack runs in the containers.
An easy way to find out is to use the `logs/undercloud/home/zuul/overcloud_containers.yaml.txt.gz` log file and
see which tag was deployed.

For example::

  container_images:
  - imagename: docker.io/tripleomaster/centos-binary-ceilometer-central:ac82ea9271a4ae3860528eaf8a813da7209e62a6_28eeb6c7
    push_destination: 192.168.24.1:8787

So we know the tag is `ac82ea9271a4ae3860528eaf8a813da7209e62a6_28eeb6c7`.
The tag is actually a Delorean hash. You can find out the versions
of packages by using this tag.
For example, `ac82ea9271a4ae3860528eaf8a813da7209e62a6_28eeb6c7` tag,
is in fact using this `Delorean repository`_.

..  _Delorean repository: https://trunk.rdoproject.org/centos7-master/ac/82/ac82ea9271a4ae3860528eaf8a813da7209e62a6_28eeb6c7/
