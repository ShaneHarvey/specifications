==========================
Awaitable isMaster Command
==========================

:Spec: XXXX
:Title: Awaitable isMaster Command
:Author: Shane Harvey
:Advisors: Matt broadstone
:Status: Accepted
:Type: Informational
:Version: 1.0
:Last Modified: 2020-02-20

.. contents::

--------

Abstract
========

As of MongoDB 4.4 the isMaster command can wait to reply until there is a
topology change or a maximum time has elapsed. Clients opt in to this
"awaitable isMaster" feature by passing new isMaster parameters
"topologyVersion" and "maxAwaitTimeMS". Exhaust support has also been added,
which clients can enable in the usual manner by setting the OP_MSG
"exhaustAllowed" flag.

Clients use the awaitable isMaster feature as the basis of the streaming
heartbeat protocol to learn much sooner about stepdowns, elections, reconfigs,
and other events.

This feature also introduces a new data structure, the "topologyVersion".
Clients and servers exchange topologyVersions to determine the relative
freshness of topology information in concurrent messages.

Specification
=============

-----------
Definitions
-----------

META
----

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in `RFC 2119 <https://www.ietf.org/rfc/rfc2119.txt>`_.

Terms
-----

Awaitable isMaster
^^^^^^^^^^^^^^^^^^

The new isMaster command feature, which allows the server to wait for a
significant topology change or timeout before replying.

Streaming isMaster
^^^^^^^^^^^^^^^^^^

The new isMaster command feature, in which the server streams isMaster replies
without awaiting further requests.

Client
^^^^^^

A process that initiates a connection to a MongoDB server. This includes
mongod and mongos processes in a replica set or sharded cluster, as well as
drivers, the shell, tools, etc.

SDAM
^^^^

The Server Discovery and Monitoring Spec.

Server
^^^^^^

A process that accepts connections. This includes mongod, and mongos's
implementation of the server side of the MongoDB wire protocol.

Significant topology change
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A change in the server's state that is relevant to the client's view of the
server, e.g. a change in the server's replica set member state, or its replica
set tags. In SDAM terms, a significant topology change on the server means the
client's ServerDescription is out of date. Standalones and mongos do not
currently experience significant topology changes but they may in the future.

NotMaster Error
^^^^^^^^^^^^^^^

A server reply document indicating one of the following errors:

- InterruptedDueToReplStateChange
- NotMaster
- NotMasterNoSlaveOk
- NotMasterOrSecondary
- PrimarySteppedDown

These errors are referred to as "not master" errors in SDAM, which defines
an algorithm for parsing these errors from any MongoDB version.

---------------
topologyVersion
---------------

A server that supports the awaitable isMaster includes a new "topologyVersion"
field in all isMaster replies and State Change Error replies. The
topologyVersion helps the client and server determine the relative freshness
of topology information in concurrent messages.

The topologyVersion is a subdocument with two fields, "processId" and
"counter":

.. code:: typescript

    {
        topologyVersion: {processId: <ObjectId>, counter: <int64>},
        ( ... other fields ...)
    }


processId
---------

An ObjectId maintained in memory by the server. It is reinitialized by the
server using the standard ObjectId logic each time this server process starts.

counter
---------

An int64 State change counter, maintained in memory by the server. It begins
at 0 when the server starts, and it is incremented whenever there is a
significant topology change.

--------------
maxAwaitTimeMS
--------------

To enable awaitable isMaster, the client includes a new int64 field
"maxAwaitTimeMS" in the isMaster request. This field determines the maximum
duration in milliseconds a server will wait for a significant topology change
before replying.

-----------------
Feature Discovery
-----------------

To discover if the connected server supports awaitable isMaster, a client
check any isMaster command reply. If the reply includes "topologyVersion" then
the server supports awaitable isMaster.

------------------
Awaitable isMaster
------------------

To initiate an awatible isMaster command, the client includes both
maxAwaitTimeMS and topologyVersion in the request, for example:

.. code:: typescript

    {
        isMaster: 1,
        maxAwaitTimeMS: 10000,
        topologyVersion: {processId: <ObjectId>, counter: <int64>},
        ( ... other fields ...)
    }

Clients MAY additionally set the OP_MSG exhaustAllowed flag to enable
streaming isMaster. With streaming isMaster, the server MAY send multiple
isMaster responds without waiting for further requests.

A server that implements the new protocol follows these rules:

- Always include the server's topologyVersion in isMaster and NotMaster Error
  replies.
- If the request includes topologyVersion without maxAwaitTimeMS or vice versa,
  return an error.
- If the request omits topologyVersion and maxAwaitTimeMS, reply immediately.
- If the request includes topologyVersion and maxAwaitTimeMS, then reply
  immediately if the server's topologyVersion.processId does not match the
  request's, otherwise reply when the server's topologyVersion.counter is
  greater than the request's, or maxAwaitTimeMS elapses, whichever comes first.
- Following the OP_MSG spec, if the request omits the exhaustAllowed bit, the
  server MUST NOT set the moreToCome bit on the reply. If the request's
  exhaustAllowed bit is set, the server MAY set the moreToCome bit on the
  reply. If the server sets moreToCome, it MUST continue streaming replies
  without awaiting further requests. Between replies it MUST wait until the
  server's topologyVersion.counter is incremented or maxAwaitTimeMS elapses,
  whichever comes first.
- On a topology change that changes the horizon parameters, the server will
  return a SplitHorizonChange error. This behavior will be the same for both
  the exhaust and non-exhaust cases of streamable isMaster. In the exhaust
  case, returning an error will close the exhaust stream between the server
  and client.

Example awaitable isMaster conversation:

+---------------------------------------+--------------------------------+
| Client                                | Server                         |
+=======================================+================================+
| isMaster handshake ->                 |                                |
+---------------------------------------+--------------------------------+
|                                       | <- reply with topologyVersion  |
+---------------------------------------+--------------------------------+
| isMaster as OP_MSG with               |                                |
| maxAwaitTimeMS and topologyVersion -> |                                |
+---------------------------------------+--------------------------------+
|                                       | wait for change or timeout     |
+---------------------------------------+--------------------------------+
|                                       | <- OP_MSG with topologyVersion |
+---------------------------------------+--------------------------------+
| ...                                   |                                |
+---------------------------------------+--------------------------------+

Example streaming isMaster conversation (awaitable isMaster with exhaust):

+---------------------------------------+--------------------------------+
| Client                                | Server                         |
+=======================================+================================+
| isMaster handshake ->                 |                                |
+---------------------------------------+--------------------------------+
|                                       | <- reply with topologyVersion  |
+---------------------------------------+--------------------------------+
| isMaster as OP_MSG with               |                                |
| exhaustAllowed, maxAwaitTimeMS,       |                                |
| and topologyVersion ->                |                                |
+---------------------------------------+--------------------------------+
|                                       | wait for change or timeout     |
+---------------------------------------+--------------------------------+
|                                       | <- OP_MSG with moreToCome      |
|                                       | and topologyVersion            |
+---------------------------------------+--------------------------------+
|                                       | wait for change or timeout     |
+---------------------------------------+--------------------------------+
|                                       | <- OP_MSG with moreToCome      |
|                                       | and topologyVersion            |
+---------------------------------------+--------------------------------+
|                                       | ...                            |
+---------------------------------------+--------------------------------+

Changelog
=========

- 2020-02-20 Published 1.0.0
