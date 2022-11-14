Core maintainers
================

The intention of this document is to give developers some information
regarding what is expected from core maintainers and hopefully provide some
guidance to those aiming for this role.

Teams
-----

The TripleO Core team is responsible for reviewing all changes proposed to
repositories that are under the `governance of TripleO <tripleo_governance_>`_.

.. _tripleo_governance: https://governance.openstack.org/tc/reference/projects/tripleo.html

The TripleO Upgrade core reviewers maintain the `tripleo_upgrade`_ project.

.. _tripleo_upgrade: https://opendev.org/openstack/tripleo-upgrade

The TripleO Validation team maintains the Validation Framework in TripleO.

The TripleO CI team maintains the TripleO CI related projects (tripleo-ci,
tripleo-quickstart, tripleo-quickstart-extras, etc).

We also have contributors with a specific area of expertise who have been
granted core reviews on their area. Example: a Ceph integration expert would
have core review on the Ceph related patches in TripleO.

Because Gerrit doesn't allow such granularity, we trust people to understand
which patches they can use their core reviewer status or not.
If one is granted core review access on an area, there is an expectation that
it'll only be used in this specific area.
The grant is usually done for all the TripleO repositories but we expect
SME cores to use +/- 2 for their area of expertise otherwise the regular +/- 1.

.. note::
   Everyone is warmly encouraged to review incoming patches in TripleO, even
   if you're not (yet) a member of these teams.
   Participating in the review process will be a major task on the road to join
   the core maintainer teams.

Adding new members
------------------

Each team mentioned above should be aware of who is active in their respective
project(s).

In order to add someone in one of these groups, it has to be discussed
between other cores and the TripleO PTL.

It is a good practice to reach out to the nominee before proposing the
candidate, to make sure about their willingness to accept this position and its
responsibilities.

In real life, it usually happens by informal discussions, but the official
proposals have to be sent with an email to the openstack-discuss mailing list.
It is strongly recommended to have this initial informal agreement before
going public, in case there are some disagreements which could cause
unpleasant discussions which could harm the nominee.

This discussion can be initiated by any core, and only the existing cores votes
will weight into whether or not the proposal is granted.
Of course anyone is welcome to share their feedback and opinions.

Removing members
----------------

It is normal for developers to reduce their activity and work on something
else. If they don't reach out by themselves, it is the responsibility of the
teams to remove them from the core list and inform about the change on the
mailing-list and privately when possible.

Also if someone doesn't respect the TripleO rules or doesn't use the core
permission correctly, this person will be removed from the core list with
a private notice at least.

Core membership expectations
----------------------------

Becoming a core member is a serious commitment and it is not granted easily.
Here are a non-exhaustive list of things that are expected:

* The time invested on the project is consistent.

* (Nearly) Daily participation in core reviews.

.. note::
   Core reviewers are expected to provide thoroughly reviews on the code,
   which doesn't only mean +1/-1, but also comments the code that confirm
   that the patch is ready (or not) to be merged into the repository.
   This capacity to provide these kind of reviews is strongly evaluated when
   recruiting new core reviewers. It is preferred to provide quality reviews
   over quantity. A negative review needs productive feedback and harmful
   comments won't help to build credibility within the team.

* Quality of technical contributions: bug reports, code, commit messages,
  specs, e-mails, etc.

* Awareness of discussions happening within the project (mailing-list, specs).

* Best effort participation on IRC #tripleo (when timezone permits),
  to provide support to our dear users and developers.

* Gain trust with other core members, engage collaboration and be nice with
  people. While mainly maintained by Red Hat, TripleO remains a friendly
  project where we hope people can have fun while maintaining a project which
  meets business needs for the OpenStack community.

* Understand the `Expedited Approvals <expedited_approvals_>`_ policy.

.. _expedited_approvals: https://specs.openstack.org/openstack/tripleo-specs/specs/policy/expedited-approvals.html

Final note
----------

The goal of becoming core must not be intimidating. It should be reachable to
anyone well involved in our project with has good intents and enough technical
level. One should never hesitate to ask for help and mentorship when needed.
