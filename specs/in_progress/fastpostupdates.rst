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

=======================================
Resolving limitations of fast-POST
=======================================

The purpose of this document is to describe the requirements to enable
``object_post_as_copy = false`` in the proxy config as a
reasonable deployment configuration in Swift once again, without
sacrificing the features enabled by ``object_post_as_copy = true``.

For brevity we shall use the term 'fast-POST' to refer to the mode of operation
enabled by setting ``object_post_as_copy = false``, and 'POST-as-COPY' to refer
to the mode when ``object_post_as_copy = true``.

Currently using fast-POST incurs the following limitations:

#. Users can not update the content-type of an object with a POST.
#. The change in last-modified time of an object due to a POST is not reflected
   in the container listing.
#. Container-Sync does not "sync" objects after their metadata has been changed
   by a POST. This is a consequence of the container listing not being updated
   and the Container-Sync process therefore not detecting that the object's
   state has changed.

The solution is to implement fast-POST such that a POST to an object will
trigger a container update.

This will require all of the current semantics of container updates from a PUT
(or a DELETE) to be extended into POST and similarly cover all failure
scenarios.  In particular container updates from a POST must be serialize-able
(in the log transaction sense, see :ref:`container_server`) so that
out-of-order metadata updates via POST and data updates via PUT and DELETE can
be replicated and reconciled across the container databases.

Additionally the new ssync replication engine has less operational testing with
fast-POST.  Some behaviors are not well understood. Currently it seems ssync
with fast-POST has the following limitations:

#. A t0.data with a t2.meta on the sender can overwrite a t1.data file on the
   receiver.
#. The whole .data file is transferred to sync only a metadata update.

If possible, or as follow on work (see :ref:`ssync`), ssync should
preserve the semantic differences of syncing updates to .meta and .data
files.

Problem Description
===================

The Swift client API describes that Swift allows an object's "metadata" to be
"updated" via the POST verb.

The client API also describes that a container listing includes, for each
object, the following specific items of metadata: name, size, hash (Etag),
last_modified timestamp and content-type.  If any of these metadata values are
updated then the client expectation is that the named entry in the container
listing for that object should reflect their new values.

For example if an object is uploaded at t0 with a size s0 and etag m0, and then
later at t1 an object with the same name is successfully stored with a size of
s1 and etag m1, then the container listing should *eventually* reflect the new
values s1 and m1 for the named object last_modified at t1.

These two API features can both be satisfied by either:

#. Not allowing POST to change any of the metadata values tracked in the
   container.
#. Ensuring that if a POST changes one of those metadata values then the
   container is also updated.

It is reasonable to argue that some of the object metadata items stored in the
container should not be allowed to be changed by a POST - the object name, size
and hash should be considered immutable for a POST because a POST is restricted
from modifying the body of the object - from which both the etag and size are
derived.

However, it can reasonably be argued that content-type should be allowed to
change on a POST. It is also reasonable to argue that the last_modified time of
an object as reported by the container listing should be equal to the timestamp
of the most recent POST or PUT.

If content-type changes are to be allowed on a POST then the container listing
must be updated, in order to satisfy the client API expectations, but the
current implementation lacks support for container updates triggered by a POST:

#. The object-server POST path does not issue container update requests, or
   store async pendings.

#. The container-server's PUT /object path has no semantics for a
   partial update of an object row - meaning there is no way to change the
   content-type of an object without creating a new record to replace the
   old one. However, because a POST only describes a transformation of an
   object, and not a complete update, an object server cannot reliably provide
   the entire object state required to generate a new container record under a
   single timestamp.

   For example, an object server handling a POST may not have the most recent
   object size and/or hash, and therefore should not include those items in a
   container update under the timestamp of the POST.

#. The backend container replication process similarly does not support
   replication of partially updated object records.

Consequently, updates to object metadata using the fast-POST mode results in an
inconsistency between the object state and the container listing: the
Last-Modified header returned on an HEAD request for an object will reflect the
last time of the last POST, while the value in the container listing will
reflect the time of the last PUT.

