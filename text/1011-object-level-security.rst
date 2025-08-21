::

    Status: Accepted
    Type: Feature
    Created: 2022-04-18
    Authors: Elvis Pranskevichus <elvis@edgedb.com>, Victor Petrovykh <victor@edgedb.com>
    RFC PR: `edgedb/rfcs#0054 <https://github.com/edgedb/rfcs/pull/54>`_

===============================
RFC 1011: Object-Level Security
===============================

This RFC proposes adding a mechanism to declare per-type access control
rules in schemas to automatically enforce data access and modification on
per-object level.  This is analogous to row-level security (RLS) in SQL
systems.


Motivation
==========

Object-level security is a powerful feature which allows universal leak-proof
enforcement of data access policies across all uses of a database.  This is
important for security and compliance, but would also be tremendously useful
for backend-less applications that interact with EdgeDB via HTTP (either via
GraphQL or EdgeQL-over-HTTP).  A secondary motivation is that this will allow
implementation of `temporal databases <temporal>`_.

In order to make it easier to write and read these object-level security
policies it is useful to group them based on some common underlying semantics
(e.g. all user-access policies vs. policies for specific tools, etc.).
Explicit semantic grouping of policies may enable sharing of some common
policy features or meta-information.


Requirements
============

Although there is no direct dependency, this RFC requires
`RFC 1010 <1001-global-vars.rst>`_ (globals) to be implemented to be
practical.
See `Interaction with globals`_ below for details.


Specification
=============

This RFC proposes to add a new schema item type -- *Access Policy* -- that is
defined in the context of an object type and specifies the rules of rewriting
queries that refer to the enclosing object type.

The access policy is the basic unit of object-level security. Multiple access
policies can be grouped together into an *Access Group*. Policies within the
same group can share a common condition when they're applicable (reducing
code-duplication in individual policy expressions). They can also omit
individual policy names as long as all access policies can be identified by
the action they govern. Finally *access groups* would allow schema designers
to explicitly keep semantically related policies together to make maintenance
easier.


CREATE ACCESS POLICY
--------------------

Define a new access control policy for a given object type.

Required capabilities: DDL.

Synopsis::

    {CREATE|ALTER} OBJECT TYPE <type-name> "{"
        CREATE ACCESS POLICY <name>
            { ALLOW | DENY }
            { ALL | UPDATE | SELECT | UPDATE READ | UPDATE WRITE | INSERT | DELETE } [ , ... ]
            [ USING (<expr>) ]
    "}"

If at least one access rule is defined for a type, then all elements become
invisible/immutable by default and must be made visible/mutable by at least
one ``ALLOW`` rule.

An ``ALLOW`` rule *adds* all objects, for which the check expression ``<expr>``
evaluates to true, to the set of visible objects.  ``ALLOW`` rules are
combined using the ``OR`` operator, i.e. they are mutually additive.

A ``DENY`` rule removes objects for which the check expression ``<expr>``
evaluates to true from the set of visible objects.  ``DENY`` rules are combined
using the ``AND`` operator.

An ``UPDATE`` policy kind is an abbreviation for ``UPDATE READ, UPDATE WRITE``
while an ``ALL`` policy is an abbreviation for all five policy kinds.

Per ``SELECT``, ``UPDATE READ``, and ``DELETE`` policies, objects that
are outside of the visible set are silently skipped in any ``SELECT``,
``UPDATE`` or ``DELETE`` expression that scans the relevant type.
Note that every ``DELETE`` and ``UPDATE`` does an implicit ``SELECT``
to produce the set to be modified, and so ``SELECT`` policies restrict
the objects that can be modified by DML as well.

A ``UPDATE WRITE`` and ``INSERT`` policies specify a *validity* check
for new or updated objects and affects ``INSERT`` and ``UPDATE``
expressions, respectively: if the proposed object is outside of the
visible set, an error is raised immediately and the query is aborted.

