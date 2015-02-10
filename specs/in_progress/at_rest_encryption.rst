::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

******************
At-Rest Encryption
******************

1. Summary
==========

To better protect the data in their clusters, Swift operators may wish
to have objects stored in an encrypted form. This spec describes a
plan to add an operator-managed encryption capability to Swift while
remaining completely transparent to clients.

Goals
-----

Swift objects are typically stored on disk as files in a standard
POSIX filesystem; in the typical 3-replica case, an object is
represented as 3 files on 3 distinct filesystems within the cluster.

An attacker may gain access to disks in a number of ways. When a disk
fails, it may be returned to the manufacturer under warranty; since it
has failed, erasing the data may not be possible, but the data may
still be present on the platters. When disks reach end-of-life, they
are discarded, and if not properly wiped, may still contain data. An
insider might steal or clone disks from the data center.

Goal 1: an attacker who gains read access to Swift's object servers'
filesystems should gain as little useful data as possible. This
provides confidentiality for users' data.

Goal 2: when a keymaster implementation allows for secure deletion of keys,
then the deletion of an object's key shall render the object irrecoverable.
This provides a means to securely delete an object.

Not Goals / Possible Future Work
--------------------------------

There are other ways to attack a Swift cluster, but this spec does not
address them. In particular, this spec does not address these threats:

  * an attacker gains access to Swift's internal network
  * an attacker compromises the key database
  * an attacker modifies Swift's code (on the Swift nodes) for evil

If these threats are mitigated at all, it is a fortunate byproduct, but it is
not the intent of this spec to address them.


2. Encryption and Key Management
================================

There are two logical parts to at-rest encryption. The first part is
the crypto engine; this performs the actual encryption and decryption
of the data and metadata.

The second part is key management. This is the process by which the
key material is stored, retrieved and supplied to the crypto engine.
The process may be split with an agent responsible for storing key
material safely (sometimes a Hardware Security Module) and an agent
responsible for retrieving key material for the crypto engine. Swift
will support a variety of key-material retrievers, called
"keymasters", via Python's entry-points mechanism. Typically, a Swift
cluster will use only one keymaster.

2.1 Request Path
----------------

The crypto engine and the keymaster shall be implemented as three
separate pieces of middleware. The crypto engine shall have both
"decrypter" and "encrypter" filter-factory functions, and the
keymaster filter shall sit between them. Example::

    [pipeline:main]
    pipeline = catch_errors gatekeeper ... decrypter keymaster encrypter proxy-logging proxy-server

The encrypter middleware is responsible for encrypting the object's
data and metadata on a PUT or POST request.

The decrypter middleware is responsible for three things. First, it
decrypts the object's data and metadata on an object GET or HEAD
response. Second, it decrypts the container listing entries and the
container metadata on a container GET or HEAD response. Third, it
decrypts the account metadata on an account GET or HEAD response.

DELETE requests are unaffected by encryption, so neither
the encrypter nor decrypter need to do anything. The keymaster may
wish to delete any key or keys associated with the deleted entity.

OPTIONS requests should be ignored entirely by the crypto engine, as
OPTIONS requests and responses contain neither user data nor user
metadata.

The decrypter, encrypter, and keymaster will all use the hook in the
WSGI environment at "swift.copy_hook" to ensure that COPY requests
result in the new object being encrypted with its own key, not with
the source object's key (assuming that they differ).

2.1.1 Large Objects
-------------------

In Swift, large objects are composed of segments, which are plain old
objects, and a manifest, which is a special object that ties the
segments together. Here, "special" means "has a particular header
value".

Large-object support is implemented in middlewares ("dlo" and "slo").
The encrypter/keymaster/decrypter trio must be placed to the right of
the dlo and slo middlewares in the proxy's middleware pipeline. This
way, the encrypter and decrypter do not have to do any special
processing for large objects; rather, each request is for a plain old
object, container, or account.

2.1.2 Etag Validation
---------------------

With unencrypted objects, the object server is responsible for
validating any Etag header sent by the client on a PUT request; the
Etag header's value is the MD5 hash of the uploaded object data.

