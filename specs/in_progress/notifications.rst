::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

==============================================
Send notifications on PUT/POST/DELETE requests
==============================================

Swift should be able to send out notifications if new objects are uploaded,
metadata has been changed or data has been deleted.

Problem Description
===================

Currently there is no way to detect changes in a given container except listing
it's contents and comparing timestamps. This makes it difficult and slow in case
there are a lot of objects stored, and it is not very efficient at all.
Some external services might be interested when an object got uploaded, updated
or deleted; for example to store the metadata in an external database for
searching or to trigger specific events like computing on object data.

Proposed Change
===============

A new middleware should be added that can be configured to run inside the proxy
server pipeline.

Alternatives
------------
Another option might be to analyze logfiles and parsing them, aggregating data
into notifications per account and sending batches of updates to an external
service. However, performance is most likely worse since there is a lot of
string parsing involved, and a central logging service might be required to send
notifications in order.

Implementation
==============

Sending out notifications should happen when an object got modified. That means
every successful object change (PUT, POST, DELETE) should trigger an action and
send out an event notification.
It should be configurable on either an account or container level that
notifications should be sent; this leaves it up to the user to decide where they
end up and if a possible performance impact is acceptable.
An implementation should be developed as an additional middleware inside the Swift
proxy, and make use of existing queuing implementations within OpenStack,
namely Zaqar (https://wiki.openstack.org/wiki/Zaqar).
It needs to be discussed if metadata that is stored along the object should be
included in the notification or not; if there is a lot of metadata the
notifications are getting quite large. A possible trade off might be a threshold
for included metadata, for example only the first X bytes. Or send no metadata
at all, but only the account/container/objectname.

Assignee(s)
-----------

Primary assignee:
    cschwede

Work Items
----------

Develop middleware for the Swift proxy server including functional tests.

Update Swift functional test VMs to include Zaqar service for testing.

Repositories
------------

None

Servers
-------

Functional tests require either a running Zaqar service on the testing VM, or a
dummy implementation that acts like a Zaqar queue.

DNS Entries
-----------

None

Documentation
-------------

Add documentation for new middleware

Security
--------

Notifications should be just enabled or disabled per-container, and the
receiving server should be set only in the middleware configuration setting.
This prevents users from forwarding events to an own, external side that the
operator is not aware of.

Enabling or disabling should be restricted to account owners.

Sent notifications include the account/container/objectname, thus traffic should
be transmitted over a private network or SSL-encrypted.

Testing
-------

Unit and functional testing shall be included in a patch.

Dependencies
============

- python-zaqarclient: https://github.com/openstack/python-zaqarclient
- zaqar service running on the gate (inside the VM)
