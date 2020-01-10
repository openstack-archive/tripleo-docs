Tempest
=======

This is a set of integration tests to be run against a live OpenStack cluster.
Tempest has batteries of tests for OpenStack API validation, scenarios, and
other specific tests useful in validating an OpenStack deployment.

Current State of Tempest
------------------------

Source code : https://opendev.org/openstack/tempest/

Tempest Version release wise:
+++++++++++++++++++++++++++++
* Ocata : 16.1.0
* Pike  : 17.2.0
* Queens : 18.0.0
* Master : master

What Tempest provides?
----------------------

* Tempest provides a set of stable apis/interfaces which are used in tempest
  tests and tempest plugins to keep backward compatibility.

  * Below is the list of stable interfaces:

    * tempest.lib.*
    * tempest.config
    * tempest.test_discover.plugins
    * tempest.common.credentials_factory
    * tempest.clients
    * tempest.test

* Tempest contains API tests for Nova, Glance, Cinder, Swift, Keystone as well
  as scenario tests for covering these components and these tests are used for
  InterOp certifications as validating the OpenStack deployment for the above
  services.

* The test which do not fit within the Tempest testsuite will go under
  respective service specific tempest plugins.

Tempest Plugins
---------------

Tempest plugins contain the API and scenario tests for specific OpenStack
services.
Here is the detailed list of `tempest plugins consumed`_ in a TripleO deployment.

.. _tempest plugins consumed: ./tempest_plugins.html

Packages provided by RDO
------------------------

* Tempest related RPMs

  * python-tempest: this package contains the tempest python library and is
    consumed as a dependency for out of tree tempest plugins i.e. for Horizon
    and Designate tempest plugins.
  * python-tempestconf: It provides the :command:`discover-tempest-config`
    utility through which we can generate tempest config.
  * openstack-tempest: this package contains a set of integration tests to be
    run against a live OpenStack cluster and required executables for running
    tempest. Packages `python-tempest` and `python-tempestconf` mentioned above
    are dependencies of `openstack-tempest` package.
  * openstack-tempest-all: It will install openstack-tempest as well as all
    the tempest plugins on the system.

* Test Runners:

  * python-stestr: It is a parallel python test runner built around subunit.
    It is used by Tempest to run tempest tests under the hood.
  * python-os-testr: It is another test runner wrapped around stestr. It is
    also used to run tempest tests.

* Kolla based tempest container

  * RDO also provides Kolla based container images for Tempest. It has
    openstack-tempest and all the required tempest plugins installed in it.
  * Run the following command to pull the tempest container Image::

    $ sudo docker pull docker.io/tripleomaster/centos-binary-tempest


Some housekeeping rules
-----------------------

* **Always** install tempest and its dependencies from **RPM**.
* Make sure the right package with **correct version** is installed
  (openstack-tempest RPM and its plugins are well tested in CI).
* **Never ever** mix pip and RPM in an openstack deployment.
* Please **read** the documentation fully before running tempest.
* openstack-tempest rpm **does not** install tempest plugins, they need to be
  installed separately.
* Additional configuration for tempest plugins **may need** to be set.
* **python-tempestconf** is installed by **openstack-tempest** rpm itself. It's
  not needed to install it separately.
* openstack-tempest is installed **on the undercloud**.
* Source **openstackrc file** for undercloud or overcloud when running Tempest
  from undercloud.
* openstack-tempest is currently used **to validate** undercloud as well as
  overcloud.
* Use Tempest **container image** to avoid installing tempest plugins on the
  deployed cloud.


Using TripleO-QuickStart to run Tempest
---------------------------------------

TripleO project provides validate-tempest ansible role through which Tempest is
used to validate undercloud and overcloud.
Set your workspace and path to a config file that contains the node
configuration, the following is the default::

    CONFIG=config/general_config/minimal.yml
    WORKSPACE=/home/centos/.quickstart

* Running tempest against overcloud::

    $ cd <path to triplo-quickstart repo>

    $ bash quickstart.sh \
      --bootstrap \
      --tags all \
      --config $CONFIG \
      --working-dir $WORKSPACE/ \
      --no-clone \
      --release master-tripleo-ci \
      --extra-vars test_ping=False \
      --extra-vars run_tempest=True  \
      $VIRTHOST

  The above command will run smoke tests on overcloud and use tempest rpm.

