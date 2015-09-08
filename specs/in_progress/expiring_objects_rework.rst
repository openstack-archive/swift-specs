::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

Scaling Expiring Objects
========================

Problem description
-------------------
The object expirer daemon does not process objects fast enought
when there are a large amount of files that need to expire.
This leads to situtions like:

- Objects that give back 404s upon requests, but are still in showing in the
  container listing.
- Objects not being deleted in a timely manner.
- Object expirer passes never completing.

Problem Example
---------------
Imagine a client is PUTting 1000 object a second spread out over 10 containers into
the cluster. First on the PUT we are using double the container resources of the
cluster, because of the extra PUT to the .expiring_objects account. Then when we
start deleting the objects we double the strain of the container layer again. The
customer’s containers now have to handle the 100 PUTs/sec and 100 DELETEs a second
from the expirer daemon. If it can’t keep up the daemon begins to gets behind.
If there are no changes to this system the daemon will never catch up- in addition
to this other customers will begin to be starved for resources as well.

Proposed change(s)
------------------
There will need to be two changes needed to fix the problem described.

1.) Allow for the container databases to know whether an object is expired.
This will allow for the container replicater to keep the object counts correct.

2.) Allow the auditor to delete objects that have expired during its pass.
This will allow for the removal of the object expirer daemon.

Implementation Plan
-------------------

The object table, in the container database, will have a 'expire_at'column added.
The expire_at column will be a timestamp that reflect when/if an object should
expire. When a container listing request is made objects whos 'expire_at' times
are before the request time will not be returned.

The container replicator will remove the object rows from the container databases
when the expire_at and reclaim age have passed.

Once the container updater runs and updates that stats for the containers the
objects that are expired will no longer be considered in the bytes_used or the
object counts in the account database.

The object auditor as it makes its pass will remove any expired objects.
When the object auditor inspects an object's metadata, if the X-Delete-At is
before the current time, the auditor will delete the object. Due to slow auditor
passes, the cluster will have extra data until the objects get processed.

Rollout Plan
------------

When deploying this change the current expirer deamon can contiue to run until
all objects are removed from the '.expiring_objects' account. Once that is done
the deamon can be stopped.

Also, a script for updating the container databases with the 'expire_at' times
for all the objects with will be created.


Assignee(s)
-----------
Primary assignee:
  (aerwin3) Alan Erwin alan.erwin@rackspace.com