Access policies on other types apply to ``using`` expressions, to prevent
information leaks through that channel.

The check expression ``<expr>`` may be omitted, which implies that the policy
matches all objects, e.g. this is equivalent to specifying ``using (true)``.

Example read policy::

    type Movie {
      property rating -> str;
      # Allow all movie objects to be read by default
      access policy default allow select;
      # But deny those that are rated 'R' to users aged under 17.
      access policy age_appropriate
        when ((global current_user).age < 17)
        deny select using (.rating = 'R');
    }

Example read/write policy::

    type Post {
      property author -> User;

      # Only allow reading to the author, but also
      # ensure that a user cannot set the `author` link
      # to anything but themselves.
      access policy author_only
        allow all using (.author = global current_user);
    }

Another example of combination of allow/deny policies::

    abstract type Owned {
      link owner -> User;

      # permit read access to owner
      access policy owner_only
        allow all using (.owner = global current_user);
    }

    abstract type Shared extending Owned {
      # allow read access to friends
      access policy friends_can_read
        allow select using (global current_user in .owner.friends);
    }

    # Post inherits policies from Shared
    # which allow access to either owner
    # or friends initially...
    type Post extending Shared {
      property private -> bool;

      # ... but restrict access to private posts to owner only
      # regardless of what permissions were granted in parent types
      access policy private_owner_only
        when (.private)
        deny all using (.owner != global current_user);
    }



ALTER ACCESS POLICY
-------------------

Alter the definition of an access control policy.

Required capabilities: DDL.

Synopsis::

    ALTER OBJECT TYPE <type-name> "{"
        ALTER ACCESS POLICY <name>
        [ "{" <subcommand>; [...] "}" ];
    "}"

    # where <subcommand> is one of

      CREATE ANNOTATION <annotation-name> := <value>
      ALTER ANNOTATION <annotation-name> := <value>
      DROP ANNOTATION <annotation-name>
      USING (<expr>)
      { ALLOW | DENY } { ALL | UPDATE | SELECT | UPDATE READ | UPDATE WRITE | INSERT | DELETE } [ , ... ]


DROP ACCESS POLICY
------------------

Remove an access control policy.

Required capabilities: DDL.

Synopsis::

    ALTER OBJECT TYPE <type-name> "{"
        DROP ACCESS POLICY <name>;
    "}"


CREATE ACCESS GROUP
-------------------

Define a new access group for a given object type.

Required capabilities: DDL.

Synopsis::

    {CREATE|ALTER} OBJECT TYPE <type-name> "{"
        CREATE ACCESS GROUP <name> "{"
            [ WHEN <condition> ]
            [ {CREATE|ALTER|DROP} ACCESS POLICY [<policy-name>] ... ]
        "}"
    "}"


The optional ``<condition>`` expression is evaluated for every object affected
by the statement and the policies from this group are applied only if the
expression evaluates to *true*.  It is essentially equivalent to joining
``<condition>`` with ``<expr>`` for each individual policy with an ``AND``
operator.  The reason for a standalone clause is that it makes it easier to
separate common conditions of *when* multiple policies are applied from the
more specific details of those policies.

Access policies on other types apply to both ``when`` and ``using``
expressions, to prevent information leaks through that channel.

The ``<policy-name>`` for each individual ``access policy`` is optional if
there is no more than one policy with the same "applicability", i.e. the
combination of ``allow``/``deny`` action and the specific kind of access
``ALL``, ``UPDATE``, ``SELECT``, ``UPDATE READ``, ``UPDATE WRITE``,
``INSERT``, ``DELETE``. This means that it's possible to group multiple
anonymous policies as long as they focus on a separate facet of access.

Example of access group::

    type Feature {
      property author -> User;

      # Only allow reading to the author, but also
      # ensure that a user cannot set the `author` link
      # to anything but themselves.
      access group user_access {
        # Restrict features access to owners.
        when (.author ?= global current_user);

        # Allow the owner to do anything they like to
        # their own features
        access policy allow all;
        # ... except delete them
        access policy deny delete;
      }
    }


