
::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

====================
Swift Symbolic Links
====================

1. Problem description
======================

With the advent of storage policies and erasure codes, moving an
object between containers is becoming increasingly useful. However, we
don't want to break existing references to the object when we do so.

For example, a common object lifecycle has the object starting life
"hot" (i.e. frequently requested) and gradually "cooling" over time
(becoming less frequently requested). The user will want an object to
start out replicated for high requests-per-second while hot, but
eventually transition to EC for lower storage cost once cold.

A completely different use case is when an application is sharding
objects across multiple containers, but finds that it needs to use
even more containers; for example, going from 256 containers up to
4096 as write rate goes up. The application could migrate to the new
schema by creating 4096-sharded references for all 256-sharded
objects, thus avoiding a lot of data movement.

Yet a third use case is a user who has large amounts of
infrequently-accessed data that is stored replicated (because it was
uploaded prior to Swift's erasure-code support) and would like to
store it erasure-coded instead. The user will probably ask for Swift
to allow storage-policy changes at the container level, but as that is
fraught with peril, we can offer them this instead.


2. Proposed change
==================

Swift will gain the notion of a symbolic link ("symlink") object. This
object will reference another object. GET, HEAD, POST, and OPTIONS
requests for a symlink object will operate on the referenced object.
DELETE and PUT requests for a symlink object will operate on the
symlink object, not the referenced object, and will delete or
overwrite it, respectively.

GET, HEAD, POST, and OPTIONS requests can operate on a symlink object
instead of the referenced object by adding a query parameter
``?symlink=true`` to the request.

The aim is for Swift symlinks to operate analogously to Unix symbolic
links (except where it does not make sense to do so).


2.1. Alternatives
-----------------

One could use a single-segment SLO manifest to achieve a similar
effect. However, the ETag of a SLO manifest is the MD5 of the ETags of
its segments, so using a single-segment SLO manifest changes the ETag
of the object. Also, object metadata (X-Object-Meta-\*) would have to
be copied to the SLO manifest since metadata from SLO segments does
not appear in the response. Further, SLO manifests contain the ETag of
the referenced segments, and if a segment changes, the manifest
becomes invalid. This is not a desirable property for symlinks.

A DLO manifest does not validate ETags, but it still fails to preserve
the referenced object's ETag and metadata, so it is also unsuitable.
Further, since DLOs are based on object name prefixes, the upload of a
new object (e.g. ``thesis.doc``, then later ``thesis.doc.old``) could
cause corrupted downloads.

Also, DLOs and SLOs cannot use each other as segments, while Swift
symlinks can reference DLOs and SLOs *and* act as segments in DLOs and
SLOs.

3. Client-facing API
====================

Clients create a Swift symlink by performing a zero-length PUT request
with the query parameter ``?symlink=true`` and the header
``X-Object-Symlink-Target-Object: <object>``.

For a cross-container symlink, also include the header
``X-Object-Symlink-Target-Container: <container>``. If omitted, it defaults to
the container of the symlink object.

For a cross-account symlink, also include the header
``X-Object-Symlink-Target-Account: <account>``. If omitted, it defaults to
the account of the symlink object.

Symlinks must be zero-byte objects. Attempting to PUT a symlink
with a nonempty request body will result in a 400-series error.

The referenced object need not exist at symlink-creation time. This
mimics the behavior of Unix symbolic links. Also, if we ever make bulk
uploads work with symbolic links in the tarballs, then we'll have to
avoid validation. ``tar`` just appends files to the archive as it
finds them; it does not push symbolic links to the back of the
archive. Thus, there's a 50% chance that any given symlink in a
tarball will precede its referent.


3.1 Example: Move an object to EC storage
-----------------------------------------

Assume the object is /v1/MY_acct/con/obj

1. Obtain an EC-storage-policy container either by finding a
   pre-existing one or by making a container PUT request with the
   right X-Storage-Policy header.

1. Make a COPY request to copy the object into the EC-policy
   container, e.g.::

    COPY /v1/MY_acct/con/obj
    Destination: ec-con/obj

1. Overwrite the replicated object with a symlink object::

    PUT /v1/MY_acct/con/obj?symlink=true
    X-Object-Symlink-Target-Container: ec-con
    X-Object-Symlink-Target-Object: obj

4. Interactions With Existing Features
======================================

4.1 COPY requests
-----------------

If you copy a symlink without ``?symlink=true``, you get a copy of the
referenced object. If you copy a symlink with ``?symlink=true``, you
get a copy of the symlink; it will refer to the same object,
container, and account.

However, if you copy a symlink without
``X-Object-Symlink-Target-Container`` between containers, or a symlink
without ``X-Object-Symlink-Target-Account`` between accounts, the new
symlink will refer to a different object.

4.2 Versioned Containers
------------------------

These will definitely interact. We should probably figure out how.


4.3 Object Expiration
---------------------

There's nothing special here. If you create the symlink with
``X-Delete-At``, the symlink will get deleted at the appropriate time.

If you use a plain POST to set ``X-Delete-At`` on a symlink, it gets
set on the referenced object just like other object metadata. If you
use POST with ``?symlink=true`` to set ``X-Delete-At`` on a symlink,
it will be set on the symlink itself.


4.4 Large Objects
-----------------

Since we'll almost certainly end up implementing symlinks as
middleware, we'll order the pipeline like this::

  [pipeline:main]
  pipeline = catch_errors ... slo dlo symlink ... proxy-server

This way, you can create a symlink whose target is a large object
*and* a large object can reference symlinks as segments.

This also works if we decide to implement symlinks in the proxy
server, though that would only happen if a compelling reason were
found.


4.5 User Authorization
----------------------

Authorization will be checked for both the symlink and the referenced
object. If the user is authorized to see the symlink but not the
referenced object, they'll get a 403, same as if they'd tried to
access the referenced object directly.


4.6. Quotas
-----------

Nothing special needed here. A symlink counts as 1 object toward an
object-count quota. Since symlinks are zero bytes, they do not count
toward a storage quota, and we do not need to write any code to make
that happen.


4.7 list_endpoints / Hadoop / ZeroVM
------------------------------------

If the application talks directly to the object server and fetches a
symlink, it's up to the application to deal with it. Applications that
bypass the proxy should either avoid use of symlinks or should know
how to handle them.

The same is true for SLO, DLO, versioning, erasure codes, and other
services that the Swift proxy server provides, so we are not without
precedent here.


4.8 Container Sync
------------------

Symlinks are synced like every other object. If the referenced object
in cluster A has a different container name than in cluster B, then
the symlink will point to the wrong place in one of the clusters.

Intra-container symlinks (those with only
``X-Object-Symlink-Target-Object``) will work correctly on both
clusters. Also, if containers are named identically on both clusters,
inter-container symlinks (those with
``X-Object-Symlink-Target-Object`` and
``X-Object-Symlink-Target-Container``) will work correctly too.


4.9 Bulk Uploads
----------------

Currently, bulk uploads ignore all non-file members in the uploaded
tarball. This could be expanded to also process symbolic-link members
(i.e. those for which ``tarinfo.issym() == True``) and create symlink
objects from them. This is not necessary for the initial
implementation of Swift symlinks, but it would be nice to have.

4.10 Swiftclient
----------------

python-swiftclient could download Swift symlinks as Unix symlinks if a
flag is given, or it could upload Unix symlinks as Swift symlinks in
some cases. This is not necessary for the initial implementation of
Swift symlinks, and is mainly mentioned here to show that
python-swiftclient was not forgotten.
