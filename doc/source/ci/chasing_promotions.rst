Chasing CI promotions
=====================

The purpose of this document is to go into more detail about the TripleO
promotion from the point of view of the ci-squad `ruck|rover`_.

There is other documentation in this repo which covers the stages of the
Tripleo-CI promotion pipeline in promotion-stages-overview_ and also about
relevant tooling such as the dlrn-api-promoter_.

Ensuring promotions are happening regularly (including for all current
stable/ branches) is one of the biggest responsibilities of the ruck|rover. As
explained in promotion-stages-overview_ the CI promotion represents the point
at which we test all the tripleo-* things against the rest of openstack. The
requirement is that there is a successful promotion (more on that below) at
least once a week. Otherwise the branch will be considered 'in the red' as in
"master promotion is red" or "we are red for stein promotion" meaning was no
promotion in (at least) 7 days for that branch.

Successful promotion
--------------------

So what does it actually mean to have a "successful promotion". In short:

  * The TripleO periodic jobs have to run to completion and
  * The periodic jobs in the promotion criteria must pass and
  * The promoter server must be running in order to actually notice
    the job results and promote!

Each of these is explained in more detail below.

TripleO periodic jobs
---------------------

The TripleO periodic jobs are `ci jobs`_ that are executed in one of the TripleO
periodic pipelines. At time of writing we have four periodic pipelines defined
in the `config repo zuul pipelines`_::

  * openstack-periodic-master
  * openstack-periodic-latest-released
  * openstack-periodic-24hr
  * openstack-periodic-wednesday-weekend

These pipelines are *periodic* because unlike the check and gate pipelines
(see `ci jobs`_ for more on those) jobs that run on each submitted code review,
periodic jobs are executed *periodically*, at an interval given in cron syntax
as you can see in the definitions at `config repo zuul pipelines`_)::

  - pipeline:
    name: openstack-periodic-master
    post-review: true
    description: Jobs in this queue are triggered to run every few hours.
    manager: independent
    precedence: high
    trigger:
      timer:
        - time: '10 0,12,18 * * *'

As can be seen at time of writing the openstack-periodic-master jobs
will run three times every day, at 10 minutes after midnight, noon and 6pm.

The four pipelines correspond to the four latest releases of OpenStack.
The openstack-periodic-master_ runs jobs for master promotion,
openstack-periodic-latest-released_ runs jobs for the latest stable branch
promotion, openstack-periodic-24hr_ runs jobs for the stable branch before that
and finally openstack-periodic-wednesday-weekend_ runs jobs for the stable
branch before that.

You can see the full list of jobs that are executed in the pipelines
in the `rdo-infra periodic zuul layout`_.

It is important to finally highlight a common pattern in the pipeline layout.
In each case the first job that must complete is the
'promote-consistent-to-tripleo-ci-testing' which is where we take the latest
consistent hash and mark it as tripleo-ci-testing to become our new candidate
(see promotion-stages-overview_) to be used by the rest of the jobs in our
pipeline. You will note that this is the only job that doesn't have any dependency::

        ...
        - periodic-tripleo-ci-rhel-8-ovb-3ctlr_1comp-featureset001-master:
            dependencies:
              - periodic-tripleo-rhel-8-buildimage-ironic-python-agent-master
              - periodic-tripleo-rhel-8-master-containers-build-push
        - periodic-tripleo-centos-7-master-promote-consistent-to-tripleo-ci-testing
        ...

Then the containers and overcloud image build jobs must complete and only then
we finally run the rest of the jobs. These ordering requirements are expressed
using dependencies in the layout::

        ...
        - periodic-tripleo-rhel-8-buildimage-overcloud-full-master:
            dependencies:
              - periodic-tripleo-centos-7-master-promote-consistent-to-tripleo-ci-testing
        - periodic-tripleo-rhel-8-buildimage-ironic-python-agent-master:
            dependencies:
              - periodic-tripleo-centos-7-master-promote-consistent-to-tripleo-ci-testing
        - periodic-tripleo-ci-centos-7-ovb-1ctlr_1comp-featureset002-master-upload:
            dependencies:
              - periodic-tripleo-centos-7-master-containers-build-push
        ..

As can be seen above the build image jobs depend on the promote-consistent job
and then everything else in the layout depends on the container build job.