ALTER ACCESS GROUP
------------------

Alter the definition of an access group.

Required capabilities: DDL.

Synopsis::

    ALTER OBJECT TYPE <type-name> "{"
        ALTER ACCESS GROUP <name>
        [ "{" <subcommand>; [...] "}" ];
    "}"

    # where <subcommand> is one of

      CREATE ANNOTATION <annotation-name> := <value>
      ALTER ANNOTATION <annotation-name> := <value>
      DROP ANNOTATION <annotation-name>
      WHEN (<condition>)
      RESET WHEN
      CREATE ACCESS POLICY [<policy-name>] ...
      ALTER ACCESS POLICY [<policy-name>] ...
      DROP ACCESS POLICY [<policy-name>]


DROP ACCESS GROUP
-----------------

Remove an access group.

Required capabilities: DDL.

Synopsis::

    ALTER OBJECT TYPE <type-name> "{"
        DROP ACCESS GROUP <name>;
    "}"


Interaction with globals
========================

Access policies are especially powerful when combined with RFC 1010
globals, because then data visibility can be globally adjusted with a single
``SET GLOBAL`` statement, which is very useful for authenticated/authorized
data access control.

Example::

    global user_id -> uuid;

    abstract object type Owned {
      required link owner -> User;

      access policy owner_only
        allow all (.owner.id = global user_id)
    }

    object type Purchase extending Owned;

    ...

    set global user_id := <uuid-1>;
    select count(Purchase);
    # 9
    set global user_id := <uuid-2>
    select count(Purchase);
    # 1


Bypassing policies
==================

A superuser can bypass the execution of query rewrite policies by setting
the ``apply_access_policies`` session configuration setting to ``false``.


Mandatory Role-based Access Control (RBAC)
==========================================

Coupled with the role-based permission system (discussed in a future RFC),
object-level security provides reliable mandatory RBAC, where an
``access policy`` is protected by role permissions and cannot be disabled
by unauthorized users.


Introspection
=============

Add an abstract ``schema::AccessSpec`` to represent both access policies as
well as access groups::

    abstract type schema::AccessSpec
      extending schema::InheritingObject, schema::AnnotationSubject {
      property expr -> std::str;
    };

Policies can be introspected via a new ``schema::AccessPolicy`` in the
introspection schema that is linked from ``schema::ObjectType`` or
``schema::AccessGroup`` via the ``access_policies`` link.  The
``schema::AccessPolicy`` is exposed as follows::

    type schema::AccessPolicy extending schema::AccessSpec {
      required property action -> schema::AccessPolicyAction;
      required multi property access_kinds -> schema::AccessKind;
      overloaded required property expr -> std::str;
    };

Access group can be introspected via a new ``schema::AccessGroup`` in the
introspection schema that is linked from ``schema::ObjectType`` via the
``access_policies`` link.  The ``schema::AccessGroup`` is exposed as follows::

    type schema::AccessGroup extending schema::AccessSpec {
      required multi property access_policies -> schema::AccessPolicy;
    };

The new ``access_policies`` link on ``schema::ObjectType`` is actually
targeting ``schema::AccessSpec`` to accommodate both the access policies and
the groups.


Implementation considerations
=============================

Access policies primarily affect what IR is generated for a given EdgeQL query.
``READ`` and ``DELETE`` rules wrap set references and transform every ``Foo``
reference into ``(SELECT Foo FILTER <allow-deny-filter>)``.

``WRITE`` actions insert an intermediate shape into ``INSERT`` and ``UPDATE``,
e.g.::

    INSERT Foo { prop := <value> }

is roughly transformed into::

    WITH
      input := { prop := <value> },
      checked := input {
        prop := prop IF (SELECT _ := <check_expr> FILTER _) ELSE raise()
      }
    INSERT Foo { prop := checked.prop }

