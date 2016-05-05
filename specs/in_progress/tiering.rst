::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

*************************
Automated Tiering Support
*************************

1. Problem Description
======================
Data hosted on long-term storage systems experience gradual changes in
access patterns as part of their information lifecycles. For example,
empirical studies by companies such as Facebook show that as image data
age beyond their creation times, they become more and more unlikely to be
accessed by users, with access rates dropping exponentially at times [1].
Long retention periods, as is the case with data stored on cold storage
systems like Swift, increase the possibility of such changes.

Tiering is an important feature provided by many traditional file & block
storage systems to deal with changes in data “temperature”. It enables
seamless movement of inactive data from high performance storage media to
low-cost, high capacity storage media to meet customers’ TCO (total cost of
ownership) requirements. As scale-out object storage systems like Swift are
starting to natively support multiple media types like SSD, HDD, tape and
different storage policies such as replication and erasure coding, it becomes
imperative to complement the wide range of available storage tiers (both
virtual and physical) with automated data tiering.


2. Tiering Use Cases in Swift
=============================
Swift users and operators can adapt to changes in access characteristics of
objects by transparently converting their storage policies to cater to the
goal of matching overall business needs ($/GB, performance, availability) with
where and how the objects are stored.

Some examples of how objects can be moved between Swift containers of different
storage policies as they age.

[SSD-based container] --> [HDD-based container]

[HDD-based container] --> [Tape-based container]

[Replication policy container] -->  [Erasure coded policy container]

In some customer environments, a Swift container may not be the last storage
tier. Examples of archival-class stores lower in cost than Swift include
specialized tape-based systems [2], public cloud archival solutions such as
Amazon Glacier and Google Nearline storage. Analogous to this proposed feature
of tiering in Swift, Amazon S3 already has the in-built support to move
objects between S3 and Glacier based on user-defined rules. Redhat Ceph has
recently added tiering capabilities as well.


3. Goals
========
The main goal of this document is to propose a tiering feature in Swift that
enables seamless movement of objects between containers belonging to different
storage policies. It is “seamless” because users will not experience any
disruption in namespace, access API, or availability of the objects subject to
tiering.

Through new Swift API enhancements, Swift users and operators alike will have
the ability to specify a tiering relationship between two containers and the
associated data movement rules.

The focus of this proposal is to identify, create and bring together the
necessary building blocks towards a baseline tiering implementation natively
within Swift. While this narrow scope is intentional, the expectation is that
the baseline tiering implementation will lay the foundation and not preclude
more advanced tiering features in future.

4. Feature Dependencies
=======================
The following in-progress Swift features (aka specs) have been identified as
core dependencies for this tiering proposal.

1. Swift Symbolic Links [3]
2. Changing Storage Policies [4]

A few other specs are classified as nice-to-have dependencies, meaning that
if they evolve into full implementations we will be able to demonstrate the
tiering feature with advanced use cases and capabilities. However, they are
not considered mandatory requirements for the first version of tiering.

3. Metadata storage/search [5]
4. Tape support in Swift [6]

5. Implementation
=================
The proposed tiering implementation depends on several building blocks, some
of which are unique to tiering, like the requisite API changes. They will be
described in their entirety. Others like symlinks are independent features and
have uses beyond tiering. Instead of re-inventing the wheel, the tiering
implementation aims to leverage specific constructs that will be available
through these in-progress features.

5.1 Overview
------------
For a quick overview of the tiering implementation, please refer to the Figure
(images/tiering_overview.png). It highlights the flow of actions taking place
within the proposed tiering engine.

1. Swift client creates a tiering relationship between two Swift containers by
marking the source container with appropriate metadata.
2. A background process named tiering-coordinator examines the source container
and iterates through its objects.
3. Tiering-coordinator identifies candidate objects for movement and de-stages
each object to target container by issuing a copy request to an object server.
4. After an object is copied, tiering-coordinator replaces it by a symlink in
the source container pointing to corresponding object in target container.


5.2 API Changes
---------------
Swift clients will be able to create a tiering relationship between two
containers, i.e., source and target containers, by adding the following
metadata to the source container.

