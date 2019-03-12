os_tempest
==========

TripleO CI group collaborates on `os_tempest` development. To see what
`os_tempest` is and the reasons why it was started have a look
`at this page <https://docs.openstack.org/openstack-ansible-os_tempest/latest/overview.html>`_.

At this moment we run `os_tempest` in `tripleo-ci-centos-7-standalone-os-tempest`
job where it is used for cloud validation.

To find out more about `os_tempest` default variables follow to its
`official documentation <https://docs.openstack.org/openstack-ansible-os_tempest/latest/user/default.html>`_


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

    This page assumes that the reader is familier with
    `TripleO CI jobs <https://docs.openstack.org/tripleo-docs/latest/ci/ci_primer.html>`_
    and with the procedures of
    `adding new TripleO jobs <https://docs.openstack.org/tripleo-docs/latest/ci/check_gates.html>`_.

By default, `tripleo-ci-centos-7-standalone-os-tempest` sets the following
variables for controlling behaviour of `os_tempest`:

.. code-block:: yaml

    vars:
        tempest_install_method: distro
        tempest_cloud_name: 'standalone'

and runs `tempest.yaml` playbook which sets the rest of the `os_tempest`
variables needed for execution on top of an environment deployed by one of the
TripleO CI jobs. The
`content of the playbook can be seen here <https://git.openstack.org/cgit/openstack/tripleo-quickstart-extras/tree/playbooks/tempest.yml>`_.

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
