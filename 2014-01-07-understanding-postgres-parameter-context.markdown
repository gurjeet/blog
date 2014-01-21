---
layout: post
title: "Understanding Postgres Parameter Context"
date: 2014-01-07 21:55:21 +0000
comments: true
categories:
- Postgres
- Administration
---

Postgres' parameters have an associated context, which determines when that
parameter can be changed.

You can see the context of every parameter using the following query. A sample
output is also shown.

    select name, context from pg_settings order by category;

                  name               |  context   
    ---------------------------------+------------
     autovacuum_freeze_max_age       | postmaster
     autovacuum_max_workers          | postmaster
     autovacuum_vacuum_threshold     | sighup
     ...
     IntervalStyle                   | user
     ...
     server_encoding                 | internal
     ...
     lc_messages                     | superuser
     ...
     local_preload_libraries         | backend
     ...

The possible values of context are:

- internal (called `PGC_INTERNAL` in source code)
- postmaster (`PGC_POSTMASTER`)
- sighup (`PGC_SIGHUP`)
- backend (`PGC_BACKEND`)
- superuser (`PGC_SUSET`)
- user (`PGC_USERSET`)

The above list is in order of when a parameter can be set; if a parameter can be
changed in a certain context, then it can be changed at any of the earlier
contexts as well.

### internal

The `internal` parameters cannot be changed; these are usually compile-time
constants. If you want to change any of these, you'll have to change it in
Postgres source code and compile a new set of Postgres executables.

### postmaster

The `postmaster` parameters can be set at Postgres startup, or during source code
compilation. (Postmaster is the parent process of all the Postgres processes,
hence the context's name).

These parameters can be set in the `postgresql.conf` file or on the command-line
when starting the Postgres server.

### sighup

The `sighup` parameters can be changed while the server is running, at Postgres
startup, or during code compilation.

To change such a parameter, you can change it in the `postgresql.conf` file and send a
`SIGHUP` signal to the Postmaster process. An easy way to send the `SIGHUP` signal
to the Postmaster process is to use `pg_ctl` or your distribution's Postgres-init
script, like so:

    pg_clt -D $PGDATA reload
OR
    sudo service postgresql-9.3 reload

### backend

The `backend` parameters can be changed/set while making a new connection to
Postgres, and never after that (and these can be changed by `SIGHUP`, at Postgres
startup, or during code compilation).

Usually the applications set these parameters while making the initial connection.

An example is the `local_preload_libraries` parameter. Say, if you want to try a
plugin for just one session, then you can initiate a `psql` session, with that
plugin loaded for the connection, like so:

    PGOPTIONS="-c local_preload_libraries=my_plugin" psql

The above method of changing parameters is possible for any application that
uses `libpq` library to connect to Postrges (for eg. [pgAdmin](http://pgadmin.org/)), since the `PGOPTIONS`
environment variable is recognized and honored by `libpq`. Other applications/libraries
may have their own methods to allow changing parameters during connection initiation.

### superuser

To change a `superuser` parameter, one needs to have `superuser` privileges in Postgres.
These parameters can be changed while a session is in progress, during a backend
startup, using `SIGHUP`, at Postgres startup, or during code compilation.

Note that normal users cannot change these parameters.

### user

The parameters with `user` context can be changed by any user, at any time, to
affect the current session they are connected to. Needless to say that since this
is the last context in the list, a parameter that is marked as `user` context, can be changed
using any of the methods shown for the other contexts.

The `SET` command can be used to change a `user` context parameter's value, for eg.:

    SET work_mem = '32 MB';

## Using context in queries

Although, as explained above, there is a certain order in the values of `context`,
there is no built-in way for one to see this order, and exploit that knowledge
using queries.

Say, if one wants to see a list of all parameters that cannot be
changed by a normal user, there's no straightforward way to do it. To that end,
I create the following `enum` type and use it in queries to extract that information easily:

    create type guc_context as enum (
        'internal',
        'postmaster',
        'sighup',
        'backend',
        'superuser',
        'user');

    select name as cannot_be_changed_by_user, context
    from pg_settings
    where context::guc_context < 'user';

Other useful information that can now be easily extracted using this `enum`:

    select name as parameter,
        context_enum > 'internal' as can_be_changed,
        context_enum = 'postmaster' as change_requires_restart,
        context_enum >= 'sighup' as can_be_changed_by_reload
    from (select name, context::guc_context as context_enum
        from pg_settings) as v;

                parameter            | can_be_changed | change_requires_restart | can_be_changed_by_reload 
    ---------------------------------+----------------+-------------------------+--------------------------
     allow_system_table_mods         | t              | t                       | f
     application_name                | t              | f                       | t
     archive_command                 | t              | f                       | t
     archive_mode                    | t              | t                       | f
     archive_timeout                 | t              | f                       | t
     array_nulls                     | t              | f                       | t
     authentication_timeout          | t              | f                       | t
     ...


