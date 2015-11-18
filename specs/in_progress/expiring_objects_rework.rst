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
There are multiple parts to the implementation. The updating of the container
database to remove the expired objects and the removal of the object from disk.

Step 1:
A expired table will be added to the container database. There will be a
'obj_row_id' and 'expired_at' column on the table. The 'obj_row_id' column will
correlate to the row_id for an object in the objects table. The 'expired_at'
column will be an integer timestamp of when the object expires.

The container replicator will remove the object rows from objects table when
their corresponding 'expire_at' time in the expired table is before the start
time of the pass. There will be a trigger to delete row(s) in the 'expired'
table after the deletion of row(s) out of the 'objects' table. Once, the
removal of the expired objects are complete the container database will
be replicated.

Step 2:
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
