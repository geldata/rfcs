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

* We're introducing a new tool: ``gel generate``. It will:

  - simplify the implementation of code generators and improve the discoverability of them.

  - be part of our CLI and will be written in Rust. It will delegate the actual code generation to separaate tools. ``gel generate X`` will try the following:

    - check if ``gel-generate-X`` is available in ``PATH``, if so - run it.

    - check against the list of known generators, if found - run it. E.g. ``gel generate ts/querybuilder`` can attempt running ``npx @gel/ts-querybuilder``.

  - will provide the necessary CLI connection arguments and will be resolving the instance to generate code for. Once the instance is resolved, the tool can invoke the actual generator tool as a subprocess, passing the instance name/secrets via env vars.

  - will have a ``--list`` flag to list all available known generators.

* The ``.edgeql`` to ``.py`` code gen tools will be updated to customize the generated code in several ways:

  - add ``--model`` option to generate ORM-aware code. When passed (for example, ``--model=app.model``), the generated type for ``select User`` would be inherited from ``app.model.default.User.__variants__.Empty``.

  - add ``--file-suffix`` option to customize the suffix of the generated file. E.g. ``gel generate --file-suffix=blocking --target=sync`` would cause ``foo.edgeql`` to be reflected into ``foo_blocking.py`` with synchronous raw client API inside.

  - add ``--target=sync|async`` to generate code for sync and async clients respectively.

  - there will be no defaults for any of these flags.  When invoked manually the user will have to specify all of them.  Integrations like ``fastapi dev`` would auto-configure the flags internally, tuned to the current project.

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
  - ``models.default.User.__variants__.Optional`` -- a type with all links and properties marked as optional.

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

get_db() and get_async_db()
---------------------------

These will be the new API for creating ORM-aware clients. They will also be the *preferred* API.

Users will be importing these from ``app.models``:

.. code-block:: python

    from app.models import get_async_db

These will be returning ORM-aware clients with APIs being supersets of sync/async "raw" clients:

* Their ``query*()`` methods will be returning ORM objects.

* They will have a ``save()`` and ``delete()`` methods, more on them in the ORM section.

* Just like ``create_client()``, ``get_db()`` connectors will be reusing the same underlying connection pool unless explicitly created with ``detached=True``.


ORM
===

* No concept of session. We're implementing simple Django-inspired API for the masses.

* Objects will implement ``__eq__`` and ``__hash__`` based on Gel's object ID. We will always be fetching Gel's ID (implicitly). This mimics Django, Ruby's active record, etc.

  - not yet saved objects won't hash or eq, raising an error.

* ``db.save()`` will be a new method on the client object.

  - The function will attempt to insert / update the passed objects.

  - The function will be atomic, if for whatever reason one of the onbjects can't be saved -- none will be saved.

  - It's an error to attempt to mutate or ``save()``  an object that's being *currently* saved/deleted via the ORM APIs. E.g. ``user.friends += [joe]``, and 2 concurrent ``db.save(user)`` calls must not append two Joes.

* ``db.delete()`` will be another new method on the client object (similar to ``save()``).

  - The function will delete the passed objects by their IDs. So
    ``await db.delete(u1, u2)`` would be equivalent to
    ``delete (select User filter .id = u1.id or .id = u2.id)``.

  - The function will be atomic, if for whatever reason one of the onbjects can't be deleted -- none will be deleted.

  - Deleting an object that hasn't been saved will be a no-op.

  - Deleting an object that no longer exists in the database will also be a no-op.

  - Like for ``save()``, concurrent ``delete()`` calls will be rejected with an error.

* Reflected Gel types will have class methods on them to build queries, e.g. ``User.select(..)``, ``User.filter(..)``, etc.

* Python's ``User.select()`` will be equivalent to ``select User { * }`` in EdgeQL -- this is the only way how we can make Python limited typing work.

* We will also be *always* fetching all link properties by default.

* We'll be adding a new ``lazy`` annotation / field to exclude properties/computeds/link-properties from splats.

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

        weight = user.friends[0].__linkprops__.weight  # type: `int | None`

* ``.id`` is a required property in Gel, however, in Python *new* objects won't have an ``.id`` until they pass through a ``save()`` call. This presents a dilemma: should the ORM type's ``id`` property be optional or required? We say it will be required, because objects will be read more often than written, so we want to optimize for the common case. Accessing the ``id`` property on an unsaved object will raise a runtime error.

  - That said, the reflected types will have a custom ``__init__`` that will not accept an ``id`` argument (in disagreement with the type annotation for the field being "required").

  - The custom ``__init__`` will also require passing all required links, which would be in disagreement with all links being "optional" in Python by default.

* Prisma-style API for custom shapes: ``User.select(name=True)``, ``User.select(name=True, settings=User.settings.select())`` instead of ``User.select(User.name)``.

* We will prohibit shadowing links with detached types, e.g. this is correct:

  .. code-block:: python

      User.select(name=True, friends=User.friends.select())

  and this is incorrect and will raise an error:

  .. code-block:: python

      User.select(name=True, friends=User.select())

  (only in Python ORM, EdgeQL will still allow this.)

* Accessing a property or link that wasn't fetched will raise an error.

  - But, setting a property or link that wasn't fetched will *not* raise an error:

    .. code-block:: python

        user = await db.get(User.select(name=True))

        print(user.email)  # <- runtime error

        user.email = 'peter@example.com'  # <- fine!
        await db.save(user)

