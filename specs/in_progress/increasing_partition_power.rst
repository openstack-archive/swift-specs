::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode


===============================
Increasing ring partition power
===============================

This document describes a process and modifications to swift code that
together enable ring partition power to be increased without cluster downtime.

Swift operators sometimes pick a ring partition power when deploying swift
and later wish to change the partition power:

 #. The operator chooses a partition power that proves to be too small and
    subsequently constrains their ability to rebalance a growing cluster.
 #. Perhaps more likely, in an attempt to avoid the above problem, operators
    choose a partition power that proves to be unnecessarily large and would
    subsequently like to reduce it.

This proposal directly addresses the first problem by enabling partition power
to be increased. Although it does not directly address the second problem
(i.e. it does not enable ring power reduction), it does indirectly help to
avoid that problem by removing the motivation to choose large partition power
when first deploying a cluster.

Problem Description
===================

The ring power determines the partition to which a resource (account, container
or object) is mapped. The partition is included in the path under which the
resource is stored in a backend filesystem. Changing the partition power
therefore requires relocating resources to new paths in backend filesystems.

In a heavily populated cluster a relocation process could be time-consuming and
so to avoid down-time it is desirable to relocate resources while the cluster
is still operating. However, it is necessary to do so without (temporary) loss
of access to data and without compromising the performance of processes such as
replication and auditing.

Proposed Change
===============

Overview
--------

The proposed solution avoids copying any file contents during a partition power
change. Objects are 'moved' from their current partition to a new partition,
but the current and new partitions are arranged to be on the same device, so
the 'move' is achieved using filesystem links without copying data.

(It may well be that the motivation for increasing partition power is to allow
a rebalancing of the ring. Any rebalancing would occur after the partition
power increase has completed - during partition power changes the ring balance
is not changed.)

To allow the cluster to continue operating during a partition power change (in
particular, to avoid any disruption or incorrect behavior of the replicator and
auditor processes), new partition directories are created in a separate
filesystem branch from the current partition directories. When all new
partition directories have been populated, the ring transitions to using the
new filesystem branch.

During this transition, object servers maintain links to resource files from
both the current and new partition directories. However, as already discussed,
no file content is duplicated or copied. The old partition directories are
eventually deleted.

Detailed description
--------------------

The process of changing a ring's partition power comprises three phases:

1. Preparation - during this phase the current partition directories continue
   to be used but existing resources are also linked to new partition
   directories in anticipation of the new ring partition power.

2. Switchover - during this phase the ring transitions to using the new
   partition directories; proxy and backend servers rollover to using the new
   ring partition power.

3. Cleanup - once all servers are using the new ring partition power,
   resource files in old partition directories are removed.

For simplicity, we describe the details of each phase in terms of an object
ring but note that the same process can be applied to account and container
rings and servers.

Preparation phase
^^^^^^^^^^^^^^^^^

During the preparation phase two new attributes are set in the ring file:

 * the ring's `epoch`: if not already set, a new `epoch` attribute is added to
   the ring. The ring epoch is used to determine the parent directory for
   partition directories. Similar to the way in which a ring's policy index is
   appended to the `objects` directory name, the epoch will be prefixed to the
   `objects` directory name. For simplicity, the ring epoch will be a
   monotonically increasing integer starting at 0. A 'legacy' ring having no
   epoch attribute will be treated as having epoch 0.

 * the `next_part_power` attribute indicates the partition power that will be
   used in the next epoch of the ring. The `next_part_power` attribute is used
   during the preparation phase to determine the partition directory in which
   an object should be stored in the next epoch of the ring.

At this point in time no other changes are made to the ring file:
the current part power and the mapping of partitions to devices are unchanged.

The updated ring file is distributed to all servers.  During this preparation
phase, proxy servers will continue to use the current ring partition mapping to
determine the backend url for objects. Object servers, along with replicator
and auditor processes, also continue to use the current ring
parameters. However, during PUT and DELETE operations object servers will
create additional links to object files in the object's future partition
directory in preparation for an eventual switchover to the ring's next
epoch. This does not require any additional copying or writing of object
contents.

The filesystem path for future partition directories is determined as follows.
In general, the path to an object file on an object server's filesystem has the
form::

  dev/[<epoch>-]objects[-<policy>]/<partition>/<suffix>/<hash>/<ts>.<ext>

where:

 * `epoch` is the ring's epoch, if non-zero
 * `policy` is the object container's policy index, if non-zero
 * `dev` is the device to which `partition` is mapped by the ring file
 * `partition` is the object's partition,
   calculated using `partition = F(hash) >> (32 - P)`,
   where `P` is the ring partition power
 * `suffix` is the last three digits of `hash`
 * `hash` is a hash of the object name
 * `ts` is the object timestamp
 * `ext` is the filename extension (`data`, `meta` or `ts`)

