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

Goal 2: it should be possible to completely remove an object so that
it cannot be recovered by deleting the key. This requires support from
the keymaster for key deletion. This provides a guarantee that an
object is not retained (in a recoverable form) any longer than
necessary.

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
Etag header's value is the MD5 hash of the uploaded object data. With
encrypted objects, the plaintext is not available to the object
server, so the encrypter must perform the validation instead.

2.2 Inter-Middleware Communication
----------------------------------

The keymaster communicates the encryption key to the encrypter and
decrypter middlewares by placing a zero-argument callable in the WSGI
environment dictionary at the key "swift.crypto.fetch_crypto_keys".
When called, this will return the key(s) necessary to process the
current request. It must be present on any GET or HEAD request for an
account, container, or object which contains any encrypted data or
metadata. If encrypted data or metadata is encountered while
processing a GET or HEAD request but fetch_crypto_keys is not present
_or_ it does not return keys when called, then this is an error and
the client will receive a 500-series response.

On a PUT or POST request, the keymaster must place fetch_crypto_keys in
the WSGI environment during request processing; that is, before
passing the request to the remainder of the middleware pipeline. This
is so that the encrypter can encrypt the object's data in a streaming
fashion without buffering the whole object.

On a GET or HEAD request, the keymaster must place fetch_crypto_keys in
the WSGI environment before returning control to the decrypter. It
need not be done at request-handling time. This lets attributes of the
key be stored in sysmeta, for example the key ID in an external
database, or anything else the keymaster wants.

3. Cipher Choice
================

3.1. The Chosen Cipher
----------------------

Swift will use AES in CTR mode with 256-bit keys.

In order to allow for ranged GET requests, the cipher shall be used
in counter (CTR) mode.

The entire object shall be encrypted as a single byte stream. The IV
will be randomly generated and stored in system metadata.


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


4. Robustness
=============


4.1 No Key
----------

If the keymaster fails to add a key to the WSGI environment, then the
client will receive the ciphertext of the object instead of the
plaintext, which looks to the client like garbage. However, we can
tell if an object is encrypted or not by the presence of system
metadata headers, so the decrypter can prevent this by raising an
error if no key was provided for the decryption of an encrypted
object.


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


6. Metadata Encryption
======================

6.1 Background
--------------
Swift entities (accounts, containers, and objects) have three kinds of
metadata.

First, there is basic metadata, like Content-Length, Content-Type, and
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


6.2 User Metadata on Objects
----------------------------

Not only the contents of an object are sensitive; metadata is
sensitive too. User metadata will be encrypted with the same key and
IV as the rest of the metadata. Since metadata values must be valid
UTF-8 strings, the encrypted values will be suitably encoded (probably
base64) for storage. In order to preserve the current user-metadata
length limits despite the bloat caused by encoding, the encrypter will
actually store the ciphertext of the user's metadata values in system
metadata. That way, users don't see lower metadata-size limits when
encryption is in use.

This means that the encrypter is responsible for enforcing
user-metadata limits on encrypted objects, as it is the last entity in
the middleware pipeline to see plaintext user metadata.

Both metadata names and values will be encrypted. Leaving
"X-Object-Sysmeta-UM-Preferred-Contraband-Vendor: Y7JF30oF5TXTeEIqOu8="
on disk is just not good enough.

Further, the object's plaintext etag and content type are sensitive
information and will be stored encrypted as well, both in the
container listing and in the object's metadata. To accomplish this,
the proxy server will actually encrypt the etag and content type
_twice_: once with the object's key, and once with the container's
key. When the object server updates the container database after
finishing a PUT request, it will send the container-key-encrypted
values over to the container. The object-key-encrypted values will be
stored in the object's metadata. This way, the client sees the
plaintext etag and content type in container listings and in object
GET or HEAD responses, just like it would without encryption enabled,
but the plaintext values of those are not stored anywhere.


6.2.1 A Note On Etag
^^^^^^^^^^^^^^^^^^^^

In the stored object's metadata, the basic-metadata field named "Etag"
will contain the MD5 hash of the ciphertext. This is required so that
the object server will not error out on an object PUT, and also so
that the object auditor will not quarantine the object due to hash
mismatch (unless bit rot has happened).

The plaintext's MD5 hash will be stored, encrypted, in system
metadata.


6.3 System Metadata
-------------------
System metadata ("sysmeta") will not be encrypted.

Consider a middleware that uses sysmeta for storage. If, for some
reason, that middleware moves from before-crypto to after-crypto in
the pipeline, then all its previously stored sysmeta will become
unreadable garbage from its viewpoint.

Since middlewares sometimes do move, either due to code changes or to
correct an erroneous configuration, we prefer robustness of the
storage system here.


6.4 Encryption of Object Data
-----------------------------

Each object is encrypted with the key from the keymaster. The IV is
randomly generated by the encrypter, and (as mentioned earlier), is
stored in system metadata.

7. Client-Visible Changes
=========================

There are no known client-visible API behavior changes in this spec.
If any are found, they should be treated as flaws and fixed.


8. Possible Future Work
=======================

8.1 Protection of Internal Network
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


8.2 Other ciphers
-----------------

AES-256 may be considered inadequate at some point, and support for
another cipher will then be needed.


8.3 Client-Managed Keys
-----------------------

CPU-constrained clients may want to manage their own encryption keys
but have Swift perform the encryption. Amazon S3 supports something
like this. Client-managed key support would probably take the form of
a new keymaster.

8.4 Re-Keying Support
---------------------

Instead of using the object key K-obj and computing the ciphertext as
E(k-obj, plaintext), treat the object key as a key-encrypting-key
(KEK) and make up a random data-encrypting key (DEK) for each object.

Then, the object ciphertext would be E(DEK, plaintext), and in system
metadata, Swift would store E(KEK, DEK). This way, if we wish to
re-key objects, we can decrypt and re-encrypt the DEK to do it, thus
turning a re-key operation from a full read-modify-write cycle to a
simple metadata update.
