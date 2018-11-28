.. TODO: This is a template which is being
   completed. The subsections stated
   here might differ from the ones in the
   final version.

Major upgrades & Minor updates CI coverage
------------------------------------------

.. include:: links.rst

This document tries to give a detailed overview of the current
CI coverage for upgrades/updates jobs. Also, it is intended as
a guideline to understand how these jobs work, as well as giving
some tips for debugging.

Upgrades/Updates CI jobs
~~~~~~~~~~~~~~~~~~~~~~~~~

At the moment most of the upgrade jobs have been moved from upstream
infrastructure to `RDO Software Factory job definition`_ due to
runtime constraints of the OpenStack infra jobs.

Each of these jobs are defined by a `featureset file`_ and a `scenario file`_. The
featureset used in a job can be found in the last part of the job type value.
This can be found in the ci job definition::

 - '{trigger}-tripleo-ci-{jobname}-{release}{suffix}':
    jobname: 'centos-7-containers-multinode-upgrades'
    release:
      - pike
      - master
    suffix: ''
    type: 'multinode-1ctlr-featureset011'
    node: upstream-centos-7-2-node
    trigger: gate

The scenario used is referenced in the featureset file, in the example above
the `featureset011`_ makes use of the following scenarios::

  composable_scenario: multinode.yaml
  upgrade_composable_scenario: multinode-containers.yaml

As this job covers the upgrade from one release to another, we need to
specify two scenario files. The one used during deployment and the one
used when upgrading. Each of these scenario files defines the services
deployed in the nodes.

.. note::
  There is a matrix with the different features deployed per feature set
  here: `featureset matrix`_

Currently, two types of upgrade jobs exist:

- multinode-upgrade (mixed-version): In this job, an undercloud with
  release N+1 is deployed, while the overcloud is deployed with a N
  release. Execution time is reduced by not upgrading the undercloud
  , instead the heat templates from the (N+1) undercloud are used when
  performing the overcloud upgrade.

    .. note::
      If you want your patch to be tested against this job you need
      to add *RDO Third Party CI* as reviewer or reply with the comment
      *check-rdo experimental*.

- undercloud-upgrade: This job tests the undercloud upgrade from a
  major release to another. The undercloud is deployed with release
  N and upgraded to N+1 release. This job does not deploy an overcloud.

.. note::
  There is an effort to `integrate`_ the new `tripleo-upgrade`_ role into
  tripleo-quickstart that defines an unified way to upgrade and update.

Upgrade/Update CI jobs, where to look
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The best place to check the current CI jobs status is in the `CI Status`_
page. This webpage contains a log of all the TripleO CI jobs, it's result
status, link to logs, git patch trigger and statistics about the pass/fail
rates.

To check the status of the Upgrades/Updates jobs, you need to click the
`TripleO CI promotion jobs`_ link from `CI Status`_, where you will find
the RDO cloud upgrades section:

.. image:: rdo_upgrades_jobs.png

In this section the CI jobs have a color code, to show its
current status in a glance::

 - Red: CI job constantly failing.
 - Yellow: Unstable job, frequent failures.
 - Green: CI job passing consistently.

If you scroll down after pressing some of the jobs in the section
you will find the CI job statistics and the last 100 (or less, it
can be edited) job executions. Each of the job executions contains::

  - Date: Time and date the CI job was triggered
  - Length: Job duration
  - Reason: CI job result or failure reason.
  - Patch: Git ref of the patch tha triggered the job.
  - Logs: Link to the logs.
  - Branch: Release branch used to run the job.


Debugging Upgrade/Update CI jobs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When opening the logs from a CI job it might look a little chaotic
(mainly when it is for the first time). It's good to have an idea
where you can find the logs you need, so you will be able to identify
the cause of a failure or debug some issue.

.. _logs directory:

The first thing to have a look at when debugging a CI job is the
console output or full log. When clicking in the job, the following
folder structure appears::

  job-output.json.gz
  job-output.txt.gz
  logs/
  zuul-info/

The job execution log is located in the *job-output.txt.gz* file. Once
opened, a huge log will appear in front of you. What should you look
for?

