TripleO CI Promotions
=====================

This section introduces the concept of promotions in TripleO.
In short, a promotion happens when we can certify the latest version of all
packages required for a TripleO deployment of OpenStack as being in a good
state and without regressions.

The certification consists of running Zuul CI jobs with the latest packages
built from source for TripleO code (list of TripleO repos at [1]_) and
the latest packages built from source for non-tripleo code. If the tests are
successful, then the result is certified as **current-tripleo**, ready to be
consumed by the TripleO CI check and gate jobs (see [2]_ for more information
about check and gate).

This process is continuous as new code is merged into the various repos. Every
time we get a successful completion of the promotion CI jobs, the tested content
is 'promoted' to be the new **current-tripleo**, hence the name this workflow
is known by. At a given time, the latest **current-tripleo** is the baseline by
which we test all new code submissions to the TripleO project.

TripleO vs non-tripleo repos
----------------------------

All proposed code submissions across the various tripleo repos are gated by the
TripleO community which owns and manages the zuul check and gate jobs for those
repos.

However, we cannot gate changes to anything outside TripleO, including all
the OpenStack projects used by TripleO as well as any dependencies such
as Open vSwitch or Pacemaker.

Even though we cannot gate on those external repos, the promotion process
allows us to test our TripleO code with their latest versions. If there are
regressions or any other bugs (and assuming ideal test coverage) the promotion
jobs will fail accordingly allowing the TripleO CI team to investigate and file
launchpad bugs so the issue(s) can be addressed.

RDO DLRN & Promotion Criteria
-----------------------------

TripleO CI jobs consume packages built by the RDO DLRN service ('delorean') so
we first introduce it here. An overview is given on the RDO project site at
[3]_.

In short, RDO DLRN builds RPMs from source and publishes the resulting packages
and repos. Each build or repo is identifiable using a unique build ID.

RDO DLRN assigns named tags to particular build IDs. You can see all of these
named tags by browsing at the RDO DLRN package root, for example for Centos8
master branch at [4]_. Of particular importance to the TripleO promotion
workflow are::

* current
* consistent
* component-ci-testing
* promoted-components
* tripleo-ci-testing
* current-tripleo

The list of tags in the order given above gives the logical progression
through the TripleO promotion workflow.

The build ID referenced by each of those named tags is constantly updated as
new content is 'promoted' to become the new named tag.

A general pattern in DLRN is that **current** is applied to the very latest
build, that is, the latest commits to a particular repo. A new **current**
build is generated periodically (e.g. every half hour). The **consistent** tag
represents the latest version of packages where there were no errors
encountered during the build for any of those (i.e. all packages were built
successfully). The **consistent** build is what TripleO consumes as the entry
point to the TripleO promotion workflow.

One last point to be made about RDO DLRN is that after the TripleO promotion
CI jobs are executed against a particular DLRN build ID, the results are
reported back to DLRN. For example, you can query using the build ID at [5]_
to get the list of jobs that were executed
against that specific content, together with the results for each.

The list of jobs that are required to pass before we can promote a particular
build is known as the 'promotion criteria'. In order to promote, TripleO
queries the DLRN API to get the results for a particular build and compares the
passing jobs to the promotion criteria, before promoting or rejecting that
content accordingly. You can find the master centos8 promotion criteria at [6]_
for example.

The TripleO Promotion Pipelines
-------------------------------

A pipeline refers to a series of Zuul CI jobs and what we refer to as the
TripleO promotion workflow is actually a number of interconnected pipelines.
At the highest level conceptually these are grouped into either *Component*
or *Integration* pipelines. The output of the Component pipeline serves as
input to the Integration pipeline.

A Component is a conceptual grouping of packages related by functional area
(with respect to an OpenStack deployment). This grouping is enforced in
practice by the RDO DLRN server and the current list of all components can be
found at [7]_. For example, you can expect to find the 'openstack-nova-'
packages within the Compute component.

The Component pipeline actually consists of a number of individual
pipelines, one for each of the components. The starting point for each of these
is the latest **consistent** build of the component packages and we will go
into more detail about the flow inside the component pipelines in the following
section.

A successful run of the jobs for the given component allows us to certify that
content as being the new **promoted-components**, ready to be used as input to
the Integration pipeline. The Integration pipeline qualifies the result of the
components tested together and when that is successful we can promote to a new
current-tripleo. This is shown conceptually for a subset of components here:

