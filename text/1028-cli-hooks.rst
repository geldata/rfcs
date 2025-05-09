::

    Status: Draft
    Type: Feature
    Created: 2025-01-14
    Authors: Victor Petrovykh <victor@edgedb.com>,
             Aljaž Mur Eržen <aljaz@edgedb.com>

===================
RFC 1028: CLI hooks
===================

This RFC discusses addition of hooks to CLI commands. In order to make it
easier to setup and maintain some of the things dependent on the schema state
we can introduce hooks for some of the schema altering commands.


Motivation
==========

There are a number of tools that generate code based on the current schema
(such as query-building tools, or query reflection tools). They need to be
re-run after every schema change in order to keep the generated code in sync
with the database. In larger projects the number of things that may need
updating after a schema change is likely to grow. Migration hooks could make
it easier to automate these tasks.

There is also a longstanding need to implement some kind of mechanism for
creating data fixtures. Unlike the migration hooks, fixtures are more likely
to be run at project initialization or after a branch wipe command.

Also, there is a need to watch for file changes (schema, queries) and
potentially respond to these by running some scripts.


Hooks Configuration
===================

All hooks are intended to be used within an active project, not for arbitrary
CLI commands. Thus if a CLI command receives an explicit ``-I <instance>``
option, the command should not trigger any project hooks even if the instance
is linked to a project. Only commands that omit the instance and use the
current project to determine the target instance are going to trigger hooks.

The hooks will be described in the ``gel.toml`` project manifest file in the
``[hooks]`` table. We define the following hook keys:

* ``project.init.before``
* ``project.init.after``
* ``branch.switch.before``
* ``branch.switch.after``
* ``branch.wipe.before``
* ``branch.wipe.after``
* ``migration.apply.before``
* ``migration.apply.after``
* ``schema.update.before``
* ``schema.update.after``

Their values are strings that are going to be executed as shell commands.

The naming scheme is intended to mimic the command structure to clearly
indicate which commands will trigger the hooks. The exception to that is the
``schema.update`` hooks that are intended to trigger in response to any kind
of schema change, regardless of which command causes it.

All hooks will use the project root directory as the execution directory.
Hooks are executed using ``/bin/sh`` on Unix-like platforms and ``cmd.exe``
on Windows.

If the shell exits with a non-zero status code, the CLI will exit immediately,
without executing any subsequent hooks or CLI actions.

The hooks have two versions: ``before`` and ``after``. The ``before`` hooks
are intended to be triggered prior to the command making any changes. That
way, any error while executing a ``before`` hook would cause the command to
fail before it applies any changes. The ``after`` hooks are intended to
trigger after all the direct effects of the triggering command have been
resolved. The ``after`` hooks are intended to work with the new and updated
state of Gel.


Command Hooks
-------------

This category of hooks are intended for one-time setup of important fixtures
or application configuration. They are perfect for setting exporting
connection strings for non-gel tools to use. They are primarily triggered by
the corresponding CLI commands, but sometimes commands may trigger more than
one hook:

* ``gel project init`` command triggers the ``project.init.before``
  and ``project.init.after`` hook. If the migrations are applied at the end of
  the initialization, then the ``migration.apply.before``,
  ``schema.update.before``, ``migration.apply.after``, and
  ``schema.update.after`` hooks are also triggered.
* ``gel branch switch`` command triggers ``branch.switch.before``,
  ``schema.update.before``, ``branch.switch.after``, and ``schema.update.after``
  hooks in that relative order.
* ``gel branch wipe`` command triggers the ``branch.wipe.before``,
  ``schema.update.before``, ``branch.wipe.after``, and ``schema.update.after``
  hooks in that relative order.
* ``gel branch rebase`` and ``gel branch merge`` commands trigger
  ``migration.apply.before``, ``schema.update.before``,
  ``migration.apply.after``, and ``schema.update.after`` hooks in that
  relative order. Notice that although these are branch commands, but they do
  not change the current branch, instead they modify and apply migrations.
  That's why they trigger the ``migration.apply`` hooks.
* ``gel migrateion apply`` (or ``gel migrate``) command triggers
  ``migration.apply.before``, ``schema.update.before``,
  ``migration.apply.after``, and ``schema.update.after`` hooks in that
  relative order.

Overall the order in which the hooks are triggered is this:

#. ``project.init.before``
#. ``project.init.after`` (signifying that the project is initialized
   and thus ready for the rest of the tools/hooks).