(1) Find the job result

  A good string to search is *PLAY RECAP*. At this point, all the
  playbooks have been executed and a summary of the runs per node
  is displayed::

    PLAY RECAP *********************************************************************
    127.0.0.2                  : ok=9    changed=0    unreachable=0    failed=0
    localhost                  : ok=10   changed=3    unreachable=0    failed=0
    subnode-2                  : ok=3    changed=1    unreachable=0    failed=0
    undercloud                 : ok=120  changed=78   unreachable=0    failed=1

  In this case, one of the playbooks executed in the undercloud has
  failed. To identify which one, we can look for the string **fatal**.::

    fatal: [undercloud]: FAILED! => {"changed": true, "cmd": "set -o pipefail && /home/zuul/overcloud-upgrade.sh 2>&1
    | awk '{ print strftime(\"%Y-%m-%d %H:%M:%S |\"), $0; fflush(); }' > overcloud_upgrade_console.log",
    "delta": "0:00:39.175219", "end": "2017-11-14 16:55:47.124998", "failed": true, "rc": 1,
    "start": "2017-11-14 16:55:07.949779", "stderr": "", "stdout": "", "stdout_lines": [], "warnings": []}

  From this task, we can guess that something went wrong during the
  overcloud upgrading process. But, where can I find the log
  *overcloud_upgrade_console.log* referenced in the task?

(2) Undercloud logs

  From the `logs directory`_ , you need to open the *logs/*
  folder. All undercloud logs are located inside the *undercloud/*
  folder. Opening it will display the following::

    etc/      *configuration files*
    home/     *job execution logs from the playbooks*
    var/      *system/services logs*

  The log we look for is located in */home/zuul/*. Most of the tasks
  executed in tripleo-quickstart will store the full script as well as
  the execution log in this directory. So, this is a good place to
  have a better understanding of what went wrong.

  If the overcloud deployment or upgrade failed, you will also find
  two log files named::

    failed_upgrade.log.txt.gz
    failed_upgrade_list.log.txt.gz

  The first one stores the output from the debugging command::

    openstack stack failures list --long overcloud

  Which prints out the reason why the deployment or upgrade
  failed. Although sometimes, this information is not enough
  to find the root cause for the problem. The *stack failures*
  can give you a clue of which service is causing the problem,
  but then you'll need to investigate the OpenStack service logs.

(3) Overcloud logs

  From the *logs/* folder, you can find a folder named *subnode-2*
  which contains most of the overcloud logs.::

   apache/
   ceph_conf.txt.gz
   deprecations.txt.gz
   devstack.journal.gz
   df.txt.gz
   etc/
   home/
   iptables.txt.gz
   libvirt/
   listen53.txt.gz
   openvswitch/
   pip2-freeze.txt.gz
   ps.txt.gz
   resolv_conf.txt.gz
   rpm-qa.txt.gz
   sudoers.d/
   var/

  To access the OpenStack services logs, you need to go to
  *subnode-2/var/log/* when deploying a baremetal overcloud. If the
  overcloud is containerized, the service logs are stored under
  *subnode-2/var/log/containers*.


Replicating CI jobs
~~~~~~~~~~~~~~~~~~~

Thanks to `James Slagle`_ there is now a way to reproduce TripleO CI jobs in
any OpenStack cloud. Everything is enabled by the `traas`_ project,
a set of Heat templates and scripts that reproduce the TripleO CI jobs
in the same way they are being run in the Zuul gate.

When cloning the repo, you just need to set some configuration parameters. A
set of sample templates have been located under
`templates/example-environments`_. The parameters defined in this
template are::

  parameters:
    overcloud_flavor:    [*flavor used for the overcloud instance*]
    overcloud_image:     [*overcloud OS image (available in cloud images)*]
    key_name:            [*private key used to access cloud instances*]
    private_net:         [*network name (it must exist and match)*]
    overcloud_node_count:[*number of overcloud nodes*]
    public_net:          [*public net in CIDR notation*]
    undercloud_image:    [*undercloud OS image (available in cloud images)*]
    undercloud_flavor:   [*flavor used for the undercloud instance*]
    toci_jobtype:        [*CI job type*]
    zuul_changes:        [*List of patches to retrieve*]

.. note:: The CI job type toci_jobtype can be found in the job definition
          under `tripleo-ci/zuul.d`_.

A good example to deploy a multinode job in RDO Cloud is this
`sample template`_. You can test your out patches by appending
the refs patch linked with the ^ character::

    zuul_changes: <project-name>:<branch>:<ref>[^<project-name>:<branch>:<ref>]*

This allows you also to test any patch in a local environment without
consuming CI resources. Or when you want to debug an environment after
a job execution.

Once the template parameters are defined, you just need to create the stack.
If we would like to deploy the *rdo-cloud-env-config-download.yaml*
`sample template`_ we would need to run::

    cd traas/
    openstack stack create traas -t templates/traas.yaml \
        -e templates/traas-resource-registry.yaml \
        -e templates/example-environments/rdo-cloud-env-config-download.yaml

This stack will create two instances in your cloud tenant, one for undercloud
and another for the overcloud. Once created, the stack will directly call
the `traas/scripts/traas.sh`_ script which downloads all required repositories
to start executing the job.

If you want to follow up the job execution, you can ssh to the undercloud
instance and tail the content from the *$HOME/tripleo-root/traas.log*. All
the execution will be logged in that file.