* "Multi" links and properties will be represented as list-like collections in Python:

  - Properties will be represented as ``gel.List`` (the exact name is still TBD) and will allow duplicate entries.

  - Links will be represented as ``gel.DistinctList`` and will not allow duplicate entries.

  - ``List`` and ``LinkList``, like Python's ``list`` will implement the ``+=`` operator to extend the list, e.g. ``user.friends += [friend]`` (note the square brackets on the right hand side).

  - unlike Python's ``list`` they will also implement the ``-=`` operator to remove items from the list, e.g. ``user.friends -= [friend]``.

  - ``LinkList`` will have a custom-tailored ``append()`` method that would accept keyword-only arguments to set link properties, e.g. ``user.friends.append(friend, weight=10)``.

    - This approach has benefits of being succinct and allows for full type safety.

    - Internally, such an ``append()`` call will create a "proxy" object wrapping the ``friend`` object and capturing the ``weight`` value.

      - The proxy is needed to avoid the ambiguity of ``weight`` being a property of ``User`` or ``User__friends__User``.

      - It's also needed to handle edge cases like:

        .. code-block:: python

            anna = User(name='Anna')
            u.friends.append(anna, weight=10)
            db.save(u, anna)

        in the above snippet we have to ensure that ``anna`` is not saved twice. Having a proxy object, as opposed to creating a new ``User__friends__User`` object with copied values, allows ``save()`` to special-case objects with link properties and ensure correct behavior.

      - The proxy object will have dynamic getters and setters updating the object it wraps on changes.

      - The proxy object can be used prior to being saved, but will be "dead" after the save:

        .. code-block:: python

            anna = User(name='Anna')
            u.friends.append(anna, weight=10)
            proxied = u.friends[0]
            proxied.__linkprops__.weight = 11 # <- changed my mind!
            save(u)

            print(proxied.name) # <- runtime error

      - Attempting to link an object that no longer exists in the database will be an error, aborting the ``save()`` call.

    - ``LinkList`` and ``List`` can only cause updates or removals on values/objects that they have fetched:

      - Calling ``u.friends.clear()`` will generate a query to remove only the objects that were present in ``u.friends``.

      - Calling ``u.friends.remove(some_user)`` will only succeed if ``some_user`` is present in ``u.friends``, otherwise it will raise an error.

        The reason for this behavior is that we can't know for sure how the link was fetched. For example, it could have been fetched without any filters, but it could also had an access policy in effect only giving it partial visibility. In such case, having ``u.friends.clear()`` wiping the link would be a disaster. Moreover, with this simple rule the API will be more debuggable & predictable for users.

        In the future we can consider adding an explicit method for generating a filter-less ``delete`` query on ``save()``, e.g. ``user.friends.delete_all()``.

      - We specifically are going with the list-like semantics instead of set-like semantics because we want deletion of a non-existing element to be an error, unlike ``set.discard()`` which would do nothing in such case. Moreover, Gel's sets when fetched can have an order, so at the very least we'd have to create an ``OrderedSet`` type (Python sets are unordered).

* We can raise ``ResourseWarning`` when an unsaved ORM object is GCed.


ORM & transactions
==================

* ORM objects will have to be aware of transactions and reset any changes made to them if the transaction is aborted. This is important because our transaction API is designed to be repeatable. Example:

  .. code-block:: python

       u = User(name='John', email='john.doe@example.com')

       try:
           async for tx in db.transaction():
               async with tx:
                   u.email = 'john@example.com'

                   some_other_user.friends += [u]
                   await tx.save(some_other_user)  # <- Let's say this fails.
        except:
            pass

        # The email we attemoted to set inside the transaction should be
        # reverted back.
        assert u.email == 'john.doe@example.com'

* Internally this can be implemented by maintaining an indirection inside the object attribute storage. Say we have two "buckets", one is the default implicit transaction bucket, and the other can be tied to the currently active transaction object (propagated through a ``contextvar``).

  - When a transaction starts and we attempt to mutate an object created outside of it, we'll copy its attributes into the transaction bucket and then layer updates there.

  - It's an error to mutate an object in more than one "concurrent" transaction block (otherwise reconciling changes would be non-deterministic).

  - This approach should be adaptable to support nested transactions (savepoints) in the future by maintaining a stack of states for the current transaction.

  - When a transaction is committed, we'll copy the changes from the transaction bucket back to the object's main bucket.

* In the v0 implementation we can skip supporting transactions at all on the ``get_db()`` clients.


ORM & batching
==============

* ``db.save()`` and ``db.delete()`` will accept multiple objects at once, allowing for batching multiple updates into a single round-trip to the database.

* We can consider implementing ``db.batch_save()`` method that would accept an asynchronous generator of objects to save. This would allow for internal distributing of commands accross multiple network connections


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


Why get_db() and not just create_client()?
------------------------------------------

The ORM client has to be tied to the specific reflected model, because:

* for Python typing, we *might* need to add some overloads of standard client APIs for better code completion experience.

* at runtime, codecs have to know which Python type is associated with a specific Gel's ``.__type__.id``.

We considered implementing magical runtime registry that the generic ``create_client()`` could be using to get the right Python type for a given Gel's ``.__type__.id``, but having such registry would complicate the code and open the door to weird edge cases (duplicate type IDs for different Gel branches or schemas, etc.)


Future work
===========

* Allow folding ``gel.toml`` into ``pyproject.toml``. This will simplify the workflow for pure-Python projects and potentially provide a good space for settings related to both FastAPI and Gel. We'll likely have to teach all client libraries to read ``pyproject.toml`` as well (``gel-python`` and ``gel-cli`` / ``gel-rust`` will *have to* anyway.)
