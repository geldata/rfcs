::

    Status: Draft
    Type: Feature
    Created: 2025-04-11
    Authors: Matt Mastracci <matt@geldata.com>

===========================================
RFC 1029: Database Pre-Authorization Mode
===========================================

This RFC proposes a new mechanism for making initial authorization setup easier
when deploying a new Gel database instance, especially in environments without
easy access to the server's local file system.

Motivation
==========

Users deploying Gel to environments such as Docker containers or cloud-based
VMs frequently encounter difficulties determining how to connect to their newly
created instance. In environments where local file system access is available,
the server's JWT secret key can be read and used to generate a signed token
that grants full access, including access to the UI.

In remote environments without file system access, this workflow is infeasible.
This RFC introduces a pre-authorization mode to simplify the onboarding
process.

Current State
=============

In previous versions, Gel required a difficult-to-track passwword to perform any
meaningful interaction. This was recently changed to allow the credentials to
provide a secret key, however that secret key is only available if we can
directly access the server's JWT keyfile.

When the JWT key is available, clients can sign tokens and connect securely.
However, this setup is not easy or practical for remote deployments.

Proposal
========

Pre-Authorization Mode
---------------------

We introduce a new server startup mode: pre-auth mode. This mode is triggered
based on a set of conditions defined below. In this mode, the server runs with
a restricted API for setting up credentials.

Triggering Conditions
--------------------

The server will enter **pre-auth mode** if, upon startup:

- JWT secret keys do not exist.
- (Alternative) Authorization configuration is entirely missing or empty.

This transition is **only evaluated on startup**. Once the preconditions are
met, the server enters pre-auth mode. If the preconditions are satisfied again
on a later restart (e.g. due to reset), the server re-enters pre-auth mode.

Behavior in Pre-Auth Mode
------------------------

Only an HTTPS listener is active. Other listeners may be available, but will
return error codes indicating that pre-auth mode must be exited before they are
fully available.

All requests must go through a pre-authorization API.

A unique one-time short code is printed to the server logs on startup. This code
must be provided by the client to authenticate initial setup. The code will be
long enough to be unlikely to brute force within 10 attempts, but short enough
to be easy to type without clipboard support. The code must avoid visually
similar characters to avoid confusion.

The initial proposal is a 10-character code using uppercase letters and digits,
separated with a hyphen in the middle. There are 34 characters that are visually
distinct (we would automatically map ``1`` and ``0`` to ``I`` and ``O``), so the
code will have 34**10 possible values. This is a little more than 2 trillion
possible codes, which is well beyond brute-forceable within 10 attempts. An
example would be ``ABC12-34XYZ``.

The CLI will use this short code to perform the initial setup exchange and
obtain credentials.

Once credentials are set (e.g. password or JWT keys), the server exits
pre-auth mode and enters standard operating mode.

JWT Key Generation
-----------------

The CLI is currently capable of generating its own JWT secret keys. However,
this is not practical for remote deployments where the server may not have
access to the local file system of the server without SSH.

As part of the setup protocol, the server will generate its own JWT secret keys
and a CLI-compatible signed access token, using the code in the gel-jwt Rust
crate. The specific JWT key type is unspecified.

The absence of these keys in the server configuration/file system is the primary
trigger for pre-auth mode.

Security Considerations
======================

To prevent abuse, pre-auth mode must not expose a server to unauthenticated
users. The required use of the short code, printed only to server logs,
acts as a basic form of authentication to prevent unauthorized access in
cloud deployments.

The short code exchange mechanism avoids exposing sensitive endpoints to
unauthenticated users without access to the instance logs.

Pre-Auth API
===========

The pre-auth HTTPS API would be defined as follows:

``POST /setup``: Accepts the short code and returns a one-time token and
correctly sets up secret JWT credentials.

``GET /status``: Returns whether the server is in normal mode, pre-auth mode,
or awaiting a restart after the setup operation.

``GET /``: Returns a styled and branded HTML page with instructions for exiting
pre-auth mode (or possibly a documentation link).

(TBD: Authentication model, retry limits, timeouts)

Custom Exception
===============

A new custom exception subtype, e.g. ``PreAuthRequiredError``, will be introduced
and returned when a user attempts to connect to a server still in pre-auth mode
using a normal EdgeDB client. This makes it easier to guide users and tooling
(e.g. the CLI) through the appropriate setup flow.

Rejected Ideas
=============

CLI Generation of JWT Keys
-------------------------

An alternative considered was allowing the CLI to provided a pre-generated JWT
key to the server. This was rejected as the server currently performs the
generation of JWT keys, and while not impossible for the CLI to perform this
task, the server will be using this generated key and is in the best position to
ensure that the key is properly generated.

REPL Access with Restricted Commands
-----------------------------------

An alternative considered was allowing REPL connections with a whitelisted
subset of commands (e.g. ``ALTER ROLE``, ``SET PASSWORD``, ``INSERT config::...``).

This was rejected due to the complexity of sandboxing the full EdgeQL language
and the risk of abuse. It would also require tight control over language
constructs and validation.

Missing Configuration to Trigger Pre-Auth Mode
--------------------------------------------

Another idea was that lack of auth-related configuration keys to trigger
pre-auth mode. This was rejected due to complexity.

Backwards Compatibility
======================

This feature is fully backward compatible. If pre-auth mode is not triggered,
the server behaves exactly as it does today.

Additional Considerations
========================

Docker images should start in pre-auth mode by default unless configured
otherwise.

We need to clearly document onboarding and setup steps. Once the various types
of error are determined, those errors should be easily Googlable to guide users
through the correct setup process.

Server logs should clearly indicate that pre-auth mode is active and provide
hints as to how to proceed.

Open Questions
=============

Should we support regeneration of JWT keys? i.e., should the server be able to
delete its own JWT keys and force a new setup using the web interface?

Should short codes expire after a time limit?

Should CLI retry setup automatically if the code is invalid?
