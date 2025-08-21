..
    Status: Draft
    Type: Feature
    Created: 2020-02-19
    RFC PR: `edgedb/rfcs#4 <https://github.com/edgedb/rfcs/pull/4>`_

======================================
RFC 0001: The WebSocket-based Protocol
======================================

This RFC describes the design and implementation of the alternate
protocol to the EdgeDB based on WebSockets for framing and the initial
handshake.


Motivation
==========

The primary motivation is to allow various browser-based exploration
and editing tools for the database.

The secondary motivation is to allow browser-based access to the
application data. Essentially allowing serverless implementation for
some applications or parts thereof. The reason this is a secondary
motivation is that it requires full featured access controls (ACLs)
which we don't have yet.

Two important side effects of implementing this RFC:
* it might also help implementing the protocol in some languages which
  we don't have language bindings yet
* existing tools like load-balancers can be used to access edgedb in
  some cases
* it enables using TLS which we don't have for the main protocol yet


Specification
=============

Changes from the Binary Protocol
--------------------------------

1. The URL to connect to is ``ws://<host>:<port>/ws/<database-name>``
2. We use single WebSocket subprotocol name ``edgedb-binary``.
3. Authentication is done at HTTP level (see Authentication_ section)
4. WebSocket framing is used as a message delimitation. The first
   byte indicates a message type, followed by message contents.
5. When WebSocket connection is established, ClientHandshake_ message
   is sent without ``database`` and ``user`` params.
6. Authentication messages are never sent from server
7. Ping/Pong WebSocket messages might be sent and must be responded.

Other than these changes, whole `protocol flow and structure`_
is the same as using binary protocol.

.. _ClientHandshake: https://edgedb.com/docs/internals/protocol/messages#ref-protocol-msg-client-handshake
.. _protocol flow and strucutre: https://edgedb.com/docs/internals/protocol/overview


Authentication
==============

TBD


Rejected Alternative Ideas
==========================

Using Subprotocol Name for Version Negotiation
----------------------------------------------

This means using string like ``edgedb-binary-0.8`` as a subprotocol name
for the handshake instead of negotiating the version in a separate
handshake message.

The upsides:
* Less roundtrips to establish connection
* External routing can be smarter (route based on the protocol version)
* Negotiation is easier to debug

The downsides are:
* Every minor protocol version should be sent as a subprotocol, i.e.
  the header would look like this:
  ``edgedb-0.8, edgedb-0.9, edgedb-1.0, edgedb-1.1``, and will grow
  over time
* Protocol extensions can't be negotiated (``Sec-WebSocket-Extensions``
  can't be used for custom extensions in the browser).


Having JSON-based Protocol
--------------------------

The JavaScript is capable for parsing binary data nowadays. And
making a fully-JSON-based protocol is large piece of work.

We don't consider this idea bad per se, but it's out of the scope of
this RFC.


Using Handshake Parameters to Specify Database Name
---------------------------------------------------

In normal binary protocol we send database name as a parameter in
the client handshake. For WebSockets the natural place for this data
is the URL, this allows nice routing between databases by proxy servers.

Also authentication must happen before handshake happens. (So handshake
is only used to establish protocol features).


Using Handshake Parameters to Specify Database User
---------------------------------------------------

In normal binary protocol we send user name as a parameter in
the client handshake. For WebSockets, the handshake is sent when
authentication is already happened.


Using Different URL Schema or Rewriting URLs
--------------------------------------------

While it may be possible that we allow changing URL schema later, it
is not our immediate goal. The described schema is good enough and
there are various proxy servers that can rewrite WebSocket URLs if
that is required for some specific application.

The ``/ws/`` path prefix is used to allow combining GraphQL,
healthcheck and other APIs under the same host/port combination in
future.


Authentication as in Binary Protocol
------------------------------------

Authentication by exchanging WebSocket frames like we do in binary
protocol is going to be wrong for the following reasons:

1. Need to keep password in browser's memory (to reconnect)
2. Is not easily introspected like cookies or headers
3. Can't use routing based on authentication data