With encrypted objects, the plaintext is not available to the object server, so
the encrypter must perform the validation instead by calculating the MD5 hash
of the object data and validating this against any Etag header sent by the
client - if the two do not match then the encrypter should immediately return a
response with status 422.

Assuming that the computed MD5 hash of plaintext is validated, the encrypter
will encrypt this value and pass to the object server to be stored as system
metadata. Since the validated value will not be available until the plaintext
stream has been completely read, this metadata will be sent using a 'request
footer', as described in section 7.2.

If the client request included an Etag header then the encrypter should also
compute the MD5 hash of the ciphertext and include this value in an Etag
request footer. This will allow the object server to validate the hash of the
ciphertext that it receives, and so complete the end-to-end validation
requirement implied by the client sending an Etag: encrypter validates client
to proxy communication, object server validates proxy to object server
communication.


2.2 Inter-Middleware Communication
----------------------------------

The keymaster is responsible for deciding if any particular resource should be
encrypted. This decision is implementation dependent but may be based, for
example, on container policy or account name. When a resource is not to be
encrypted the keymaster will set the key `swift.crypto.override` in the request
environ to indicate to the encrypter middleware that encryption is not
required.

When encryption is required, the keymaster communicates the encryption key to
the encrypter and decrypter middlewares by placing a zero-argument callable in
the WSGI environment dictionary at the key "swift.crypto.fetch_crypto_keys".
When called, this will return the key(s) necessary to process the current
request. It must be present on any GET or HEAD request for an account,
container, or object which contains any encrypted data or metadata. If
encrypted data or metadata is encountered while processing a GET or HEAD
request but fetch_crypto_keys is not present _or_ it does not return keys when
called, then this is an error and the client will receive a 500-series
response.

On a PUT or POST request, the keymaster must place
"swift.crypto.fetch_crypto_keys" in the WSGI environment during request
processing; that is, before passing the request to the remainder of the
middleware pipeline. This is so that the encrypter can encrypt the object's
data in a streaming fashion without buffering the whole object.

On a GET or HEAD request, the keymaster must place
"swift.crypto.fetch_crypto_keys" in the WSGI environment before returning
control to the decrypter. It need not be done at request-handling time. This
lets attributes of the key be stored in sysmeta, for example the key ID in an
external database, or anything else the keymaster wants.


3. Cipher Choice
================

3.1. The Chosen Cipher
----------------------

Swift will use AES in CTR mode with 256-bit keys.

In order to allow for ranged GET requests, the cipher shall be used
in counter (CTR) mode.

The entire object body shall be encrypted as a single byte stream. The
initialization vector (IV) used for encrypting the object body will be randomly
generated and stored in system metadata.


3.2. Why AES-256-CTR
--------------------

CTR mode basically turns a block cipher into a stream cipher, so
dealing with range GET requests becomes much easier. No modification
of the client's requested byte ranges is needed. When decrypting, some
padding will be required to align the requested data to AES's 16-byte
block size, but that can all be done at the proxy level.

Remember that when a GET request is made, the decrypter knows nothing
about the object. The object may or may not be encrypted; it may or
may not exist. If Swift were to allow configurable cipher modes, then
the requested byte range would have to be expanded to get enough bytes
for any supported cipher mode at all, which means taking into account
the block size and operating characteristics of every single supported
cipher/blocksize/mode. Besides the network overhead (especially for
small byteranges), the complexity of the resulting code would make it
an excellent home for bugs.

3.3 Future-Proofing
-------------------

The cipher and mode will be stored in system metadata on every
encrypted object. This way, when Swift gains support for other ciphers
or modes, existing objects can still be decrypted.

In general we must assume that any resource (account/container/object metadata
or object data) in a Swift cluster may be encrypted using a different cipher,
or not encrypted. Consequently, the cipher choice must be stored as metadata of
every encrypted resource, along with the IV. Since user metadata may be updated
independently of objects, this implies storing encryption related metadata of
metadata.


4. Robustness
=============


4.1 No Key
----------

If the keymaster fails to add "swift.crypto.fetch_crypto_keys" to the WSGI
environment of a GET request, then the client would receive the ciphertext of
the object instead of the plaintext, which looks to the client like garbage.
However, we can tell if an object is encrypted or not by the presence of system
metadata headers, so the decrypter can prevent this by raising an error if no
key was provided for the decryption of an encrypted object.