.. only:: html

  .. mermaid:: component_integration_pipelines.mmd

In the diagram above, you can see the component pipeline at the top with the
compute, cinder and security components. This feeds into the integration
pipeline in the bottom half of the diagram where promoted-components will be
tested together and if successful produce the new **current-tripleo**.

The Component Promotion Pipeline
--------------------------------

As noted above, the "Component pipeline" is actually a series of individual
pipelines, one for each component. While these all operate and promote
in the same way, they do so independently of each other.
So the latest **compute/promoted-components** may be much newer than the latest
**security/promoted-components**, if the latter is failing to promote for
example. The following flowchart shows the progression of the RDO DLRN tags
through a single component pipeline while in practice this flow is repeated in
parallel per component.

.. only:: html

  .. mermaid:: component_pipeline_tags_flow.mmd


As illustrated above, the entry point to the component pipelines
is the latest **consistent** build from RDO DLRN. Once a day a periodic job
tags the latest **consistent** build as **component-ci-testing**. For example
you can see the history for the baremetal component job at [8]_ descriptively
named
**periodic-tripleo-centos-8-master-component-baremetal-promote-consistent-to-component-ci-testing**.

After this job has completed the content marked as **component-ci-testing**
becomes the new candidate for promotion to be passed through the component CI
jobs. The **component-ci-testing** repo content is tested with the latest
**current-tripleo** repos of everything else. Remember that at a given time
**current-tripleo** is the known good baseline by which we test all new
content and the same applies to new content tested in the component pipelines.