X-Container-Tiering-Target: <target_container_name>
X-Container-Tiering-Age: <threshold_object_age >

The metadata values can be set during the creation of the source container
(PUT) operation or they can be set later as part of a container metadata
update (POST) operation. Object age refers to the time elapsed since the
object’s creation time (creation time is stored with the object as
‘X-Timestamp’ header).

The user semantics of setting the above container metadata are as follows.
When objects in the source container become older than the specified threshold
time, they become candidates for being de-staged to the target container. There
are no guarantees on when exactly they will be moved or the precise location of
the objects at any given time. Swift will operate on them asynchronously and
relocate objects based on user-specified tiering rules. Once the tiering
metadata is set on the source container, the user can expect levels of
performance, reliability, etc. for its objects commensurate with the storage
policy of either the source or target container.

One can override the tiering metadata for individual objects in the source
container by setting the following per-object metadata,

X-Object-Tiering-Target: <target_container_name>
X-Object-Tiering-Age: <object_age_in_minutes>

Presence of tiering metadata on an object will imply that it will take
precedence over the tiering metadata set on the hosting container. However,
if a container is not tagged with any tiering metadata, the objects inside it
will not be considered for tiering regardless of whether they are tagged with
any tiering related metadata or not. Also, if the tiering age threshold on the
object metadata is lower than the value set on the container, it will not take
effect until the container age criterion is met.

An important invariant preserved by the tiering feature is the namespace of
objects. As will be explained in later sections, after objects are moved they
will be replaced immediately by symlinks that will allow users to continue
foreground operations on objects as if no migrations have taken place. Please
refer to section 7 on open questions for further commentary on the API topic.

To summarize, here are the steps that a Swift user must perform in order to
initiate tiering between objects from a source container (S) to a target
container (T) over time.

1. Create containers S and T with desired storage policies, say replication
and erasure coding respectively
2. Set the tiering-related metadata (X-Container-Tiering-*) on container S
as described earlier in this section.
3. Deposit objects into container S.
4. If needed, override the default container settings for individual objects
inside container S by setting object metadata (X-Object-Tiering-*).

It will also be possible to create cascading tiering relationships between
more than two containers. For example, a sequence of tiering relationships
between containers C1 -> C2 -> C3 can be established by setting appropriate
tiering metadata on C1 and C2. When an object is old enough to be moved from
C1, it will be deposited in C2. The timer will then start on the moved object
in C2 and depending on the age settings on C2, the object will eventually be
migrated to C3.


5.3 Tiering Coordinator Process
-------------------------------
The tiering-coordinator is a background process similar to container-sync,
container-reconciler and other container-* processes running on each container
server. We can potentially re-use one of the existing container processes,
specifically either container-sync or container-reconciler to perform the job of
tiering-coordinator, but for the purposes of this discussion it will be assumed
that it is a separate process.

The key actions performed by tiering-coordinator are

(a) Walk through containers marked with tiering metadata
(b) Identify candidate objects for tiering within those containers
(c) Initiate copy requests on candidate objects
(d) Replace source objects with corresponding symlinks

We will discuss (a) and (b) in this section and cover (c) and (d) in subsequent
sections. Note that in the first version of tiering, only one metric
<object age> will be used to determine the eligibility of an object for
migration.

The tiering-coordinator performs its operations in a series of rounds. In each
round, it iterates through containers whose SQLite DBs it has direct access to
on the container server it is running on. It checks if the container has the
right X-Container-Tiering-* metadata. If present, it starts the scanning process
to identify candidate objects. The scanning process leverages a convenient (but
not necessary) property of the container DB that objects are listed in the
chronological order of their creation times. That is, the first index in the
container DB points to the object with oldest creation time, followed by next
younger object and so on. As such, the scanning process described below is
optimized for the object age criterion chosen for tiering v1 implementation.
For extending to other tiering metrics, we refer the reader to section 6.1 for
discussion.

