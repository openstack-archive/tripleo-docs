emit-releases-file and releases.sh
==================================

The emit-releases-file tool is a python script that lives in the tripleo-ci
repo under the `scripts/emit_releases_file`_ directory. This script produces
an output file called `releases.sh` containing shell variable export commands.
These shell variables set the release **name** and **hash** for the
installation and target (versions) of a given job. For example, installing
latest stable branch (currently stein) and upgrading to master. The **hash**
is the delorean repo hash from which the packages used in the job are to be
installed.

The contents of `releases.sh` will differ depending on the type of upgrade or
update operation being performed by a given job and this is ultimately
determined by the featureset. Each upgrade or update related featureset sets
boolean variables that signal the type of upgrade performed. For example
featureset050_ is used for undercloud upgrade and it sets::

  undercloud_upgrade: true

The `releases.sh` for an undercloud upgrade job looks like::

  #!/bin/env bash
  export UNDERCLOUD_INSTALL_RELEASE="stein"
  export UNDERCLOUD_INSTALL_HASH="c5b283cab4999921135b3815cd4e051b43999bce_5b53d5ba"
  export UNDERCLOUD_TARGET_RELEASE="master"
  export UNDERCLOUD_TARGET_HASH="be90d93c3c5f77f428d12a9a8a2ef97b9dada8f3_5b53d5ba"
  export OVERCLOUD_DEPLOY_RELEASE="master"
  export OVERCLOUD_DEPLOY_HASH="be90d93c3c5f77f428d12a9a8a2ef97b9dada8f3_5b53d5ba"
  export OVERCLOUD_TARGET_RELEASE="master"
  export OVERCLOUD_TARGET_HASH="be90d93c3c5f77f428d12a9a8a2ef97b9dada8f3_5b53d5ba"
  export STANDALONE_DEPLOY_RELEASE="master"
  export STANDALONE_DEPLOY_HASH="be90d93c3c5f77f428d12a9a8a2ef97b9dada8f3_5b53d5ba"
  export STANDALONE_DEPLOY_NEWEST_HASH="b4c2270cc6bec2aaa3018e55173017c6428237a5_3eee5076"
  export STANDALONE_TARGET_RELEASE="master"
  export STANDALONE_TARGET_NEWEST_HASH="b4c2270cc6bec2aaa3018e55173017c6428237a5_3eee5076"
  export STANDALONE_TARGET_HASH="be90d93c3c5f77f428d12a9a8a2ef97b9dada8f3_5b53d5ba"

As can be seen there are three different groups of keys set:
`UNDERCLOUD_INSTALL` and `UNDERCLOUD_TARGET` is one group, then
`OVERCLOUD_DEPLOY` and `OVERCLOUD_TARGET`, and finally `STANDALONE_DEPLOY` and
`STANDALONE_TARGET`. For each of those groups we have the `_RELEASE` name and
delorean `_HASH`. Since the example above is generated from an undercloud
upgrade job/featureset only the undercloud related values are set correctly.
The values for `OVERCLOUD_` and `STANDALONE_` are set to the default values
with both `_DEPLOY` and `_TARGET` referring to `master`.

Where is releases.sh used
-------------------------

The releases script is not used for all CI jobs or even for all upgrades
related jobs. There is a conditional in the
`tripleo-ci run-test role which determines`_
the list of jobs for which we `use emit-releases-file`. In future we may remove
this conditional altogether.

Once it is determined that the releases.sh file will be used, a list of extra
`RELEASE_ARGS is compiled`_ to be passed into the subsequent
`quickstart playbook invocations`_. An example of what these `RELEASE_ARGS`
looks like is::

  --extra-vars @/home/zuul/workspace/.quickstart/config/release/tripleo-ci/CentOS-7/master.yml -e dlrn_hash=be90d93c3c5f77f428d12a9a8a2ef97b9dada8f3_5b53d5ba -e get_build_command=be90d93c3c5f77f428d12a9a8a2ef97b9dada8f3_5b53d5ba

The `RELEASE_ARGS` are resolved by a helper function
get_extra_vars_from_release_. As you can see this function uses the release
name passed in via the `_RELEASE` value from the `releases.sh` to set the right
release configuration file from the tripleo-quickstart `config/release/`_
directory which sets variables for the ansible execution. It also sets the
`dlrn_hash` which is used to setup the right repo and thus versions of packages
and finally the get_build_command is used to make sure we have the right
containers for the job.

As you can see in the list of compiled `RELEASE_ARGS` the `INSTALL` or `TARGET`
are passed in to the get_extra_vars_from_release function, depending on the
playbook::

    declare -A RELEASE_ARGS=(
        ["multinode-undercloud.yml"]=$(get_extra_vars_from_release \
            $UNDERCLOUD_INSTALL_RELEASE $UNDERCLOUD_INSTALL_HASH)
        ["multinode-undercloud-upgrade.yml"]=$(get_extra_vars_from_release \
            $UNDERCLOUD_TARGET_RELEASE $UNDERCLOUD_TARGET_HASH)

So for the multinode-undercloud.yml use INSTALL_RELEASE but for
multinode-undercloud-upgrade.yml use TARGET_RELEASE and HASH.

.. _`scripts/emit_releases_file`: https://opendev.org/openstack/tripleo-ci/src/commit/91c836da76f6f28a5c7545b6a96bf6a9c0d2289e/scripts/emit_releases_file
.. _featureset050: https://opendev.org/openstack/tripleo-quickstart/src/commit/b90b5a51df5104da35adf42a7d7fb5f7bc603eca/config/general_config/featureset050.yml#L18
.. _releases_jobs: https://opendev.org/openstack/tripleo-ci/src/commit/91c836da76f6f28a5c7545b6a96bf6a9c0d2289e/roles/run-test/templates/toci_gate_test.sh.j2#L120
.. _`tripleo-ci run-test role which determines`: https://opendev.org/openstack/tripleo-ci/src/commit/93768b46eec9cf3767fef23c186806e660f69395/roles/run-test/templates/toci_gate_test.sh.j2#L124
.. _get_extra_vars_from_release: https://opendev.org/openstack/tripleo-ci/src/commit/91c836da76f6f28a5c7545b6a96bf6a9c0d2289e/roles/run-test/templates/oooq_common_functions.sh.j2#L155
.. _`RELEASE_ARGS is compiled`: https://opendev.org/openstack/tripleo-ci/src/commit/91c836da76f6f28a5c7545b6a96bf6a9c0d2289e/roles/run-test/templates/toci_quickstart.sh.j2#L66
.. _`quickstart playbook invocations`: https://opendev.org/openstack/tripleo-ci/src/commit/91c836da76f6f28a5c7545b6a96bf6a9c0d2289e/roles/run-test/templates/toci_quickstart.sh.j2#L130
.. _`config/release/`: https://opendev.org/openstack/tripleo-quickstart/src/commit/b90b5a51df5104da35adf42a7d7fb5f7bc603eca/config/release
