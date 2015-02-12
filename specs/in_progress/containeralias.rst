::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

=================
Container aliases
=================

A container alias makes it possible to link to other containers, even to
containers in different accounts.

Problem Description
===================

Currently it is more complicated to access containers in other accounts than
containers defined in the account returned as your storage URL because you
need to use a different storage URL than returned by your auth-endpoint -
which is known to not be support by all clients.  Even if the storage URL of a
shared container which you have access is known and supported by the client of
choice - shared containers are not listed when doing a GET request on the
users account, thus they are not discoverable by a regular client applications
or users.

Alias container could simplify this task. A swift account owner/admin with
permissions to create containers could create an alias onto a container which
users of the account already have access to (most likely via ACL's), and
requests rooted at or under this alias could be redirected or proxied to a
second container on a different account.

This would make it simpler to access these containers with existing clients
for different reasons.

#. A GET request on the account level would list these containers
#. Requests to an alias container are forwarded to the target container,
   making it possible to access that container without using a different
   storage URL in the client.

However, setting the alias still requires the storage URL (see
`Automatic container alias provisioning`_ for alternative future work).

Caveats
=======

Setting an alias should be impossible if there are objects in the source
container because these would become inaccessible, but still require storage
space.  There is a possible race condition if a container is created and
objects are stored within while at the same time (plus a few milliseconds?) an
alias is set.

A reconciler mechanism (similar to the one used in storage policies) might
solve this, as well as ensuring that the alias can be only set during
container creation. Un-setting alias would be denied, instead the alias
container is to be deleted.

Proposed Change
===============

New metadata to set and review, as well as sys-metadata to store - the target
container on a container alias.

Most of the required changes can be put into a separate middleware. There is an
existing patch: https://review.openstack.org/#/c/62494

.. note::

    The main problem identified with that patch was that a split brain could
    allow a container created on handoffs WITHOUT an alias to shadow a
    pre-existing alias container, and during upload could cause the user
    perception of to which location data was written to be confused and
    potentially un-resolved.

It's been purposed that reconciliation process to move objects in an alias
container to the target container could allow an eventually consistent repair
of the split-brain'd container.

Security
========

Test and verify what happens if requests are not yet authenticated; make sure
ACLs are respected and unauthorized requests to containers in other accounts is
impossible.

The change should include functional tests which validate cross-account and
non-swift owner cross-container alias's correctly respect target ACL's - even
if in some cases they appear to duplicate the storage-url based
cross-account/cross-container ACL tests.

If a background process is permitted to move object stored in a container
which is later determined to have been an alias there is likely to be
authorization implications if the ACL's on the target have changed.

Documentation
--------------
 
Update the documentation and document the behavior.

Work Items
----------

Further discussion of design.

Assignee(s)
-----------

Primary assignee:
  cschwede <christian.schwede@enovance.com>

Future Work
===========

Automatic container alias provisioning
--------------------------------------

Cross-account container sharing might be even more simplified, leading to a
better user experience.

Let's assume there are two users in different accounts:

``test:tester`` and ``test2:tester2``

If ``test:tester`` puts an ACL onto an container ``/AUTH_test/container`` to
allow access for ``test2:tester2``, the middleware could create an alias
container ``/AUTH_test2/test_container`` linking to ``/AUTH_test/container``.
This would make it possible to discover shared containers to other
users/accounts. However, there are two challenges:

1. Name conflicts: there might be an existing container
``/AUTH_test2/test_container``
2. A lookup would require to resolve an account name into the account ID

Cross realm container aliases
-----------------------------

Might be possible to tie into container sync realms (or something similar) to
allow operators the ability to let users proxy requests to other realms.

