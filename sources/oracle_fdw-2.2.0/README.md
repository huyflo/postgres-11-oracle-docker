Foreign Data Wrapper for Oracle
===============================

oracle_fdw is a PostgreSQL extension that provides a Foreign Data Wrapper for
easy and efficient access to Oracle databases, including pushdown of WHERE
conditions and required columns as well as comprehensive EXPLAIN support.

This README contains the following sections:

1. [Cookbook](#1-cookbook)
2. [Objects created by the extension](#2-objects-created-by-the-extension)
3. [Options](#3-options)
4. [Usage](#4-usage)
5. [Installation Requirements](#5-installation-requirements)
6. [Installation](#6-installation)
7. [Internals](#7-internals)
8. [Problems](#8-problems)
9. [Support](#9-support)

oracle_fdw was written by Laurenz Albe, with notable contributions from
Vincent Mora of Oslandia and Tatsuro Yamada of the NTT OSS Center.

Special thanks to Christian Ullrich for ongoing help with Windows.

1 Cookbook
==========

This is a simple example how to use oracle_fdw.  
More detailed information will be provided in the sections
[Options](#3-options) and [Usage](#4-usage).  You should also read the
[PostgreSQL documentation on foreign data][fd] and the commands referenced
there.

 [fd]: https://www.postgresql.org/docs/current/static/ddl-foreign-data.html

For the sake of this example, let's assume you can connect as operating system
user `postgres` (or whoever starts the PostgreSQL server) with the following
command:

    sqlplus orauser/orapwd@//dbserver.mydomain.com:1521/ORADB

That means that the Oracle client and the environment is set up correctly.
I also assume that oracle_fdw has been compiled and installed (see the
[Installation](#6-installation) section).

We want to access a table defined like this:


    SQL> DESCRIBE oratab
     Name                            Null?    Type
     ------------------------------- -------- ------------
     ID                              NOT NULL NUMBER(5)
     TEXT                                     VARCHAR2(30)
     FLOATING                        NOT NULL NUMBER(7,2)


Then configure oracle_fdw as PostgreSQL superuser like this:

    pgdb=# CREATE EXTENSION oracle_fdw;
    pgdb=# CREATE SERVER oradb FOREIGN DATA WRAPPER oracle_fdw
              OPTIONS (dbserver '//dbserver.mydomain.com:1521/ORADB');
    pgdb=# GRANT USAGE ON FOREIGN SERVER oradb TO pguser;


(You can use other naming methods or local connections, see the description of
the option **dbserver** below.)

Then you can connect to PostgreSQL as `pguser` and define:

    pgdb=> CREATE USER MAPPING FOR pguser SERVER oradb
              OPTIONS (user 'orauser', password 'orapwd');


(You can use external authentication to avoid storing Oracle passwords;
see below.)

    pgdb=> CREATE FOREIGN TABLE oratab (
              id        integer           OPTIONS (key 'true')  NOT NULL,
              text      character varying(30),
              floating  double precision  NOT NULL
           ) SERVER oradb OPTIONS (schema 'ORAUSER', table 'ORATAB');

(Remember that table and schema name -- the latter is optional -- must
normally be in uppercase.)

Now you can use the table like a regular PostgreSQL table.

2 Objects created by the extension
==================================

    FUNCTION oracle_fdw_handler() RETURNS fdw_handler
    FUNCTION oracle_fdw_validator(text[], oid) RETURNS void

These functions are the handler and the validator function necessary to create
a foreign data wrapper.

    FOREIGN DATA WRAPPER oracle_fdw
      HANDLER oracle_fdw_handler
      VALIDATOR oracle_fdw_validator

The extension automatically creates a foreign data wrapper named `oracle_fdw`.  
Normally that's all you need, and you can proceed to define foreign servers.
You can create additional Oracle foreign data wrappers, for example if you
need to set the **nls_lang** option (you can alter the existing `oracle_fdw`
wrapper, but all modifications will be lost after a dump/restore).

    FUNCTION oracle_close_connections() RETURNS void

This function can be used to close all open Oracle connections in this session.
See the [Usage](#4-usage) section for further description.

    FUNCTION oracle_diag(name DEFAULT NULL) RETURNS text

This function is useful for diagnostic purposes only.  
It will return the versions of oracle_fdw, PostgreSQL server and Oracle client.
If called with no argument or NULL, it will additionally return the values of
some environment variables used for establishing Oracle connections.  
If called with the name of a foreign server, it will additionally return
the Oracle server version.

3 Options
=========

Foreign data wrapper options
----------------------------

(Caution: If you modify the default foreign data wrapper `oracle_fdw`,
any changes will be lost upon dump/restore.  Create a new foreign data wrapper
if you want the options to be persistent.  The SQL script shipped with the
software contains a CREATE FOREIGN DATA WRAPPER statement you can use.)

- **nls_lang** (optional)

  Sets the NLS_LANG environment variable for Oracle to this value.  
  NLS_LANG is in the form "language_territory.charset" (for example
  AMERICAN_AMERICA.AL32UTF8).  This must match your database encoding.
  When this value is not set, oracle_fdw will automatically do the right
  thing if it can and issue a warning if it cannot. Set this only if you
  know what you are doing.  See the [Problems](#8-problems) section.

Foreign server options
----------------------

- **dbserver** (required)

  The Oracle database connection string for the remote database.  
  This can be in any of the forms that Oracle supports as long as your
  Oracle client is configured accordingly.  
  Set this to an empty string for local ("BEQUEATH") connections.

User mapping options
--------------------

- **user** (required)

  The Oracle user name for the session.  
  Set this to an empty string for *external authentication* if you don't
  want to store Oracle credentials in the PostgreSQL database (one simple way
  is to use an *external password store*).

- **password** (required)

  The password for the Oracle user.

Foreign table options
---------------------

- **table** (required)

  The Oracle table name.  This name must be written exactly as it occurs in
  Oracle's system catalog, so normally consist of uppercase letters only.

  To define a foreign table based on an arbitrary Oracle query, set this
  option to the query enclosed in parentheses, e.g.

      OPTIONS (table '(SELECT col FROM tab WHERE val = ''string'')')

  Do not set the **schema** option in this case.  
  INSERT, UPDATE and DELETE will work on foreign tables defined on simple
  queries; if you want to avoid that (or confusing Oracle error messages
  for more complicated queries), use the table option **readonly**.

- **schema** (optional)

  The table's schema (or owner).  Useful to access tables that do not belong
  to the connecting Oracle user.  This name must be written exactly as it
  occurs in Oracle's system catalog, so normally consist of uppercase letters
  only.

- **max_long** (optional, defaults to "32767")

  The maximal length of any LONG, LONG RAW and XMLTYPE columns in the Oracle
  table.  Possible values are integers between 1 and 1073741823 (the maximal
  size of a `bytea` in PostgreSQL).  This amount of memory will be allocated
  at least twice, so large values will consume a lot of memory.  
  If **max_long** is less than the length of the longest value retrieved,
  you will receive the error message `ORA-01406: fetched column value was
  truncated`.

- **readonly** (optional, defaults to "false")

  INSERT, UPDATE and DELETE is only allowed on tables where this option is
  not set to yes/on/true.  Since these statements can only be executed from
  PostgreSQL 9.3 on, setting this option has no effect on earlier versions.
  It might still be a good idea to set it in PostgreSQL 9.2 and earlier
  on tables that you do not wish to be changed, to be prepared for an upgrade
  to PostgreSQL 9.3 or later.

- **sample_percent** (optional, defaults to "100")

  This option only influences ANALYZE processing and can be useful to
  ANALYZE very large tables in a reasonable time.

  The value must be between 0.000001 and 100 and defines the percentage of
  Oracle table blocks that will be randomly selected to calculate PostgreSQL
  table statistics.  This is accomplished using the `SAMPLE BLOCK (x)`
  clause in Oracle.

  ANALYZE will fail with ORA-00933 for tables defined with Oracle queries and
  may fail with ORA-01446 for tables defined with complex Oracle views.

- **prefetch** (optional, defaults to "200")

  Sets the number of rows that will be fetched with a single round-trip between
  PostgreSQL and Oracle during a foreign table scan.  This is implemented using
  Oracle row prefetching.  The value must be between 0 and 10240, where a value
  of zero disables prefetching.

  Higher values can speed up performance, but will use more memory on the
  PostgreSQL server.

Column options (from PostgreSQL 9.2 on)
---------------------------------------

- **key** (optional, defaults to "false")

  If set to yes/on/true, the corresponding column on the foreign Oracle table
  is considered a primary key column.  
  For UPDATE and DELETE to work, you must set this option on all columns
  that belong to the table's primary key.

4 Usage
=======

Oracle permissions
------------------

The Oracle user will obviously need CREATE SESSION privilege and the right
to select from the table or view in question.

For EXPLAIN VERBOSE the user will also need SELECT privileges on V$SQL and
V$SQL_PLAN.

Connections
-----------

oracle_fdw caches Oracle connections because it is expensive to create an
Oracle session for each individual query.  All connections are automatically
closed when the PostgreSQL session ends.

The function `oracle_close_connections()` can be used to close all cached
Oracle connections.  This can be useful for long-running sessions that don't
access foreign tables all the time and want to avoid blocking the resources
needed by an open Oracle connection.  
You cannot call this function inside a transaction that modifies Oracle data.

Columns
-------

When you define a foreign table, the columns of the Oracle table are mapped
to the PostgreSQL columns in the order of their definition.

oracle_fdw will only include those columns in the Oracle query that are
actually needed by the PostgreSQL query.

The PostgreSQL table can have more or less columns than the Oracle table.
If it has more columns, and these columns are used, you will receive a warning
and NULL values will be returned.

If you want to UPDATE or DELETE, make sure that the `key` option is set on all
columns that belong to the table's primary key.  Failure to do so will result
in errors.

Data types
----------

You must define the PostgreSQL columns with data types that oracle_fdw can
translate (see the conversion table below).  This restriction is only enforced
if the column actually gets used, so you can define "dummy" columns for
untranslatable data types as long as you don't access them (this trick only
works with SELECT, not when modifying foreign data).  If an Oracle value
exceeds the size of the PostgreSQL column (e.g., the length of a varchar
column or the maximal integer value), you will receive a runtime error.

These conversions are automatically handled by oracle_fdw:

    Oracle type              | Possible PostgreSQL types
    -------------------------+--------------------------------------------------
    CHAR                     | char, varchar, text
    NCHAR                    | char, varchar, text
    VARCHAR                  | char, varchar, text
    VARCHAR2                 | char, varchar, text, json
    NVARCHAR2                | char, varchar, text
    CLOB                     | char, varchar, text, json
    LONG                     | char, varchar, text
    RAW                      | uuid, bytea
    BLOB                     | bytea
    BFILE                    | bytea (read-only)
    LONG RAW                 | bytea
    NUMBER                   | numeric, float4, float8, char, varchar, text
    NUMBER(n,m) with m<=0    | numeric, float4, float8, int2, int4, int8,
                             |    boolean, char, varchar, text
    FLOAT                    | numeric, float4, float8, char, varchar, text
    BINARY_FLOAT             | numeric, float4, float8, char, varchar, text
    BINARY_DOUBLE            | numeric, float4, float8, char, varchar, text
    DATE                     | date, timestamp, timestamptz, char, varchar, text
    TIMESTAMP                | date, timestamp, timestamptz, char, varchar, text
    TIMESTAMP WITH TIME ZONE | date, timestamp, timestamptz, char, varchar, text
    TIMESTAMP WITH           | date, timestamp, timestamptz, char, varchar, text
       LOCAL TIME ZONE       |
    INTERVAL YEAR TO MONTH   | interval, char, varchar, text
    INTERVAL DAY TO SECOND   | interval, char, varchar, text
    XMLTYPE                  | xml, char, varchar, text
    MDSYS.SDO_GEOMETRY       | geometry (see "PostGIS support" below)

If a NUMBER is converted to a boolean, 0 means `false`, everything else `true`.

Inserting or updating XMLTYPE only works with values that do not exceed the
maximum length of the VARCHAR2 data type (4000 or 32767, depending on the
`MAX_STRING_SIZE` parameter).

NCLOB is currently not supported because Oracle cannot automatically convert
it to the client encoding.

If you need conversions exceeding the above, define an appropriate view in
Oracle or PostgreSQL.

WHERE conditions and ORDER BY clauses
-------------------------------------

PostgreSQL will use all applicable parts of the WHERE clause as a filter
for the scan.  The Oracle query that oracle_fdw constructs will contain a WHERE
clause corresponding to these filter criteria whenever such a condition can
safely be translated to Oracle SQL.  This feature, also known as *push-down
of WHERE clauses*, can greatly reduce the number of rows retrieved from Oracle
and may enable Oracle's optimizer to choose a good plan for accessing the
required tables.

Similarly, ORDER BY clauses will be pushed down to Oracle from PostgreSQL 9.2
on wherever possible. Note that no ORDER BY condition that sorts by a character
string will be pushed down as the sort orders in PostgreSQL an Oracle cannot be
guaranteed to be the same.

To make use of that, try to use simple conditions for the foreign table.
Choose PostgreSQL column data types that correspond to Oracle's types,
because otherwise conditions cannot be translated.

The expressions `now()`, `transaction_timestamp()`, `current_timestamp`,
`current_date` and `localtimestamp` will be translated correctly.

The output of EXPLAIN will show the Oracle query used, so you can see which
conditions were translated to Oracle and how.

Joins between foreign tables
----------------------------

From PostgreSQL 9.6 on, oracle_fdw can push down joins to the Oracle server,
that is, a join between two foreign tables will lead to a single Oracle query
that performs the join on the Oracle side.

There are some restrictions when this can happen:

- Both tables must be defined on the same foreign server.
- Joins between three or more tables won't be pushed down.
- The join must be in a SELECT statement.
- oracle_fdw must be able to push down all join conditions and WHERE clauses.
- Cross joins without join conditions are not pushed down.
- If a join is pushed down, ORDER BY clauses will not be pushed down.

It is important that table statistics for both foreign tables have been
collected with ANALYZE for PostgreSQL to determine the best join strategy.

Modifying foreign data
----------------------

From PostgreSQL 9.3 on, oracle_fdw supports INSERT, UPDATE and DELETE on
foreign tables.  This is allowed by default (also in databases upgraded
from an earlier PostgreSQL release) and can be disabled by setting the
**readonly** table option.

For UPDATE and DELETE to work, the columns corresponding to the primary
key columns of the Oracle table must have the **key** column option set.
These columns are used to identify a foreign table row, so make sure that
the option is set on *all* columns that belong to the primary key.

If you omit a foreign table column during INSERT, that column is set to
the value defined in the DEFAULT clause on the PostgreSQL foreign table
(or NULL if there is no DEFAULT clause). DEFAULT clauses on the
corresponding Oracle columns are not used.
If the PostgreSQL foreign table does not include all columns of the
Oracle table, the Oracle DEFAULT clauses will be used for the columns not
included in the foreign table definition.

The RETURNING clause on INSERT, UPDATE and DELETE is supported except
for columns with Oracle data types LONG and LONG RAW (Oracle doesn't support
these data types in the RETURNING clause).

Triggers on foreign tables are supported from PostgreSQL 9.4.
Triggers defined with AFTER and FOR EACH ROW require that the foreign table
has no columns with Oracle data type LONG or LONG RAW.  This is because
such triggers make use of the RETURNING clause mentioned above.

While modifying foreign data works, the performance is not particularly
good, specifically when many rows are affected, because (owing to the way
foreign data wrappers work) each row has to be treated individually.

Transactions are forwarded to Oracle, so BEGIN, COMMIT, ROLLBACK and
SAVEPOINT work as expected.  Prepared statements involving Oracle are
not supported.  See the [Internals](#7-internals) section for details.

Since oracle_fdw uses serialized transactions, it is possible that
data modifying statements lead to a serialization failure:

    ORA-08177: can't serialize access for this transaction

This can happen if concurrent transactions modify the table and gets more
likely in long running transactions.  Such errors can be identified by their
SQLSTATE (40001). An application using oracle_fdw should retry transactions
that fail with this error.

EXPLAIN
-------

PostgreSQL's EXPLAIN will show the query that is actually issued to Oracle.
EXPLAIN VERBOSE will show Oracle's execution plan (that will not work with
Oracle server 9i or older, see [Problems](#8-problems)).

ANALYZE
-------

From PostgreSQL version 9.2 on, you can use ANALYZE to gather statistics
on a foreign table.  This is supported by oracle_fdw.  

Without statistics, PostgreSQL has no way to estimate the row count for
queries on a foreign table, which can cause bad execution plans to be chosen.

PostgreSQL will *not* automatically gather statistics for foreign tables
with the autovacuum daemon like it does for normal tables, so it is
particularly important to run ANALYZE on foreign tables after creation
and whenever the remote table has changed significantly.

Keep in mind that analyzing an Oracle foreign table will result in a full
sequential table scan.  You can use the table option **sample_percent** to
speed this up by using only a sample of the Oracle table.

PostGIS support
---------------

The data type `geometry` is only available when PostGIS is installed.

The only supported geometry types are POINT, LINE, POLYGON,
MULTIPOINT, MULTILINE and MULTIPOLYGON in two and three dimensions.
Empty PostGIS geometries are not supported because they have no equivalent
in Oracle Spatial.

NULL values for Oracle SRID will be converted to 0 and vice versa.
For other conversions between Oracle SRID and PostGIS SRID, create a file
`srid.map` in the PostgreSQL `share` directory.  Each line of this file
shall contain an Oracle SRID and the corresponding PostGIS SRID, separated
by whitespace.  Keep the file small for good performance.

Support for IMPORT FOREIGN SCHEMA
---------------------------------

From PostgreSQL 9.5 on, IMPORT FOREIGN SCHEMA is supported to bulk import
table definitions for all tables in an Oracle schema.  
In addition to the documentation of IMPORT FOREIGN SCHEMA, consider the
following:

- IMPORT FOREIGN SCHEMA will create foreign tables for all objects found in
  ALL_TAB_COLUMNS.  That includes tables, views and materialized views,
  but not synonyms.

- There are three supported options for IMPORT FOREIGN SCHEMA:
  - **case**: controls case folding for table and column names during import.  
    The possible values are:
    - `keep`: leave the names as they are in Oracle, usually in upper case.
    - `lower`: translate all table and column names to lower case.
    - `smart`: only translate names that are all upper case in Oracle
               (this is the default).
  - **collation**: the collation used for case folding for the `lower` and
    `smart` options of **case**.  
    The default value is `default` which is the database's default collation.
    Only collations in the `pg_catalog` schema are supported.
    See the `collname` values in the `pg_collation` catalog for a list of
    possible values.
  - **readonly** (boolean): controls if imported tables can be modified.
    If set to `true`, all imported tables are created with the foreign
    table option **readonly** set to `true` (see the [Options](#3-options)
    section).  
    The default is `false`.

- The Oracle schema name must be written exactly as it is in Oracle, so
  normally in upper case.  Since PostgreSQL translates names to lower case
  before processing, you must protect the schema name with double quotes
  (for example `"SCOTT"`).

- Table names in the LIMIT TO or EXCEPT clause must be written as they
  will appear in PostgreSQL after the case folding described above.

Note that IMPORT FOREIGN SCHEMA does not work with Oracle server 8i;
see the [Problems](#8-problems) section for details.

5 Installation Requirements
===========================

oracle_fdw should compile and run on any platform supported by PostgreSQL and
Oracle client, although I could only test it on Linux and Windows.

PostgreSQL 9.1 or better is required.  
Support for ANALYZE is available from PostgreSQL 9.2 on.  
Support for INSERT, UPDATE and DELETE is available from PostgreSQL 9.3 on.

Due to an API break in a PostgreSQL minor release, the following PostgreSQL
versions cannot be used: 9.6.0 to 9.6.8 and 10.0 to 10.3

oracle_fdw is written for standard open source PostgreSQL.
Forks of PostgreSQL, such as "PostgresPro" and "Postgres-XL", are likely to
be incompatible.
If you want to try it nonetheless, you'll have to build oracle_fdw from source.
If you encounter problems while using such a PostgreSQL-derived server,
please try with the original version before reporting an issue.

Oracle client version 11.2 or better is required.
oracle_fdw can be built and used with Oracle Instant Client as well as with
Oracle Client and Server installations installed with Universal Installer.
Binaries compiled with Oracle Client 11 can be used with later client versions
without recompilation or relink.

The supported Oracle server versions depend on the used client version (see the
Oracle Client/Server Interoperability Matrix in Oracle Support document
207303.1).  PostgreSQL and Oracle need to have the same architecture.
For example, you cannot have 32-bit software for the one and 64-bit software
for the other.

It is advisable to use the latest Patch Set on both Oracle client and server,
particularly with desupported Oracle versions.  
For a list of Oracle bugs that are known to affect oracle_fdw's usability,
see the [Problems](#8-problems) section.  
Consult the oracle_fdw Wiki (https://github.com/laurenz/oracle_fdw/wiki)
for tips about Oracle installation and configuration and share your own
knowledge there.

6 Installation
==============

If you are using a binary distribution of oracle_fdw, skip to "Installing the
extension" below.

Building oracle_fdw on platforms other than Windows:
----------------------------------------------------

oracle_fdw has been written as a PostgreSQL extension and uses the Extension
Building Infrastructure PGXS on all platforms except Windows.  It should be
easy to install.  For building on Windows, see the next section.

You will need PostgreSQL headers and PGXS installed (if your PostgreSQL was
installed with packages, install the development package).  
You need to install Oracle's C header files as well (SDK package for Instant
Client).  If you use the Instant Client ZIP files provided by Oracle,
you will have to create a symbolic link from `libclntsh.so` to the
actual shared library file yourself.

Make sure that PostgreSQL is configured `--without-ldap` (at least the server).
See the [Problems](#8-problems) section.

Make sure that `pg_config` is in the PATH (test with `pg_config --pgxs`).  
Set the environment variable ORACLE_HOME to the location of the Oracle
installation.

Unpack the source code of oracle_fdw and change into the directory.  
Then the software installation should be as simple as:

    $ make
    $ make install

For the second step you need write permission on the directories where
PostgreSQL is installed.

If you want to build oracle_fdw in a source tree of PostgreSQL, use

    $ make NO_PGXS=1

Building oracle_fdw on Windows:
-------------------------------

To build oracle_fdw on Windows, you need:

- PostgreSQL headers and libraries. These can be found in the PostgreSQL
  installation directory.
- Oracle headers and libraries (SDK package for Instant Client).
- Microsoft Visual Studio 2013 or later.

To build, either open `oracle_fdw\msvc\oracle_fdw.sln` in the IDE, or:

- Open a development command prompt (either x86 or x64 depending on your
  PostgreSQL installation) and change to the `oracle_fdw\msvc` directory.
- Run (single command line):

      > msbuild oracle_fdw.sln /p:Configuration=(Debug or Release) ^
            /p:Platform=(Win32 or x64) ^
            /p:OracleClient=(path to Oracle Client/SDK) ^
            /p:PostgreSQL=(path to PostgreSQL installation)

  (The "^"s are line continuations; you can just put everything on a single
line instead.)

When the build is complete, you will find `oracle_fdw.dll` in a subdirectory
named for your build options, e.g. `x64\Release`.

If you use Visual Studio 2015 or later and you get errors about missing
header files including `sys/types.h`, you must install the Universal CRT
SDK (part of Visual Studio).

Installing the extension:
-------------------------

Make sure that the oracle_fdw shared library is installed in the PostgreSQL
library directory and that oracle_fdw.control and the SQL files are in
the PostgreSQL extension directory.

Since the Oracle client shared library is probably not in the standard
library path, you have to make sure that the PostgreSQL server will be able
to find it.  How this is done varies from operating system to operating
system; on Linux you can set LD_LIBRARY_PATH or use `/etc/ld.so.conf`.

Make sure that all necessary Oracle environment variables are set in the
environment of the PostgreSQL server process (ORACLE_HOME if you don't use
Instant Client, TNS_ADMIN if you have configuration files, etc.)

To install the extension in a database, connect as superuser and

    CREATE EXTENSION oracle_fdw;

That will define the required functions and create a foreign data wrapper.

To upgrade from an oracle_fdw version before 1.0.0, use

    ALTER EXTENSION oracle_fdw UPDATE;

Note that the extension version as shown by the psql command `\x` or the
system catalog `pg_available_extensions` is *not* the installed version
of oracle_fdw.  To get the oracle_fdw version, use the function `oracle_diag`.

Running the regression tests:
-----------------------------

Unless you are developing oracle_fdw or want to test its functionality
on an exotic platform, you don't have to do this.

For the regression tests to work, you must have a PostgreSQL cluster
(9.3 or better) and an Oracle server (10.2 or better with Locator or Spatial)
running, and the oracle_fdw binaries must be installed.  
The regression tests will create a database called `contrib_regression` and
run a number of tests.  For the PostGIS regression tests to succeed,
the PostGIS binaries must be installed.

The Oracle database must be prepared as follows:
- A user `scott` with password `tiger` must exist (unless you want to edit
  the regression test scripts).  The user needs CREATE SESSION and CREATE TABLE
  system privileges and enough quota on its default tablespace, as well as
  SELECT privileges on V$SQL and V$SQL_PLAN.
- Two tables must be created as follows:
  
        CREATE TABLE scott.typetest1 (
           id  NUMBER(5)
              CONSTRAINT typetest1_pkey PRIMARY KEY,
           c   CHAR(10 CHAR),
           nc  NCHAR(10),
           vc  VARCHAR2(10 CHAR),
           nvc NVARCHAR2(10),
           lc  CLOB,
           r   RAW(10),
           u   RAW(16),
           lb  BLOB,
           lr  LONG RAW,
           b   NUMBER(1),
           num NUMBER(7,5),
           fl  BINARY_FLOAT,
           db  BINARY_DOUBLE,
           d   DATE,
           ts  TIMESTAMP WITH TIME ZONE,
           ids INTERVAL DAY TO SECOND,
           iym INTERVAL YEAR TO MONTH
        ) SEGMENT CREATION IMMEDIATE;

        CREATE TABLE scott.gis (
           id  NUMBER(5) PRIMARY KEY,
           g   MDSYS.SDO_GEOMETRY
        ) SEGMENT CREATION IMMEDIATE;

- Gather statistics for the two tables:

        EXEC DBMS_STATS.GATHER_TABLE_STATS ('SCOTT', 'TYPETEST1', NULL, 100);
        EXEC DBMS_STATS.GATHER_TABLE_STATS ('SCOTT', 'GIS', NULL, 100);

Set the environment for the PostgreSQL server so that it can establish an
Oracle connection without connect string:
If the Oracle server is on the same machine, set the environment variables
ORACLE_SID and ORACLE_HOME appropriately, for a remote server set the
environment variable TWO_TASK (or LOCAL on Windows) to the connect string.

The regression tests are run as follows:

    $ make installcheck

7 Internals
===========

oracle_fdw sets the MODULE of the Oracle session to `postgres` and the
ACTION to the backend process number.  This can help identifying the Oracle
session and allows you to trace it with DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE.

oracle_fdw uses Oracle's result prefetching to avoid unnecessary client-server
round-trips.  The prefetch row count can be configured with the **prefetch**
table option and is set to 200 by default.

Rather than using a PLAN_TABLE to explain an Oracle query (which would require
such a table to be created in the Oracle database), oracle_fdw uses execution
plans stored in the library cache.  For that, an Oracle query is *explicitly
described*, which forces Oracle to parse the query.  The hard part is to find
the SQL_ID and CHILD_NUMBER of the statement in V$SQL because the SQL_TEXT
column contains only the first 1000 bytes of the query.
Therefore, oracle_fdw adds a comment to the query that contains an MD5 hash
of the query text.  This is used to search in V$SQL.
The actual execution plan or cost information is retrieved from V$SQL_PLAN.

oracle_fdw uses transaction isolation level SERIALIZABLE on the Oracle side,
which corresponds to PostgreSQL's REPEATABLE READ.  This is necessary because
a single PostgreSQL statement can lead to multiple Oracle queries (e.g. during
a nested loop join) and the results need to be consistent.  
Unfortunately the Oracle implementation of SERIALIZABLE has certain quirks;
see the [Problems](#8-problems) section for more.

The Oracle transaction is committed immediately before the local transaction
commits, so that a completed PostgreSQL transaction guarantees that the Oracle
transaction has completed.  However, there is a small chance that the
PostgreSQL transaction cannot complete even though the Oracle transaction
is committed.  This cannot be avoided without using two-phase transactions
and a transaction manager, which is beyond what a foreign data wrapper
can reasonably provide.  
Prepared statements involving Oracle are not supported for the same reason.

8 Problems
==========

Encoding
--------

Characters stored in an Oracle database that cannot be converted to the
PostgreSQL database encoding will silently be replaced by *replacement
characters*, typically a normal or inverted question mark, by Oracle.
You will get no warning or error messages.

If you use a PostgreSQL database encoding that Oracle does not know
(currently, these are EUC_CN, EUC_KR, LATIN10, MULE_INTERNAL,
WIN874 and SQL_ASCII), non-ASCII characters cannot be translated
correctly. You will get a warning in this case, and the characters
will be replaced by replacement characters as described above.

You can set the **nls_lang** option of the foreign data wrapper to force a
certain Oracle encoding, but the resulting characters will most likely be
incorrect and lead to PostgreSQL error messages.  This is probably only
useful for SQL_ASCII encoding if you know what you are doing.
See the [Options](#3-options) section.

Limited functionality in old Oracle versions
--------------------------------------------

- The definition of the Oracle system catalogs V$SQL and V$SQL_PLAN has
  changed with Oracle 10.1.  Using EXPLAIN VERBOSE with older Oracle server
  versions will result in errors like:

      ERROR:  error describing query: OCIStmtExecute failed to execute
              remote query for sql_id
      DETAIL:  ORA-00904: "LAST_ACTIVE_TIME": invalid identifier

  There is no plan to fix this, since Oracle 9i has been out of Extended Support
  since 2010 and the functionality is not essential.

- IMPORT FOREIGN SCHEMA throws the following error with Oracle server 8i:

      ERROR:  error importing foreign schema: OCIStmtExecute failed to execute
              column query
      DETAIL:  ORA-00904: invalid column name

  This is because the view ALL_TAB_COLUMNS lacks the column CHAR_LENGTH,
  which was added in Oracle 9i.

LDAP libraries
--------------

The Oracle client shared library comes with its own LDAP client
implementation conforming to [RFC 1823](http://www.rfc-base.org/rfc-1823.html),
so these functions have the same names as OpenLDAP's.  This will lead to a
name collision when the PostgreSQL server was configured `--with-ldap`.

The name collision will not be detected, because oracle_fdw is loaded at
runtime, but trouble will happen if anybody calls an LDAP function.
Typically, OpenLDAP is loaded first, so if Oracle calls an LDAP function
(for example if you use *directory naming* name resolution), the backend
will crash.  This can lead to messages like the following (seen on Linux)
in the PostgreSQL server log:

    ../../../libraries/libldap/getentry.c:29: ldap_first_entry:
     Assertion `( (ld)->ld_options.ldo_valid == 0x2 )' failed.

The best thing is to configure PostgreSQL `--without-ldap`.  This is the only
safe way to avoid this problem.  
Even when PostgreSQL is built `--with-ldap`, it may work as long as you don't
use any LDAP client functionality in Oracle.  
On some platforms, you can force Oracle's client shared library to be loaded
before the PostgreSQL server is started (LD_PRELOAD on Linux).  Then Oracle's
LDAP functions should get used.  In that case, Oracle may be able to use
LDAP functionality, but using LDAP from PostgreSQL will crash the backend.

You cannot use LDAP functionality both in PostgreSQL and in Oracle, period.

Serialization errors
--------------------

In Oracle 11.2 or above, inserting the first row into a newly created
Oracle table with oracle_fdw will lead to a serialization error.

This is because of an Oracle feature called *deferred segment creation* which
defers allocation of storage space for a new table until the first row
is inserted.  This causes a serialization failure with serializable
transactions (see document 1285464.1 in Oracle's knowledge base).

This is no serious problem; you can work around it by either ignoring that
first error or creating the table with SEGMENT CREATION IMMEDIATE.

A much nastier problem is that concurrent inserts can sometimes cause
serialization errors when an index page is split concurrently with a
modifying serializable transaction (see Oracle document 160593.1).

Oracle claims that this is not a bug, and the suggested solution is to retry
the transaction that got a serialization error.

Oracle bugs
-----------

This is a list of Oracle bugs that have affected oracle_fdw in the past.

Bug 2728408 can cause `ORA-8177 cannot serialize access for this transaction`
even if no modification of remote data is attempted.  
It can occur with Oracle server 8.1.7.4 (install one-off patch 2728408) or
Oracle server 9.2 (install Patch Set 9.2.0.4 or better).

Missing Oracle client DLL (on Windows only)
-------------------------------------------

The following error message (from any query involving an Oracle foreign
table) indicates that PostgreSQL cannot find the Oracle client library:

    ERROR:  Oracle client library (oci.dll) not found
    DETAIL:  No Oracle client is installed, or your system is configured incorrectly.
    HINT:  Verify that the PATH variable includes the Oracle client.

Make sure that the path to `oci.dll` is in the PATH environment variable of
the user running the PostgreSQL server.  If it is running as a Windows service,
this is the system environment, and you must restart the service after
changing it.

If updating the environment does not work, you may be using a PostgreSQL
distribution that provides its own environment variables, hiding the Oracle
client.  See the bug report at https://github.com/laurenz/oracle_fdw/issues/160
for more information.

9 Support
=========

If you want to report a problem with oracle_fdw, and the name of the
foreign server is (for example) "ora_serv", please include the output of

    SELECT oracle_diag('ora_serv');

in your problem report.
If that causes an error, please also include the output of

    SELECT oracle_diag();

If you have a problem or question or any kind of feedback, the preferred
option is to open an issue on GitHub:  
https://github.com/laurenz/oracle_fdw/issues  
This requires a GitHub account.
