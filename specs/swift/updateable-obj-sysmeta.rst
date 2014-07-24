::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=========================
Updateable Object Sysmeta
=========================

The original system metadata patch ( https://review.openstack.org/#/c/51228/ )
supported only account and container system metadata.

There are now patches in review that store middleware-generated metadata
with objects, e.g.:

* on demand migration https://review.openstack.org/#/c/64430/
* server side encryption https://review.openstack.org/#/c/76578/1

Object system metadata should not be stored in the x-object-meta- user
metadata namespace because (a) there is a potential name conflict with
arbitrarily named user metadata and (b) system metadata in the x-object-meta-
namespace will be lost if a user sends a POST request to the object.

A patch is under review ( https://review.openstack.org/#/c/79991/ ) that will
persist system metadata that is included with an object PUT request,
and ignore system metadata sent with POSTs.

The goal of this work is to enable object system metadata to be persisted
AND updated. Unlike user metadata, it should be possible to update
individual items of system metadata independently when making a POST request
to an object server.

This work applies to fast-POST operation, not POST-as-copy operation.

Problem Description
===================

Item-by-item updates to metadata can be achieved by simple changes to the
metadata read-modify-write cycle during a POST to the object server: read
system metadata from existing data or meta file, merge new items,
write to a new meta file. However, concurrent POSTs to a single server or
inconsistent results between multiple servers can lead to multiple meta
files containing divergent sets of system metadata. These must be preserved
and eventually merged to achieve eventual consistency.

Proposed Change
===============

The proposed new behavior is to preserve multiple meta files in the obj_dir
until their system metadata is known to have been read and merged into a
newer meta file.

When constructing a diskfile object, all existing meta files that are newer
that the data file (usually just one) should be read for potential system
metadata contributions. To enable a per-item most-recent-wins semantic when
merging contributions from multiple meta files, system metadata should be
stored in meta files as `key: (value, timestamp)` pairs. This is not
necessary when system metadata is stored in a data file because the
timestamp of those items is known to be that of the data file.

When writing the diskfile during a POST, the merged set of system metadata
should be written to the new meta file, after which the older meta files can
be deleted.

This requires a change to the diskfile cleanup code (`hash_cleanup_listdir`).
After creating a new meta file, instead of deleting all older meta files,
only those that were either older than the data file or read during
construction of the new meta file are deleted.

In most cases the result will be same, but if a second concurrent request
has written a meta file that was not read by the first request handler then
this meta file will be left in place.

Similarly, a change is required in the async cleanup process (called by the
replicator daemon). The cleanup process should merge any existing meta files
into the most recent file before deleting older files. To reduce workload,
this merge process could be conditional upon a threshold number of meta
files being found.

Replication considerations
--------------------------

As a result of failures, object servers may have different existing meta
files for an object when a POST is handled and a new (merged) metadata set
is written to a new meta file. Consequently, object servers may end up with
identically timestamped meta files having different system metadata content.

rsync:

To differentiate between these meta files it is proposed to include a hash
of the metadata content in the name of the meta file. As a result,
meta files with differing content will be replicated between object servers
and their contents merged to achieve eventual consistency.

The timestamp part of the meta filename is still required in order to (a)
allow meta files older than a data or tombstone file to be deleted without
being read and (b) to continue to record the modification time of user
metadata.

ssync - TBD

Deleting system metadata
------------------------

An item of system metadata with key `x-object-sysmeta-x` should be deleted
when a header `x-object-sysmeta-x:""` is included with a POST request. This
can be achieved by persisting the system metadata item in meta files with an
empty value, i.e. `key : ("", timestamp)`, to indicate to any future metadata
merges that the item has been deleted. This guards against inclusion of
obsolete values from older meta files at the expense of storing the empty
value. The empty-valued system metadata may be finally removed during a
subsequent merge when it is observed that some expiry time has passed since
its timestamp (i.e. any older value that the empty value is overriding would
have been replicated by this time, so it is safe to delete the empty value).

Example
-------

Consider the following scenario. Initially the object dir on each object
server contains just the original data file::

    obj_dir:
        t1.data:
            x-object-sysmeta-p: ('p1', t0)

Two concurrent POSTs update the object on servers A and B,
with timestamps t2 and t3, but fail on server C. One POST updates
`x-object-sysmeta-p` and adds `x-object-sysmeta-y`. The other POST adds
`x-object-sysmeta-z`. These POSTs result in two meta files being added to the
object directory on A and B::

    obj_dir:
        t1.data:
            x-object-sysmeta-p: ('p1', t0)
        t2.h2.meta:
            x-object-sysmeta-p: ('p2', t2)
            x-object-sysmeta-x: ('x1', t2)
            x-object-sysmeta-y: ('y1', t2)
        t3.h3.meta:
            x-object-sysmeta-p: ('p1', t0)
            x-object-sysmeta-x: ('x2', t3)
            x-object-sysmeta-z: ('z1', t3)

(`hx` in filename represents hash of metadata)

A response to a subsequent HEAD request would contain the composition of the
two meta files' system metadata items::

    x-object-sysmeta-p: 'p2'
    x-object-sysmeta-x: 'x2'
    x-object-sysmeta-y: 'y1'
    x-object-sysmeta-z: 'z1'

A further POST request received at t4 deletes `x-object-sysmeta-p`. This
causes the two meta files to be read, their contents merged and a new meta
file to be written. This POST succeeds on all servers,
so on servers A and B we have::

     obj_dir:
        t1.data :
            x-object-sysmeta-p: ('p1', t0)
        t4.h4a.meta:
            x-object-sysmeta-p: ('', t4)
            x-object-sysmeta-x: ('x3', t3)
            x-object-sysmeta-z: ('z1', t3)
            x-object-sysmeta-y: ('y1', t2)

whereas on server C we have::

     obj_dir:
        t1.data :
            x-object-sysmeta-p: ('p1', t0)
        t4.h4b.meta:
            x-object-sysmeta-p: ('', t4)

Eventually the meta files will be replicated between servers and merged,
leaving all servers with::

     obj_dir:
        t1.data :
            x-object-sysmeta-p: ('p1', t0)
        t4.h4a.meta:
            x-object-sysmeta-p: ('', t4)
            x-object-sysmeta-x: ('x3', t3)
            x-object-sysmeta-z: ('z1', t3)
            x-object-sysmeta-y: ('y1', t2)

Alternatives
------------

One alternative approach would be to preserve all meta files that are newer
than a data or tombstone file and never merge their contents. This removes
the need to include a hash in the meta file name, but has the obvious
disadvantage of accumulating an increasing number of files, each of which
needs to be read when constructing a diskfile.

Another alternative would store system metadata in separate `sysmeta` file.
It may then be possible to discard the timestamp from the filename (if the
`timestamp.hash` format is deemed too long).


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Alistair Coles (acoles)


Work Items
----------

TBD

Repositories
------------

None

Servers
-------

None

DNS Entries
-----------

None

Documentation
-------------

No change to external API docs. Developer docs would be updated to make
developers aware of the feature.

Security
--------

None

Testing
-------

Additional unit tests will be required for diskfile.py, object server. Probe
tests will be useful to verify replication behavior.

Dependencies
============

Patch for object system metadata on PUT only:
 https://review.openstack.org/#/c/79991/

Spec for updating containers on fast-POST:
 https://review.openstack.org/#/c/102592/

There is a mutual dependency between this spec and the spec to update
containers on fast-POST: the latter requires content-type to be treated as
an item of mutable system metadata, which this spec aims to enable. This
spec assumes that fast-POST becomes usable, which requires consistent
container updates to be enabled.