5. Multiple Keymasters
======================

5.1 Coexisting Keymasters
-------------------------

Just as Swift supports multiple simultaneous auth systems, it can
support multiple simultaneous keymasters. With auth, each auth system
claims a subset of the Swift namespace by looking at accounts starting
with their reseller prefix. Similarly, multiple keymasters may
partition the Swift namespace in some way and thus coexist peacefully.

5.2 Keymasters in Core Swift
----------------------------

5.2.1 Trivial Keymaster
^^^^^^^^^^^^^^^^^^^^^^^

Swift will need a trivial keymaster for functional tests of the crypto
engine. The trivial keymaster will not be suitable for production use
at all. To that end, it should be deliberately kept as small as
possible without regard for any actual security of the keys.

Perhaps the trivial keymaster could use the SHA-256 of a configurable
prefix concatenated with the object's full path for the cryptographic
key. That is,::

    key = SHA256(prefix_from_conf + request.path)

This will allow for testing of the PUT and GET paths, the COPY path
(the destination object's key will differ from the source object's),
and also the invalid key path (by changing the prefix after an object
is PUT).


5.2.2 Barbican Keymaster
^^^^^^^^^^^^^^^^^^^^^^^^

Swift will probably want a keymaster that stores things in Barbican at
some point.


6 Encryption of Object Body
===========================

Each object is encrypted with the key from the keymaster. A new IV is
randomly generated by the encrypter for each object body.

The IV and the choice of cipher is stored using sysmeta. For the following
discussion we shall refer to the choice of cipher and IV collectively as
"crypto metadata".

The crypto metadata for object body can be stored as an item of sysmeta that
the encrypter adds to the object PUT request headers, e.g.::

  X-Object-Sysmeta-Crypto-Meta: "{'iv': 'xxx', 'cipher': 'AES_CTR_256'}"

.. note::
    Here, and in following examples, it would be possible to omit the
    ``'cipher'`` keyed item from the crypto metadata until a future
    change introduces alternative ciphers. The existence of any crypto metadata
    is sufficient to infer use of the 'AES_CTR_256' unless otherwise specified.


7. Metadata Encryption
======================

7.1 Background
--------------

Swift entities (accounts, containers, and objects) have three kinds of
metadata.

First, there is basic object metadata, like Content-Length, Content-Type, and
Etag. These are always present and user-visible.

Second, there is user metadata. These are headers starting with
X-Object-Meta-, X-Container-Meta-, or X-Account-Meta- on objects,
containers, and accounts, respectively. There are per-entity limits on
the number, individual sizes, and aggregate size of user metadata.
User metadata is optional; if present, it is user-visible.

Third and finally, there is system metadata, often abbreviated to
"sysmeta". These are headers starting with X-Object-Sysmeta-,
X-Container-Sysmeta-, and X-Account-Sysmeta-. There are _no_ limits on
the number or aggregate sizes of system metadata, though there may be
limits on individual datum sizes due to HTTP header-length
restrictions. System metadata is not user-visible or user-settable; it
is intended for use by Swift middleware to safely store data away from
the prying eyes and fingers of users.


7.2 Basic Object Metadata
-------------------------

An object's plaintext etag and content type are sensitive information and will
be stored encrypted, both in the container listing and in the object's
metadata. To accomplish this, the encrypter middleware will actually encrypt
the etag and content type _twice_: once with the object's key, and once with
the container's key.

There must be a different IV used for each different encrypted header.
Therefore, crypto metadata will be stored for the etag and content_type::

  X-Object-Sysmeta-Crypto-Meta-ct: "{'iv': 'xxx', 'cipher': 'AES_CTR_256'}"
  X-Object-Sysmeta-Crypto-Meta-Etag: "{'iv': 'xxx', 'cipher': 'AES_CTR_256'}"

The object-key-encrypted values will be sent to the object server using
``X-Object-Sysmeta-Crypto-Etag`` and ``Content-Type`` headers that will be
stored in the object's metadata.