Furthermore, the container-sync process is unable to detect when object state
has been changed by a POST, since it relies on a new row being created in the
container database whenever an object changes.

Code archeology seems to support that the primary motivations for the
POST-as-COPY mode of operation were allowing content-type to be
modified without re-uploading the entire object with a PUT from the client,
and enabling container-sync to sync object metadata updates.

Proposed Changes
================

The changes proposed below contribute to achieving the property that all Swift
internal services which track the state of an object will eventually reach a
consistent view of the object metadata, which has three components:

#. immutable metadata (i.e. name, size and hash) that can only be set at the
   time that the object data is set i.e. by a PUT request
#. content-type that is set by a PUT and *may* be modified by a POST
#. mutable metadata such as custom user metadata, which is set by a PUT or POST

Since each of these components could be set at different times on different
nodes, it follows that an object's state must include three timestamps, all or
some of which may be equal:

#. the 'data-timestamp', describing when the immutable metadata was set, which
   is less than or equal to:
#. the 'content-type-timestamp', which is less than or equal to:
#. the 'metadata-timestamp' which describes when the object's mutable metadata
   was set, and defines the Last-Modified time of the object.

We assert that to guarantee eventual consistency, Swift internal processes must
track the timestamp of each metadata component independently. Some or all of
the three timestamps will often be equal, but a Swift process should never
assert such equality unless it can be inferred from state generated by a client
request.

Proxy-server
------------

No changes required - the proxy server already includes container update
headers with backend object POST requests.

Object-server
-------------

#. The DiskFile class will be modified to allow content-type to
   be updated and written to a .meta file. When content-type is updated by a
   POST, a content-type-timestamp value equal to the POST request timestamp
   will also be written to the .meta file.
#. The DiskFile class will be modified so that existing content-type and
   content-type-timestamp values will be copied to a new .meta file if no new
   values are provided.
#. The DiskFile interface will be modified to provide methods to access the
   object's data-timestamp (already stored in the .data file), content-type
   timestamp (as described above) and metadata-timestamp (already stored in the
   .meta file).
#. The DiskFile class will be modified to support using encoded timestamps as
   .meta file names (see :ref:`rsync` and :ref:`timestamp_encoding`).
#. The object-server POST path will be updated to issue container-update
   requests with fallback to the async pending queue similar to the PUT path.
#. Container update requests triggered by a POST will include all three of
   the object's timestamp values: the data-timestamp, the content-type
   timestamp and the metadata-timestamp. These timestamps will either be sent
   as separate headers or encoded into a single timestamp header
   (:ref:`timestamp_encoding`) header value.

.. _container_server:

Container-server
----------------

#. The container-server 'PUT /<object>' path will be modified to support three
   timestamp values being included in the update item that are stored in the
   pending file and eventually passed to the database merge_items method.
#. The merge_items method will be modified so that any existing row for an
   updated object is merged with the object update to produce a new row that
   encodes the most recent of each of the metadata components and their
   respective timestamps i.e. the row will encode three tuples::

    (data-timestamp, size, name, hash)
    (content-type-timestamp, content-type)
    (metadata-timestamp)

   This requires storing two additional timestamps which will be achieved by
   either encoding all three timestamps in a single string stored in the
   existing created_at column (:ref:`timestamp_encoding`) value stored in same
   field as the existing (data) timestamp or by adding new columns to the
   objects table. Note that each object will continue to have only one row in
   the database table.

#. The container listing code will be modified to use the object's metadata
   timestamp as the value for the reported last-modified time.

.. note::
   With this proposal, new container db rows do not necessarily store all of
   the attributes sent with a single object update. Each new row is now
   comprised of the most recent metadata components from the update and any
   existing row.


Container-replicator
--------------------

#. The container-replicator will be modified to ensure that all three object
   timestamps are included in replication updates. At the receiving end these
   are handled by the same merge_items method as described above.

.. _rsync:

rsync object replication
------------------------

With the proposed changes, .meta files may now contain a content-type value set
at a different time to the other mutable metadata. Unlike :ref:`ssync`, the
rsync based replication process has no visibility of the contents of the object
files. The replication process cannot therefore distinguish between two meta
files which have the same name but may contain different content-type and
content-type-timestamp values.