* Running tempest against undercloud::

    $ bash quickstart.sh \
      --bootstrap \
      --tags all \
      --config $CONFIG \
      --working-dir $WORKSPACE/ \
      --no-clone \
      --release master-tripleo-ci \
      --extra-vars test_ping=False \
      --extra-vars run_tempest=True  \
      --extra-vars tempest_overcloud=False \
      --extra-vars tempest_undercloud=True \
      --extra-vars tempest_white_regex='tempest.api.(identity|compute|network|image)' \
      $VIRTHOST

  The above command will run Identity, Compute, Network and Image api tests on
  undercloud.

* Running Tempest against undercloud using containerized tempest::

    $ bash quickstart.sh \
      --bootstrap \
      --tags all \
      --config $CONFIG \
      --working-dir $WORKSPACE/ \
      --no-clone \
      --release master-tripleo-ci \
      --extra-vars test_ping=False \
      --extra-vars run_tempest=True  \
      --extra-vars tempest_overcloud=False \
      --extra-vars tempest_undercloud=True \
      --extra-vars tempest_format=container \
      --extra-vars tempest_white_regex='tempest.api.(identity|compute|network|image)' \
      $VIRTHOST

  The above command will run Identity, Compute, Network and Image api tests on
  undercloud using containerized tempest.

.. note::
  Here is the list of
  `validate-tempest role variables <https://opendev.org/openstack/tripleo-quickstart-extras/src/branch/master/roles/validate-tempest/README.md>`_
  which can be modified using extra-vars.


Running Tempest manually
------------------------

Required resources before running Tempest
+++++++++++++++++++++++++++++++++++++++++

The following resources are needed to be created, only if Tempest is run
manually.

* If Tempest is run against undercloud, then source the stackrc file::

    $ source stackrc

    $ export OS_AUTH_URL="$OS_AUTH_URL/v$OS_IDENTITY_API_VERSION"

* If Tempest is run against overcloud, then source the overcloudrc file::

    $ source overcloudrc

* Create *Member* role for undercloud/overcloud, it will be used by tempest
  tests::

    $ openstack role create --or-show Member

* Create a public network having external connectivity, will be used by tempest
  tests when running tempest tests against overcloud

  * Create a public network::

        $ openstack network create public \
            --external \
            --provider-network-type flat \
            --provider-physical-network datacentre

  * Create/Attach subnet to it::

        $ openstack subnet create ext-subnet \
            --subnet-range 192.168.24.0/24 \
            --allocation-pool start=192.168.24.150,end=192.168.24.250 \
            --gateway 192.168.24.1 \
            --no-dhcp \
            --network public


Installing Tempest RPM and its plugins
++++++++++++++++++++++++++++++++++++++

Install openstack-tempest::

    $ sudo yum -y install openstack-tempest

Install tempest plugins

* Find out what are the openstack services configured on overcloud/undercloud.
* Then install the respective plugins on undercloud using yum command.

Getting the list of tempest rpms and tempest plugins installed on undercloud::

    $ rpm -qa | grep tempest


Tempest workspace
+++++++++++++++++

Create a tempest workspace::

    $ tempest init tempest_workspace

tempest_workspace directory will be created automatically in the location where
the above command is executed.
It will create three folders within tempest_workspace directory.

* etc - tempest configuration file tempest.conf will resides here.
* logs - tempest.log file will be here
* tempest_lock - It holds the lock for tempest workspace.
* .stestr.conf - It is used to load all the tempest tests.

List tempest workspaces::

    $ tempest workspace list

The tempest workspace information is found in ~/.tempest folder.


Generating tempest.conf using discover-tempest-config
+++++++++++++++++++++++++++++++++++++++++++++++++++++

For running Tempest a tempest configuration file called ``tempest.conf`` needs
to be created. Thanks to that file Tempest knows the configuration of the
environment it will be run against and can execute the proper set of tests.

The tempest configuration file can be generated automatically by
:command:`discover-tempest-config` binary, which is provided by
``python-tempestconf`` package installed by ``openstack-tempest`` rpm.
:command:`discover-tempest-config` queries the cloud and discovers cloud
configuration.

.. note::
  To know more about ``python-tempestconf`` visit
  `python-tempestconf's documentation. <https://docs.openstack.org/python-tempestconf/latest/>`_

.. note::
  Not all of the configuration may be discovered by
  :command:`discover-tempest-config`, therefore the tempest.conf needs to be
  rechecked for correctness or tuned so that it better suits the user's needs.

All the below operations will be performed from undercloud.

For undercloud
**************

