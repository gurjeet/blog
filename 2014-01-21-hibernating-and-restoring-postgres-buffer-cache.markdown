---
layout: post
title: "Hibernating and Restoring Postgres Buffer Cache"
date: 2014-01-21 03:01:45 +0000
comments: true
categories:
- Postgres
- Administration
---
With the introduction of [`pg_prewarm`][pre_warm_commit] extension in Postgres,
it has become very easy to save and restore the contents of Postgres server's
buffer cache across a server restart. The tables, indexes and other on-disk data
structures (like [TOAST][TOAST_link] data, Visibility Map, etc.) are cached in
shared-buffers before they can be read or modified by user queries. Hence, by
saving the shared-buffers before a server restart, and restoring those buffers
after the restart, you can expect to reduce the ramp-up time, and hence expect
peak performance almost right off the bat.

[pre_warm_commit]: http://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=c32afe53c2e87a56e2ff930798a5588db0f7a516

[TOAST_link]:http://www.postgresql.org/docs/9.3/static/storage-toast.html

Following is a brain-dead way of implementing hibernation, and I am sure it can
be optimized to reduce the number of calls to `pg_prewarm`, since it allows the
caller to specify a range of blocks to be loaded, and I'm asking it to load just
one block per call.

We're going to use [`pg_buffercache`][pg_buffer_cache_93] to extract the list of
buffers currently loaded in buffer cache, and after a server restart, `pg_prewarm`
will be used to load those buffers back in.

[pg_buffer_cache_93]:http://www.postgresql.org/docs/9.3/static/pgbuffercache.html

## Declare environment variables to be used by scripts

    HIBERNATE_DESTINATION=$HOME/pg_hibernate/

    export PGUSER=postgres
    PSQL="psql"
    PSQL_TEMPL_DB="$PSQL -d template1"
    PSQL_PG_DB="$PSQL -d postgres"

    PSQL_TEMPL_NOFLUFF="$PSQL_TEMPL_DB -A -t"
    PSQL_PG_NOFLUFF="$PSQL_PG_DB -A -t"

## Prepare the databases

This is a one time operation. We avoid installing anything into `template0`
database, since it is a read-only database. But we do consciously install the
extension into `template1` database; this is so that any new databases created
after this point will get this extension pre-installed.

### Install extensions
    for db in $($PSQL_TEMPL_NOFLUFF -c 'select datname from pg_database where datname <> $$template0$$'); do
      echo Installing pg_prewarm in $db
      $PSQL_TEMPL_DB -c 'create extension if not exists pg_prewarm;'
    done

    echo Installing pg_buffercache extension in postgres database
    $PSQL_PG_DB -c 'create extension if not exists pg_buffercache;'

## Save buffer information

We are actually generating a `psql` script, that can be later fed to `psql`, as is.

    mkdir -p $HIBERNATE_DESTINATION

    for db in $($PSQL_TEMPL_NOFLUFF -c 'select datname from pg_database where datname <> $$template0$$'); do
      $PSQL_PG_NOFLUFF -c 'select    $q$select pg_prewarm((    select    oid
                                                             from      pg_class
                                                             where     pg_relation_filenode(oid) = $q$ || relfilenode || $q$)::regclass,
                                                         $$buffer$$, $q$
                                                         || case relforknumber
                                                            when 0 then $q$$$main$$$q$
                                                            when 1 then $q$$$fsm$$$q$
                                                            when 2 then $q$$$vm$$$q$
                                                            when 3 then $q$$$init$$$q$
                                                            end || $q$, $q$
                                                         || relblocknumber || $q$, $q$
                                                         || relblocknumber || $q$);$q$
                                        from     pg_buffercache
                                        where    reldatabase = (select    oid
                                                                from      pg_database
                                                                where     datname = $$'${db}'$$)
                                        order by relfilenode, relforknumber, relblocknumber;' > $HIBERNATE_DESTINATION/${db}.save
    done

## Restore the buffers

Ideally, this would be performed after a server restart, but there's no harm in
doing it without the restart either (except the performance implications, if
doing this eveicts buffers currently in use by other connections ;). Note that
if some table/index's underlying file storage has been renamed by the server
since we extracted the buffers list, some of these calls will return with `ERROR`.

We connect to each database, and simply feed the script generated earlier, into `psql`.

    for db in $($PSQL_TEMPL_NOFLUFF -c 'select datname from pg_database where datname <> $$template0$$'); do
      if [ ! -e $HIBERNATE_DESTINATION/${db}.save ]; then
        continue
      fi
      $PSQL -d $db -f $HIBERNATE_DESTINATION/${db}.save
    done

At this point, the Postgres shared-buffers should contain all the buffers that
were present when we extracted the buffer list from `pg_buffercache`.

PS: When reviewing the `pg_prewarm` code, I did not think through the user-experience
aspect of this extension. But after it was committed, the more I thought about
how it's going to be used, the less appealing this solution became. As is evident
from above, the administrator needs the help of two extensions, and then a storage
location where to store the list of buffers. Ideally, as a DBA, I would like to
see a feature which doesn't require me to muck around with catalogs etc. I have
a design for such an extension, and may start coding it some time soon.