The naming of .meta files must therefore be modified so that the filename
indicates both the metadata-timestamp and the content-type-timestamp. The
current proposal is to use an encoding of the content-type-timestamp and
metadata-timestamp as the .meta file name. Specifically:

 * if the the .meta file contains a content-type value, its name shall be
   the encoding of the metadata-timestamp followed by the (older or equal)
   content-type-timestamp, with a `.meta` extension.
 * if the the .meta file does not contain a content-type value, its name shall
   be the metadata-timestamp, with a `.meta` extension.

Other options for .meta file naming are discussed in :ref:`alternatives`.

The hash_cleanup_listdir function will be modified so that the decision as to
whether a particular meta file should be deleted will no longer be based on a
lexicographical sort of the file names - the file names will be decomposed into
a content-type-timestamp and a metadata-timestamp and the one (or two) file(s)
having the newest of each will be retained.

In addition the DiskFile implementation must be changed to preserve, and read,
up to two meta files in the object directory when their names indicate that one
contains the most recent content-type and the other contains the most recent
metadata.

Multiple .meta files will only exist until the next PUT or POST request is
handled. On a PUT, all older .meta files are deleted - their content is
obsolete. On a newer POST, the multiple .meta files are read and their contents
merged, taking the newest of user metadata and content-type. The merged
metadata is written to a single newer .meta file and all older .meta files are
deleted.

For example, consider an object directory that after rsync has the following
files (sorted)::

    t0_offset.meta      - unwanted
    t2.data             - wanted, most recent data-timestamp
    enc(t6, t2).meta    - wanted, most recent metadata-timestamp
    enc(t4, t3).meta    - unwanted
    enc(t5, t5).meta    - wanted, most recent content-type-timestamp

If a POST occurs at t7 with new user metadata but no new content-type value,
the contents of the directory after handling the post will be::

    t2.data
    enc(t7, t5).meta

Note that the when an object merges content-type and metadata-timestamp from
two .meta files, it is reconstructing the same state that will already have
been propagated to container servers. There is no need for object servers to
send container updates in response to replication events (i.e. no change to
current behavior in that respect).

.. _ssync:

Updates to ssync
----------------

Additionally we should endeavor to enumerate the required changes to ssync to
support the preservation of semantic difference between a POST and PUT.  For
example:

#. The missing check request sent by the ssync_sender should include enough
   information for the ssync_receiver to determine which of the object's state
   is out of date i.e. none, some or all of data, content-type and metadata.
#. The missing check response from ssync_receiver should include enough
   information for the ssync_sender to differentiate between a hash that
   is "missing" and out-of-date content-type and/or metadata update.
#. When handling ssync_sender's send_list during the UPDATES portion, in
   addition to sending PUT and DELETE requests the sender should be able
   to send a pure metadata POST update
#. The ssync_receiver's updates method must be prepared to dispatch POST
   requests to the underlying object-server app in addition to PUT and
   DELETE requests.

The current ssync implementation seems to indicate that it was originally
intended to be optimized for the default POST-as-COPY configuration, and it
does not handle some corner cases with fast-POST as well as rsync replication.
Because ssync is still described as experimental, improving ssync support
should not be a requirement for resolving the current limitations of fast-POST
for rsync deployments.  However ssync is still actively being developed and
improved, and remains a key component to a number of other efforts improve and
enhance Swift.  Full ssync support for fast-POST should be a requirement for
making fast-POST the default.

.. _container-sync:

Container Sync
--------------

Container Sync will require both the ability to discover that an
object has changed, and the ability to request that object.

Because each object update via fast-POST will trigger a container
update, there will be a new row (and timestamp) in the container
databases for every update to an object (just like with POST-as-COPY
today!)

The metadata-timestamp in the database will reflect a complete version
of an object and metadata transformation.  The exact version of the
object retrieved can be verified with X-Backend-Timestamp.

.. _x-newest:

X-Newest
--------

X-Newest should be updated to use X-Backend-Timestamp.