Each container DB will have two persistent markers to track the progress of
tiering – tiering_sync_start and tiering_sync_end. The marker tiering_sync_start
refers to the starting index in the container DB upto which objects have already
been processed. The marker tiering_sync_end refers to the index beyond which
objects have not yet been considered for tiering. All the objects that fall
between the two markers are the ones for which tiering is currently in progress.
Note that the presence of persistent markers in the container DB helps with
quickly resuming from previous work done in the event of container server
crash/reboot.

When a container is selected for tiering for the first time, both the markers
are initialized to -1. If the first object is old enough to meet the
X-Container-Tiering-Age criterion, tiering_sync_start is set to 0. Then the
second marker tiering_sync_end is advanced to an index that is lesser than
the two values  - (i) tiering_sync_start + tier_max_objects_per_round (latter
will be a configurable value in /etc/swift/container.conf) or (ii) largest
index in the container DB whose corresponding object meets the tiering age
criterion.

The above marker settings will ensure two invariants. First, all objects
between (and including) tiering_sync_start and tiering_sync_end are candidates
for moving to the target container. Second, it will guarantee that the number
of objects processed on the container in a single round is bound by the
configuration parameter (tier_max_objects_per_round, say = 200). This will
ensure that the coordinator process will round robin effectively amongst all
containers on the server per round without spending undue amount of time on
only a few.

After the markers are fixed, tiering-coordinator will issue a copy request
for each object within the range. When the copy requests are completed, it
updates tiering_sync_start = tiering_sync_end and moves on to the next
container. When tiering-coordinator re-visits the same container after
completing the current round, it restarts the scanning routine described
above from tiering_sync_start = tiering_sync_end (except they are not both
-1 this time).

In a typical Swift cluster, each container DB is replicated three times and
resides on multiple container servers. Therefore, without proper
synchronization, tiering-coordinator processes can end up conflicting with
each other by processing the same container and same objects within. This
can potentially lead to race conditions with non-deterministic behavior. We
can overcome this issue by adopting the approach of divide-and-conquer
employed by container-sync process. The range of object indices between
(tiering_sync_start, tiering_sync_end) can be initially split up into as
many disjoint regions as the number of tiering-coordinator processes
operating on the same container. As they work through the object indices,
each process might additionally complete others’ portions depending on the
collective progress. For a detailed description of how container-sync
processes implicitly communicate and make group progress, please refer
to [7].

5.4 Object Copy Mechanism
-------------------------
For each candidate object that the tiering-coordinator deems eligible to move to
the target container, it issues an ‘object copy’ request using an API call
supported by the object servers. The API call will map to a method used by
object-transferrer daemons running on the object servers. The
tiering-coordinator can select any of the object servers (by looking up the ring
datastructure corresponding to the object in source container policy) as a
destination for the request.

The object-transferrer daemon is supposed to be optimized for converting an
object from one storage policy to another. As per the ‘Changing policies’ spec,
the object-transferrer daemon will be equipped with the right techniques to move
objects between Replication -> EC, EC -> EC, etc. Alternatively, in the absence
of object-transferrer, the tiering coordinator can simply make use of the
server-side ‘COPY’ API that vanilla Swift exposes to regular clients. It can
send the COPY request to a swift proxy server to clone the source object into
the target container. The proxy server will perform the copy by first reading in
(GET request) the object from any of the source object servers and creating a
copy (PUT request) of the object in the target object servers. While this will
work correctly for the purposes of the tiering coordinator, making use of the
object-transferrer interface is likely to be a better option. Leveraging the
specialized code in object-transferrer through a well-defined interface for
copying an object between two different storage policy containers will make the
overall tiering process efficient.

Here is an example interface represented by a function call in the
object-transferrer code:

def  copy_object(source_obj_path, target_obj_path)

The above method can be a wrapper over similar functionality used by the
object-transferrer daemon. The tiering-coordinator will use this interface to
call the function through a HTTP call.

copy_object(/A/S/O, /A/T/O)

where S is the source container and T is the target container. Note that the
object name in the target container will be the same as in the source container.

