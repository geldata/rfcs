::

    Status: Draft
    Type: Feature
    Created: 2025-04-15
    Authors: Yury Selivanov <yury@geldata.com>

==================================
RFC 1042: New FastAPI+Gel workflow
==================================

Installation
============

We're introducing a new command: ``gel init`` -- an alias to ``gel project init``.

Python+FastAPI development workflow starts with:

.. code-block:: bash

    $ uvx gel init

followed by

.. code-block:: bash

    $ fastapi dev

We're plugging into ``fastapi dev`` / ``fastapi run`` commands to automatically run ``gel watch`` under the hood. ``gel watch``, in turn, will run codegen and schema migrations.

Before making an automatic schema migration, the tooling will be automatically creating a snapshot of the branch, making it easy to return to the previous state.


Project structure
=================

::
    project/
        .git/
        +dbschema/
            default.gel
        app/
            +models/
                default.py
            main.py
        +gel.toml
        pyproject.toml

Where Gel-related files are prefixed with ``+``. In short:

* ``dbschema/`` is where the ``.gel`` files go.
* They will be automatically picked up by ``fastapi dev`` and reflected into Python files into ``app/models``.
* ``gel.toml`` is the Gel project marker will be generated automatically by ``gel init``.

Codegen
=======

* Gel's modules should be reflected into Python files inside ``app/models``.

* We will be using a hard dependancy on **pydantic**.

* A reflected Gel module should be represented by two files, e.g.

* ``default.gel`` -> ``default.py`` and ``__default_extras.py``, where:

  - ``default.py`` should only contain 1:1 mapping of Gel types to Python dataclasses. Nothing extra, we optimize for maximum readability.

  - ``__default_extras.py`` should contain all additional FastAPI-helper classes etc.

  - ``default.py`` as the last line can have ``from .__default_extras import *``, followed by ``__all__ += (...)``.

* ``app/models/__init__.py`` will re-export all top-level namespaces, so that ``from . import models; print(models.default.User)`` works.

* ``app/models/__init__.py`` will export ``get_db()`` and ``get_async_db()`` factories for sync and async ORM-aware clients respectively.

* Generated Python types will have a special ``__variants__`` class property to access "building bricks" -- fragments of types that can be re-shaped into different useful dataclasses, e.g.:

  - ``models.default.User.__variants__.Empty`` -- an empty type that has ``id`` and ``__tid__`` attributes.
  - ``models.default.User.__variants__.Optional`` -- a type with all links and properties marked as optiona.

* Generated Python types will *also* have a special ``__typeof__`` class property to access types of fields, useful for defining custom dataclasses without copy/pasting. E.g.:

  .. code-block:: python

    class Foo(models.default.User.__variants__.Empty):

        name: models.default.User.__typeof__.name

Client connection API
=====================

create_client() and create_async_client()
-----------------------------------------

These will be returning "raw" clients, where results of the queries will be records.

The main change will be that ``create_client()`` and ``create_async_client()`` will be idempotent. IOW they will re-use the same underlying connection pool unless explicitly created with ``detached=True``.

create_db() and create_async_db()
---------------------------------

These will be the new API for creating ORM-aware clients. They will also be the *preferred* API.

Users will be importing these from ``app.models``:

.. code-block:: python

    from app.models import create_async_db

These will be returning ORM-aware clients with APIs being supersets of sync/async "raw" clients:

* Their ``query*()`` methods will be returning ORM objects.

* They will have a ``save()`` method to sync changes to Python objects with the database.

* Just like ``create_client()``, ``create_db()`` connectors will be reusing the same underlying connection pool unless explicitly created with ``detached=True``.


ORM
===

* No concept of session. We're implementing simple Django-inspired API for the masses.

* Objects will implement ``__eq__`` and ``__hash__`` based on Gel's object ID. We will always be fetching Gel's ID (implicitly). This mimics Django, Ruby's active record, etc.

  - not yet saved objects won't hash or eq, raising an error.

* Reflected Gel types will have class methods on them to build queries, e.g. ``User.select(..)``, ``User.filter(..)``, etc.