The container-key-encrypted etag and content-type values will be sent to the
object server using header names ``X-Backend-Container-Update-Override-Etag``
and ``X-Backend-Container-Update-Override-Content-Type`` respectively. Existing
object server behavior is to then use these values in the ``X-Etag`` and
``X-Content-Type`` headers included with the container update sent to the
container server.

When handling a container GET request, the decrypter must process the container
listing and decrypt every occurrence of an Etag or Content-Type using the
container key. When handling an object GET or HEAD, the decrypter must decrypt
the values of ``X-Object-Sysmeta-Crypto-Etag`` and
``X-Object-Sysmeta-Crypto-Content-Type`` using the object key and copy these
value to the ``Etag`` and ``Content-Type`` headers returned to the client.

This way, the client sees the plaintext etag and content type in container
listings and in object GET or HEAD responses, just like it would without
encryption enabled, but the plaintext values of those are not stored anywhere.

.. note::
    The encrypter will not know the value of the plaintext etag until it has
    processed all object content. Therefore, unless the encrypter buffers the
    entire object ciphertext (!) it cannot send the encrypted etag headers to
    object servers before the request body. Instead, the encrypter will emit a
    multipart MIME document for the request body and append the encrypted etag
    as a 'request footer'. This mechanism will build on the use of
    multipart MIME bodies in object server requests introduced by the Erasure
    Coding feature [1].

For basic object metadata that is encrypted (i.e. etag and content-type), the
object data crypto metadata will apply, since this basic metadata is only set
by an object PUT. However, the encrypted copies of basic object metadata that
are forwarded to container servers with container updates will require
accompanying crypto metadata to also be stored in the container server DB
objects table. To avoid significant code churn in the container server, we
propose to append the crypto metadata to the basic metadata value string.

For example, the Etag header value included with a container update will have
the form::

  Etag: E(CEK, <etag>); meta={'iv': 'xxx', 'cipher': 'AES_CTR_256'}

where ``E(CEK, <etag>)`` is the ciphertext of the object's etag encrypted with
the container key (``CEK``).

When handling a container GET listing, the decrypter will need to parse each
etag value in the listing returned from the container server and transform its
value to the plaintext etag expected in the response to the client. Since a
'regular' plaintext etag is a fixed length string that cannot contain the ';'
character, the decrypter will be able to easily differentiate between an
unencrypted etag value and an etag value with appended crypto metadata that by
design is always longer than a plaintext etag.

The crypto metadata appended to the container update etag will also be valid
for the encrypted content-type ``E(CEK, <content-type>)`` since both are set at
the same time. However, other proposed work [2] makes it possible to update the
object content-type with a POST, meaning that the crypto metadata associated
with content-type value could be different to that associated with the etag. We
therefore propose to similarly append crypto metadata in the content-type value
that is destined for the container server:

   Content-Type: E(CEK, <content-type>); meta="{'iv': 'yyy', 'cipher': 'AES_CTR_256'}"

In this case the use of the ';' separator character will allow the decrypter to
parse content-type values in container listings and remove the crypto metadata
attribute.

7.2.1 A Note On Etag
^^^^^^^^^^^^^^^^^^^^

In the stored object's metadata, the basic-metadata field named "Etag"
will contain the MD5 hash of the ciphertext. This is required so that
the object server will not error out on an object PUT, and also so
that the object auditor will not quarantine the object due to hash
mismatch (unless bit rot has happened).

The plaintext's MD5 hash will be stored, encrypted, in system
metadata.


7.3 User Metadata
-----------------

Not only the contents of an object are sensitive; metadata is sensitive too.
Since metadata values must be valid UTF-8 strings, the encrypted values will be
suitably encoded (probably base64) for storage. Since this encoding may
increase the size of user metadata values beyond the allowed limits, the
metadata limit checking will need to be implemented by the encrypter
middleware. That way, users don't see lower metadata-size limits when
encryption is in use. The encrypter middleware will set a request environ key
`swift.constraints.override` to indicate to the proxy-server that limit
checking has already been applied.

