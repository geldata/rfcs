::

    Status: Draft
    Type: Feature
    Created: 2025-06-17
    Authors: Scott Trinh <scott@geldata.com>

==================================================
RFC 1043: Email-Based One-Time Code Authentication
==================================================

This RFC proposes an extension to the ``ext::auth`` module to support one-time codes (OTCs) as a configurable method for the existing email-based passwordless authentication flow. The primary motivation is to solve common cross-device usability issues inherent in "magic link" style authentication.


Motivation
==========

PKCE friction
-------------

The current ``ext::auth`` passwordless flow uses a "magic link" sent to a user's email. This flow is protected by a Proof Key for Code Exchange (PKCE) session that is initiated on the client device before the link is sent. This creates a significant usability problem in a common scenario:

1. A user starts the login process on their **desktop** (Device A). The client on Device A starts a PKCE session.
2. The user receives the magic link in their email and opens it on their **phone** (Device B).
3. The authentication attempt on Device B fails because it does not have the PKCE session information from Device A.

This forces users to ensure they open the link on the same device where they started the process, which is often inconvenient and leads to a frustrating user experience. A one-time code, which can be easily transcribed between devices, elegantly solves this problem.

Multi-factor authentication
---------------------------

You might want to trigger a one-time code authentication flow as part of a multi-factor authentication flow. For example, you might want to send a code to a user's email as part of a two-factor authentication flow, or securing a sensitive action. This would be implemented by the developer as having /both/ a primary factor (like Email+Password or OAuth), and a secondary factor (like Magic Link with code verification).

.. code-block:: esdl

  type User {
    multi identities: ext::auth::Identity {
      constraint exclusive;
    };
  };

.. code-block:: python

  async def something_sensitive(code: str, email: str):
      # Ensure this code is valid for this email address
      await user_service.verify_code(email, code)
      # Do something sensitive


Invite codes
------------

You might have an authentication setup where an admin user will invite other users to join a team or organization. This would be implemented by the developer as having /both/ a primary factor (like Email+Password or OAuth), and a secondary factor (like Magic Link with code verification), but the admin would only set up the secondary factor and when the user accepts the invite, they link a primary factor to the same user.

.. code-block:: esdl

  type User {
    multi identities: ext::auth::Identity {
      constraint exclusive;
    };
  };

And some pseudo-code:

.. code-block:: python

  async def invite_user(email: str, team_id: str):
      # Create the user and associate it with the team
      await user_service.create_team_user(team_id, email)

      # Create a new Magic Link based factor directly for the user
      await user_service.register_magic_link(email)

  async def accept_invite(email: str, code: str, primary_identity_id: str):
      # Verify the code
      user = await user_service.verify_code(email, code)

      # Link the user's primary factor to the secondary factor's user
      await user_service.link_primary_factor(user.id, primary_identity_id)


High-Level Proposal
===================

We will introduce a new method for email+password email verification and magic link called ``Code``. This will be a configurable alternative to the existing ``Link`` method.

The core of this proposal is to **decouple the PKCE session from the authentication initiation step**. When the ``Code`` method is enabled:

1.  The initiation request (to send a code to an email) will be PKCE-agnostic. It simply triggers the sending of an email.
2.  The verification step (submitting the code) will be responsible for initiating and completing the PKCE flow.

This allows a user to easily transcribe the one-time code from their email (which might be opened on a different device) to the device where they initiated the login process. The PKCE session remains correctly scoped to the device performing the final verification, ensuring security while improving usability.


Detailed Design
===============

Configuration
-------------

A new enum and configuration option will be added to the ``ext::auth`` module to allow developers to select the desired passwordless flow.

For the email password provider, we will add a new property to the provider config to allow the developer to select the verification method.

.. code-block:: esdl

    create scalar type ext::auth::VerificationMethod extending std::enum<Link, Code>;

    create type ext::auth::EmailPasswordProviderConfig
        extending ext::auth::ProviderConfig {
        # ... existing properties ...

        create required property verification_method: ext::auth::VerificationMethod {
            set default := ext::auth::VerificationMethod.Link;
        };
    };

And for the magic link provider, we will add a new property to the provider config to allow the developer to select the verification method.