Source the stackrc file::

    $ source stackrc

Use :command:`discover-tempest-config` to generate ``tempest.conf``
automatically::

    $ cd <path to tempest workspace>

    $ discover-tempest-config --out etc/tempest.conf \
      --image <path to cirros image> \
      --debug \
      --create \
      auth.use_dynamic_credentials true \
      auth.tempest_roles Member \
      network-feature-enabled.port_security true \
      compute-feature-enabled.attach_encrypted_volume False \
      validation.image_ssh_user cirros \
      validation.ssh_user cirros \
      compute-feature-enabled.console_output true


For overcloud
*************

Source the overcloudrc file::

    $ source overcloudrc

Use :command:`discover-tempest-config` to generate tempest.conf automatically::

    $ cd <path to tempest workspace>

    $ discover-tempest-config --out etc/tempest.conf \
      --deployer-input ~/tempest-deployer-input.conf \
      --network-id $(openstack network show public -f value -c id) \
      --image <path/url to cirros image to use> \
      --debug \
      --remove network-feature-enabled.api_extensions=dvr \
      --create \
      auth.use_dynamic_credentials true \
      auth.tempest_roles Member \
      network-feature-enabled.port_security true \
      compute-feature-enabled.attach_encrypted_volume False \
      network.tenant_network_cidr 192.168.0.0/24 \
      compute.build_timeout 500 \
      volume-feature-enabled.api_v1 False \
      validation.image_ssh_user cirros \
      validation.ssh_user cirros \
      network.build_timeout 500 \
      volume.build_timeout 500 \
      object-storage-feature-enabled.discoverability False \
      service_available.swift False \
      compute-feature-enabled.console_output true \
      orchestration.stack_owner_role Member

On the successful execution of above command, the tempest.conf will be get
generated in <path to tempest workspace>/etc/tempest.conf.

Things to keep in mind while using discover-tempest-config
**********************************************************
* tempest.conf values may be overridden by passing [section].[key] [value]
  arguments.
  For example: when **compute.allow_tenant_isolation true** is passed to
  :command:`discover-tempest-config` that value will be set in tempest.conf and will
  override the value set by discovery.
  `More about override options. <https://docs.openstack.org/python-tempestconf/latest/user/usage.html#override-values>`_

* If OpenStack was deployed using TripleO/Director, pass the deployment input
  file tempest-deployer-input.conf to the :command:`discover-tempest-config` command with
  ``--deployer-input`` option. The file contains some version specific values set
  by the installer. More about the argument can be found in
  `python-tempestconf's CLI documentation. <https://docs.openstack.org/python-tempestconf/latest/cli/cli_options.html>`_

* ``--remove`` option can be used to remove values from tempest.conf,
  for example: ``--remove network-feature-enabled.api_extensions=dvr``.
  The feature is useful when some values in tempest.conf are automatically
  set by the discovery, but they are not wanted to be printed to tempest.conf.
  More about the feature can be found
  `here <https://docs.openstack.org/python-tempestconf/latest/user/usage.html#prevent-some-key-value-pairs-to-be-set-in-tempest-conf>`_.


Always save the state of resources before running tempest tests
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
In order to be able to use tempest utility to clean up resources after running
tests, it's needed to initialize the state of resources before running the
tests::

    $ tempest cleanup --init-saved-state

It will create **saved_state.json** file in tempest workspace containing all
the tenants and resources information present on the system under test. More
about the feature can be found in
`Tempest documentation <https://docs.openstack.org/tempest/latest/cleanup.html>`_.

List tempest plugins installed on undercloud
++++++++++++++++++++++++++++++++++++++++++++

Since we install the required tempest plugins on undercloud, use tempest
command to find out::

    $ tempest list-plugins

List tempest tests
++++++++++++++++++

Go to tempest workspace and run the following command to get the list::

    $ cd <path to tempest workspace>
    $ tempest run -l

To grep a list of specific tests like all compute tests::

    $ tempest run -l | grep compute

Running Tempest tests
+++++++++++++++++++++

**tempest run** utility is used to run tempest tests. It will use the configs
defined in tempest.conf to run tests against the targeted host.

* For running all api/scenario tempest tests::

    $ tempest run -r '(api|scenario)'

* For running smoke tests for basic sanity of the deployed cloud::

    $ tempest run --smoke

* For running specific tempest plugin tests like: keystone_tempest_plugin tests::

    $ tempest run --regex '(keystone_tempest_plugin)'

