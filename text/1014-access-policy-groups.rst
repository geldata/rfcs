::

    Status: Draft
    Type: Feature
    Created: 2022-07-21
    Authors: Victor Petrovykh <victor@edgedb.com>
    RFC PR:

==============================
RFC 1014: Access Policy Groups
==============================

This RFC proposes adding a mechanism to group per-type access control
rules in schemas.


Motivation
==========

In order to make it easier to write and read these object-level security
policies it is useful to group them based on some common underlying semantics
(e.g. all user-access policies vs. policies for specific tools, etc.).
Explicit semantic grouping of policies may enable sharing of some common
policy features or meta-information.


Requirements
============

This RFC requires `RFC 1011 <1001-object-level-security.rst>`_.


Specification
=============

This RFC proposes to add a new schema item type -- *Access Group* -- that is
defined in the context of an object type and can contain access policies
applicable to the enclosing object type.

The access policy is the basic unit of object-level security. Multiple access
policies can be grouped together into an *Access Group*. Policies within the
same group can share a common condition when they're applicable (reducing
code-duplication in individual policy expressions). They can also omit
individual policy names as long as all access policies can be identified by
the action they govern. Finally *access groups* would allow schema designers
to explicitly keep semantically related policies together to make maintenance
easier.


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


Introspection
=============

Add an abstract ``schema::AccessSpec`` to represent both access policies as
well as access groups::

    abstract type schema::AccessSpec
      extending schema::InheritingObject, schema::AnnotationSubject {
      property expr -> std::str;
    };

Update the ``schema::AccessPolicy`` in the introspection schema to be linked
from ``schema::ObjectType`` or ``schema::AccessGroup`` via the
``access_policies`` link.  The new ``schema::AccessPolicy`` becomes as
follows::

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


Rejected Alternative Ideas
==========================

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
