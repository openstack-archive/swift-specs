::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Improve Erasure Coding Efficiency for Global Cluster
=====================================================

This SPEC describes an improvement of efficiency for Global Cluster with
Erasure Coding. It proposes a way to improve the PUT/GET performance
in the case of Erasure Coding with more than 1 regions ensuring original
data even if a region is lost.

Problem description
===================

Swift now supports Erasure Codes (EC) which ensures higher durability and lower
disk cost than the replicated case for a one region cluster. However, currently
if Swift were running EC over 2 regions, using < 2x data redundancy
(e.g. ec_k=10, ec_m=4) and then one of the regions is gone due to some unfortunate
reasons (e.g. huge earthquake, fire, tsunami), there is a chance data would be lost.
That is because, assuming each region has an even available volume of disks, each
region should have around 7 fragments, less than ec_k, which is not enough data
for the EC scheme to rebuild the original data.

To protect stored data and to ensure higher durability, Swift has to keep >= 1
data size for each region (i.e. >= 2x  for 2 regions) by employing larger ec_m
like ec_m=14 for ec_k=10. However, this increase sacrifices encode performance.
In my measurements running PyECLib encode/decode on an Intel Xeon E5-2630v3 [1], the
benchmark result was as follows:

+----------------+----+----+---------+---------+
|scheme          |ec_k|ec_m|encode   |decode   |
+================+====+====+=========+=========+
|jerasure_rs_vand|10  |4   |7.6Gbps  |12.21Gbps|
+----------------+----+----+---------+---------+
|                |10  |14  |2.67Gbps |12.27Gbps|
+----------------+----+----+---------+---------+
|                |20  |4   |7.6Gbps  |12.87Gbps|
+----------------+----+----+---------+---------+
|                |20  |24  |1.6Gbps  |12.37Gbps|
+----------------+----+----+---------+---------+
|isa_lrs_vand    |10  |4   |14.27Gbps|18.4Gbps |
+----------------+----+----+---------+---------+
|                |10  |14  |6.53Gbps |18.46Gbps|
+----------------+----+----+---------+---------+
|                |20  |4   |15.33Gbps|18.12Gbps|
+----------------+----+----+---------+---------+
|                |20  |24  |4.8Gbps  |18.66Gbps|
+----------------+----+----+---------+---------+

Note that "decode" uses (ec_k + ec_m) - 2 fragments so performance will
decrease less than when encoding as is shown in the results above.

In the results above, comparing ec_k=10, ec_m=4 vs ec_k=10, ec_m=14, the encode
performance falls down about 1/3 and other encodings follow a similar trend.
This demonstrates that there is a problem when building a 2+ region EC cluster.

1: http://ark.intel.com/ja/products/83356/Intel-Xeon-Processor-E5-2630-v3-20M-Cache-2_40-GHz

Proposed change
===============

Add an option like "duplication_factor". Which will create duplicated (copied)
fragments instead of employing a larger ec_m.

For example, with a duplication_factor=2, Swift will encode ec_k=10, ec_m=4 and
store 28 fragments (10x2 data fragments and 4x2 parity fragments) in Swift.

This requires a change to PUT/GET and the reconstruct sequence to map from the
fragment index in Swift to actual fragment index for PyECLib but IMHO we don't
need to make an effort to build much modification for the conversation among
proxy-server <-> object-server <-> disks.

I don't want describe the implementation in detail in the first patch of the spec
because it should be an idea to improve Swift. More discussion on the implementation
side will following in subsequent patches.

Considerations of acutal placement
----------------------------------
Placement of these doubled fragments are important. If the same fragments,
original and copied, appear in the same region and the second region fails,
then we would be in the same situation where we couldn't rebuild the original
object as we were in the smaller parity fragments case.`

e.g:

- duplication_factor=2, k=4, m=2
- 1st Region: [0, 1, 2, 6, 7, 8]
- 2nd Region: [3, 4, 5, 9, 10, 11]
- (Assuming actual indices to rebuild mapped as index // (k+m))

In this case, 1st region has only fragments consisting of fragment index 0, 1, 2
and 2nd has only 3, 4, 5. Therefore, it is not able to rebuild the original object
from the fragments in only one region because the fragment uniqueness in the
region is less than k. The worst case scenario, like this, will cause significant data
loss as would happen with no duplication factor.

i.e. In fact, data durability will be

- "no duplication" < "with duplication" < "more unique parities"

In future work, we can find a way to tie a fragment index to a region,
something like "1st subset should be in 1st Region and 2nd subset
should be ..." but so far this is beyond this spec.

Alternatives
------------

We can find a way to use container-sync as a solution to the problem rather
then employing my proposed change.
This section will describe the pros/cons for my "proposed change" and "container-sync".

Proposed Change
^^^^^^^^^^^^^^^
Pros:

- Higher performance way to spread objects across regions (No need to re-decode/encode for transferring across regions)
- No extra configuration other than storage policy is needed for users to turn on the global replication. (strictly global erasure coding?)
- Able to use other global cluster efficiency improvements (affinity control)

Cons:

- Need to employ more complex handling around ECObjecController

Container-Sync
^^^^^^^^^^^^^^
Pros:

- Simple and able to reuse existing swift mechanisms
- Less data transfer between regions

Cons:

- Re-decode/encode is required when transferring objects to another region
- Need to set the sync option for each container
- Impossible to retrieve/reconstruct an object when > ec_m disks unavailable (includes ip unreachable)


Implementation
==============

- Proxy-Server PUT/GET path
- Object-Reconstructor
- (Optional) Ring placement strategy

Questions and Answers
=====================

- TBD

Assignee(s)
-----------

Primary assignee:
  kota\_ (Kota Tsuyuzaki)

Work Items
----------

Develop codes around proxy-server and object-reconstructor

Repositories
------------

None

Servers
-------

None

DNS Entries
-----------

None

Dependencies
============

None