* Running multiple tests::

    $ tempest run --regex '(test_regex1 | test_regex2 | test_regex 3)'

* Use ``--black-regex`` argument to skip specific tests::

    $ tempest run -r '(api|scenario)' --black-regex='(keystone_tempest_plugin)'

  The above will skip all keystone_tempest_plugin tests.

Using whitelist file for running selective tests
++++++++++++++++++++++++++++++++++++++++++++++++

Writing long test regex seems to be boring, let's create a simple whitelist file
and use the same with tempest run to run those specific whitelist tests.

* Create a whitelist.txt file in tempest workspace::

    $ touch whitelist.txt

* Append all the all tests in a newline which we want to run in whitelist.txt
  file::

    $ cat whitelist.txt
      keystone_tempest_plugin.*
      # networking bgpvpn tempest tests
      networking_bgpvpn_tempest.tests*

  .. note::
    Use **#** to add comments in the whitelist/blacklist file.

* Running tempest tests present in whitelist file::

    $ tempest run -w <path to whitelist file>


Using blacklist file to skipping multiple tests
+++++++++++++++++++++++++++++++++++++++++++++++

If we want to skip multiple tests, we can blacklist file for the same.

* Create a skip_test.txt file in tempest workspace::

    $ touch skip_test.txt


* Append all the all tests in a newline which we want to skip in skip_test.txt
  file::

    $ cat whitelist.txt
      keystone_tempest_plugin.*
      # networking bgpvpn tempest tests
      networking_bgpvpn_tempest.tests*

* Use *-b* optuon with tempest run to skip/blacklist tests::

    $ tempest run -w <path to whitelist_file> -b <path to skip tests>

Running Tempest tests serially as well as in parallel
+++++++++++++++++++++++++++++++++++++++++++++++++++++

* All test methods within a TestCase are assumed to be executed serially.
* To run tempest tests serially::

    $ tempest run --serial

* Run the tests in parallel (this is the default)::

    $ tempest run --parallel

* Specify the number of workers to use when running tests in parallel::

    $ tempest run -r '(test_regex)' --concurrency <numbers of workers>

* The default number of workers is equal to the number of CPUs on the system
  under test.

Generating HTML report of tempest tests
+++++++++++++++++++++++++++++++++++++++

* In order to generate tempest subunit files in v2 format, use ``--subunit``
  flag with tempest run::

    $ tempest run -r '(test_regex)' --subunit

* Generating html output from it::

    $ subunit2html .stestr/<run number file> tempest.html

* subunit2html command is provided by python-subunit rpm package.


Where are my tempest tests results?
+++++++++++++++++++++++++++++++++++

Once tempest run finishes, All the tests results are stored in subunit file
format under **.stestr** folder under tempest workspace.

* 0,1,<list of tempest run> files contains the tempest run output.
* **failing** contains the list of failed tests with detailed api responses.
* All the tests executions api responses is logged in **tempest.log** file in
  tempest workspace.


Status of Tempest tests after tempest run
+++++++++++++++++++++++++++++++++++++++++

After the execution of tempest tests, It will generate 3 status

* **PASSED**: The test successfully run.
* **FAILED**: The test got failed due to specific reasons.
* **SKIPPED**: If a tempest tests is skipped, it will give a reason why it is
  skipped.


Cleaning up environment after tempest run
+++++++++++++++++++++++++++++++++++++++++
More about this feature can be found in
`Tempest documentation <https://docs.openstack.org/tempest/latest/cleanup.html>`

* Get a report of resources and tenants which got created/modified after tempest tests run::

    $ tempest cleanup --dry-run

  It will create a dry_run.json file in tempest workspace.
* Cleaning up the environment::

    $ tempest cleanup

* We can force delete the tempest resources and as well as associated admin
  tenants::

    $ tempest cleanup --delete-tempest-conf-object


Running containerized Tempest manually
--------------------------------------
This section shows how to run Tempest from a container against overcloud or
undercloud on undercloud. The required resources for running containerized
Tempest are the same as for running the non-containerized one.
To find out which resources are needed, see
`Required resources before running Tempest`_.

All the steps below use **stack user** as an example. You may be ssh-ed as a
different user but in that case you **have to** change all of the paths below
accordingly (instead of stack user user your $USER)

Prepare the tempest container
+++++++++++++++++++++++++++++
* Change to `/home/stack` directory::

    $ cd /home/stack

* Download a container::

    $ docker pull docker.io/tripleomaster/centos-binary-tempest:current-tripleo-rdo

