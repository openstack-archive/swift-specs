::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===============================
PACO Single Process deployments
===============================

Since the release of the DiskFile API, there's been a number of different
implementations providing the ability of storing Swift objects in the
third-party storage systems. Commonly these systems provide the durability and
availability of the objects (e.g., GlusterFS, GPFS), thus requiring the object
ring to be created with only one replica.

A typical deployment style for this configuration is a "PACO" deployment,
where the proxy, account, container and object services are running on the same
node. The object ring is built in a such a way that the proxy server always
send requests to the local object server. The object server (with it's 
third-party DiskFile) is then responsible for writing the data to the underlying
storage system which will then distribute the data according to its own
policies.

Problem description
===================

In a typical swift deployment, proxy nodes send data to object
servers running on different nodes and the object servers write the data
directly to disk. In the case of third-party storage systems, the object server
typically makes another network connection to send the object to that storage
system, adding some latency to the data path.

Even when the proxy and object servers are on the same node, latency is still
introduced due to RPC communication over local network.

Proposed change
===============

For the scenario of single replica - PACO deployments, the proxy server would
be sending data directly to the third-party storage systems. To accomplish this
we would like to call the object wsgi application directly from
the proxy process instead of making the additional network connection.

This proposed solution focuses on reducing the proxy to object server latency
Proxy to account and/or container communications would stay the same for now
and be addressed on later patch.

Assignee(s)
-----------

Primary assignee:
  thiago@redhat.com

Work Items
----------

A WiP patch has been submitted: https://review.openstack.org/#/c/159285/.
The work that has been done recently to the Object Controllers in the proxy
servers provides the ability for a very nice separation of the code.

TODOs and where further investigation is needed:

* How to load the object WSGI application instance in the proxy process?
* How to add support for multiple storage policies?

Prototype
---------

To test patch `159285 <https://review.openstack.org/#/c/159285/>`_ follow these
steps:

#. Create new single replica storage system. Update swift.conf and create new
   ring. The port provided during ring creation will not be used for anything. 
#. Create an object-server config file: ``/etc/swift/single-process.conf``.
   This configuration file can look like any other object-server configuration
   file, just make sure it specifies the correct device the object server
   should be writing to. For example, in the case of `Swift-on-File <https://github.com/stackforge/swiftonfile>`_
   object server, the device is the mountpoint of the shared filesystem (i.e.,
   Gluster, GPFS).
#. Start the proxy.
