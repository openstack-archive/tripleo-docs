Reproduce CI jobs for debugging and development
===============================================

Knowing that at times ( perhaps always ) manipulating zuul jobs to do
your bidding can be frustrating. Perhaps you are trying to reproduce a
bug, test a patch, or just bored on a Sunday afternoon. I wanted to
briefly remind folks of their options.

`RDO's zuul: <https://review.rdoproject.org/sf/welcome.html>`__
---------------------------------------------------------------

RDO's zuul is setup to directly inherit from upstream zuul. Any TripleO
job that executes upstream should be rerunable in RDO's zuul. A distinct
advantage here is that you can ask RDO admins to hold the job for you,
get your ssh keys on the box and debug the live environment. It's good
stuff. To hold a node, ask your friends in #rhos-ops

Use testproject: Some documentation can be found
`here <https://docs.openstack.org/tripleo-docs/latest/ci/chasing_promotions.html#hack-the-promotion-with-testproject>`__:

upstream job example:
^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

    - project:
        name: testproject
        check:
          jobs:
            - tripleo-ci-centos-8-content-provider
            - tripleo-ci-centos-8-containers-multinode:
                dependencies:
                  - tripleo-ci-centos-8-content-provider

        gate:
          jobs: []

periodic job, perhaps recreating a CIX issue example:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

   - project:
      name: testproject
      check:
         jobs:
            - tripleo-ci-centos-8-scenario002-standalone:
               vars:
                  timeout: 22000
            - periodic-tripleo-ci-centos-8-standalone-full-tempest-scenario-master:
               vars:
                  timeout: 22000
                  force_periodic: true
            - periodic-tripleo-ci-centos-8-standalone-full-tempest-scenario-victoria:
               vars:
                  timeout: 22000
                  force_periodic: true
            - periodic-tripleo-ci-centos-8-standalone-full-tempest-scenario-ussuri:
               vars:
                  timeout: 22000
                  force_periodic: true

      gate:
         jobs: []




Remember that depends-on can bring in any upstream changes.
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- Here is an example commit message:

Test jobs with new ovn package

.. code-block:: yaml

   Test jobs with new ovn package

   Depends-On: https://review.opendev.org/c/openstack/openstack-tempest-skiplist/+/775493

   Change-Id: I7b392acc4690199caa78cac90956e717105f4c6e

`Local zuul: <https://github.com/rdo-infra/ansible-role-tripleo-ci-reproducer>`__
---------------------------------------------------------------------------------

Setting up zuul and friends locally is a much heavier lift than your
first option.  Instructions and scripts to help you are available in any upstream
TripleO job, and
`here <https://github.com/rdo-infra/ansible-role-tripleo-ci-reproducer>`__

A basic readme for the logs can be found directly in the logs directory
of any tripleo job.

-  `Basic
   Readme <https://opendev.org/openstack/tripleo-ci/src/branch/master/docs/tripleo-quickstart-logs.html>`__
-  `Job
   reproduce <https://opendev.org/openstack/tripleo-quickstart-extras/src/branch/master/roles/create-zuul-based-reproducer/templates/README-reproducer-zuul-based-quickstart.html.j2>`__

If you are familiar w/ zuul and friends, containers, etc.. this could be
a good option for you and your team. There are a lot of moving parts and
it's complicated, well because it's complicated. A good way to become
more familiar with zuul would be to try out zuul's tutorial

`zuul-runner: <https://zuul-ci.org/docs/zuul/reference/developer/specs/zuul-runner.html>`__
-------------------------------------------------------------------------------------------

A long hard fought battle of persuasion and influence has been fought
with the maintainers of the zuul project. The blueprints and specs have
merged. The project's status is not complete as there are many
unmerged patches to date.

Other Options:
--------------

Finally, if you are not attempting to recreate, test, play with an
upstream tripleo job and just want to develop code there is another
option. A lot of developers find `tripleo-lab <https://github.com/cjeanner/tripleo-lab>`__ to be quite useful. Many
devels have their own patterns as well, what works for you is fine.

Summary:
--------

For what it's worth imho using testproject jobs is an efficient, low
barrier to getting things done with upstream TripleO jobs. I'll be
updating the documentation and references to try and help over the next
few days, patches are welcome :)
