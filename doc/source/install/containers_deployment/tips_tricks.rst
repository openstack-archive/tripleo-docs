Tips and Tricks for containerizing services
===========================================

This document contains a list of tips and tricks that are useful when
containerizing an OpenStack service.

Monitoring containers
---------------------

It's often useful to monitor the running containers and see what has been
executed and what not. The puppet containers are created and removed
automatically unless they fail. For all the other containers, it's enough to
monitor the output of the command below::

    $ watch -n 0.5 docker ps -a --filter label=managed_by=docker-cmd

.. _debug-containers:

Viewing container logs
----------------------

You can view the output of the main process running in a container by running::

    $ docker logs $CONTAINER_ID_OR_NAME

Ideally all containerized processes would log everything to
stdout/stderr and the above command would suffice. Not all services
are quite there yet, so we export traditional logs from containers
into the `/var/log/containers` directory on the host, where you can
look at them.

Debugging container failures
----------------------------

The following commands are useful for debugging containers.

* **inspect**: This command allows for inspecting the container's structure and
  metadata. It provides info about the bind mounts on the container, the
  container's labels, the container's command, etc::

    $ docker inspect $CONTAINER_ID_OR_NAME

  There's no shortcut for *rebuilding* the command that was used to run the
  container but, it's possible to do so by using the `docker inspect` command
  and the format parameter::

   $ docker inspect --format='{{range .Config.Env}} -e "{{.}}" {{end}} {{range .Mounts}} -v {{.Source}}:{{.Destination}}{{if .Mode}}:{{.Mode}}{{end}}{{end}} -ti {{.Config.Image}}' $CONTAINER_ID_OR_NAME

  Copy the output from the command above and append it to the one below, which
  will run the same container with a random name and remove it as soon as the
  execution exits::

    $ docker run --rm $OUTPUT_FROM_PREVIOUS_COMMAND /bin/bash

* **exec**: Running commands on or attaching to a running container is extremely
  useful to get a better understanding of what's happening in the container.
  It's possible to do so by running the following command::

    $ docker exec -ti $CONTAINER_ID_OR_NAME /bin/bash

  Replace the `/bin/bash` above with other commands to run oneshot commands. For
  example::

    $ docker exec -ti mysql mysql -u root -p $PASSWORD

  The above will start a mysql shell on the mysql container.

* **export** When the container fails, it's basically impossible to know what
  happened. It's possible to get the logs from docker but those will contain
  things that were printed on the stdout by the entrypoint. Exporting the
  filesystem structure from the container will allow for checking other logs
  files that may not be in the mounted volumes::

    $ docker export $CONTAINER_ID_OR_NAME | tar -C /tmp/$CONTAINER_ID_OR_NAME -xvf -

Using docker-toool
------------------

In addition to the above, there is also now a json file that is generated
that contains all the information for all the containers and how they
are run.  This file is `/var/lib/docker-container-startup-configs.json`.

`docker-toool` was written to read from this file and start containers
for debugging purposes based on those commands.  For now this utility
is in the tripleo-heat-templates repo in the `docker/` directory.

By default this tool lists all the containers that are started and
their start order.

If you wish to see the command line used to start a given container,
specify it by name using the --container (or -c) argument.  --run (or
-r) can then be used with this to actually execute docker to run the
container.  To see all available options use::

    ./docker-toool --help

Other options listed allow you to modify this command line for
debugging purposes.  For example::

    ./docker-toool -c swift-proxy -r -e /bin/bash -u root -i -n test

The above command will run the swift proxy container with all the volumes,
permissions etc as used at runtime but as the root user, executing /bin/bash,
named 'test', and will run interactively (eg -ti).  This allows you to enter
the container and run commands to see what is failing, perhaps install strace
and strace the command etc.  You can also verify configurations or any other
debugging task you may have.

Debugging docker-puppet.py
--------------------------

The :ref:`docker-puppet.py` script manages the config file generation and
puppet tasks for each service.  This also exists in the `docker` directory
of tripleo-heat-templates.  When writing these tasks, it's useful to be
able to run them manually instead of running them as part of the entire
stack. To do so, one can run the script as shown below::

  CONFIG=/path/to/task.json /path/to/docker-puppet.py

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
        config_image:
          list_join:
            - '/'
            - [ {get_param: DockerNamespace}, {get_param: DockerGlanceApiImage} ]


Would generated a json file called `/var/lib/docker-puppet-tasks2.json` that looks like::

    [
        {
            "config_image": "tripleoupstream/centos-binary-glance-api:latest",
            "config_volume": "glance_api",
            "puppet_tags": "glance_api_config,glance_api_paste_ini,glance_swift_config,glance_cache_config",
            "step_config": "include ::tripleo::profile::base::glance::api\n"
        }
    ]


Setting the path to the above json file as value to the `CONFIG` var passed to
`docker-puppet.py` will create a container using the
`centos-binary-glance-api:latest` image and it and run puppet on a catalog
restricted to the given puppet `puppet_tags`.

As mentioned above, it's possible to create custom json files and call
`docker-puppet.py` manually, which makes developing and debugging puppet steps
easier.

`docker-puppet.py` also supports the environment variable `SHOW_DIFF`,
which causes it to print out a docker diff of the container before and
after the configuration step has occurred.

By default `docker-puppet.py` runs things in parallel.  This can make
it hard to see the debug output of a given container so there is a
`PROCESS_COUNT` variable that lets you override this.  A typical debug
run for docker-puppet might look like::

    SHOW_DIFF=True PROCESS_COUNT=1 ./docker-puppet.py

Testing in CI
-------------

When new service containers are added, ensure to update the image names in
`container-images/overcloud_containers.yaml` tripleo-common repo. These service
images are pulled in and available in the local docker registry that the
containers ci job uses::

    uploads:
        - imagename: tripleoupstream/centos-binary-example:latest