Upon receiving the copy request, the object server will first check if the
source path is a symlink object. If it is a symlink, it will respond with an
error to the tiering-coordinator to indicate that a symlink already exists.
This behavior will ensure idempotence and guard against situations where
tiering-coordinator crashes and retries a previously completed object copy
request. Also, it avoids tiering for sparse objects such as symlinks created
by users. Secondly, the object server will check if the source object has
tiering metadata in the form of X-Object-Tiering-* that overrides the default
tiering settings on the source container. It may or may not perform the object
copy depending on the result.

5.5 Symlink Creation
--------------------
After an object is successfully copied to the destination container, the
tiering-coordinator will issue a ‘symlink create’ request to proxy server to
replace the source object by a reference to the destination object. Waiting
until the object copy is completed before replacing it by a symlink ensures
safety in case of failures. The system could end up with an extra target
object without a symlink pointing to it, but not the converse which
constitutes data loss. Note that the symlink feature is currently
work-in-progress and will also be available as an external API to swift clients.

When the symlink is created by the tiering-coordinator, it will need to ensure
that the original object’s ‘X-Timestamp’ value is preserved on the symlink
object. Therefore, it is proposed that in the symlink creation request, the
original time field can be provided (tiering-coordinator can quickly read the
original values from container DB entry) as object user metadata, which is
translated internally to a special sysmeta field by the symlink middleware.
On subsequent user requests, the sysmeta field storing the correct creation
timestamp will be sent to the user.

With the symlink successfully created, Swift users can continue to issue object
requests like GET, PUT to the original namespace /Account/Container/Object. The
Symlink middleware will ensure that the swift users do not notice the presence
of a symlink object unless a query parameter ‘?symlink=true’ [3] is explicitly
provided with the object request.

Users can also continue to read and update object metadata as before. It is not
entirely clear at the time of this writing if the symlink object will store a
copy of user metadata in its own extended attributes or if it will fetch the
metadata from the referenced object for every HEAD/GET on the object. We will
defer to whichever implementation that the symlink feature chooses to provide.

An interesting race condition is possible due to the time window between object
copy request and symlink creation. If there is an interim PUT request issued by
a swift user between the two, it will be overwritten by the internal symlink
created by the tiering-coordinator. This is an incorrect behavior that we need
to protect against. We can use the same technique [8] (with help of a second
vector timestamp) that container-reconciler uses to resolve a similar race
condition. The tiering-coordinator, at the time of symlink creation, can detect
the race condition and undo the COPY request. It will have to delete the object
that was created in the destination container. Though this is wasted work in
the face of such race conditions, we expect them to be rare scenarios. If the
user conceives tiering rules properly, there ought to be little to no
foreground traffic for the object that is being tiered.

6. Future Work
===============

6.1 Other Tiering Criteria
--------------------------
The first version of tiering implementation will be heavily tailored (especially
the scanning mechanism of tiering-coordinator) to the object age criterion. The
convenient property of container DBs that store objects in the same order as
they are created/overwritten lends to very efficient linear scanning for
candidate objects.

In the future, we should be able to support advanced criteria such as read
frequency counts, object size, metadata-based selection, etc. For example,
consider the following hypothetical criterion:

"Tier objects from container S to container T if older than 1 month AND size >
1GB AND tagged with metadata ‘surveillance-video’"

When the metadata search feature [5] is available in Swift, tiering-coordinator
should be able to run queries to quickly retrieve the set of object names that
match ad-hoc criteria on both user and system metadata. As the metadata search
feature evolves, we should be able to leverage it to add custom metadata such
as read counts, etc for our purposes.

6.2 Integration with External Storage Tiers
-------------------------------------------
The first implementation of tiering will only support object movement between
Swift containers. In order to establish a tiering relationship between a swift
container and an external storage backend, the backend must be mounted in Swift
as a native container through the DiskFile API or other integration mechanisms.
For instance, a target container fully hosted on GlusterFS or Seagate Kinetic
drives can be created through Swift-on-file or Kinetic DiskFile implementations
respectively.

The Swift community believes that a similar integration approach is necessary
to support external storage systems as tiering targets. There is already work
underway to integrate tape-based systems in Swift. In the same vein, future
work is needed to integrate external systems like Amazon Glacier or vendor
archival products via DiskFile drivers or other means.