If specified, the ``WHEN`` conditions must be taken into account, e.g by
combining directly with the ``ALLOW/DENY`` filters and check expressions.


Rejected Alternative Ideas
==========================

Generalized policy based query rewrite
--------------------------------------

A `previous version of this RFC <https://github.com/edgedb/rfcs/pull/50>`_
proposed a generic "query rewrite" mechanism allowing, besides security,
also trigger-like functionality, but such bundling and generality was
deemed to be too complex, and the decision was made to add explicit mechanisms
for object-level security and (in a future RFC) support for trigger actions.

Use database views (a.k.a. contexts) to implement security
-------------------------------------------------------------------------

A proposal was made to implement security on schema-level instead of
type-level, e.g::

    context Authenticate (auth_method -> AuthMethod, token_id -> str) {
      type view User using (
        SELECT User
        FILTER .session.auth_method = global auth_method
               AND .session.token_id = global token_id);
      type view Sessions using (
        SELECT Sessions
        FILTER .auth_method = global auth_method
               AND .token_id = global token_id );
    }

    context User (user_id -> uuid) {
      type view User using (
        SELECT User Filter .user_id = global user_id);
      type view Article using (
        SELECT Article FILTER .owner.id = global user_id);
      type view PublicArticle using (
        SELECT Article FILTER .public);
    }

Context would then need to be activated::

    SET CONTEXT User { user_id: = <uuid>$user_id };

This proposal was rejected because this design poses significant challenges to
composition, i.e. composing several levels of security without the need to
duplicate large chunks of schema, as well as lack of support for mandatory
access control, as contexts are application-centric and are opt-in.


Factoring out common expression
-------------------------------

The previous iteration of the RFC proposed a ``when`` clause for each ``access
policy``. However, in that implementation it was effectively an arbitrary
splitting of the ``using`` expression into two parts, without a clear
advantage.

With the introduction of ``access group`` the ``when`` clause moved there
instead and is now applicable to all the individual policies within the group.
This allows for factoring out of common expressions that are relevant to all
access (e.g. based on the current user) and potentially simplifies the
expressions used by the policies themselves.


Adding grouping functionality to access policy
----------------------------------------------

The idea of allowing multiple sub-policies under a single ``access policy``
was rejected in favor of ``access group``. There are two major factors
contributing to the rejection:

1) The DDL changes necessary to support fine-tuning of sub-policies clash with
the current implementation in RC2. Currently, specifying ``alter access policy
my_policy allow select`` would effectively *drop* all other kinds of
sub-policies and either *create* a new ``select`` sub-policy or *alter* the
expression for an existing one. This would have to be side-by-side with
explicit individual commands like ``create allow select`` or ``alter allow
select``. The danger is that forgetting the ``create``/``alter``/``drop``
keyword would result in a valid command, but with very different meaning.

2) The ``access group`` can have all the same benefits as extending the
functionality of ``access policy``: the semantically meaningful ``when``
condition and making the need to name each policy individually unnecessary. It
also has the added benefit of actually allowing multiple *named* policies to
target the same type of access and still be grouped together. This increases
the flexibility of the feature and makes it possible for schema designers to
organize access based on type of access (e.g. all different ``select`` access
rules grouped together, etc.).


Backwards compatibility
=======================

The ``when`` clause in ``access policy`` is backwards incompatible with the
v2.0-rc2 implementation. We can leave it as allowed syntax for the purpose of
migrations and interpret it simple as an expression that must be added to the
``using`` expression with a conjunction.

Thus this migration command::

  create access policy owner_only
    # Must be logged in
    when (exists global user_id)
    # Allow viewing your own stuff
    allow select (.owner.id ?= global user_id);

... would be translated into this::

  create access policy owner_only
    allow select ((exists global user_id) and .owner.id ?= global user_id);
