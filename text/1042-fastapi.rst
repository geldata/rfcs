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

.. code-block:: bash

    $ uvx gel project init

ideally followed by

.. code-block:: bash

    $ fastapi dev

Codegen
=======

* Gel's modules should be reflected into Python files.

* We will be using a hard dependancy on **pydantic**.

* A reflected Gel module should be represented by two files, e.g.

``default.gel`` -> ``default.py`` and ``__default_extras.py``, where:

* ``default.py`` should only contain 1:1 mapping of Gel types to Python dataclasses. Nothing extra, we optimize for maximum readability.

* ``__default_extras.py`` should contain all additional FastAPI-helper classes etc.

* ``default.py`` as the last line can have ``from .__default_extras import *``, followed by ``__all__ += (...)``.


Client connection API
=====================

* ``client.query()`` and the other ``query*`` methods will be returning ORM objects.

* ``create_client()`` and ``create_async_client()`` are to become idempotent.

* Use ``create_client(detached=True)`` for creating a client with a separate internal pool from the rest.


ORM
===

* No concept of session. We're implementing simple Django-inspired API for the masses.

* Objects will implement ``__eq__`` and ``__hash__`` based on Gel's object ID. We will always be fetching Gel's ID (implicitly). This mimics Django, Ruby's active record, etc.
  - not yet saved objects won't hash or eq.

* Reflected Gel types will have class methods on them to build queries, e.g. ``User.select(..)``, ``User.filter(..)``, etc.

* Python's ``User.select()`` will be equivalent to ``select User { * }`` in EdgeQL -- this is the only way how we can make Python limited typing work.

* We'll be adding a new ``lazy`` annotation / field to exclude properties/computeds from splats.

* Objects can be mutated and saved: ``user.name = 'Peter'; await client.save(User(name='Anna'), user)``

* For every Gel's type we'll be generating a number of Python classes:
  - one for "vanilla" top-level select / insert.
  - one for every link this type can be reached for -- this is needed for properly typing link properties. XXX1
  - link properties will be shadowing regular link/properties? XXX2

* Types will have a ``.with_link_props(link, **link_props)`` instance method, e.g. ``User(name='Anna').with_link_props(User.friends, weight=10)``

* Prisma-style API for custom shapes: ``User.select(name=True)``, ``User.select(name=True, settings=Settings.select())``? XXX3

* Non-prisma style API: ``User.select(User.name, User.settings.select(User.settings.background_color))``


Open questions
==============

1. Shorter alias for ``project init``? Maybe ``uvx gel init``? Why have ``project anyway``?
2. Status of Matt's work?
3. Elvis, we only have two weeks -- we need to figure out a PR to FastAPI in just a day or two max. We have to plug into ``fastapi dev``.
4. Where do we put the reflected schema files? ``.edgeql`` files? They need to be importable.
5. ``querySQL()`` -- what will it return? Do we want to support SQL records in our ORM?
6. Do we add ``lazy`` now? Or in 7?
7. XXX1 -- how exactly would they be used? We can't infer those types statically from a QB expression?
8. XXX2 -- did we agree on this?
9. Verify that our QB will be compatible with the new path factoring
10. XXX3 -- agree on this?
