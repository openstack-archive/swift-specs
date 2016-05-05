======================
Swift Specs Repository
======================

This archive is no longer active. Content is kept for historic purposes.
========================================================================

Documents in this repo are a collection of ideas. They are not
necissarily a formal design for a feature, nor are they docs for a
feature, nor are they a roadmap for future features.

This is a git repository for doing design review on enhancements to
OpenStack Swift.  This provides an ability to ensure that everyone
has signed off on the approach to solving a problem early on.

Repository Structure
====================
The structure of the repository is as follows::

  specs/
      done/
      in_progress/

Implemented specs will be moved to :ref:`done-directory`
once the associated code has landed.

The Flow of an Idea from your Head to Implementation
====================================================
First propose a spec to the ``in_progress`` directory so that it can be
reviewed. Reviewers adding a positive +1/+2 review in gerrit are promising
that they will review the code when it is proposed. Spec documents should be
approved and merged as soon as possible, and spec documents in the
``in_progress`` directory can be updated as often as needed. Iterate on it.

#. Have an idea
#. Propose a spec
#. Reviewers review the spec. As soon as 2 core reviewers like something,
   merge it. Iterate on the spec as often as needed, and keep it updated.
#. Once there is agreement on the spec, write the code.
#. As the code changes during review, keep the spec updated as needed.
#. Once the code lands (with all necessary tests and docs), the spec can be
   moved to the ``done`` directory. If a feature needs a spec, it needs
   docs, and the docs must land before or with the feature (not after).

Spec Lifecycle Rules
====================
#. Land quickly: A spec is a living document, and lives in the repository
   not in gerrit. Potential users, ops and developers will look at
   http://specs.openstack.org/openstack/swift-specs/ to get an idea of what's
   being worked on, so they need to be quick to land.

#. Initial version is an idea not a technical design: That way the merits of
   the idea can be discussed and landed and not stuck in gerrit limbo land.

#. Second version is an overview of the technical design: This will aid in the
   technical discussions amongst the community.

#. Subsequent versions improve/enhance technical design: Each of these
   versions should be relatively small patches to the spec to keep rule #1. And
   keeps the spec up to date with the progress of the implementation.

How to ask questions and get clarifications about a spec
========================================================
Naturally you'll want clarifications about the way a spec is written. To ask
questions, propose a patch to the spec (via the normal patch proposal tools)
with your question or your understanding of the confusing part. That will
raise the issue in a patch review and allow everyone to answer or comment.

Learn As We Go
==============
This is a new way of attempting things, so we're going to be low in
process to begin with to figure out where we go from here. Expect some
early flexibility in evolving this effort over time.