* Create directories which will be used for exchanging data between the host
  machine and the container::

    $ mkdir container_tempest tempest_workspace

* We'll use container_tempest as a source of files for the container, so let's
  copy there all needed files::

    $ cp stackrc overcloudrc tempest-deployer-input.conf container_tempest

* List available images::

    $ docker image list

  or::

    $ docker images

  you should see something like::

    REPOSITORY                                      TAG                     IMAGE ID            CREATED             SIZE
    docker.io/tripleomaster/centos-binary-tempest   current-tripleo-rdo     881f7ac24d8f        10 days ago         1.09 GB


How to execute commands within the container?
+++++++++++++++++++++++++++++++++++++++++++++
In order to make it easier, create an alias as follows::

     $ alias docker-tempest="docker run -i \
         -v "$(pwd)"/container_tempest:/home/stack/container_tempest \
         -v "$(pwd)"/tempest_workspace:/home/stack/tempest_workspace \
         docker.io/tripleomaster/centos-binary-tempest:current-tripleo-rdo \
         /bin/bash"

When mounting the directories, make sure that **absolute** paths are used.

* If you want to check available tempest plugins in the container, run::

    $ docker-tempest -c "tempest list-plugins"

* For getting a list of tempest related rpms installed within the tempest
  container run::

    $ docker-tempest -c "rpm -qa | grep tempest"


Generate tempest.conf and run tempest tests within the container
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
* Let's create a tempest script which will be later executed within the
  container in order to generate tempest.conf and run tempest tests::

    $ cat <<'EOF'>> /home/stack/container_tempest/tempest_script.sh
    # Set the exit status for the command
    set -e

    # if you want to run tempest against overcloud, overcloudrc file needs
    # to be sourced and in case of undercloud it's stackrc
    # NOTE: the files need to be copied to /home/stack/container_tempest
    # directory in order to have it accessible from the container
    source /home/stack/container_tempest/overcloudrc

    # Create a tempest workspace, use the shared directory so that the files
    # in it are accessible from the host as well.
    tempest init /home/stack/tempest_workspace

    # change directory to tempest_workspace
    pushd /home/stack/tempest_workspace

    # export TEMPESTCONF environment variable for easier later usage
    export TEMPESTCONF="/usr/bin/discover-tempest-config"
    # Execute the discover-tempest-config in order to generate tempest.conf
    # Set --out to /home/stack/tempest_workspace/tempest.conf so that the
    # tempest.conf file is later accessible from host machine as well.
    # Set --deployer-input to point to the tempest-deployer-input.conf
    # located in the shared directory.
    $TEMPESTCONF \
      --out /home/stack/tempest_workspace/etc/tempest.conf \
      --deployer-input /home/stack/container_tempest/tempest-deployer-input.conf \
      --debug \
      --create \
      object-storage.reseller_admin ResellerAdmin

    # Run for example smoke tests
    tempest run --smoke

    EOF

  .. note::

    * Apart from arguments passed to python-tempestconf showed above, any other
      wanted arguments can be specified there. See
      `Generating tempest.conf using discover-tempest-config`_.
    * Instead of running smoke tests, other types of tests can be ran,
      see `Running Tempest tests`_ section.
    * `Always save the state of resources before running tempest tests`_.
    * If you **already have** a `tempest.conf` file and you want to just run
      tempest tests, **omit** TEMPESTCONF from the script above and replace it
      with a command which copies your `tempest.conf` from `container_tempest`
      directory to `tempest_workspace/etc` directory::

        $ cp /home/stack/container_tempest/tempets.conf /home/stack/tempest_workspace/etc/tempest.conf

* Set executable privileges to the `tempest_script.sh` script::

    $ chmod +x container_tempest/tempest_script.sh

* Run the tempest script from the container as follows::

     $ docker run -i \
         -v "$(pwd)"/container_tempest:/home/stack/container_tempest \
         -v "$(pwd)"/tempest_workspace:/home/stack/tempest_workspace \
         docker.io/tripleomaster/centos-binary-tempest:current-tripleo-rdo \
         /bin/bash \
         -c 'set -e; /home/stack/container_tempest/tempest_script.sh'

* In case you want to rerun the tempest tests, clean tempest workspace first::

    $ sudo rm -rf /home/stack/tempest_workspace
    $ mkdir /home/stack/tempest_workspace

  .. note::
    It's done with sudo because tempest in containers creates the files
    as root.