.. note::

    We should fix the sync daemon from using the row[‘created_at’] value
    to set the x-timestamp of the object PUT to the peer container, and
    have it instead use the X-Timestamp from the object being synced.

.. _timestamp_encoding:

Multiple Timestamp Encoding
---------------------------

If required, multiple timestamps t0, t1 ... will be encoded into a single
timestamp string having the form::

  <t0[_offset]>[<+/-><offset_to_t1>[<+/-><offset_to_t2>]]

where:

* t0 may include an offset, if non-zero, with leading zero's removed from the
  offset, e.g. 1234567890.12345_2
* offset_to_t1 is the difference in units of 10 microseconds between t0 and
  t1, in hex, if non-zero
* offset_to_t2 is the difference in units of 10 microseconds between t1 and
  t2, in hex, if non-zero

An example of encoding three monotonically increasing timestamps would be::

  1234567890.12345_2+9f3c+aa322

An example of encoding of three equal timestamps would be::

  1234567890.12345_2

i.e. identical to the shortened form of t0.

An example of encoding two timestamps where the second is older would be::

  1234567890.12345_2-9f3c

Note that a lexicographical sort of encoded timestamps is not required to
result in any chronological ordering.


Example Scenarios
=================

In the following examples we attempt to enumerate various failure conditions
that would require making decisions about how the implementation serializes or
merges out-of-order metadata updates.

These examples use the current proposal for encoding multiple timestamps
:ref:`timestamp_encoding` in .meta file names and in the container db
`created_at` column. For simplicity we use the shorthand `t2-t1` to represent
the encoding of timestamps t2 and t1 in this form, but note that the `-t1` part
is in fact a time difference and not the absolute value of the t2 timestamp.

(The exact format of the .meta file name is still being discussed.)

Consider initial state for an object that was PUT at time t1::

  Obj server  1,2,3: /t1.data {etag=m1, size=s1, c_type=c1}
  Cont server 1,2,3: {ts=t1, etag=m1, size=s1, c_type=c1}

Happy Path
----------

All servers initially consistent, successful fast-POST at time t2 that
modifies an object’s content-type. When all is well our object
servers will end up in a consistent state::

  Obj server 1,2,3: /t1.data {etag=m1, size=s1, c_type=c1}
                    /t2+t2.meta {c_type=c2}

The proposal is for the fast-POST to trigger a container update that is
a combination of the existing metadata from the .data file and the new
content-type::

  Cont server 1,2,3: {ts=t1+t2+t2, etag=m1, size=s1, c_type=c2}


.. note::

    A container update will be issued for every POST even if the
    content-type is not updated to ensure that the container listing
    last-modified time is consistent with the object state, and to ensure
    that a new row is created for container sync.

Now consider some failure scenarios...

Object node down
----------------

In this case only a subset of object nodes would receive the metadata
update::

  Obj server 1,2: /t1.data {etag=m1, size=s1, c_type=c1}
                  /t2+t2.meta {c_type=c2}
  Obj server   3: /t1.data {etag=m1, size=s1, c_type=c1}

Normal object replication will copy the metadata update t2 to the failed object
server 3, bringing its state in line with the other object servers.

Because the failed object node would not have updated it's respective
container server, that will be out of date as well::

  Cont server 1,2: {ts=t1+t2+t2, etag=m1, size=s1, c_type=c2}
  Cont server   3: {ts=t1, etag=m1, size=s1, c_type=c1}

During replication, row merging on server 3 would merge the content-type update
at t2 with the existing row to create a new row identical to that on servers 1
and 2..

Container update fails
----------------------

If a container server is offline while an object server is handling a POST then
the object server will store an async_pending of the update record in the same
as for PUTs and DELETEs.

Object node missing .data file
------------------------------

POST will return 404 and not process the request if the object does not
exist::

  Obj server 1,2: /t1.data {etag=m1, size=s1, c_type=c1}
                  /t2+t2.meta {c_type=c2}
  Obj server   3: 404

