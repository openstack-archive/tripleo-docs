tripleo.sh
==========

`tripleo.sh
<http://git.openstack.org/cgit/openstack/tripleo-common/tree/scripts/tripleo.sh>`_
is a script that can be used to help bootstrap development environments. It
automates many of the steps in this documentation to help get setup faster.
It's opinionated automation around other production tooling
(`python-tripleoclient
<http://git.openstack.org/cgit/openstack/python-tripleoclient>`_ , etc).
tripleo.sh is also used by tripleo-ci to test TripleO patches.

Get tripleo.sh
--------------

tripleo.sh is from the `tripleo-common
<http://git.openstack.org/cgit/openstack/tripleo-common>`_ project. git clone
tripleo-common, and the script is under the scripts/ directory::

  git clone https://git.openstack.org/openstack/tripleo-common
  tripleo-common/scripts/tripleo.sh --help


Using tripleo.sh
----------------

The tripleo.sh script is intended to run on an new instack undercloud setup.
That is, you would follow the `environment setup <http://docs.openstack.org/developer/tripleo-docs/environments/environments.html#environment-setup>`_ docs through to and including
`instack-virt-setup` (for a virt setup), ssh onto the resulting undercloud
node and then run tripleo.sh with the options identified below.

Options
^^^^^^^

The help text shows what options are available, and the options are listed in
corresponding order with how a TripleO deployment is done.

Repository setup::

  tripleo-common/scripts/tripleo.sh --repo-setup

Installing the undercloud::

  tripleo-common/scripts/tripleo.sh --undercloud

Building overcloud images::

  tripleo-common/scripts/tripleo.sh --overcloud-images

Registering nodes::

  tripleo-common/scripts/tripleo.sh --register-nodes

Introspect nodes::

  tripleo-common/scripts/tripleo.sh --introspect-nodes

Deploy overcloud::

  tripleo-common/scripts/tripleo.sh --overcloud-deploy

Alternatively, all of the above options can be execute at once with::

  tripleo-common/scripts/tripleo.sh --all

Environment variables
^^^^^^^^^^^^^^^^^^^^^

Certain values and assumptions can be changed via environment variables. See
the `tripleo.sh
<http://git.openstack.org/cgit/openstack/tripleo-common/tree/scripts/tripleo.sh>`_
source code for details.
