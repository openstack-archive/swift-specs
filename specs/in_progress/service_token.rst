::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Composite Tokens and Service Accounts
=====================================

This is a proposal for how Composite Tokens can be used by services such
as Glance and Cinder to store objects in project-specific accounts yet
retain control over how those objects are accessed.

This proposal uses the "Service Token Composite Authorization" support in
the auth_token Keystone middleware
(http://git.openstack.org/cgit/openstack/keystone-specs/plain/specs/keystonemiddleware/service-tokens.rst).

Problem Description
===================

Swift is used by many OpenStack services to store data on behalf of users.
There are typically two approaches to where the data is stored:

* *Single-project*. Objects are stored in a single dedicated Swift account
  (i.e., all data belonging to all users is stored in the same account).

* *Multi-project*. Objects are stored in the end-user's Swift account (project).
  Typically, dedicated container(s) are created to hold the objects.

There are advantages and limitations with both approaches as described in the
following table:

==== ==========================================  ==========    ========
Item Feature/Topic                               Single-       Multi-
                                                 Project       Project
---- ------------------------------------------  ----------    --------
1    Fragile to password leak (CVE-2013-1840)    Yes           No
2    Fragile to token leak                       Yes           No
3    Fragile to container deletion               Yes           No
4    Fragile to service user deletion            Yes           No
5    "Noise" in Swift account                    No            Yes
6    Namespace collisions (user and service      No            Yes
     picking same name)
7    Guarantee of consistency (service           Yes           No
     database vs swift account)
8    Policy enforcement (e.g., Image Download)   Yes           No
==== ==========================================  ==========    ========

Proposed change
===============

It is proposed to put service data into a seperate account to the end-user's
"normal" account. Although the account has a different name, the account
is linked to the end-user's project. This solves issues with noise
and namespace collisions. To remove fragility and improve consistency
guarantees, it is proposed to use the composite token feature to manage
access to this account.

In summary, there are three related changes:

* Support for Composite Tokens

* The authorization logic can require authenticated information from the
  composite tokens

* Support for multiple reseller prefixes, each with their own configuration

The effect is that access to the data must be made through the service.
In addition, the service can only access the data when it is processing
a request from the end-user (i.e, when it has an end-user's token).

The changes are described one by one in this document. The impatient can
skip to "Composite Tokens in the OpenStack Environment" for a complete,
example.

Composite Tokens
================

The authentication system will validate a second token. The token is stored
in the X-Service-Token header so is known as the service token (name chosen
by Keystone).

The core functions of the token authentication scheme is to determine who the
user is, what account is being accessed and what roles apply. Keystoneauth
and Tempauth have slightly different semantics, so the tokens are combined
in slightly different ways as explained in the following sections.

Combining Roles in Keystoneauth
-------------------------------

The following rules are used when a service token is present:

* The user_id is the user_id from the first token (i.e., no change)

* The account (project) is specified by the first token (i.e., no change)

* The user roles are initialy determined by the first token (i.e., no change).

* The roles from the service token are made available in service_roles.

Example 1 - Combining Roles in keystoneauth
-------------------------------------------

In this example, the <token-two> is scoped to a different project than
the account/project being accessed::

      Client
        |                               <user-token>: project-id: 1234
        |                                                user-id: 9876
        |                                                  roles: Admin
        | X-Auth-Token: <user-token>
        | X-Service-Token: <token-two>
        |
        |                                <token-two>: project-id: 5678
        v                                                user-id: 5432
      Swift                                                roles: service
        |
        v
      Combined identity information:
           user_id: 9876
           project_id: 1234
           roles: Admin
           service_roles: service

Combining Groups in Tempauth
----------------------------

The user groups from both tokens are simply combined into one list. The
following diagram gives an example of this::


      Client
        |                               <user-token>: from "joe"
        |
        |
        | X-Auth-Token: <user-token>
        | X-Service-Token: <token-two>
        |
        |                               <token-two>: from "glance"
        v
      Swift
        |
        |  [filter:tempauth]
        |  user_joesaccount_joe: joespassword .admin
        |  user_glanceaccount_glance: glancepassword .service
        |
        v
      Combined Groups: .admin .service

Support for multiple reseller prefixes
======================================

The reseller_prefix will now support a list of prefixes. For example,
the following supports both ``AUTH_`` and ``SERVICE_``::

    [...]
    reseller_prefix = AUTH_ SERVICE_

For backward compatibility, the default remains as ``AUTH_``.

All existing configuration parameters are assumed to apply to the first
item in the list. However, to indicate which prefix a paramter applies to,
put the prefix in front of the parameter name. This applies to the
following parameters:

* reseller_admin_role (keystoneauth)
* operator_roles (keystoneauth)
* allow_overrides (tempauth, keystoneauth)

Other paramters (logging, storage_url_scheme, etc.) are not specific to
the reseller prefix.

For example, this shows two prefixes and some parameters::

    [filter:keystoneauth]
    reseller_prefix = AUTH_ SERVICE_
    reseller_admin_role = ResellerAdmin     <= old style, applies to AUTH_
    AUTH_operator_roles = auth              <= new style
    SERVICE_reseller_admin_role = ResellerAdmin
    SERVICE_operator_roles = auth
    SERVICE_allow_overrides = false

Support for composite authorization
===================================

We will add a parameter called "service_roles" to keystoneauth. If
present, composite tokens must be used and the service_roles (as explained
earler) must contain the listed roles. Here is an example where the
``AUTH_`` namespace requires the "admin" role, whereas the ``SERVICE_``
namespace requires a "service" role::

    [filter:keystoneauth]
    AUTH_operator_roles = admin
    SERVICE_operator_roles = admin
    SERVICE_service_roles = service

In tempauth, the ".admin" group has a special (built-in) meaning. It is
proposed to make this explicit. This allows different prefixes to require
different groups(s). The "admin_groups" paramter is used for this purpose.
If the value is a list, the user must be a member of all the groups in the
list. The following shows an example::

    [filter:tempauth]
    AUTH_admin_groups: .admin    <= not really needed since this is the default
    SERVICE_admin_groups: .admin .service


Composite Tokens in the OpenStack Environment
=============================================

This section presents a simple configuration showing the flow from client
through an OpenStack Service to Swift. We use glance in this example, but
the principal is the same for all services. See later for a more
complex service-specific setup.

The flow is as follows::

     Client
        |                               <user-token>: project-id: 1234
        |                                                user-id: 9876
        | (request)                                        roles: Admin
        | X-Auth-Token: <user-token>
        |
        v
     Glance
        |
        | PUT /v1/SERVICE_1234/container/object
        | X-Auth-Token: <user-token>
        | X-Service-Token: <glance-token>
        |
        |                             <glance-token>: project-id: 5678
        v                                                user-id: 5432
      Swift                                                roles: service
        |
        v
      Combined identity information:
           user_id: 9876
           project-id: 1234
           roles: Admin
           service_roles: service

           [filter:keystoneauth]
           reseller_prefix = AUTH_ SERVICE_
           AUTH_operator_roles = Admin
           AUTH_reseller_admin_roles = ResellerAdmin
           SERVICE_operator_roles = Admin
           SERVICE_service_roles = service
           SERVICE_reseller_admin_roles = ResellerAdmin

The authorization logic is as follows::

         /v1/SERVICE_1234/container/object
             -------
                |
               in?
                |
        reseller_prefix = AUTH_ SERVICE_
                \
                Yes
                  \
            Use SERVICE_* configuation
                   |
                   |
            /v1/SERVICE_1234/container/object
                        ----
                         |
                     same as? project-id: 1234
                         \
                         Yes
                           \
                       roles: admin
                           |
                          in? SERVICE_operator_roles = Admin
                           \
                           Yes
                             \
                           service_roles: service
                             |
                            in? SERVICE_SERVICE_ROLES = service
                             \
                             Yes
                               \
                                ----> swift_owner = True


Other Aspects
=============

Tempurl, FormPOST, Container Sync
---------------------------------

These work on the principal that the secret key is stored in a *privileged*
header. No change is proposed as the account controls described in this
document continue to use this concept. However, an additional use-case
becomes possible: it should be possible to use temporary URLs to
allow a client to upload or download objects to or from a service
account.

Service Catalog
---------------

The Keystone service catalog will reflect multiple accounts as shown in the
following example::

    {
      "name": "Object Storage",
      "type": "object-store",
      "endpoints": [
      {
        "publicURL": "https://hostname/v1/AUTH_1234",
        "serviceURL": "https://hostname/v1/SERVICE_1234"
      }
    }

Service-Specific Accounts
-------------------------

Using a common ``SERVICE_`` namespace means that all OpenStack Services share
the same account. A simple alternative is to use multiple accounts -- with
corresponding reseller_prefixes and service catalog entries. For example,
Glance could use ``IMAGE_`` and Cinder could use ``VOLUME_``. There is nothing
in this proposal that limits this option. Here is an example of a
possible configuration::

    [filter:keystoneauth]
    reseller_prefix = AUTH_ IMAGE_ VOLUME_
    IMAGE_SERVICE_ROLES = glance_service
    VOLUME_SERVICE_ROLES = cinder_service

python-swiftclient
------------------

No changes are needed in python-swiftclient to support this feature.

Service Changes To Use ``SERVICE_`` Namespace
---------------------------------------------

Services (such as Glance, Cinder) need to be enhanced as follows to use
the ``SERVICE_`` namespace:

* Use the serviceURL path. Services have access to ``HTTP_SERVICE_CATALOG`` in
  their environment so it is easy to construct the appropriate path.
* Add their token to the X-Service-Token header
* They should have the "service" role for this token
* They should include their service type (e.g., image) as a prefix to any
  container names they create. This will prevent conflict between services
  sharing the account.

Upgrade Implications
====================

The Swift software must be upgraded before Services attempt to use the
``SERVICE_`` namespace. Since Services use configurable options
to decide how they use Swift, this should be easy to sequence (i.e., upgrade
software first, then change the Service's configuration options).

How Services handle existing legacy data is beyond the scope of this
proposal.

Alternatives
============

*Account ACL*

An earlier draft proposed extending the account ACL. It also proposed to
add a default account ACL concept. On review, it was decided that this
was unnecessary for this use-case (though that work might happen in it's
own right).

*Co-owner sysmeta*

An earlier draft proposed new sysmeta that established "co-ownership"
rules for containers.

*policy.xml File*:

The Keystone Composite Authorization scheme has use cases for other Openstack
projects. The OSLO incubator policy checker module may be extended to support
roles acquired from X-Service-Token. However, this will only be used in
Swift if keystoneauth already uses a policy.xml file.

If policy files are adapted by keystoneauth, it should be easy to apply. In
effect, a different policy.xml file would be supplied for each reseller prefix.

*Proxy Logging*:

The proxy-logging middleware logs the value of X-Auth-Token. No change is
proposed.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
    donagh.mccabe@hp.com

To be fully effective, changes are needed in other projects:

* Keystone Middleware. Done

* OSLO. As mentioned above, probably not needed or depended on.

* Glance. stuart.mclaren@hp.com will make the Glance changes.

* Cinder. Unknown.

* Devstack. The Swift change by itself will probably not require Devstack
  changes. The Glance and Cinder services may need additional configuration
  parameters to enable the X-Service-Token feature.
  Assignee: Unknown

* Tempest. In principal, no changes should be needed as the proposal is
  intended to be transparent to end-users. However, it may be possible
  that some tests incorrectly access images or volume backups directly.
  Assignee: Unknown

* Ceilometer (for ``SERVICE_`` namespace). It is not clear if any
  changes are needed or desirable.

Work Items
----------

* swift/common/middleware/tempauth.py is modified to support multiple
  reseller prefixes, a configurable name for .admin and to process the
  X-Service-Token header

* swift/common/middleware/keystoneauth.py is modified to support multiple
  reseller prefixes and the service_roles parameter.

* Write unit tests

* Write functional tests

Repositories
------------

No new git repositories will be created.

Servers
-------

No new servers are created. The keystoneauth middleware is used by the
proxy-server.

DNS Entries
-----------

No DNS entries will to be created or updated.

Dependencies
============

* "Service Token Composite Authorization"
   https://review.openstack.org/#/c/96315