After object replication the object servers should have the same files. This
requires no change to rsync replication. ssync replication will be modified to
send a PUT with t1 (including content-type=c1) followed by a POST with t2
(including content-type=c2), i.e. ssync will replicate the requests received by
the healthy servers.

Object node stale .data file
----------------------------

If one object server has an older .data file then the composite timestamp sent
with it's container update will not match that of the other nodes::

  Obj server 1,2: /t1.data {etag=m1, size=s1, c_type=c1}
                  /t2+t2.meta {c_type=c2}
  Obj server   3: /t0.data {etag=m0, size=s0, c_type=c0}
                  /t2+t2.meta {c_type=c2}

After object replication the object servers should have the same files. This
requires no change to rsync replication. ssync replication will be modified to
send a PUT with t1, i.e. ssync will replicate the request missed by the failed
server.

Assuming container server 3 was also out of date, the container row will be
updated to::

  Cont server 1,2: {ts=t1+t2+t2, etag=m1, size=s1, c_type=c2}
  Cont server   3: {ts=t0+t2+t2, etag=m0, size=s0, c_type=c2}

During container replication on server 3, row merging will apply the later data
timestamp at t1 to the existing row to create a new row that matches servers 2
and 3.

Assuming container server 3 was also up to date, the container row will be
updated to::

  Cont server 1,2: {ts=t1+t2+t2, etag=m1, size=s1, c_type=c2}
  Cont server   3: {ts=t1+t2+t2, etag=m1, size=s1, c_type=c2}

Note that in this case the row merging has applied the content-type from the
update but ignored the immutable metadata from the update which is older than
the values in the existing db row.

Newest .data file node down
---------------------------

If none of the nodes that have the t1 .data file are available to handle the
POST at the time of the client request the the metadata may only be applied on
nodes having a stale .data file::

  Obj server 1,2: /t0.data {etag=m0, size=s0, c_type=c0}
                  /t2+t2.meta {c_type=c2}
  Obj server   3: /t1.data {etag=m1, size=s1, c_type=c1}

Object replication will eventually make the object servers consistent.

The containers may be similarly inconsistent::

  Cont server 1,2: {ts=t0+t2+t2, etag=m0, size=s0, c_type=c2}
  Cont server   3: {ts=t1, etag=m1, size=s1, c_type=c1}

During container replication on server 3, row merging will apply the
content-type update at t2 to the existing row but ignore the data-timestamp and
immutable metadata, since the existing row on server 3 has newer data
timestamp.

During replication on container servers 1 and 2, row merging will apply the
data-timestamp and immutable metadata updates from server 3 but ignore the
content-type update since they have a newer content-type-timestamp.

Additional POSTs with Content-Type to overwrite metadata
--------------------------------------------------------

If the initial state already includes a metadata update, the content-type may
have been overridden::

  Obj server 1,2,3: /t1.data {etag=m1, size=s1, c_type=c1}
                    /t2+t2.meta {c_type=c2}

In this case the container's would also reflect the content-type of the
metadata update::

  Cont server 1,2,3: {ts=t1+t2+t2, etag=m1, size=s1, c_type=c2}

When another POST occurs at t3 which includes a content-type update, the final
state of the object server would overwrite the last metadata update entirely::

  Obj server 1,2,3: /t1.data {etag=m1, size=s1, c_type=c1}
                    /t3+t3.meta {c_type=c3}


Additional POSTs without Content-Type to overwrite metadata
-----------------------------------------------------------

If the initial state already includes a metadata update, the content-type may
have been overridden::

  Obj server 1,2,3: /t1.data {etag=m1, size=s1, c_type=c1}
                    /t2+t2.meta {c_type=c2}

In this case the container's would also reflect the content-type of the
metadata update::

  Cont server 1,2,3: {ts=t1+t2+t2, etag=m1, size=s1, c_type=c2}

When another POST occurs at t3 which does not include a content-type update,
the object server will merge its current record of the content-type with the
new metadata and store in a new .meta file, the name of which indicates that it
contains state modified at two separate times::

  Obj server 1,2,3: /t1.data {etag=m1, size=s1, c_type=c1}
                    /t3-t2.meta {c_type=c2}

