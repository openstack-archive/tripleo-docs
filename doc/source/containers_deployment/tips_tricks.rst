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

    $ watch docker ps -a --filter label=managed_by=docker-cmd

.. _debug-containers:

Debugging container failures
----------------------------

The following commands are useful for debugging containers.

* **inspect**: This command allows for inspecting the container's structure and
  metadata. It provides info about the bind mounts on the container, the
  container's labels, the container's command, etc::

    $ docker inspect $CONTAINER_ID_OR_NAME

  There's no shortcut for *rebuilding* the command that was used to run the
  container but, it's possible to do so by using the `docker inspect` command
  and the format parameter:::

   $ docker inspect --format='{{range .Config.Env}} -e "{{.}}" {{end}} {{range .Mounts}} -v {{.Source}}:{{.Destination}}{{if .Mode}}:{{.Mode}}{{end}}{{end}} -ti {{.Config.Image}}' $CONTAINER_ID_OR_NAME

  Copy the output from the command above and append it to the one below, which
  will run the same container with a random name and remove it as soon as the
  execution exits:::

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


Debugging docker-puppet.py
--------------------------

The :ref:`docker-puppet.py` script manages the config file generation and puppet
tasks for each service. When writing these tasks, it's useful to be able to run
them manually instead of running them as part of the entire stack. To do so, one
can run the script as shown below::

  CONFG=/path/to/task.json $THT_ROOT/docker/docker-puppet.py

The json file must follow the following form::

    [
        [
            $CONFIG_VOLUME,
            $PUPPET_TAGS,
            $PUPPET_MANIFEST,
            $DOCKER_IMAGE,
            $DOCKER_VOLUMES
        ]
    ]


Using a more realistic example. Given a `docker_puppet_task` section like this::

      docker_puppet_tasks:
        step_2:
          - 'mongodb_init_tasks'
          - 'mongodb_database,mongodb_user,mongodb_replset'
          - 'include ::tripleo::profile::base::database::mongodb'
          - list_join:
            - '/'
            - [ {get_param: DockerNamespace}, {get_param: DockerMongodbImage} ]
          - - "mongodb:/var/lib/mongodb"
            - "logs:/var/log/kolla:ro"


Would generated a json file called `/var/lib/docker-puppet-tasks2.json` that looks like::

    [
        [
            mongodb_init_tasks,
            "mongodb_database,mongodb_user,mongodb_replset",
            "include ::tripleo::profile::base::database::mongodb",
            "tripleoupstream/centos-binary-mongodb:latest",
            [
                "mongodb:/var/lib/mongodb",
                "logs:/var/log/kolla:ro"
            ]
        ]
    ]


Setting the path to the above json file as value to the `CONFIG` var passed to
`docker-puppet.py` will create a container using the
`centos-binary-mongodb:latest` image and it'll run the puppet puppet tags listed
in the second item of the array.

As mentioned above, it's possible to create custom json files and call
`docker-puppet.py` manually, which makes developing and debugging puppet steps
easier.