As an example of the component CI jobs, you can see the history for the
baremetal component standalone job at [9]_. If you navigate to the
*logs/undercloud/etc/yum.repos.d/*
directory for one of those job runs you will see (at least) the following
repos:

* delorean.repo - which provides the latest current-tripleo content
* baremetal-component.repo - which provides the 'component-ci-testing' content
  that we are trying to promote.

You may notice that the trick allowing the baremetal-component.repo to have
precedence for the packages it provides is to set the repo priority accordingly
(*1* for the component and *20* for delorean.repo).

Another periodic job checks the result of the **component-ci-testing** job runs
and if the component promotion criteria is satisfied the candidate content is
promoted and tagged as the new **promoted-components**. You can find the
promotion criteria for Centos8 master components at [10]_.

As an example the history for the zuul job that handles promotion to
promoted-components for the cinder component can be found at [11]_

You can explore the latest content tagged as **promoted-components** for the
compute component at [12]_. All the component **promoted-components** are
aggregated into one repo that can be found at [13]_ and looks
like the following::

    [delorean-component-baremetal]
    name=delorean-openstack-ironic-9999119f737cd39206df3d73e23e5f47933a6f32
    baseurl=https://trunk.rdoproject.org/centos8/component/baremetal/99/99/9999119f737cd39206df3d73e23e5f47933a6f32_1b0aff0d
    enabled=1
    gpgcheck=0
    priority=1

    [delorean-component-cinder]
    name=delorean-openstack-cinder-482e6a3cc5cca697b54ee1d853a4eca6e6f3cfc7
    baseurl=https://trunk.rdoproject.org/centos8/component/cinder/48/2e/482e6a3cc5cca697b54ee1d853a4eca6e6f3cfc7_ae00ff8c
    enabled=1
    gpgcheck=0
    priority=1

Every time a component promotes a new **component/promoted-components** the
aggregated **promoted-components** delorean.repo on the RDO DLRN server is
updated with the new content.

This **promoted-components** repo is used as the starting point for the TripleO
Integration promotion pipeline.

The Integration Promotion Pipeline
----------------------------------

The Integration pipeline as the name suggests is the integration point where
we test new content from all components together. The consolidated
**promoted-components** delorean.repo produced by the component pipeline
is tested with a series of CI jobs. If the jobs listed in the promotion
criteria pass, we promote that content and tag it as **current-tripleo**.

.. only:: html

  .. mermaid:: promotions.mmd

As can be seen in the flowchart above, the **promoted-components** content
is periodically promoted (pinned) to **tripleo-ci-testing**, which becomes the
new promotion candidate to be tested. You can find the build history
for the job that promotes to **tripleo-ci-testing** for Centos 8 master,
descriptively named
**periodic-tripleo-centos-8-master-promote-promoted-components-to-tripleo-ci-testing**,
at [14]_.

First the **tripleo-ci-testing** content is used to build containers and
overcloud deployment images and these are pushed to RDO cloud to be used by
the rest of the jobs in the integration pipeline.

The periodic promotion jobs are then executed with the results being reported
back to DLRN. If the right jobs pass according to the promotion criteria
then the **tripleo-ci-testing** content is promoted and tagged to become the
new **current-tripleo**.

An important distinction in the integration pipeline compared to the promotion
pipeline is in the final promotion of content. In the component pipeline
the **promoted-components** content is tagged by a periodic Zuul job as
described above. For the Integration pipeline however, the promotion to
**current-tripleo** happens with the use of a dedicated service. This service
is known to the tripleo-ci squad by a few names including
'the promotion server', 'the promoter server' and 'the promoter'.

In short the promoter periodically queries delorean for the results of the last
few tripleo-ci-testing runs. It compares the results to the promotion criteria
and if successful it re-tags the container and overcloud deployment images as
**current-tripleo** and pushes back to RDO cloud (as well as to the quay.io and
docker registries). It also talks to the DLRN server and retags the
successful **tripleo-ci-testing** repo as the new **current-tripleo**.
You can read more about the promoter with links to the code at [15]_.

References
~~~~~~~~~~

.. [1] `List of TripleO repos <https://releases.openstack.org/teams/tripleo.html>`_
.. [2] `TripleO Check and Gate jobs <https://docs.openstack.org/tripleo-docs/latest/ci/ci_primer.html#zuul-queues-gate-vs-check>`_
.. [3] `RDO DLRN Overview @ rdoproject.org <https://www.rdoproject.org/what/dlrn/>`_
.. [4] `Index of RDO DLRN builds for Centos 8 master @ rdoproject.org <https://trunk.rdoproject.org/centos8-master/>`_
.. [5] `Query RDO DLRN by build ID @ rdoproject.org <https://trunk.rdoproject.org/api-centos8-master-uc/api/civotes_agg_detail.html?ref_hash=4c59f98669a605fd62278142ef6b8939>`_
.. [6] `Centos8 current-tripleo promotion criteria at time of writing <https://github.com/rdo-infra/ci-config/blob/a3120dbefa9034b1f0c0057bec74623e32bd4ac3/ci-scripts/dlrnapi_promoter/config/CentOS-8/master.ini#L21>`_
.. [7] `Centos8 RDO DLRN components @ rdoproject.org <https://trunk.rdoproject.org/centos8-master/component/>`_
.. [8] `Zuul job history "periodic-tripleo-centos-8-master-component-baremetal-promote-consistent-to-component-ci-testing" <https://review.rdoproject.org/zuul/builds?job_name=periodic-tripleo-centos-8-master-component-baremetal-promote-consistent-to-component-ci-testing>`_
.. [9] `Zuul job history "periodic-tripleo-ci-centos-8-standalone-baremetal-master" <https://review.rdoproject.org/zuul/builds?job_name=periodic-tripleo-ci-centos-8-standalone-baremetal-master>`_
.. [10] `Centos8 master promoted-components promotion critiera at time of writing <https://github.com/rdo-infra/ci-config/blob/a22de83f4c0f78f3d3555bfba2511fedc3919d3e/ci-scripts/dlrnapi_promoter/config/CentOS-8/component/master.yaml#L18-L21>`_
.. [11] `Zuul job history "periodic-tripleo-centos-8-master-component-cinder-promote-to-promoted-components" <https://review.rdoproject.org/zuul/builds?job_name=periodic-tripleo-centos-8-master-component-cinder-promote-to-promoted-components>`_
.. [12] `Compute promoted-components @ rdoproject.org <https://trunk.rdoproject.org/centos8-master/component/compute/promoted-components>`_
.. [13] `Centos8 master promoted-components delorean.repo @ rdoproject.org  <https://trunk.rdoproject.org/centos8-master/promoted-components/delorean.repo>`_
.. [14] `Zuul job history "periodic-tripleo-centos-8-master-promote-promoted-components-to-tripleo-ci-testing" <https://review.rdoproject.org/zuul/builds?job_name=periodic-tripleo-centos-8-master-promote-promoted-components-to-tripleo-ci-testing>`_
.. [15] `TripleO CI docs "Promotion Server and Criteria" <https://docs.openstack.org/tripleo-docs/latest/ci/chasing_promotions.html#promotion-server-and-criteria>`_
