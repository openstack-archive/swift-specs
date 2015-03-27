::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Changing Policy of Containers
=============================

Our proposal is to give swift users power to change storage policies of
containers and objects which are contained in those containers.

Problem description
===================

Swift currently prohibits users from changing containers' storage policies so
this constraint raises at least two problems.

One problem is the flexibility. For example, there is an organization using
Swift as a backup storage of office data and all data is archived monthly in a
container named after date like 'backup-201502'. Older archive becomes less
important so users want to reduce the consumed capacity to store it. Then Swift
users will try to change the storage policy of the container into cheaper one
like '2-replica policy' or 'EC policy' but they will be strongly
disappointed to find out that they cannot change the policy of the container
once created. The workaround for this problem is creating other new container
with other storage policy then copying all objects from an existing container
to it but this workaround raises another problem.

Another problem is the reachability. Copying all files to other container
brings about the change of all files' URLs. That makes users confused and
frustrated. The workaround for this problem is that after copying all files to
new container, users delete an old container and create the same name container
again with other storage policy then copy all objects back to the original name
container. However this obviously involves twice as heavy workload and long
time as a single copy.

Proposed change
===============

The ring normally differs from one policy to another so 'a/c/o' object of
policy 1 is likely to be placed in devices of different nodes from 'a/c/o'
object of policy 0. Therefore, objects replacement associated with the policy
change needs very long time and heavy internal traffic. For this reason,
an user request to change a policy must be translated
into asynchronous behavior of transferring objects among storage nodes which is
driven by background daemons. Obviously, Swift must not suspend any
user's requests to store or get information during changing policies.

We need to add or modify Swift servers' and daemons' behaviors as follows:

**Servers' changes**

1. Adding POST container API to send a request for changing a storage policy
   of a container
#. Adding response headers for GET/HEAD container API to notify how many
   objects are placed in a new policy or still in an old policy
#. Modifying GET/HEAD object API to get an object even if replicas are placed
   in a new policy or in an old policy

**Daemons' changes**

1. Adding container-replicator a behavior to watch a container which is
   requested to change its storage policy
#. Adding a new background daemon which transfers objects among storage nodes
   from an old policy to a new policy

Servers' changes
----------------

1. Add New Behavior for POST Container
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Currently, Swift returns "204 No Content" for the user POST container request
with X-Storage-Policy header. This indicates "nothing done." For the purpose
of maintaining backward compatibility and avoiding accidental execution, we
prefer to remain this behavior unchanged. Therefore, we propose introducing the
new header to 'forcibly' execute policy changing as follows.

.. list-table:: Table 1: New Request Header to change Storage Policy
   :widths: 30 8 12 50
   :header-rows: 1

   * - Parameter
     - Style
     - Type
     - Description
   * - X-Forced-Change-Storage-Policy: <policy_name> (Optional)
     - header
     - xsd:string
     - Change a storage policy of a container to the policy specified by
       'policy_name'. This change accompanies asynchronous background process
       to transfer objects.

Possible responses for this API are as follows.

.. list-table:: Table 2: Possible Response Codes for the New Request
   :widths: 2 8
   :header-rows: 1

   * - Code
     - Notes
   * - 202 Accepted
     - Accept the request properly and start to prepare objects replacement.
   * - 400 Bad Request
     - Reject the request with a policy which is deprecated or is not defined
       in a configuration file.
   * - 409 Conflict
     - Reject the request because another changing policy process is not
       completed yet (relating to 3-c change)

When a request of changing policies is accepted (response code is 202), a
target container stores following two sysmetas.

.. list-table:: Table 3: Container Sysmetas for Changing Policies
   :widths: 2 8
   :header-rows: 1

   * - Sysmeta
     - Notes
   * - X-Container-Sysmeta-Prev-Index: <int>
     - "Pre-change" policy index. It will be used for GET or DELETE objects
       which are not transferred to the new policy yet.
   * - X-Container-Sysmeta-Objects-Queued: <bool>
     - This will be used for determining the status of policy changing by
       daemon processes. If False, policy change request is accepted but not
       ready for objects transferring. If True, objects have been queued to the
       special container for policy changing so those are ready for
       transferring. If undefined, policy change is not requested to that
       container.

This feature should be implemented as middleware 'change-policy' because of
the following two reasons:

