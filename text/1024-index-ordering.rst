::

    Status: Draft
    Type: Feature
    Created: 2023-08-09
    Authors: Victor Petrovykh <victor@edgedb.com>

========================
RFC 1024: Index Ordering
========================

This RFC outlines an improvement to EdgeDB indexes that should make it easier
for users to write queries that actually use the index speedups.


Motivation
==========

In Postgres the indexes support "ordering" specificaion, so one can write:
``CREATE INDEX test3_desc_index ON test3 (id DESC NULLS LAST);`` or ``CREATE
INDEX test3_xy_index ON test3 (x ASC, y DESC);``. This allows fine-tuning
indexes to the types of queries that you expect. We should replicate this
capability in EdgeDB.


Specification
=============

Instead of treating the ``on (...)`` clause as an expression, we can upgrade
it to actually be an ordering clause for the index with syntax much like order
by::

  index on (
      <expr> [ acs | desc ] [ empty { first | last } ] [ then ... ]
  )
  [ except ( <except-expr> ) ]
  [ "{" <annotation-declarations> "}" ] ;

For example::

  type Foo {
    text: str;
    x: int64;
    required y: int64;

    index on (.text asc empty last);
    index on (.x desc empty last then .y asc);
  }

Notice that this approach works well with indexes that use more than one field
and potentially need different ordering for different fields.

Additionally, since this syntax naturally allows specifying several fields for
indexing ``index on (.a then .b then .c)``, we no longer need to use a tuple
expression to do that, removing the special case and making the index
expression more semantically consistent.


Implementation
--------------

This change maps very naturally onto Postgres indexes. In particular, it
removes the need to have a special case handling for tuples as index
expression as we no longer need to use that format to create an index over
multiple fields::

  index on (.x desc empty last then .y asc);

can be translated into::

  CREATE INDEX foo_index ON "Foo" (x DESC NULLS LAST, y ASC);


Backwards Compatibility
=======================

This change is *syntactically* backwards compatible since all of the extra
specification for the index are optional. However, sematically, we should not
interpret index on a tuple expression as index on several fields anymore. We
should urge people to update their schemas to use this new version of index
instead of a tuple expression. In theory this would involve a migration that
only actually changes the schema text (and therefore the hash), but doesn't
affect existing indexes.


Security Implications
=====================

There are no security implications.


Rejected Alternative Ideas
==========================

. . .