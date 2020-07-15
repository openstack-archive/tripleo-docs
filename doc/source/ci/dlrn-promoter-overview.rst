How the TripleO-RDO Pipelines' Promotions Work
==============================================

Building consumable RDO repos and images involves various stages.
Each stage takes inputs and outputs artifacts. This document explains the
stages comprising the promotion pipelines, and the tools used to create
and manage the resulting artifacts.

What is DLRN?
-------------

DLRN is a tool to build RPM packages from each commit to a set of
OpenStack-related git repositories that are included in RDO.
DLRN builds are run through CI and to detect packaging issues with the
upstream branches of these Openstack projects.

DLRN Artifacts - Hashes and Repos
---------------------------------

When a DLRN build completes, it produces a new hash and related repo version.
For example, the Pike builds on CentOS are available at:
https://trunk.rdoproject.org/centos7-pike/.
The builds are placed in directories by DLRN hash. Each directory contains
the RPMs as well as a repo file
https://trunk.rdoproject.org/centos7-pike/current-tripleo/delorean.repo
and a ``commit.yaml`` file
https://trunk.rdoproject.org/centos7-pike/current-tripleo/commit.yaml.

There are some standard links that are updated as the builds complete and pass
stages of CI. Examples are these links are:

- https://trunk.rdoproject.org/centos7-pike/current/
- https://trunk.rdoproject.org/centos7-pike/consistent/
- https://trunk.rdoproject.org/centos7-pike/current-tripleo/
- https://trunk.rdoproject.org/centos7-pike/current-tripleo-rdo/
- https://trunk.rdoproject.org/centos7-pike/current-tripleo-rdo-internal/
- https://trunk.rdoproject.org/centos7-pike/tripleo-ci-testing/

The above links will be referenced in the sections below.

Promoting through the Stages - DLRN API
---------------------------------------

DLRN API Client
```````````````

`The DLRN API
<https://github.com/softwarefactory-project/DLRN/blob/master/doc/source/api.rst>`_
`client <https://github.com/softwarefactory-project/dlrnapi_client/>`_
enables users to query repo status, upload new hashes and create promotions.
Calls to the dlrnapi_client retrieve the inputs to stages and upload artifacts
after stages.

For example:

::

    $ dlrnapi --url https://trunk.rdoproject.org/api-centos-master-uc \
      promotion-get --promote-name tripleo-ci-testing

    [{'commit_hash': 'ec650aa2c8ce952e4a33651190301494178ac562',
     'distro_hash': '9a7acc684265872ff288a11610614c3b5739939b',
     'promote_name': 'tripleo-ci-testing',
     'timestamp': 1506427440},
     {'commit_hash': 'ec650aa2c8ce952e4a33651190301494178ac562',
    [..]


    $ dlrnapi --url https://trunk.rdoproject.org/api-centos-master-uc \
      repo-status --commit-hash ec650aa2c8ce952e4a33651190301494178ac562 \
     --distro-hash 9a7acc684265872ff288a11610614c3b5739939b

    [{'commit_hash': 'ec650aa2c8ce952e4a33651190301494178ac562',
     'distro_hash': '9a7acc684265872ff288a11610614c3b5739939b',
     'in_progress': False,
     'job_id': 'consistent',
     'notes': '',
     'success': True,
     'timestamp': 1506409403,
     'url': ''},
     {'commit_hash': 'ec650aa2c8ce952e4a33651190301494178ac562',
     'distro_hash': '9a7acc684265872ff288a11610614c3b5739939b',
     'in_progress': False,
     'job_id': 'periodic-singlenode-featureset023',
     'notes': '',
     'success': True,
     'timestamp': 1506414726,
     'url': 'https://logs.rdoproject.org/openstack-periodic-4hr/periodic-tripleo-centos-7-master-containers-build/8a76883'},
     {'commit_hash': 'ec650aa2c8ce952e4a33651190301494178ac562',
    [..]


DLRN API Promoter
`````````````````

`The DLRN API Promoter script
<https://github.com/rdo-infra/ci-config/tree/master/ci-scripts/dlrnapi_promoter>`_
is a Python script that, based on the information in an input config file,
will promote an existing DLRN link to another link, provided the required tests
return successful results.

For example,
`the master ini config file
<https://github.com/rdo-infra/ci-config/blob/master/ci-scripts/dlrnapi_promoter/config/master.ini>`_
is passed to the `promoter script
<https://github.com/rdo-infra/ci-config/blob/master/ci-scripts/dlrnapi_promoter/dlrnapi_promoter.py>`_
to promote the ``current-tripleo`` link to ``current-tripleo-rdo``. See the
sections above where both these links (for Pike) were shown.

In the RDO Phase 1 pipeline, the tests listed under the ``[current-tripleo-rdo]``
are run with the ``current-tripleo`` hash. Each test reports its ``success`` status to the
DLRN API endpoint for the Master release, ``api-centos-master-uc``.

If each test reports ``SUCCESS: true``, the content of the ``current-tripleo``
will become the new content of the ``current-tripleo-rdo`` hash.

For complete documentation on how to run the Promoter script see:
https://github.com/rdo-infra/ci-config/blob/master/ci-scripts/dlrnapi_promoter/README.md


Pushing RDO containers to ``docker.io``
```````````````````````````````````````

The DLRN Promoter script calls the `container push playbook
<https://github.com/rdo-infra/ci-config/blob/master/ci-scripts/container-push/container-push.yml>`_
to push the RDO containers at each stage to `docker.io
<https://hub.docker.com/r/tripleopike/centos-binary-heat-api/tags/>`_.
Note that the above ``docker.io`` link shows containers tagged with ``tripleo-ci-testing``,
``current-tripleo`` and ``current-tripleo-rdo``.


DLRN API Promoter Server
````````````````````````

It is recommended that the Promoter script is run from a dedicated server.
`The promoter-setup repo
<https://github.com/rdo-infra/ci-config/tree/master/ci-scripts/promoter-setup>`_
contains the Ansible playbook used to setup the promoter-server in the RDO
Cloud environment. This playbook allows the promoter script server to be
rebuilt as required.

