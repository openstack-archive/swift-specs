::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

=====================================
tempurls with a prefix-based scope
=====================================

The tempurl middleware should be allowed to use a prefix-based signature, which grants access for
all objects with this specific prefix. This allows access to a whole container or pseudofolder
with one signature, instead of using a new signature for each object.


Problem Description
===================

At the moment, if one wants to share a large amount of objects inside a container/pseudofolder
with external people, one has to create temporary urls for each object. Additionally, objects which
are placed inside the container/pseudofolder after the generation of the signature cannot
be accessed with the same signature.
Prefix-based signatures would  allow to reuse the same signature for a large amount of objects
which share the same prefix. 

Use cases:

1. We have one pseudofolder with 1000000 objects. We want to share this pseudofolder with external
   partners. Instead of generating 1000000 different signatures, we only need to generate one
   signature.
2. We have an webbased-application on top of swift like the swiftbrowser
   (https://github.com/cschwede/django-swiftbrowser), which acts as a filebrowser. We want to
   support the sharing of temporary pseudofolders with external people. We do not know in advance
   which and how many objects will live inside the pseudofolder.
   With prefix-based signatures, we could develop the webapplication in a way so that the user
   could generate a temporary url for one pseudofolder, which could be used by external people
   for accessing all objects which will live inside it
   (this use-case additionaly needs a temporary container listing, to display which objects live
   inside the pseudofolder and a modification of the formpost middleware, please see spec
   https://review.openstack.org/#/c/225059/).


Proposed Change
===============

The temporary url middleware should be changed. The code change should not be too big.
If the client desires to use a prefix-based signature, he can append an URL parameter
"temp_url_prefix"  with the desired prefix (an empty prefix would specify the whole container),
and the middleware would only use the container path + prefix for calculating the signature.
Furthermore, the middleware would check if the object path really contains this prefix.

Lets have two examples. In the first example, we want to allow a user to upload a bunch of objects
in a container c.
He first creates a tempurl, for example using the swift command line tool
(modified version which supports tempurls on container-level scope):
::

 $swift tempurl --container-level PUT 86400 /v1/AUTH_account/c/ KEY
 /v1/AUTH_account/c/?temp_url_sig=9dd9e9c318a29c6212b01343a2d9f9a4c9deef2d&temp_url_expires=1439280760&temp_url_prefix=

The user then uploads a bunch of files using each time the same container-level signature:
::

 $curl -XPUT --data-binary @file1 https://example.host/v1/AUTH_account/c/o1?temp_url_sig=9dd9e9c318a29c6212b01343a2d9f9a4c9deef2d&temp_url_expires=1439280760&temp_url_prefix=
 $curl -XPUT --data-binary @file2 https://example.host/v1/AUTH_account/c/o2?temp_url_sig=9dd9e9c318a29c6212b01343a2d9f9a4c9deef2d&temp_url_expires=1439280760&temp_url_prefix=
 $curl -XPUT --data-binary @file3 https://example.host/v1/AUTH_account/c/p/o3?temp_url_sig=9dd9e9c318a29c6212b01343a2d9f9a4c9deef2d&temp_url_expires=1439280760&temp_url_prefix=

In the next example, we want to allow an external user to download a whole pseudofolder p:
::

 $swift tempurl --container-level GET 86400 /v1/AUTH_account/c/p KEY

 /v1/AUTH_account/c/p?temp_url_sig=4e755839d19762e06c12d807eccf46ff3224cb3f&temp_url_expires=1439281346&temp_url_prefix=p

 $curl https://example.host/v1/AUTH_account/c/p/o1?temp_url_sig=4e755839d19762e06c12d807eccf46ff3224cb3f&temp_url_expires=1439281346&temp_url_prefix=p
 $curl https://example.host/v1/AUTH_account/c/p/o2?temp_url_sig=4e755839d19762e06c12d807eccf46ff3224cb3f&temp_url_expires=1439281346&temp_url_prefix=p
 $curl https://example.host/v1/AUTH_account/c/p/p2/o3?temp_url_sig=4e755839d19762e06c12d807eccf46ff3224cb3f&temp_url_expires=1439281346&temp_url_prefix=p

Following requests would be denied, because of missing/wrong prefix:
::

 $curl https://example.host/v1/AUTH_account/c/o4?temp_url_sig=4e755839d19762e06c12d807eccf46ff3224cb3f&temp_url_expires=1439281346&temp_url_prefix=p
 $curl https://example.host/v1/AUTH_account/c/p3/o5?temp_url_sig=4e755839d19762e06c12d807eccf46ff3224cb3f&temp_url_expires=1439281346&temp_url_prefix=p


Alternatives
------------

A new middleware could be introduced. But it seems that this would only lead to a lot of
code-copying, as the changes are really small in comparison to the original middleware.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bartz

Work Items
----------

Add modifications to tempurl and respective test module.

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

Modify documentation for tempurl middleware.

Security
--------
The calculation of the signature uses the hmac module (https://docs.python.org/2/library/hmac.html)
in combination with the sha1 hash function.
The difference of a prefix-based signature to the current object-path-based signature is, that
the path is shrunk to the prefix. The remaining part of the calculation stays the same.
A shorter path induces a shorter message as input to the hmac calculation, which should not reduce
the cryptographic strength. Therefore, I do not see security-related problems with introducing
a prefix-based signature.

Testing
-------

Tests should be added to the existing test module.

Dependencies
============

None