Promotion Server and Criteria
-----------------------------

The promotion server is maintained by the Tripleo-CI squad at a secret location
(!) and it runs the code from the `DLRN API Promoter`_ as a service. In short,
the job of this service is to fetch the latest hashes from the `RDO delorean
service`_ and then query the state of the periodic jobs using that particular
hash.

The main input to the promotion server is the configuration which defines
the `promotion criteria`_. This is the list of jobs that must pass so that we
can declare a successful promotion::

        [current-tripleo]
        periodic-tripleo-centos-7-master-containers-build-push
        periodic-tripleo-ci-centos-7-ovb-3ctlr_1comp-featureset001-master
        periodic-tripleo-ci-centos-7-ovb-1ctlr_1comp-featureset002-master-upload
        periodic-tripleo-ci-centos-7-multinode-1ctlr-featureset010-master
        periodic-tripleo-ci-centos-7-scenario001-standalone-master
        periodic-tripleo-ci-centos-7-scenario002-standalone-master
        periodic-tripleo-ci-centos-7-scenario003-standalone-master
        ...

The promoter service queries the delorean service for the results of those
jobs (for a given hash) and if they are all found to be in SUCCESS then the
hash can be promoted to become the new current-tripleo_.

It is a common practice for TripleO CI ruck or rover to check the
`indexed promoter service logs`_ to see why a given promotion is not successful
for example or when debugging issues with the promotion code itself.

Hack the promotion with testproject
-----------------------------------

Finally testproject_ and the ability to run individual periodic jobs on
demand is an important part of the ruck|rover toolbox. In some cases you may
want to run a job for verification of a given launchpad bug that affects
perioric jobs.

However another important use is when the ruck|rover notice that one of the
jobs in criteria failed on something they (now) know how to fix, or on some
unrelated/transient issue. Instead of waiting another 6 or however many hours
for the next periodic to run, you can try to run the job yourself using
testproject. If the job is successful in testproject and
it is the only job missing from criteria then posting the testproject review
can also mean directly causing the promotion to happen.

You first need to checkout testproject::

    git clone https://review.rdoproject.org/r/testproject
    cd testproject
    vim .zuul.layout

To post a testproject review you simply need to add a .zuul.layout_ file::

    - project:
        check:
          jobs:
            - periodic-tripleo-centos-7-train-containers-build-push:
                vars:
                  force_periodic: true

So the above would run the periodic-tripleo-centos-7-train-containers-build-push.
Note the required *force_periodic* variable which causes the job to run as
though it is in the periodic pipeline, rather than in the check pipeline which
you will use in testproject.

An `example is there`_ and if you need to include a known fix you can simply
have a Depends-On in the commit message.

Specifying a particular hash
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Jobs in the periodic promotion pipelines are using the tripleo-ci-testing
repo as described in the promotion-stages-overview_, since that is the candidate
we are trying to promote to current-tripleo. The tripleo-ci-testing and all
other named tags in tripleo, are associated with a particular *hash* that
identifies the delorean repo. For example looking at `centos7 master tripleo-ci-testing`_
at time of writing we see::

    [delorean]
    name=delorean-tripleo-ansible-544864ccc03b053317f5408b0c0349a42723ce73
    baseurl=https://trunk.rdoproject.org/centos7/54/48/544864ccc03b053317f5408b0c0349a42723ce73_ebb98bd9
    enabled=1
    gpgcheck=0
    priority=1

So the centos7 master tripleo-ci-testing *hash* is
*544864ccc03b053317f5408b0c0349a42723ce73_ebb98bd9a*. The corrresponding repo
is given by the baseurl above and if you navigate to that URL with your
browser you can see the list of packages used in the jobs. Thus, the job
specified in the example above for testproject
*periodic-tripleo-centos-7-train-containers-build-push* would use whatever
the current tripleo-ci-testing points to.

However it is possible to override the particular hash (and thus repo) used by
a job you run with testproject, using the dlrn_hash_tag featureset_override::

    - project:
        check:
          jobs:
            - periodic-tripleo-ci-centos-7-ovb-1ctlr_1comp-featureset002-train-upload:
                vars:
                  force_periodic: true
                  featureset_override:
                    dlrn_hash_tag: 4b32d316befe0919fd98a147d84086bc0907677a_046903a2