User metadata names will *not* be encrypted. Since a different IV (or indeed a
different cypher) may be used each time metadata is updated by a POST request,
encrypting metadata names would make it impossible for Swift to delete
out-dated metadata items. Similarly, if encryption is enabled on an existing
Swift cluster, encrypting metadata names would prevent previously unencrypted
metadata being deleted when updated.

For each piece of user metadata on objects we need to store crypto metadata,
since all user metadata items are encrypted with a different IV. This cannot
be stored as an item of sysmeta since sysmeta cannot be updated by an object
POST. We therefore propose to modify the object server to persist the headers
``X-Object-Massmeta-Crypto-Meta-*`` with the same semantic as ``X-Object-Meta-*``
headers i.e. ``X-Object-Massmeta-Crypto-Meta-*`` will be updated on every POST
and removed if not present in a POST. The gatekeeper middleware will prevent
``X-Object-Massmeta-Crypto-Meta-*`` headers ever being included in client
requests or responses.

The encrypter will add a ``X-Object-Massmeta-Crypto-Meta-<key>`` header 
to object PUT and POST request headers for each piece of user metadata, e.g.::

  X-Object-Massmeta-Crypto-Meta-<key>: "{'iv': 'zzz', 'cipher': 'AES_CTR_256'}"

.. note::
   There is likely to be value in adding a generic mechanism to persist *any*
   header in the ``X-Object-Massmeta-`` namespace, and adding that prefix to
   those blacklisted by the gatekeeper. This would support other middlewares
   (such as a keymaster) similarly annotating user metadata with middleware
   generated metadata.

For user metadata on containers and accounts we need to store crypto metadata
for each item of user metadata, since these can be independently updated by
POST requests. Here we can use sysmeta to store the crypto metadata items,
e.g. for a user metadata item with key ``X-Container-Meta-Color`` we would
store::

  X-Container-Sysmeta-Crypto-Meta-Color: "{'iv': 'ccc', 'cipher': 'AES_CTR_256'}"

7.4 System Metadata
-------------------

System metadata ("sysmeta") will not be encrypted.

Consider a middleware that uses sysmeta for storage. If, for some
reason, that middleware moves from before-crypto to after-crypto in
the pipeline, then all its previously stored sysmeta will become
unreadable garbage from its viewpoint.

Since middlewares sometimes do move, either due to code changes or to
correct an erroneous configuration, we prefer robustness of the
storage system here.

7.5 Summary
-----------