* Python's ``User.select()`` will be equivalent to ``select User { * }`` in EdgeQL -- this is the only way how we can make Python limited typing work.

* We will also be *always* fetching all link properties by default.

* We'll be adding a new ``lazy`` annotation / field to exclude properties/computeds from splats.

* Objects can be mutated and saved: ``user.name = 'Peter'; await db.save(User(name='Anna'), user)``

* For every Gel's type we'll be generating a number of Python classes:

  - one for "vanilla" top-level select / insert.

  - one for every link this type can be reached for -- this is needed for properly typing link properties. E.g. in the following example::

    .. code-block:: python

        User.select(friends=User.friends.select())

    a. ``User`` will be the "vanilla" Gel's ``User`` type:

       .. code-block:: python

           class User:

               name: str
               friends: list[User__friends__User]

    b. where ``User__friends__User`` will be a special variant of ``User`` reachable by traversing the ``User.friends`` link:

       .. code-block:: python

           class User__friends__User(User):

               class __linkprops__:
                   weight: int | None

  - This enables link properties to always be fetched and never shadow regular properties or links. Accessing them will be possible via the ``__linkprops__`` property, e.g.:

    .. code-block:: python

        user = await db.get(User.select(friends=User.friends.select()))

        weight = user.friends.__linkprops__.weight  # type: `int | None`

* Prisma-style API for custom shapes: ``User.select(name=True)``, ``User.select(name=True, settings=User.settings.select())`` instead of ``User.select(User.name)``.

* We will prohibit shadowing links with detached types, e.g. this is correct:

  .. code-block:: python

      User.select(name=True, friends=User.friends.select())

  and this is incorrect and will raise an error:

  .. code-block:: python

      User.select(name=True, friends=User.select())

  (only in Python ORM, EdgeQL will still allow this.)


Changes to the CLI
==================

* ``gel init`` is an alias to ``gel project init``

* ``gel watch`` to gain a new flag to run auto-backups on schema changes (will be set by ``fastapi dev``)

* ``gel watch`` to automatically shutdown when the parent process dies

* ``gel backup`` and ``gel restore`` to gain incremental backup feature for *local* instances

* ``gel instance list-backups`` will gain support for listing backups for a *local* instance


Other decisions
===============

query_sql() will return records
-------------------------------

We can't make it return ORM objects.

lazy
----

We'll be adding a new ``lazy`` "field" to Gel types in 7.0. Setting ``lazy := True`` on a link, property, or computed will exclude if from ``select Type { * }``, ``select Type { ** }``, as well as ``Type.select()`` in Python query builder API.

GH issue: https://github.com/geldata/gel/issues/8601

Hard dependency on pydantic
---------------------------

It seems that the ``pydantic`` is the way for the community, let's enable it.


Always fetching link properties
-------------------------------

This allows us to simplify the query builder API, e.g. this is possible:

.. code-block:: python

    User.select(friends=User.friends.select())

instead of:

.. code-block:: python

    User.select(friends=User.friends.with_link_props().select())

where the hypothetical ``.with_link_props()`` would have a lot of problems, including being extremely verbose, but also total lack of Python type safety.


Open questions
==============

* Should we allow incremental development backups of *cloud* instances? What if a user prefers to develop against a cloud instance?

* If clients created by ``create_client()`` and ``create_async_client()`` are "raw", and ``create_db()`` is an ORM-aware client, how do we reflect ``.edgeql`` files into Python? "raw" clients will use types incompatible with ORM-aware clients, and so will be the generated code tailored for them.

* Can ``lazy`` be set on *link properties*? This can be a way to opt out from automatically fetching them by our query builder.

* Will our links be empty lists when not fetched or ``None``?

* Should ``.id`` be optional or required? For every object fetched form the database we will always have an ID, but for newly created objects we won't until they are saved.


Future work
===========

* Allow folding ``gel.toml`` into ``pyproject.toml``. This will simplify the workflow for pure-Python projects and potentially provide a good space for settings related to both FastAPI and Gel. We'll likely have to teach all client libraries to read ``pyproject.toml`` as well (``gel-python`` and ``gel-cli`` / ``gel-rust`` will *have to* anyway.)
