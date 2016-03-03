::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

==================================================
Swift Request Tagging for detailed logging/tracing
==================================================

URL of your blueprint:

None.

To tag a particular request/every 'x' requests, which would undergo more detailed logging.

Problem Description
===================
Reasons for detailed logging:

- A Swift user is having problems, which we cannot recreate but could tag this user request for more logging.

- In order to better investigate a cluster for bottlenecks/problems - Internal user (admin/op) wants additional info on some situations where the client is getting inconsistent container listings. With the Swift-inspector, we can tell what node is not returning the correct listings.

Proposed Change
===============

Existing: Swift-Inspector (https://github.com/hurricanerix/swift-inspector ) currently
provides middleware in Proxy and Object servers. Relays info about a request back to the client with the assumption that the client is actively making a decision to tag a request to trigger some action that would not otherwise occur.
Current Inspectors:

- Timing -‘Inspector-Timing’: gives the amount of time it took for the proxy-server to process the request
- Handlers – ‘Inspector-Handlers’: not implemented (meant to return the account/container/object servers that were contacted in the request) ‘Inspector-Handlers-Proxy’: returns the proxy that handled the request
- Nodes - ‘Inspector-Nodes’: returns what account/container/object servers the path resides on ‘Inspector-More-Nodes’: returns extra nodes for handoff.

Changes:

- Add logging inspector to the above inspectors , which would enable detailed logging for tagged requests.
- Add the capability to let the system decide (instead of the client) to tag a request and nice to add rules to trigger actions like extra logging etc.

Possible Tagging criteria: Tagging

- every 'x' requests/ a % of all requests.

- based on something in the request/response headers (e.g.if the HTTP method is DELETE, or the response is sending a specific status code back)

- based on a specific account/container/object/feature.

Alternatives
------------
- Logging: log collector/log aggregator like logstash.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~shashirekha-j-gundur

Work Items
----------

- To add an Inspector– ‘Logging’ to existing inspectors , to enable the logs.

- Add rules to tag decide which requests to be tagged

- Trigger actions like logging.

- Restrict the access of nodes/inventory list displayed to admins/ops only.

- Figure out hmac_key access (Inspector-Sig) and ‘Logging’ work together?

Repositories
------------

Will any new git repositories need to be created? Yes.

Servers
-------

Will any new servers need to be created? No.

What existing servers will be affected? Proxy and Object servers.

DNS Entries
-----------

Will any other DNS entries need to be created or updated? No.

Documentation
-------------

Will this require a documentation change? Yes , Swift-inspector docs.

Will it impact developer workflow? No.

Will additional communication need to be made? No.

Security
--------

None.

Testing
-------

Unit tests.

Dependencies
============

- Swift-Inspector https://github.com/hurricanerix/swift-inspector

- Does it require a new puppet module? No.
