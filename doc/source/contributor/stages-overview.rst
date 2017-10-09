Promotion Stages
================

The list below shows each stage within the RDO promotion workflow.
Each stage shows the inputs taken and the artifacts produced.

.. note:: All the links shown in the diagram refer to the Master release.

          Links for stable branches would include the stable release name,
          for example, `Pike stable release <https://trunk.rdoproject.org/centos7-pike/tripleo-ci-testing/>`_.

1. **Upstream TripleO**

    `CI DLRN Master consistent
    <https://trunk.rdoproject.org/centos7-master/consistent/>`_
    is generated every 30 mins from upstream commits in case of no packaging errors.

    The ``current`` build contains the latest packages that can be built,
    even if some other packages are failing to build. The ``current`` and
    ``consistent`` builds will be equivalent where there are no problems with
    the build. See https://trunk.rdoproject.org/centos7-master/report.html
    (for Master) for the result of builds.

2. **Upstream Promotion Pipeline**

    `rdoproject.org zuul <https://review.rdoproject.org/zuul/>`_

    *Update* from consistent -> tripleo-ci-testing
    https://trunk.rdoproject.org/centos7-master/tripleo-ci-testing/

    *Push containers* to ``docker.io`` tagged with ``tripleo-ci-testing``
    https://hub.docker.com/r/tripleomaster/centos-binary-heat-api/tags/

    Run tests  - if tests report success ->
    *Promote* from tripleo-ci-testing -> current-tripleo using DLRN Promoter
    https://trunk.rdoproject.org/centos7-master/current-tripleo/

    *Push containers* to trunk.registry.rdoproject.org/master/ (tripleo-ci-testing)
    and upon promotion, also tag them with ``current-tripleo``
    https://review.rdoproject.org/jenkins/job/periodic-tripleo-centos-7-master-containers-build

    *Update images* using DLRN Promoter
    https://images.rdoproject.org/master/rdo_trunk/current-tripleo/stable/
    periodic-tripleo-ci-centos-7-ovb-3ctlr_1comp-featureset002-master-upload
    See artifacts referenced:
    https://github.com/openstack/tripleo-quickstart/blob/master/config/release/master.yml
    https://images.rdoproject.org/master/rdo_trunk/current-tripleo
    RH1 mirror server for images http://66.187.229.139/

    *Push containers* to ``docker.io`` tagged with ``current-tripleo``
    https://hub.docker.com/r/tripleomaster/centos-binary-heat-api/tags/

    Logs from the DLRN Promoter can be accessed on http://38.145.33.13/
    `A Sova instance <http://38.145.34.234/>`_ is used to monitor these jobs

3. **RDO Phase 1**

    https://ci.centos.org/job/rdo_trunk-promote-master-current-tripleo/

    Get https://trunk-primary.rdoproject.org/centos7-pike/current-tripleo/delorean.repo

    Run tests  - if tests report success ->
    *Promote* from current-tripleo -> current-tripleo-rdo using DLRN Promoter
    https://trunk.rdoproject.org/centos7-master/current-tripleo-rdo/

    *Promote the images* to
    https://images.rdoproject.org/master/rdo_trunk/current-tripleo-rdo/

    *Tag the containers* as ``current-tripleo-rdo``

    *Push containers* to ``docker.io`` tagged with ``current-tripleo-rdo``
    https://hub.docker.com/r/tripleomaster/centos-binary-heat-api/tags/

4. **RDO Phase 2**

    Get RDO Phase 1 generated image (RDO on CentOS) from
    https://images.rdoproject.org/master/$BUILD_SYS/$PIN/$hash_id
    and cache internally

    Build http://<internal>/ci-images/master (RDO on RHEL)

    Run tests (RDO on CentOS for baremetal and RDO on RHEL)
    If tests report success ->
    *Update symlink* using DLRN Promoter
    https://trunk.rdoproject.org/centos7-master/current-tripleo-rdo-internal/

    *Update the images link* as
    https://images.rdoproject.org/master/rdo_trunk/current-tripleo-rdo-internal

    *Tag the containers* as ``current-tripleo-rdo-internal``

    Run futher downstream jobs (scale etc.)
    rdo-promote-master-rdo_trunk-nonvoting

5. **OSP Phase 0**

    *Hand off* to RHOSP builds
    https://access.redhat.com/documentation/en/red-hat-openstack-platform/