7. Open Issues
==============
This section is structured as a series of questions and possible answers. With
more feedback from the swift community, the open issues will be resolved and
merged into the main document.

Q1: Can the target container exist on a different account than the source
container?

Ans: The proposed API assumes that the target container is always on the same
account as the source container. If this restriction is lifted, the proposed
API needs to be modified appropriately.

Q2: When the client sets the tiering metadata on the source container, should
the target container exist at that time? What if the user has no permissions on
the target container? When is all the error checking done?

Ans: The error checking can be deferred to the tiering-coordinator process. The
background process, upon detecting that the target container is unavailable can
skip performing any tiering activity on the source container and move on to the
next container. However, it might be better to detect errors in the client path
and report early. If the latter approach is chosen, middleware functionality is
needed to sanity check tiering metadata set on containers.

Q3: How is the target container presented to the client? Would it be just like
any other container with read/write permissions?

Ans: The target container will be just like any other container. The client is
responsible for manipulating the contents in the target container correctly. In
particular, it should be aware that there might be symlinks in source container
pointing to target objects. Deletions or overwrites of objects directly using
the target container namespace could render some symlinks useless or obsolete.

Q4: What is the behavior when conflicting tiering metadata are set over a
period of time. For example, if the tiering age threshold is increased on a
container with a POST metadata operation, will previously de-staged objects
be brought back to the source container to match the new tiering rule?

Ans: Perhaps not. The new tiering metadata should probably only be applied to
objects that have not yet been processed by tiering-coordinator. Previous
actions performed by tiering-coordinator based on older metadata need not be
reversed.

Q5: When a user issues a PUT operation to an object that has been de-staged to
the target container earlier, what is the behavior?

Ans: The default symlink behavior should apply but it’s not clear what it will
be. Will an overwrite PUT cause the symlink middleware to delete both the
symlink and the object being pointed to?

Q6: When a user issues a GET operation to an object that has been de-staged to
the target container earlier, will it be promoted back to source container?

Ans: The proposed implementation does not promote objects back to an upper tier
seamless to the user. If needed, such a behavior can be easily added with help
of a tiering middleware in the proxy server.

Q7: There is a mention of the ability to set cascading tiering relationships
between multiple containers, C1 -> C2 -> C3. What if there is a cycle in this
relationship graph?

Ans: A cycle should be prevented, else we can run into atleast one complicated
situation where a symlink might be pointing to an object on the same container
with the same name, thereby overwriting the symlink ! It is possible to detect
cycles at the time of tiering metadata creation in the client path with a
tiering-specific middleware that is entrusted with the cycle detection by
iterating through existing tiering relationships.

Q8: Are there any unexpected interactions of tiering with existing or new
features like SLO/DLO, encryption, container sharding, etc ?

Ans: SLO and DLO segments should continue to work as expected. If an object
server receives an object copy request for a SLO manifest object from a
tiering-coordinator, it will iteratively perform the copy for each constituent
object. Each constituent object will be replaced by a symlink. Encryption
should also work correctly as it is almost entirely orthogonal to the tiering
feature. Each object is treated as an opaque set of bytes by the tiering engine
and it does not pay any heed to whether the object is cipher text or not.
Dealing with container sharding might be tricky. Tiering-coordinator expects
to linearly walk through the indices of a container DB. If the container DB
is fragmented and stored in many different container servers, the scanning
process can get complicated. Any ideas there?

8. References
=============

1.  http://www.enterprisetech.com/2013/10/25/facebook-loads-innovative-cold-storage-datacenter/
2.  http://www-03.ibm.com/systems/storage/tape/
3.  Symlinks in Swift. https://review.openstack.org/#/c/173609/
4.  Changing storage policies in Swift. https://review.openstack.org/#/c/168761/
5.  Add metadata search in Swift. https://review.openstack.org/#/c/180918/
6.  Tape support in Swift. https://etherpad.openstack.org/p/liberty-swift-tape-storage
7.  http://docs.openstack.org/developer/swift/overview_container_sync.html
8.  Container reconciler section at http://docs.openstack.org/developer/swift/overview_policies.html
