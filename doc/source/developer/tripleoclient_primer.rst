Primer python-tripleoclient and tripleo-common
==============================================

This document gives an overview of how python-tripleoclient_ provides the
cli interface for TripleO. In particular it focuses on two key aspects of
TripleO commands: where they are defined and how they (very basically) work.

Whilst python-tripleoclient provides the CLI for TripleO, it is in
tripleo-common_ that the logic behind a given command resides. So interfacing
with OpenStack services such as Heat, Nova or Mistral typically happens in
tripleo-common.

For this primer we will use a specific example command but the same applies to
any TripleO cli command to be found in the TripleO documentation or in any
local deployment (or even in TripleO CI) logfiles.

The example used here is::

    openstack overcloud container image build

This command is used to build the container images listed in the
tripleo-common file overcloud_containers.yaml_ using Kolla_.

See the `Building Containers Deploy Guide <building_containers_deploy_guide_>`_ for more information on
how to use this command as an operator.

.. _building_containers_deploy_guide: https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/3rd_party.html

One of the TripleO CI jobs that executes this command is the
tripleo-build-containers-centos-7_ job. This job invokes the overcloud container
image build command in the build.sh.j2_ template::

    openstack overcloud container image build \
    --config-file $TRIPLEO_COMMON_PATH/container-images/overcloud_containers.yaml \
    --kolla-config-file {{ workspace }}/kolla-build.conf \

The relevance of showing this is simply to serve as an example in the following
sections. First we see how to identify *where* in the tripleoclient code a given
command is defined, and then *how* the command works, highlighting a recurring
pattern common to all TripleO commands.

.. _python-tripleoclient: https://opendev.org/openstack/python-tripleoclient/
.. _tripleo-common: https://opendev.org/openstack/tripleo-common/
.. _overcloud_containers.yaml: https://opendev.org/openstack/tripleo-common/src/branch/master/container-images/overcloud_containers.yaml?id=827af753884e15326863ff2207b2ac95d4ad595b#n1
.. _Kolla: https://opendev.org/openstack/kolla
.. _tripleo-build-containers-centos-7: http://zuul.opendev.org/builds?job_name=tripleo-build-containers-centos-7
.. _build.sh.j2: https://opendev.org/openstack-infra/tripleo-ci/src/branch/master/playbooks/tripleo-buildcontainers/templates/build.sh.j2?id=69212e1cd8726396c232b493f1aec79480459666#n5
.. _setup.cfg: https://opendev.org/openstack/python-tripleoclient/src/branch/master/setup.cfg?id=73cc43898cfcc8b99ce736f734fc5b514f5bc6e9#n46


TripleO commands: *where*
-------------------------

Luckily the location of all TripleO commands is given in the list of
``entry_points`` in the python-tripleoclient_ setup.cfg_ file. Each *key=value*
pair has a key derived from the TripleO command. Taking the command, omit
the initial *openstack* and link subcommands with underscore instead of
whitespace. That is, for the
**openstack overcloud container image build** command the equivalent entry is
**overcloud_container_image_build**::

    [entry_points]
    openstack.cli.extension =
        tripleoclient = tripleoclient.plugin

    openstack.tripleoclient.v1 =
    ...
        overcloud_container_image_build = tripleoclient.v1.container_image:BuildImage

The value in each *key=value* pair provides us with the file and class name
used in the tripleoclient namespace for this command. For **overcloud_container_image_build** we have
**tripleoclient.v1.container_image:BuildImage**, which means this command is
defined in a class called **BuildImage** inside the `tripleoclient/v1/container_image.py`_
file.

.. _`tripleoclient/v1/container_image.py`: https://opendev.org/openstack/python-tripleoclient/src/branch/master/tripleoclient/v1/container_image.py?id=0132e7d08240d8a9d5839cc4345574d44ec2b278#n100

TripleO commands: *how*
-----------------------

Obviously each TripleO command 'works' differently in that they are doing
different things - deploy vs upgrade the undercloud vs overcloud etc.
However there **is** at least one commonality which we highlight in this section.
Each TripleO command class defines a get_parser_ function and a take_action_
function.

The get_parser_ is where all command line arguments are defined and
take_action_ is where tripleo-common is invoked to perform the task at hand,
building container images in this case.

Looking inside the **BuildImage** class we find::

    def get_parser(self, prog_name):
    ...
        parser.add_argument(
            "--config-file",
            dest="config_files",
            metavar='<yaml config file>',
            default=[],
            action="append",
            help=_("YAML config file specifying the images to build. May be "
                   "specified multiple times. Order is preserved, and later "
                   "files will override some options in previous files. "
                   "Other options will append. If not specified, the default "
                   "set of containers will be built."),
        )
        parser.add_argument(
            "--kolla-config-file",

Here we can see where the two arguments shown in the introduction above are
defined: **--config-file** and **--kolla-config-file**. You can see the default
values and all other attributes for each of the command parameters there.

Finally we can look for the take_action_ function to learn more about how the
command actually 'works'. Typically the take_action function will have some
validation of the provided arguments before calling out to tripleo-common to
actually 'do' the work (build container images in this case)::

    from tripleo_common.image import kolla_builder
    ...
    def take_action(self, parsed_args):
    ...
        try:
            builder = kolla_builder.KollaImageBuilder(parsed_args.config_files)
            result = builder.build_images(kolla_config_files,

Here we can see the actual image build is done by the **kolla_builder.KollaImageBuilder**
class **build_images** function. Looking in tripleo-common we can follow that
python namespace to find the definition of **build_images** in the
`tripleo_common/image/kolla_builder.py`_ file::

    def build_images(self, kolla_config_files=None, excludes=[],
                 template_only=False, kolla_tmp_dir=None):
        cmd = ['kolla-build']
    ...

.. _get_parser: https://opendev.org/openstack/python-tripleoclient/src/branch/master/tripleoclient/v1/container_image.py?id=0132e7d08240d8a9d5839cc4345574d44ec2b278#n119
.. _take_action:  https://opendev.org/openstack/python-tripleoclient/src/branch/master/tripleoclient/v1/container_image.py?id=0132e7d08240d8a9d5839cc4345574d44ec2b278#n184
.. _`tripleo_common/image/kolla_builder.py`: https://opendev.org/openstack/tripleo-common/src/branch/master/tripleo_common/image/kolla_builder.py?id=3db41939a370ef3bbd2c6b60ca24e6e8e4b6e30a#n441
