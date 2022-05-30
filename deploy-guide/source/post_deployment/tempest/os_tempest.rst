os_tempest
==========

os_tempest is the unified ansible role for installing, configuring and running tempest
tests as well as processing tempest results using stackviz.

TripleO CI group collaborates on `os_tempest` development. To see what
`os_tempest` is and the reasons why it was started have a look
`at the os-tempest documentation <https://docs.openstack.org/openstack-ansible-os_tempest/latest/overview.html>`_.

Installation on a manually deployed TripleO Standalone Deployment
-----------------------------------------------------------------
Follow the `os_tempest Installation guide
<https://docs.openstack.org/openstack-ansible-os_tempest/latest/user/installation.html>`_.

If the installation was successful you can expect ansible-galaxy to list the following roles: config_template, python_venv_build, os_tempest.

.. code-block:: shell

    $ ansible-galaxy list
    - config_template, master
    - python_venv_build, master
    - os_tempest, (unknown version)

Running os_tempest role using the playbook
------------------------------------------
In order to run os_tempest role on target host, you must first ensure you have a working clouds.yaml.

Then, we can create a `tempest.yaml` playbook with the following vars:

.. code-block:: yaml

    ---
    - hosts: localhost
      name: Run Tempest on Standalone
      vars:
        ansible_become: true
        tempest_run: 'yes'
        tempest_install_method: 'distro'
        tempest_cloud_name: 'standalone'
        tempest_workspace: "/home/centos/tempest"
        tempest_services:
          - neutron
        tempest_public_net_physical_type: 'datacentre'
        tempest_private_net_provider_type: 'geneve'
        tempest_service_setup_host: '{{ inventory_hostname }}'
        tempest_public_subnet_cidr: '192.168.0.0/24'
        tempest_public_subnet_gateway_ip: '{{ tempest_public_subnet_cidr|nthhost(1) }}'
        tempest_public_subnet_allocation_pools: '{{ tempest_public_subnet_cidr|nthhost(100) ~ "-" ~ tempest_public_subnet_cidr|nthhost(120) }}'
        tempest_use_tempestconf: true
        tempest_run_stackviz: false
        tempest_tempest_conf_overrides:
          auth.tempest_roles: "Member"
        tempest_test_whitelist:
          - 'tempest.api.identity.v3'
      gather_facts: true
      roles:
        - os_tempest

What are these above vars:
++++++++++++++++++++++++++

* `ansible_become: true`: os_tempest requires root permission for installation and creation of tempest related directories
* `tempest_run: 'yes'`: For running os_tempest role, by default, It is set to `no`.
* `tempest_install_method: 'distro'`: Set to `distro` for installing tempest and it's plugins from distro packages
* `tempest_workspace`: It is the full directory path where we want to create tempest workspace.
* `tempest_cloud_name: 'standalone'`: Name of the cloud name from clouds.yaml file for using to create tempest related resources on target host.
* `tempest_services`: For installing tempest plugins as well as creating pre tempest resources like networks for tempest tests.
* `tempest_public_net_physical_type`:
   The name of public physical network. For standalone tripleo deployment, it can found under `/var/lib/config-data/
   puppet-generated/neutron/etc/neutron/plugins/ml2/ml2_conf.ini` and then look for the value of `flat_networks`.
* `tempest_private_net_provider_type`:
   The Name of the private network provider type, in case of ovn deployment, it should be `geneve`.
   It can be found under `/var/lib/config-data/puppet-generated/neutron/etc/neutron/plugins/ml2/ml2_conf.ini` and then look for `type_drivers`.
* `tempest_service_setup_host`: It should be set to ansible inventory hostname. For some operation, the ansible role delegates to inventory hostname.
* `tempest_public_subnet_cidr`: Based on the standalone deployment IP, we need to pass a required cidr.
* `tempest_public_subnet_gateway_ip and tempest_public_subnet_allocation_pools`:
   Subnet Gateway IP and allocation pool can be calculated based on the value of `tempest_public_subnet_cidr` nthhost value.
* `tempest_use_tempestconf`: For generating tempest.conf, we use python-tempestconf tool. By default It is set to false. Set it to `true` for using it
* `tempest_run_stackviz`: Stackviz is very useful in CI for analyzing tempest results, for local use, we set it to false. By default it is set to true.
* `tempest_tempest_conf_overrides`: In order to pass additional tempest configuration to python-tempestconf tool, we can pass a dictionary of values.
* `tempest_test_whitelist`: We need to pass a list of tests which we wish to run on the target host as a list.
* `tempest_test_blacklist`: In order to skip tempest tests, we can pass the list here.
* `gather_facts`: We need to set gather_facts to true as os_tempest rely on targeted environment facts for installing stuff.


Here are the `defaults vars of os_tempest role <https://docs.openstack.org/openstack-ansible-os_tempest/latest/user/default.html>`_.

How to run it?
++++++++++++++
We can use `ansible-playbook` command to run the `tempest.yaml` playbook.

.. code-block:: shell

  $ ansible-playbook tempest.yaml

Once the playbook run finishes, we can find the tempest related directories in the tempest workspace.
within `tempest_workspace/etc/` dir, we can find following files:

* tempest.conf
* tempest_whitelist.txt
* tempest_blacklist.txt

within `/var/log/tempest` dir, we can find the tempest tests results in html format.

* stestr_results.html
* test_list.txt

Create your own os_tempest job
-------------------------------

We are going to use `tripleo-ci-centos-7-standalone-os-tempest` job, which
uses the role for validating the cloud.

Create a job definition in your `.zuul.yaml` file putting
`tripleo-ci-centos-7-standalone-os-tempest` as a parent of the job:

.. code-block:: yaml

    - job:
        name: our-tripleo-os-tempest-job
        parent: tripleo-ci-centos-7-standalone-os-tempest

.. note::

    More about Zuul job definitions can be found in
    `the official Zuul documentation <https://zuul-ci.org/docs/zuul/user/config.html>`_.

.. note::

    This page assumes that the reader is familiar with
    `TripleO CI jobs <https://docs.openstack.org/tripleo-docs/latest/ci/ci_primer.html>`_
    and with the procedures of
    `adding new TripleO jobs <https://docs.openstack.org/tripleo-docs/latest/ci/check_gates.html>`_.

By default, `tripleo-ci-centos-7-standalone-os-tempest` sets the following
variables for controlling behaviour of `os_tempest`:

.. code-block:: yaml

    vars:
        tempest_install_method: distro
        tempest_cloud_name: 'standalone'

It runs `tempest.yaml` playbook which sets the rest of the `os_tempest`
variables needed for execution on top of an environment deployed by one of the
TripleO CI jobs. The
`content of the playbook can be seen here <https://opendev.org/openstack/tripleo-quickstart-extras/src/branch/master/playbooks/tempest.yml>`_.

If you want to set some of the variables mentioned above differently you need
to override them by adding those variables to your job definition.

Let's say you would like to change `tempest_cloud_name` and
`tempest_public_net_physical_type`. After setting the variables your job
definition should look like:

.. code-block:: yaml

    - job:
        name: our-tripleo-os-tempest-job
        parent: tripleo-ci-centos-7-standalone-os-tempest
        vars:
            tempest_cloud_name: <your cloud name>
            tempest_public_net_physical_type: <your net type>

To see configuration options, please, follow
`this page <https://docs.openstack.org/openstack-ansible-os_tempest/latest/user/configuration.html>`_
of official documentation of `os_tempest` role.