The container server updates will now encode three timestamps which will cause
row merging on the container servers to apply the metadata-timestamp to their
existing rows and create a new row for the object::

  Cont server 1,2,3: {ts=t1+t2+t3, etag=m1, size=s1, c_type=c2}


Resolving conflicts with multiple metadata overwrites
-----------------------------------------------------

If a previous content-type update is not consistent across all nodes then a
subsequent metadata update at t3 that does not include a content-type value
will result in divergent metadata sets across the nodes::

  Obj server 1,2: /t1.data {etag=m1, size=s1, c_type=c1}
                  /t3-t2.meta {c_type=c2}
  Obj server   3: /t1.data {etag=m1, size=s1, c_type=c1}
                  /t3.meta

Even worse, if subsequent POSTs are not successfully handled successfully on
all nodes then we can end up with no single node having completely up to date
metadata::

  Obj server 1,2: /t1.data {etag=m1, size=s1, c_type=c1}
                  /t3-t2.meta {c_type=c2}
  Obj server   3: /t1.data {etag=m1, size=s1, c_type=c1}
                  /t4.meta

With rsync replication, each object server will eventually have a consistent
set of files, but will have two .meta files::

  Obj server 1,2,3: /t1.data {etag=m1, size=s1, c_type=c1}
                    /t3-t2.meta {c_type=c2}
                    /t4.meta

When the diskfile is opened, both .meta files are read to retrieve the most
recent content-type and the most recent mutable metadata.

With ssync replication, the inconsistent nodes will exchange POSTs that will
eventually result in a consistent single .meta file on each node::

  Obj server 1,2,3: /t1.data {etag=m1, size=s1, c_type=c1}
                    /t4-t2.meta {c_type=c2}


.. _alternatives:

Alternatives
============

Alternative .meta file naming
-----------------------------

#. Encoding the content-type timestamp followed by the metadata timestamp (i.e.
   reverse the order w.r.t. the proposal. This would result in encodings that
   always have a positive offset which is consistent with the
   enc(data-timestamp, content-type-timestamp, metadata-timestamp) form used
   in container updates. However, having the proposed encoding order ensures
   that files having *some* content newer than a data file will always sort
   ahead of the data file, which reduces the churn in diskfile code such as
   hash_cleanup_listdir, and is arguably more intuitive for human inspection
   ("t2-offset.meta is preserved in the dir with t1.data because t2 is later
   than t1", rather than "t0+offset is preserved in the dir with t1.meta
   because the sum of t0 and offset is later than t1).

#. Using a two vector timestamp with the 'normal' part being the content-type
   timestamp and the offset being the time delta to the metadata-timestamp.

   (It is the author's understanding that it is safe to use a timestamp offset
   to represent the metadata-timestamp in this way because .meta files will
   never be assigned a timestamp offset by the container-reconciler, since the
   container-reconciler only uses timestamp offsets to imposing an internal
   ordering on object PUTs and DELETEs having the same external timestamp.)

   This is in principle the same as the proposed option but possibly results
   in a less compact filename and may create confusion with two vector
   timestamps.

#. Using a combination of the metadata-timestamp and a hash of the .meta file
   contents to form a name for the .meta file. The timestamp part allows for
   cleanup of .meta files that are older than a .data or .ts file, while the
   hash part distinguishes .meta that contain different Content-Type and/or
   Content-Type timestamp values. During replication, all valid .meta files are
   preserved in the object directory (the worst case number being capped at the
   number of replicas in the object ring). When DiskFile loads the metadata,
   all .meta files will be read and the most recent values merged into the
   metadata dict. When the merged metadata dict is written, all contributing
   .meta files may be deleted.

   This option is more general in that it allows other metadata items to also
   have individual timestamps (without requiring an unbounded number of
   timestamps to be encoded in the .meta filename). It therefore supports
   other potential new features such as updatable object sysmeta and
   updatable user metadata. Any such feature is of course beyond the scope of
   proposal.


Just use POST-as-COPY
---------------------

POST-as-COPY has some limitations that make it ill-suited for some workloads.

#. POST to large objects is slow
#. POST during failure can result in stale data being copied over fresher data.