Thus, in the example above the periodic-tripleo-ci-centos-7-ovb-1ctlr_1comp-featureset002-train-upload
job would run with the hash: *4b32d316befe0919fd98a147d84086bc0907677a_046903a2*
regardless of the current value of tripleo-ci-testing.

The most common reason for overriding the hash in this way is when we notice
that a particular job failed during one of the recent periodic pipeline runs.
Looking at one of the `indexed promoter service logs`_ you may notice something
like the following text::

    2020-02-21 03:57:07,458 31360 INFO promoter Skipping promotion of centos7-master
    {'timestamp': 1582243926, 'distro_hash': 'ebb98bd9545e026f033683143ae39e9e236b3671',
    'promote_name': 'tripleo-ci-testing', 'user': 'review_rdoproject_org',
    'repo_url': 'https://trunk.rdoproject.org/centos7/54/48/544864ccc03b053317f5408b0c0349a42723ce73_ebb98bd9',
    'full_hash': '544864ccc03b053317f5408b0c0349a42723ce73_ebb98bd9',
    'repo_hash': '544864ccc03b053317f5408b0c0349a42723ce73_ebb98bd9',
    'commit_hash': '544864ccc03b053317f5408b0c0349a42723ce73'}
    from tripleo-ci-testing to current-tripleo,
    missing successful jobs: [u'periodic-tripleo-ci-centos-7-ovb-3ctlr_1comp-featureset035-master',
    u'periodic-tripleo-ci-centos-7-ovb-3ctlr_1comp-featureset001-master']

In particular note the last line 'missing successful jobs'. This means that
for the hash *544864ccc03b053317f5408b0c0349a42723ce73_ebb98bd9* a promotion
could not happen, because in this particular run, those two identified jobs
were failed. If the jobs were fixed in the meantime or you now know
how to fix them and get a good result, you could re-run those with testproject
specifying the particular hash. If they execute successfully then on the next
run the promoter will promote that hash to become the new current-tripleo.


.. _promotion-stages-overview: stages-overview.html
.. _dlrn-api-promoter: dlrn-promoter-overview.html
.. _`ruck|rover`: ruck_rover_primer.html
.. _`ci jobs`: https://docs.openstack.org/tripleo-docs/latest/ci/ci_primer.html#where-do-tripleo-promotion-jobs-live
.. _`config repo zuul pipelines`: https://github.com/rdo-infra/review.rdoproject.org-config/blob/0fd16d0badb13e02460d3b2e3213db4af7f027e0/zuul.d/upstream.yaml#L84-L157
.. _openstack-periodic-master: https://review.rdoproject.org/zuul/builds?pipeline=openstack-periodic-master
.. _openstack-periodic-latest-released: https://review.rdoproject.org/zuul/builds?pipeline=openstack-periodic-latest-released
.. _openstack-periodic-24hr: https://review.rdoproject.org/zuul/builds?pipeline=openstack-periodic-24hr
.. _openstack-periodic-wednesday-weekend: https://review.rdoproject.org/zuul/builds?pipeline=openstack-periodic-wednesday-weekend
.. _`rdo-infra periodic zuul layout`: https://github.com/rdo-infra/review.rdoproject.org-config/blob/0fd16d0badb13e02460d3b2e3213db4af7f027e0/zuul.d/tripleo.yaml#L74-L424
.. _`DLRN API Promoter`: https://github.com/rdo-infra/ci-config/blob/master/ci-scripts/dlrnapi_promoter/README.md
.. _`RDO delorean service`: https://trunk.rdoproject.org/centos7-master-head/report.html
.. _`promotion criteria`: https://github.com/rdo-infra/ci-config/blob/4bc3261c4ce644829a317c1bd85c1d645cb96cbd/ci-scripts/dlrnapi_promoter/config/CentOS-7/master.ini#L16
.. _current-tripleo: https://trunk.rdoproject.org/centos7-master/current-tripleo/delorean.repo
.. _testproject: https://review.rdoproject.org/r/#/q/project:testproject
.. _`example is there`: https://review.rdoproject.org/r/#/c/23502/
.. _`indexed promoter service logs`: http://promoter.rdoproject.org/
.. _`centos7 master tripleo-ci-testing`: https://trunk.rdoproject.org/centos7-master/tripleo-ci-testing/delorean.repo