The encrypter will set the following headers on PUT requests to object
servers::

  Etag = MD5(ciphertext) (IFF client request included an etag header)
  X-Object-Sysmeta-Crypto-Meta-Etag = {'iv': <iv>, 'cipher': <C_req>}

  Content-Type = E(OEK, content-type)
  X-Object-Sysmeta-Crypto-Meta-ct = {'iv': <iv>, 'cipher': <C_req>}

  X-Object-Sysmeta-Crypto-Meta = {'iv': <iv>, 'cipher': <C_req>}
  X-Object-Sysmeta-Crypto-Etag = E(OEK, MD5(plaintext))

  X-Backend-Container-Update-Override-Etag = \
      E(CEK, MD5(plaintext); meta={'iv': <iv>, 'cipher': <C_req>}
  X-Backend-Container-Update-Override-Content-Type = \
      E(CEK, content-type); meta={'iv': <iv>, 'cipher': <C_req>}

where ``OEK`` is the object encryption key, ``iv`` is a randomly chosen
initialization vector and ``C_req`` is the cipher used while handling this
request.

Additionally, on object PUT or POST requests that include user defined
metadata headers, the encrypter will set::

  X-Object-Meta-<user_key> = E(OEK, <user_value>}  for every <user-key>
  X-Object-Massmeta-Crypto-Meta-<user_key> = {'iv': <iv>, 'cipher': <C_req>}

On PUT or POST requests to container servers, the encrypter will set the
following headers for each user defined metadata header::

  X-Container-Meta-<user_key> = E(CEK, <user_value>}
  X-Container-Sysmeta-Crypto-Meta-<user_key> = {'iv': <iv>, 'cipher': <C_req>}

Similarly, on PUT or POST requests to account servers, the encrypter will set
the following headers for each user defined metadata header::

  X-Account-Meta-<user_key> = E(AEK, <user_value>}
  X-Account-Sysmeta-Crypto-Meta-<user_key> = {'iv': <iv>, 'cipher': <C_req>}

where ``AEK`` is the account encryption key.


8. Client-Visible Changes
=========================

There are no known client-visible API behavior changes in this spec.
If any are found, they should be treated as flaws and fixed.


9. Possible Future Work
=======================

9.1 Protection of Internal Network
----------------------------------

Swift's security model is perimeter-based: the proxy server handles
authentication and authorization, then makes unauthenticated requests
on a private internal network to the storage servers. If an attacker
gains access to the internal network, they can read and modify any
object in the Swift cluster, as well as create new ones. It is
possible to use authenticated encryption (e.g. HMAC, GCM) to detect
object tampering.

Roughly, this would involve computing a strong hash (e.g. SHA-384
or SHA-3) of the object, then authenticating that hash. The object
auditor would have to get involved here so that we'd have an upper
bound on how long it takes to detect a modified object.

Also, to prevent an attacker from simply overwriting an encrypted
object with an unencrypted one, the crypto engine would need the
ability to notice a GET for an unencrypted object and return an error.
This implies that this feature is primarily good for clusters that
have always had encryption on, which (sadly) excludes clusters that
pre-date encryption support.


9.2 Other ciphers
-----------------

AES-256 may be considered inadequate at some point, and support for
another cipher will then be needed.


9.3 Client-Managed Keys
-----------------------

CPU-constrained clients may want to manage their own encryption keys
but have Swift perform the encryption. Amazon S3 supports something
like this. Client-managed key support would probably take the form of
a new keymaster.

9.4 Re-Keying Support
---------------------

Instead of using the object key K-obj and computing the ciphertext as
E(k-obj, plaintext), treat the object key as a key-encrypting-key
(KEK) and make up a random data-encrypting key (DEK) for each object.

Then, the object ciphertext would be E(DEK, plaintext), and in system
metadata, Swift would store E(KEK, DEK). This way, if we wish to
re-key objects, we can decrypt and re-encrypt the DEK to do it, thus
turning a re-key operation from a full read-modify-write cycle to a
simple metadata update.


Alternatives
============

Storing user metadata in sysmeta
--------------------------------

To avoid the need to check metadata header limits in the encrypter, encrypted
metadata values could be stored using sysmeta, which is not subject to the same
limits. When handling a GET or HEAD response, the decrypter would need to
decrypt metadata values and copy them back to user metadata headers.

This alternative was rejected because object sysmeta cannot be updated by a
POST request, and so Swift would be restricted to operating in the POST-as-copy
mode when encryption is enabled.

Enforce a single immutable cipher choice per container
------------------------------------------------------

We could avoid storing cipher choice as metadata on every resource (including
individual metadata items) if the choice of cipher were made immutable for a
container or even for an account. Unfortunately it is hard to implement an
immutable property in an eventually consistent system that allows multiple
concurrent operations on distributed replicas of the same resource.

Container storage policy is 'eventually immutable' (any inconsistency is
eventually reconciled across replicas and no replica's policy state may be
updated by a client request). If we made cipher choice a property of a policy
then the cipher for a container could be similarly 'eventually immutable'.
However, it would be possible for objects in the same container to be encrypted
using different ciphers during the any initial window of policy inconsistency
immediately after the container is first created. The existing container policy
reconciler process would need to re-encrypt any object found to have used the
'wrong' cipher, and to do so it would need to know which cipher had been used
for each object, which leads back to cipher choice being stored per-object.

It should also be noted that the IV would still need to be stored for every
resource, so this alternative would not mitigate the need to store crypto
metadata in general.

Furthermore, binding cipher choice to container policy does not provide a means
to guarantee an immutable cipher choice for account metadata.

Implementation
==============

Assignee(s)
-----------

Primary assignees:

|    jrichli@us.ibm.com
|    alistair.coles@hp.com


References
==========
[1] http://specs.openstack.org/openstack/swift-specs/specs/done/erasure_coding.html

[2] Updating containers on object fast-POST: https://review.openstack.org/#/c/102592/