#. ``branch.switch.before``
#. ``branch.wipe.before``
#. ``migration.apply.before``
#. ``schema.update.before``
#. ``branch.switch.after``
#. ``branch.wipe.after``
#. ``migration.apply.after``
#. ``schema.update.after``


Schema Hooks
------------

These hooks are a bit more abstract than the ones corresponding to specific
CLI commands. The ``schema.update.before`` and ``schema.update.after`` hooks
are intended for scripts that sync up the codebase with the schema state
(regardless of what caused the schema state to change). It may be re-compiling
the query-builder or reflected individual queries. It may be generating an ORM
compatibility layer. It could also perform any project-specific custom
operation, like integration tests, type-checkers, or linters.

The effects produced by these hook scripts should be idempotent, so that
running them several times on the same schema (or nearly same schema with
trivial changes) is safe.

The ``gel watch`` command will also trigger these hooks (without a need of a
separate TOML config section). This is due to the fact that the ``watch``
command conceptually monitors the schema file changes and automatically
creates and applied migrations. So the schema file changes would trigger
``migration.apply.before``, ``schema.update.before``,
``migration.apply.after``, and ``schema.update.after`` hooks in that relative
order.


Watch Configuration
===================

Sometimes in a Gel project there's a need to respond to some file changes: run
a migration due to schema change or run code generators due to query file
change. The CLI already supports watching the schema for changes and applying
them, but we can generalize this mechanism and make it more flexible, much
like the hooks that respond to CLI commands.

The idea is to have a mechanism for specifying a file system path you want to
watch for modifications and a script that gets triggered by those changes.
Unlike hooks, there are no "before" and "after" triggers here. Only one kind
of response is possible: trigger when a change to the watched entity is
detected.

The watch configuration will be described in the ``gel.toml`` project manifest
file in the ``[[watch]]`` table array. Each ``[[watch]]`` element represents a
single watch trigger.

Each watch trigger should have the following structure:

* ``files = ["<path-string>"]`` - specify file(s) being watched; required
* ``script = "<command>"`` - command to be executed by the shell; required

We may add more configuration options in the future to the watch triggers to
expand functionality.

The ``gel watch`` command (without needing further options) would then be used
to start the watch process and monitor whatever is specified in the
``gel.toml``. By default ``gel watch`` will only watch the files specified in
the ``gel.toml`` config and execute the trigger scripts.

Running ``gel watch --migrate`` will additionally monitor schema changes and
perform real-time migrations. The ``--migrate`` option also will automatically
retry applying migrations if it encounters network errors.

This is a backwards incompatible change compared to how ``edgedb watch`` operates
now.

The output of the scripts will then appear in the same TTY as the ``gel
watch`` command. The output should be prefixed by the ``files`` value so that
we can indicate which trigger produced the output when several scripts run in
parallel.


Trigger Execution
-----------------

Due to the nature of monitoring changes, we cannot guarantee any particular
order in which the watch scripts will be executed when multiple changes occur
at once. This is independent of the source of multiple triggers: whether it is
due to multiple files being updated or multiple watch rules matching the same
file. The scripts are effectively triggered asynchronously and the exact order
is an implementation detail that should not be relied upon.

It is possible to trigger the same watch target multiple times in a row either
by multiple consecutive update events to the same file or by virtue of the
watch trigger monitoring several files which have been modified. We want to
reduce unnecessary repeated execution of the same trigger script by debouncing
multiple file system change events into a single trigger event.

By default we want all trigger events to have a 100ms debounce delay before
executing. This value may be configurable in the future and be different for
each trigger.

Another consideration affecting debouncing is whether the same trigger script
is already currently running. It means that ``gel watch`` must keep track of
all the currently running scripts so that when a new script from the trigger
queue becomes eligible to run an additional check can be made. If there is
already a copy of the same trigger event script running, the newly eligible
copy is auto-debounced while the previous copy is still running. This ensures
that only one copy of any trigger script is actually running at the same time,
further reducing unnecessary churn.

Another consequence of executing triggered scripts in the background is that
sometimes the changes are invalid in some way and the script will fail. This
is considered part of normal operation (e.g. syntax error in a saved query
file) and the scripts should be such that failure does not create some
non-recoverable state.


Files
-----

The ``[[watch]]`` elements should contain at least ``files`` and
``script`` keys. The ``files`` value is an array of file system paths. It
can also contain common glob patterns (such as provided by `this library
<https://docs.rs/globset/latest/globset/#syntax>`_).

The paths should always use ``/`` (*nix-style) as path separators. This
uniformity ensures that the ``gel.toml`` config file is not system-dependent
(at least as long as it relies on relative paths). It is the job of the CLI
tools to correctly convert *nix paths into Windows paths internally when
necessary. The relative file paths are assumed to start at the project root.

