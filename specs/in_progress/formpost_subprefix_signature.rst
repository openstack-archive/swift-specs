::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

================================================
formpost should allow subprefix-based signatures
================================================

The signature used by formpost to validate a file upload should also be considered valid,
if the object_prefix, which is used to calculate the signature, is a real subprefix of the
object_prefix used in the action url of the form.
With this, sharing of data with external people is made much easier
via webbased applications, because just one signature is needed to create forms for every
pseudofolder in a container.


Problem Description
===================

At the moment, if one wants to use a form to upload data, the signature of the form must be
calculated using the same object_prefix as the object_prefix in the url of the action attribute
of the form.
We propose to allow dynamically created forms, which are valid for all object_prefixes which contain
a common prefix.

With this, one could generate one signature, which is valid for all pseudofolders in a container.
This signature could be used in a webapplication, to share every possible pseudofolder
of a container with external people. The user who wants to share his container would not be obliged
to generate a signature for every pseudofolder.


Proposed Change
===============

The formpost middleware should be changed. The code change would be really small.
If a subprefix-based signature is desired, the hmac_body of the signature must contain a "subprefix"
field to make sure that the creator of the signature explicitly allows uploading of objects into
sub-pseudofolders. Beyond that, the form must contain a hidden field "subprefix", too.
Formpost would use the value of this field to calculate a hash based on that
value. Furthermore, the middleware would check if the object path really contains this prefix.

Lets have one example: A user wants to share the pseudofolder "folder" with external users in
a web-based fashion. He (or a webapplication) calcluates the signature with the path
"/v1/my_account/container/folder" and subprefix "folder":
::

    import hmac
    from hashlib import sha1
    from time import time
    path = '/v1/my_account/container/folder'
    redirect = 'https://myserver.com/some-page'
    max_file_size = 104857600
    max_file_count = 10
    expires = int(time() + 600)
    key = 'MYKEY'
    hmac_body = '%s\n%s\n%s\n%s\n%s\n%s' % (path, redirect,
    max_file_size, max_file_count, expires, "folder")
    signature = hmac.new(key, hmac_body, sha1).hexdigest()

If an external user is willing to post to the subfolder folder/subfolder/, a form which contains
the above calculated signature and the hidden field subprefix would be used:
::

    <![CDATA[
    <form action="https://myswift/v1/my_account_container/folder/subfolder/"
        method="POST"
        enctype="multipart/form-data">
        <input type="hidden" name="redirect" value="REDIRECT_URL"/>
        <input type="hidden" name="max_file_size" value="BYTES"/>
        <input type="hidden" name="max_file_count" value="COUNT"/>
        <input type="hidden" name="expires" value="UNIX_TIMESTAMP"/>
        <input type="hidden" name="signature" value="HMAC"/>
        <input type="hidden" name="subprefix" value="folder"
        <input type="file" name="FILE_NAME"/>
        <br/>
        <input type="submit"/>
    </form>
    ]]>


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bartz

Work Items
----------

Add modifications to formpost and respective test module.

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

Modify documentation for formpost middleware.

Security
--------

None

Testing
-------

Tests should be added to the existing test module.

Dependencies
============

None