Given `next_part_power` and `epoch` in the ring file, it is possible to
calculate::

  future_partition = F(hash) >> (32 - next_part_power)
  next_epoch = epoch + 1

The future partition directory is then::

  dev/<next_epoch>-objects[-<policy>]/<next_partition>/<suffix>/<hash>/<ts>.<ext>

For example, consider a ring in its first epoch, with current partition power
P, containing an object currently in partition X, where 0 <= X < 2**P. If the
partition power increases by a factor of 2, the object's future partition will
be either 2X or 2X+1 in the ring's next epoch. During a DELETE an additional
filesystem link will be created at one of::

  dev/1-objects/<2X>/<suffix>/<hash>/<ts>.ts
  dev/1-objects/<2X+1>/<suffix>/<hash>/<ts>.ts

Once object servers are known to be using the updated ring file a new relinker
process is started. The relinker prepares an object server's filesystem for a
partition power change by crawling the filesystem and linking existing objects
to future partition directories. The relinker determines each object's future
partition directory in the same way as described above for the object server.

The relinker does not remove links from current partition directories. Once the
relinker has successfully completed, every existing object should be linked
from both a current partition directory and a future partition directory. Any
subsequent object PUTs or DELETEs will be reflected in both the current and
future partition directory as described above.

To avoid newly created objects being 'lost', it is important that an object
server is using the updated ring file before the relinker process starts in
order to guarantee that either the object server or the relinker create future
partition links for every object. This may require object servers to be
restarted prior to the relinker process being started, or to otherwise report
that they have reloaded the ring file.

The relinker will report successful completion in a file
`/var/cache/swift/relinker.recon` that can be queried via (modified) recon
middleware.

Once the relinker process has successfully completed on all object servers, the
partition power change process may move on to the switchover phase.

Switchover phase
^^^^^^^^^^^^^^^^

To begin the switchover to using the next partition power, the ring file is
updated once more:

 * the current partition power is stored as `previous_part_power`
 * the current partition power is set to `next_partition_power`
 * `next_partition_power` is set to None
 * the ring's `epoch` is incremented
 * the mapping of partitions to devices is re-created so that partitions 2X and
   2X+1 map to the same devices to which partition X was mapped in the previous
   epoch. This is a simple transformation. Since no object content is moved
   between devices the actual ring balance remains unchanged.

The updated ring file is then distributed to all proxy and object servers.

Since ring file distribution and loading is not instantaneous, there is a
window of time during which a proxy server may direct object requests to either
an old partition or a current partition (note that the partitions previously
referred to as 'future' are now referred to as 'current').  Object servers will
therefore create additional filesystem links during PUT and DELETE requests,
pointing from old partition directories to files in the current partition
directories. The paths to the old partition directories are determined in the
same way as future partition directories were determined during the preparation
phase, but now using the `previous_part_power` and decrementing the current
ring `epoch`.

This means that if one proxy PUTs an object using a current partition, then
another proxy subsequently attempts to GET the object using the old partition,
the object will be found, since both current and old partitions map to the same
device. Similarly if one proxy PUTs an object using the old partition and
another proxy then GETs the object using the current partition, the object will
be found in the current partition on the object server.

The object auditor and replicator processes are restarted to force reloading of
the ring file and commence to operate using the current ring parameters.

Cleanup phase
^^^^^^^^^^^^^

The cleanup phase may start once all servers are known to be using the updated
ring file. Once again, this may require servers to be restarted or to report
that they have reloaded the ring file during switchover.

A final update is made to the ring file: the `previous_partition_power`
attribute is set to `None` and the ring file is once again distributed. Once
object servers have reloaded the update ring file they will cease to create
object file links in old partition directories.

At this point the old partition directories may be deleted - there is no need
to create tombstone files when deleting objects in the old partitions since
these partition directories are no longer used by any swift process.

A cleanup process will crawl the filesystem and delete any partition
directories that are not part of the current epoch or a future epoch. This
cleanup process should repeat periodically in case any devices that were
offline during the partition power change come back online - the old epoch
partition directories discovered on those devices may be deleted. Normal
replication may cause current epoch partition directories to be created on a
resurrected disk.

(The cleanup function could be added to an existing process such as the
auditor).

Other considerations
--------------------

swift-dispersion-[populate|report]
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The swift-dispersion-[populate|report] tools will need to be made epoch-aware.
After increasing partition power, swift-dispersion-populate may need to be
run to achieve the desired coverage. (Although initially the device coverage
will remain unchanged, the percentage of partitions covered will have reduced
by whatever factor the partition power has increased.)

Auditing
^^^^^^^^

During preparation and switchover, the auditor may find a corrupt object. The
quarantine directory is not in the epoch partition directory filesystem branch,
so a quarantined object will not be lost when old partitions are deleted.