The values corresponding to the keys are strings that are going to be executed
as shell commands, much like for the hooks.

All watch scripts will use the project root directory as the execution
directory. They are executed using ``/bin/sh`` on Unix-like platforms and 
using ``cmd.exe`` on Windows.

An example of this configuration::

    [[watch]]
    files = ["queries/*.edgeql"]
    script = "npx @edgedb/generate queries"

Only files in the project directories can be watched. If the ``gel.toml``
config specifies files outside of the project to be watched it should cause an
error for the ``gel watch`` command. The invalid spec should not be ignored
(silently or with a warning).


Project Interactions
====================

All of the above settings are intended as *project* settings. Which means that
the hooks can only be triggered by *project-specific* commands. If a command
overrides the default project settings (custom instance, branch, etc.) we can
no longer assume that the project hook is valid for that command and no hooks
will be triggered.

Similarly, the watch settings are invalid if they attempt to monitor files
outside of the project directory structure.

There should be a way to execute CLI commands without triggering any hooks. We
propose the ``--skip-hooks`` boolean flag to be added to the ``gel`` CLI
top-level options. If present it signals to the CLI that the current command
must not trigger any hooks and must behave as if no hooks are set up. Same
effect can be achieved by setting ``GEL_SKIP_HOOKS`` environment variable
to a truthy value.


Watch Interactions
==================

The proposed ``gel watch`` process is intended to help project development and
so we must consider how the ``gel watch`` monitoring and triggering interacts
with other tools likely to be used in a project. Primarily the concern is that
``gel watch --migrate`` causes changes in the database in response to changes
in files on disk. So we need to consider how the database environment can be
out of sync with the project code.

The danger is that reacting to a schema declaration change can result in an
unintended destructive (if things were removed from the schema) change in the
actual data.

Internal Gel Tooling
--------------------

The main source of changes to the database environment is from our own ``gel``
tools. We need to make sure that ``gel watch`` does not drift from what the
rest of the ``gel`` tools are doing.

Environment variables generally are shell-dependent and there is no reasonable
expectation that any such changes propagate without restarting whatever
application depends on them.

We have ``gel.toml`` configuration file that can be edited. In particular, it
can be edited to change the scripts that the hooks run. ``gel watch`` is
responsible for executing some of these triggered scripts and so it must
monitor ``gel.toml`` for changes and update its own behavior accordingly
(based on whether new files need to be monitored or hooks have been added or
removed, etc.). In the event that it detects a configurtation change that
cannot be satisfied by "reloading", the ``gel watch`` process should terminate
with an error explaning what change caused it and a recommendation to restart
manually.