.. code-block:: esdl

    create type ext::auth::MagicLinkProviderConfig
        extending ext::auth::ProviderConfig {
        # ... existing properties ...

        create required property verification_method: ext::auth::VerificationMethod {
            set default := ext::auth::VerificationMethod.Link;
        };
    };

The default value will be ``Link`` to ensure full backwards compatibility.

Schema
------

To manage the state of an in-progress OTC authentication, a new transient type will be introduced.

.. code-block:: esdl

    create type ext::auth::OneTimeCode {
        create required property code_hash: std::str {
            create constraint exclusive;
            create annotation std::description :=
                "The securely hashed one-time code.";
        };
        create required property expires_at: std::datetime {
            create annotation std::description :=
                "The date and time when the code expires.";
        };

        create required link factor: ext::auth::Factor {
            on target delete delete source;
        };

        create property max_attempts: int16 {
            default := 5;
        };
        create property attempts: int16 {
            default := 0;
        };
    };

The ``OneTimeCode`` object will be created when the flow is initiated and deleted immediately upon successful verification. We can also delete all expired ``OneTimeCode`` objects when the server makes a verification attempt to avoid needing a separate cleanup job. That means you could have a situation where you've only created a single ``OneTimeCode`` object, it expires and the user never verifies it, and it never gets deleted, but is still invalid, so the only cost is the storage of the object itself.

PKCE Flow and Authentication Ceremony
-------------------------------------

The key innovation of this proposal is the adjustment of the PKCE flow.

**Phase 1: Code Initiation (PKCE Agnostic)**

The client sends a request to initiate the flow for an email address. This request does not require a PKCE session. The server generates a user-friendly code, stores its hash in a ``OneTimeCode`` record, and sends the code to the user's email.

**Phase 2: Code Verification (PKCE Mandatory)**

The client sends a request to verify the code, which **must** include a PKCE ``code_challenge``. The server validates the code against the stored hash. If successful, it completes the PKCE flow and issues the auth token.

This decoupled flow enables two essential client-side patterns:

Same-Device Flow (Session Reuse)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1.  A user on Device A initiates the flow. The client immediately starts a PKCE session and stores the ``code_verifier``.
2.  The client sends the initiation request.
3.  The user enters the received code on Device A.
4.  The client sends the verification request, including the ``code_challenge`` from the **existing** PKCE session.

Cross-Device Flow (New Session)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1.  A user on Device A initiates the flow. The client sends the request without needing a PKCE session.
2.  The user receives the code and enters it into the client on Device B.
3.  The client on Device B now starts a **new** PKCE session and immediately sends the verification request with its ``code_challenge``.
4.  The authentication succeeds because the PKCE session is correctly scoped to Device B, which is performing the verification.


Design Considerations and Rejected Ideas
========================================

*   **A Separate ``CodeFactor``:** We considered introducing a new, distinct factor for OTC. This was rejected because this feature is fundamentally a different *method* for existing factors (e.g. Email+Password, Magic Link), not a new identity type. Reusing the existing factor simplifies the conceptual model and configuration.

*   **Separate HTTP Endpoints:** An initial idea was to create new endpoints like ``/send-code`` and ``/verify-code``. This was rejected in favor of augmenting the existing ``/verify`` and ``/magic-link/authenticate`` endpoints. A unified API is cleaner and makes the feature a true drop-in alternative.

*   **Client-Provided Codes:** We rejected any design that would allow a client to specify the code. The code must be generated and managed entirely server-side to prevent security vulnerabilities when the HTTP server for the extension is exposed to the public internet.


Out of Scope
============

*   **Time-Based One-Time Passwords (TOTP):** TOTP (e.g., from an authenticator app) relies on a long-lived shared secret and is a distinct authentication factor. It is considered out of scope for this RFC and may be addressed in a future proposal.

*   **Other Delivery Channels:** This RFC focuses exclusively on adding OTC to existing email-based factors. This allows us to leverage the existing SMTP and webhook infrastructure. Support for other channels like SMS or in-app notifications is out of scope, but can be implemented using the existing webhook infrastructure.


Backwards Compatibility
=======================

This proposal is fully backwards-compatible. The new functionality is opt-in via the ``verification_method`` configuration option, which defaults to ``Link``. Existing projects will continue to function without any changes.