1. This operation probably should be authorized only to limitted group
   (e.g., swift cluster's admin (reseller_admin)) because this operation
   occurs heavy internal traffic.
   Therefore, authority of this operation should be managed in the middleware
   level.
#. This operation needs to POST sysmetas to the container. Sysmeta must be
   managed in middleware level according to Swift's design principle

2. Add Response Headers for GET/HEAD Container
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Objects will be transferred gradually by backend processes. From the viewpoint
of Swift operators, it is important to know the progress of policy changing,
that is, how many objects are already transferred or still remain
untransferred. This can be accomplished by simply exposing policy_stat table of
container DB file for each storage policy. Each policy's stat will be exposed
by ``X-Container-Storage-Policy-<Policy_name>-Bytes-Used`` and
``X-Container-Storage-Policy-<Policy_name>-Object-Count`` headers as follows::

  $ curl -v -X HEAD -H "X-Auth-Token: tkn" http://<host>/v1/AUTH_test/container
  < HTTP/1.1 200 OK
  < X-Container-Storage-Policy-Gold-Object-Count: 3
  < X-Container-Storage-Policy-Gold-Bytes-Used: 12
  < X-Container-Storage-Policy-Ec42-Object-Count: 7
  < X-Container-Storage-Policy-Ec42-Bytes-Used: 28
  < X-Container-Object-Count: 10
  < X-Container-Bytes-Used: 40
  < Accept-Ranges: bytes
  < X-Storage-Policy: ec42
  < ...

Above response indicates 70% of object transferring is done.

3. Modify Behavior of GET/HEAD object API
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In my current consideration, object PUT should be done only to the new policy.
This does not affect any object in the previous policy so this makes the
process of changing policies simple.
Therefore, the best way to get an object is firstly sending a GET request to
object servers according to the new policy's ring, and if the response code is
404 NOT FOUND, then a proxy resends GET requests to the previous policy's
object servers.

However, this behavior is in discussion because sending GET/HEAD requests twice
to object servers can increase the latency of user's GET object request,
especially in the early phase of changing policies.

Daemons' changes
----------------

1. container-replicator
^^^^^^^^^^^^^^^^^^^^^^^

To enqueue objects to the list for changing policies, some process must watch
what a container is requested for changing its policy. Adding this task to
container-replicator seems best way because container-replicator originally
has a role to seek all container DBs for sanity check of Swift cluster.
Therefore, this can minimize extra time to lock container DBs for adding this
new feature.

Container-replicator will check if a container has
``X-Container-Sysmeta-Objects-Queued`` sysmeta and its value is False. Objects
in that container should be enqueued to the object list of a special container
for changing policies. That special container is created under the special
account ``.change_policy``. The name of a special container should be unique
and one-to-one relationship with a container to which policy changing is
requested. The name of a special container is simply defined as
``<account_name>:<container_name>``. This special account and containers are
accessed by the new daemon ``object-transferrer``, which really transfers
objects from the old policy to the new policy.

2. object-transferrer
^^^^^^^^^^^^^^^^^^^^^

Object-transferrer is newly introduced daemon process for changing policies.
Object-transferrer reads lists of special containers from the account
``.change_policy`` and reads lists of objects from each special container.
Object-transferrer transfers those objects from the old policy to the new
policy by using internal client. After an object is successfully transferred
to the new policy, an object in the old policy will be deleted by DELETE
method.

If transferrer finishes to transfer all objects in a special container, it
deletes a special container and deletes sysmetas
``X-Container-Sysmeta-Prev-Index`` and ``X-Container-Sysmeta-Objects-Queued``
from a container to change that container's status from IN-CHANGING to normal
(POLICY CHANGE COMPLETED).

Example
-------

.. list-table:: Table 4: Example of data transition during changing policies
   :widths: 1 4 2 4 2
   :header-rows: 1

   * - Step
     - Description
     - Container /a/c
       objects
     - Container /a/c/ metadata
     - Container /.change_policy/a:c
       objects
   * - | 0
     - | Init.
     - | ('o1', 1)
       | ('o2', 1)
       | ('o3', 1)
     - | X-Backend-Storage-Policy-Index: 1
     - | N/A
   * - | 1
     - | POST /a/c X-Forced-Change-Storage-Policy: Pol-2
     - | ('o1', 1)
       | ('o2', 1)
       | ('o3', 1)
     - | X-Backend-Storage-Policy-Index: 2
       | X-Container-Sysmeta-Prev-Policy-Index: 1
       | X-Container-Sysmeta-Objects-Queued: False
     - | N/A
   * - | 2
     - | container-replicator seeks policy changing containers
     - | ('o1', 1)
       | ('o2', 1)
       | ('o3', 1)
     - | X-Backend-Storage-Policy-Index: 2
       | X-Container-Sysmeta-Prev-Policy-Index: 1
       | X-Container-Sysmeta-Objects-Queued: True
     - | ('o1', 0, 'application/x-transfer-1-to-2')
       | ('o2', 0, 'application/x-transfer-1-to-2')
       | ('o3', 0, 'application/x-transfer-1-to-2')
   * - | 3
     - | object-transferrer transfers 'o1' and 'o3'
     - | ('o1', 2)
       | ('o2', 1)
       | ('o3', 2)
     - | X-Backend-Storage-Policy-Index: 2
       | X-Container-Sysmeta-Prev-Policy-Index: 1
       | X-Container-Sysmeta-Objects-Queued: True
     - | ('o2', 0, 'application/x-transfer-1-to-2')
   * - | 4
     - | object-transferrer transfers 'o2'
     - | ('o1', 2)
       | ('o2', 2)
       | ('o3', 2)
     - | X-Backend-Storage-Policy-Index: 2
       | X-Container-Sysmeta-Prev-Policy-Index: 1
       | X-Container-Sysmeta-Objects-Queued: True
     - | Empty
   * - | 5
     - | object-transferrer deletes a special container and metadatas from
         container /a/c
     - | ('o1', 2)
       | ('o2', 2)
       | ('o3', 2)
     - | X-Backend-Storage-Policy-Index: 2
     - | N/A

Above table focuses data transition of a container in changing a storage policy
and a corresponding special container. A tuple indicates object info, first
element is an object name, second one is a policy index and third one, if
available, is a value of content-type, which is defined for policy changing.

Given that three objects are stored in the container ``/a/c`` as policy-1
(Step 0). When the request to change this container's
policy to policy-2 is accepted (Step 1), a backend policy index will be
changed to 2 and two sysmetas are stored in this container. In the periodical
container-replicator process, replicator finds a container with policy change
sysmetas and then creates a special container ``/.change_policy/a:c`` with
a list of objects (Step 2). Those objects have info of old policy and new policy
with the field of content-type. When object-transferrer finds this special
container from ``.change_policy`` account, it gets some objects from the old
policy (usually from a local device) and puts them to the new policy's storage
nodes (Step 3 and 4). If the special container becomes empty (Step 5), it
indicates policy changing for that container finished so the special container
is deleted and policy changing metadatas of an original container are also
deleted.

Alternatives: As Sub-Function of Container-Reconciler
-----------------------------------------------------

Container-reconciler is a daemon process which restores objects registered in
an incorrect policy into a correct policy. Therefore, the reconciling procedure
satisfies almost all of functional requirements for policy changing. The
advantage of using container-reconciler for policy changing is that we need to
modify a very few points of existing Swift sources. However, there is a big
problem to use container-reconciler. This problem is that container-reconciler
has no function to determine the completeness of changing policy of objects
contained in a specific container. As a result, this problem makes it
complicated to handle GET/HEAD object from the previous policy and to allow
the next storage policy change request. Based on discussion in Swift hack-a-thon
(held in Feb. 2015) and Tokyo Summit (held in Oct. 2015), we decided to add
object-transferrer to change container's policy.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Daisuke Morita (dmorita)

Milestones
----------

Target Milestone for completion:
  Mitaka

Work Items
----------

* Add API for Policy Changing

  * Add a middleware 'policy-change' to process Container POST request with
    "X-Forced-Change-Storage-Policy" header. This middleware stores sysmeta
    headers to target container DB for policy changing.
  * Modify container-server to add response headers for Container GET/HEAD
    request to show the progress of changing policies by exposing all the info
    from policy_stat table
  * Modify proxy-server (or add a feature to new middleware) to get object for
    referring both new and old policy index to allow users' object read during
    changing policy

* Add daemon process among storage nodes for policy changing

  * Modify container-replicator to watch a container if it should be initialized
    (creation of a corresponding special container) for changing policies
  * Write object-transferrer code
  * Daemonize object-transferrer

* Add unit, functional and probe tests to check that new code works
  intentionally and that it is OK for splitted brain cases