We also have some project-specific configuration in the project stash
``$CONFIG/projects/`` (as specified in RFC #1025). Most of the settings in the
stash are specified by ``gel project init`` and don't change. However, the
current ``branch`` is updated by ``gel`` CLI branch tools. It is imperative
that ``gel watch`` never disagrees with the rest of the Gel tools or clients
on what the current branch is when applying changes.

Lock File
---------

The current proposal is to implement a lock file ``.gel-cli.lock`` that ``gel
watch`` will create. The lock file should contain the PID of the process that
created it. This file should be deleted upon exit.

When ``gel watch`` is started it must check the lock file and terminate with
an error if the file is present and the process referenced in it is running.
If the lock file references a non-existant process, the error message can say
that the file appears to be stale. The recommendation is to still terminate
the current ``gel watch`` and let the user handle the verification of other
instances of ``gel watch`` running and manual removal of the lock file.

The lock file itself can contain a text warning that the user should verify
that the process which created the file is terminated before removing the lock
file itself.

The ``gel`` CLI should check if the lock file exists before making any changes
involving the current branch. Such changes include:

* changing current branch
* rebasing or merging where one of the branches
  involved is the current branch
* wiping a branch
* restoring the current branch from backup

If the lock file is detected the command must fail. The error message should
explain that the specific branch is protected while ``gel watch`` is running
and that the recommended procedure is to finalize the schema changes (``gel
migration create`` + ``gel migrate --dev-mode``) and to stop ``gel watch``
before proceeding.

Importantly, it does not matter whether the ``gel`` command would trigger
hooks (using project defaults) or specifies the instance directly and would
bypass the hooks. As long as it affects a branch that is associated with a
project instance with a lock file, that branch must not be altered.

In general, the lock file can be used to determine if ``gel watch`` is running
and whether it is safe to alter the schema via some different tool.

Git Tooling
-----------

External tools are most likely to affect the schema files without necessarily
affecting the underlying schema. The main focus here will be on ``git``,
however we can later generalize some of this to other versioning tools.

There are a couple of important ways in which ``git`` can change the schema
declaration files: by changing git branches or by reverting or in any other
way getting back to an earlier commit within the same git branch.

Reverting is a fairly explicitly destructive operation so drastic changes to
the schema by ``gel watch -m`` is acceptable. Backups can be used to guard
against losing dev data.

Changing a git branch, however, is not only a common practice, but also may
involve updating the current gel branch. If we expect the gel branch to
change, we should not let ``gel watch -m`` apply any of the schema changes to
the old branch. This is difficult to prevent, though.

We propose to record the current git branch when ``gel watch -m`` starts. Then
any time ``gel watch -m`` is about to change the schema it must check that the
git branch didn't change since it started. If the git branch change is
detected, ``gel watch`` must terminate with an error suggesting that it
detected a git branch change and must be restarted to make sure that it
correctly maps the schema changes to the correct gel branch.


Design Considerations
=====================

It makes sense to follow a convention of filling out the ``[hooks]``
table in order of execution priority from highest to lowest::

    [hooks]
    project.init.after="setup_dsn.sh"
    branch.wipe.after=""
    branch.switch.after="setup_dsn.sh"
    schema.update.after="gel-orm sqlalchemy --mod compat --out compat"

The order in which hook *keys* appear does not impact their priority (we don't
want people getting subtle bugs due to different key order). It would simply
be a convention for any of our examples or auto-generated TOML configs.

These are mostly intended to be development aids so we may need an additional
mechanism for distinguishing cloud *development* and *production* instances to
prevent accidental triggering of the hooks in production.


Future Possibilities
====================

The debouncer delay can be configurable in the ``[[watch]]`` section as
``deboucer-delay=<integer>`` with the value being the delay in milliseconds.
This means that it's no longer sufficient to check the head of the trigger
queue to find the script to be executed. The entire queue must be checked and
all triggers that are older than their delay time must be executed.

We can introduce a ``gel-config`` to ``[[watch]]`` spec for monitoring changes to
the various database config values and responding to them. This would make
``files`` an optional key and use ``gel-config = <string>`` as a way to specify
which config variable we are watching.

We can set up a logfile for watch scripts (because that creates a record which
survives closing of terminals or reboots) as an additional convenience
feature. This can be specified in the general ``[watch-global]`` section as
``logfile="<path-to-logfile>"``.

The process launched by a script could have additional environmental variables
such as ``GEL_HOOK_NAME`` with value, for example, ``branch.wipe.before``.

We can add the following optional keys to the ``[[watch]]`` element:

* ``retry-delay = <integer>`` - retry delay in milliseconds
* ``retry-attempts = <integer>`` - number of retry attempts

If the ``retry-delay`` key is absent we will use a delay of 5 seconds before retrying.

If the ``retry-attempts`` key is absent we will assume that there is no limit to retry attempts and the script must be retried until it exits with a success status code.

The presence of *at least one* the above keys would indicate that the watch
script must be retried if it had exited with a non-zero status code. In case
both of the keys are missing, no retry attempt should be made and the keys'
"default" values should be ignored.

An example of a valid retry config::

    [[watch]]
    files = [".env"]
    script = "./sync_with_remote.sh"
    retry-delay = "5s"

In general, it makes sense to execute triggered scripts in parallel. This
reduces the latency in case some scripts are slow-running. However, sometimes
(e.g. when scripts overlap in which files they change) it is desirable that no
two scripts are running at the same time within the project. We can add
``--serial`` flag to ``gel watch`` command in order force trigger scripts to
be executed one at a time. Note that if there are long-running scripts the may
be a significant delay between the triggering event and the actual script
execution if ``--serial`` mode is enabled.

Watch Improvements
------------------

Potentially ``gel`` CLI tools can be a lot more pro-active in verifying the
owner of the lock file and allow operations to succeed if the lock is stale.
The lock then is either removed or renewed depending on the specific command.

Ideally, we'd like to have a more robust connection between git branches and
gel branches (via git hooks). As long as we have our hooks, we no longer need
to stop ``gel watch`` after git branch changes (and we would allow ``gel
branch switch`` to work, since presumably that's what the hook does).

As stated any git branch change followed by editing and saving the schema will
cause ``gel watch`` to error out. This is suboptimal for the many cases where
the branch change was for a feature that didn't alter the schema and the
schema change happens organically later. Since we know both the original git
branch and the current one, we can check the ``dbschema`` diff between the
branches to see if ``gel watch -m`` is reacting to the result of the branch
change or to an unrelated edit. We might want to allow unrelated edits.

Technically, any version control tool other than git that has some analogue of
branches could emulate the specified behavior of stopping ``gel watch -m``
after branch changes. The ``migration.apply.before``hook could be used to
validate that the branch didn't change from the time when ``gel watch -m`` was
started if the branch info is recorded at start time (presumably by a user
script).


Rejected Ideas
==============

We don't want any extra environment variables to be setup for the hooks. You
could use whatever you would have used if you ran the scripts by hand from the
shell. For example, to get the instance name ``gel project info
--instance-name``. The idea is that the scripts wouldn't be relying on any
hook-specific magic and you could run them (and thus debug them) by hand with
the identical effects.

We don't want to provide a list of scripts to run for hooks and watch
triggers. This is because with a list of scripts we need to specify what
happens when some scripts fail. Should the rest of the list be executed or
aborted. Under different circumstances different approaches would make sense
and we would need to implement all these interaction variants. Instead any
such complexity can be handled inside a singe script that the trigger
references.

We no longer try to generalize the monitoring the schema file changes and
auto-migrating them as part of regular ``gel.toml`` watch spec. There is a bit
of special handling and safe-guards involved in that monitoring that make it a
little too special. Instead we offer ``--migrate`` flag to enable
auto-migrations.

The hooks order should not be "wrapped". Imagine the following "wrapped" order:

* ``branch.switch.before``
* ``schema.update.before``
* ``gel branch switch foo``
* ``schema.update.after``
* ``branch.switch.after``

If the ``brach.switch`` hooks are managing some branch-dependent ``.env``
changes such as the currently used Postgres connection string for a SQL tool,
then ``schema.update.after`` hook executes in the new branch, but with the old
connection string. So it could result in an inconsistent state for processing
schema changes if that requires to coordinate SQL and Gel schemas. A similar
argument can work even for ``before`` hooks, although it would be a little
less intuitive. The idea is that if an effect is a result of a cascade (i.e.
schema is about to change *because* of a branch change), it still makes sense
for the hooks to run in the order in which they logically trigger each other.
This gives the opportunity for the previous steps in this cascade to setup the
correct context for the next triggered step. For example, the
``branch.switch.before`` hook can setup a flag that tells
``schema.update.before`` to skip all validation checks since it's about to be
replaced entirely with whatever is in the new branch.

All the paths appearing in ``gel.toml`` should be using *nix-style path
separator ``/`` instead of allowing both *nix and Windows style paths. This is
to ensure that ``gel.toml`` remains compatible across different systems since
it's expected to be committed as part of a repository.

We no longer want to use a minimalistic "glob=script" setup for files being
watched. This format is hard to extend in the future if any additional config
options are deemed necessary. Instead we will use sub-tables
``[[watch]]`` for each watch trigger, making it easier to add custom
options on a per-trigger basis. The simplest watch example is still quite
short and arguably more easier to understand::

    [[watch]]
    files = ["queries/*.edgeql"]
    script = "npx @edgedb/generate queries"

Watch should not have a ``--retry-sec`` flag for controlling retry behavior
globally. Instead we should configure retry policy on a per-trigger basis in
the ``gel.toml``. This is due to the fact that auto-retry only makes sense for
non-deterministic failures such as network failures, but results only in
unnecessary churn for deterministic scripts such as compilation.

We don't suspend watching while executing scripts. If we suspend watching,
then the implementation detail of the order of triggered scripts becomes
extremely important. E.g. if I "save all changes" what happens? Do all of them
get detected on the same watch tick and multiple watch targets will be
triggered? Do some of the changes get detected first and then block other
changes from being watched? Would I get slightly different behavior from using
my key shortcuts to switch tabs and save 2-3 files in quick succession?

If we never suspend watching, then the results for the above are all the same
- all changes will be eventually noticed and all triggers will fire. If we
suspend watching, then the actual trigger pattern will be sensitive to how
exactly files were saved.


Backwards Compatibility
=======================

The ``gel watch`` command will operate in a way that is the opposite to what
it ``edgedb watch`` used to do. The functionality of ``gel watch`` to
specifically monitor schema changes and attempt to auto-apply them in real
time involves some special handling on our end and would be enabled via an
opt-in flag ``--migrate`` (or ``-m``). Thus running the ``gel watch`` command
will no longer monitor and auto-apply schema changes. To do this going forward
``gel watch --migrate`` needs to be used.