The quarantining of an object in a current partition directory will not remove
the object from a future partition, so after switchover the auditor will
discover the object again, and quarantine it again. The diskfile quarantine
renamer could optionally be made 'relinker' aware and unlink duplicate object
references when quarantining an object.


Alternatives
------------

Prior work
^^^^^^^^^^

The swift_ring_tool_ enables ring power increases while swift services are
disabled. It takes a similar approach to this proposal in that the ring
mapping is changed so that every resource remains on the same device when
moved to its new partition. However, new partitions are created in the
same filesystem branch as existing (hence the need for services to be suspended
during the relocation).

.. _swift_ring_tool: https://github.com/enovance/swift-ring-tool/

Previous proposals have been made to upstream swift:

https://bugs.launchpad.net/swift/+bug/933803 suggests a 'same-device'
partition re-mapping, as does this proposal, but did not provide for
relocation of resources to new partition directories.

https://review.openstack.org/#/c/21888/ suggests maintaining a partition power
per device (so only new devices use the increase partition power) but appears
to have been abandoned due to complexities with replication.


Create future partitions in existing `objects[-policy]` directory
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The duplication of filesystem entries for objects and creation of (potentially
duplicate) partitions during the preparation phase could have undesirable
effects on other backend processes if they are not isolated in another
filesystem branch.

For example, the object replicator is likely to discover newly created future
partition directories that appear to be 'misplaced'. The replicator will
attempt to sync these to their primary nodes (according to the old ring
mapping) which is unnecessary. Worse, the replicator might then delete the
future partitions from their current nodes, undoing the work of the relinker
process.

If the replicator were to adopt the future ring mappings from the outset of the
preparation phase then the same problems arise with respect to current
partitions that now appear to be misplaced. Furthermore, the replication
process is likely to race with the relinker process on remote nodes to
populate future partitions: if relocation proceeds faster on node A than B then
the replicator may start to sync objects from A to B, which is again
unnecessary and expensive.

The auditor will also be impacted as it will discover objects in the future
partition directories and audit them, being unable to distinguish them as
duplicates of the object still stored in the current partition.

These issues could of course be avoided by disabling replication and auditing
during the preparation phase, but instead we propose to make the future ring
partition naming be mutually exclusive from current ring partition naming, and
simply restrict the replicator and auditor to only process partitions that are
in the current ring partition set. In other words we isolate these processes
from the future partition directories that are being created by the relinker.


Use mutually exclusive future partitions in existing `objects` directory
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The current algorithm for calculating the partition for an object is to
calculate a 32 bit hash of the object and then use its P most significant bits,
resulting in partitions in the range {0, 2**P - 1}. i.e.::

  part = H(object name) >> (32 - P)

A ring with partition power P+1 will re-use all the partition numbers of a ring
with partition power P.

To eliminate overlap of future ring partitions with current ring partitions we
could change the partition number algortihm to add an offset to each partition
number when a ring's partition power is increased:

offset = 2**P part = (H(object name) >> (32 - P)) + offset

This is backwards compatible: if `offset` is not defined in a ring file then it
is set to zero.

To ensure that partition numbers remain < 2**32, this change will reduce the
maximum partition power from 32 to 31.

Proxy servers start to use the new ring at outset of relocation phase
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This would mean that GETs to backends would use the new rings partitions in
object urls. Objects may not yet have been relocated to their new partition
directory and the object servers would therefore need to fall back to looking
in the old ring partition for the object. PUTs and DELETEs to the new partition
would need to be made conditional upon a newer object timestamp not existing in
the old location. This is more complicated than the proposed method.

Enable partition power reduction
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Ring power reduction is not easily achieved with the approach presented in this
proposal because there is no guarantee that partitions in the current epoch
that will be merged into partitions in the next epoch are located on the same
device. File contents are therefore likely to need copying between devices
during a preparation phase.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  alistair.coles@hp.com

Work Items
----------

 #. modify ring classes to support new attributes
 #. modify ringbuilder to manage new attributes
 #. modify backend servers to duplicate links to files in future epoch partition
    directories
 #. make backend servers and relinker report their status in a way that recon
    can report e.g. servers report when a new ring epoch has been loaded, the
    relinker reports when all relinking has been completed.
 #. make recon support reporting these states
 #. modify code that assumes storage-directory is objects[-policy_index] to
    be aware of epoch prefix
 #. make swift-dispersion-populate and swift-dispersion-report epoch-aware
 #. implement relinker daemon
 #. document process

Repositories
------------

No new git repositories will be created.

Servers
-------

No new servers are created.

DNS Entries
-----------

No DNS entries will to be created or updated.

Documentation
-------------

Process will be documented in the administrator's guide. Additions will be made
to the ring-builder documents.

Security
--------

No security issues are foreseen.

Testing
-------

Unit tests will be added for changes to ring-builder, ring classes and
object server.

Probe tests will be needed to verify the process of increasing ring power.

Functional tests will be unchanged.


Dependencies
============

None