Also because COPY is exposed to the client first hand the semantic behavior can
always be achieved explicitly by a determined client.

Force content-type-timestamp to be same as metadata-timestamp
-------------------------------------------------------------

We can simplify the management of .meta files by requiring every POST arriving
at an object server to include the content-type, and therefore remove the need
to maintain a separate content-type-timestamp. There would be no need to
maintain multiple meta files. Container updates would still need to be sent
during an object POST in order to keep the container server in sync with the
object state. The container server still needs to be modified to merge both
content-type and metadata-timestamps with an existing row.

The requirement for content-type to be included with every POST is unreasonably
onerous on clients, but could be achieved by having the proxy server retrieve
the current content-type using a HEAD request with X-Newest = True and insert
it into the backend POST when content-type is missing from the client POST.

However, this scheme violates our assertion that no internal process should
ever assume one of an object's timestamps to be equal to another. In this case,
the proxy is forcing the content-type-timestamp to be the same as the metadata
timestamp that is due to the incoming POST request. In failure conditions, the
proxy may read a stale content-type value, associate it with the latest
metadata-timestamp and as a result erroneously overwrite a fresher content-type
value.

If, as a development of this alternative, the proxy were also to read the
'current' content-type value and its timestamp using a HEAD with X-Newest, and
add both of these items to the backend object POST, then we are get back to the
object server needing to maintain separate content-type and metadata-timestamps
in .meta file.

Further, if the newest content-type in the system is unavailable during a POST
it would be lost, and worse yet if the latest value was associated with a
datafile there's no obvious way to correctly promote it's data timestamp
values in the containers short of doing the very merging described in this
spec - so it comes out as less desirable for the same amount of work.

Use the metadata-timestamp as last modified
-------------------------------------------

This is basically what both fast-POST and POST-as-COPY do today.  When an
object's metadata is updated at t3 the x-timestamp for the transformed object
is t3.  However, fast-POST never updates the last modified in the container
listing.

In the case of fast-POST it can apply a t3 metadata update asynchronously to a
t1 .data file because it restricts metadata updates from including changes to
metadata that would require being merged into a container update.

We want to be able to update the content-type and therefore the container
listing.

In the case of POST-as-COPY it can do this because the metadata update applied
to a .data file t0 is considered "newer" than the .data file t1. The record for
the transformation applied to the t0 data file at t3 is stored in the
container, and the record of the "newer" t1 .data file is irrelevant.

Use metadata-timestamp as primary portion of two vector timestamp
-----------------------------------------------------------------

This suggests the .data file timestamp would be the offset, and merging t3_t0
and t3_t1 would prefer t3_t1.  However merging t3_t0 and t1 would prefer t3_t0
(much as POST-as-COPY does today).  The unlink old method would have to be
updated for rsync replication to ensure that a t3_t0 metadata file "guards" a
t0 data against the "newer" t1 .data file.

It's generally presumed that a stale read during POST-as-COPY resulting in data
loss is rare, the same false-hope applies equivalently to this purposed
specification for a container updating fast-POST implementation.  The
difference being this implementation would throw out the *meta* data update
with a preference to the latest .data file instead.

This alternative was rejected as workable but less desirable.

Implementation
==============

#. `Prefer X-Backend-Timestamp for X-Newest <https://review.openstack.org/133869>`_
#. `Update container on fast-POST <https://review.openstack.org/#/c/135380/>`_
#. `Make ssync compatible with fast-post meta files <https://review.openstack.org/#/c/138498/>`_


Assignee(s)
-----------

#. Alistair Coles (acoles)
#. Clay Gerrard (clayg)


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

Changes may be required to API docs if the last modified time reported in a
container listing changes to be the time of a POST rather than the time of the
PUT (there is currently an inconsistency between POST-as-COPY operation and
fast-POST operation).

We may want to deprecate POST-as-COPY after successful implementation of this
proposal.

Security
--------

None

Testing
-------

New and modified unit tests will be required for the object server and
container-sync.  Probe tests will be useful to verify behavior.

Dependencies
============

None
