AUTHOR
======

This library has been created by Federico Razzoli:
info [At) federico - razzoli . com

REQUIREMENTS
============

This library has been developed with MariaDB 10.0.

If you are able to compile the CONNECT storage engine against older versions or other forks, this library will work. However, some modifications may be necessary.

INSTALL
=======

materialize.sql contains the SQL statements which are necessary to create all needed stored procedures and tables.

First, open it and search for these lines:

```
	OPTIONS
	(
		  HOST '127.0.0.1'
		, USER 'root'
		, PASSWORD 'root'
		, PORT 3306
		, DATABASE '_'
	)
```

Replace these data with some working login credentials.

To execute it:
/path/to/mysql -u<username> -p<password> < /path/to/materialize.sql

The CONNECT storage engine will be installed.

UNINSTALL
=========

To uninstall, just execute:
DROP DATABASE _;

You may also want to uninstall the CONNECT storage engine:
UNINSTALL SONAME 'ha_connect';

SECURITY NOTE
=============

The materialize_sql() procedures, and other procedures and tables, will allow the users to execute arbitrary SQL statements on the local server. From MariaDB's point of view, these statements will be ran by the user associated to the `_` server (the data you set at the installation stage). Be sure that this user does not have too many privileges, and/or only some users have the SELECT and EXECUTE privilege on the `_` database.

MANUAL
======

ABSTRACT
========

There are several limits to the SQL statements which can be executed inside a stored routine:
* Some statements cannot be executed at all (example: CHECK TABLE).
* Some statements can be executed, but only as prepared statements, which makes the procedures even more verbose.
* Some statements can be executed, but their results are not accessible from the procedure (SHOW, EXPLAIN).
* There is no way to execute some statements and avoid to return their results to the client (CALL).

The CONNECT storage engine solves this problem. If a CONNECT table is of MYSQL type and contains a SRCDEF table option, the specified statements will be executed on the specified server (possibly the local server) each time the table is queried. Of course, the results of the statement will be returned by the queries. This mechanism is based on normal SELECTs, thus it works even inside stored routines.

However, creating these tables could make the routines very verbose and error prone: a CREATE TABLE statement needs to be dynamically composed and executed as a prepared statement.

This library provides convenient non-verbose syntaxes to achieve the same results.

_.materialize_sql()
===================

Create a CONNECT table based on the specified SQL statement. Querying the table will execute the statement on the local server. The results will be returned by the query.

Parameters:
* p_db_name: Database which will contain the CONNECT table.
* p_tab_name: Name of the CONNECT table.
* p_sql: SQL statement associated to the CONNECT table.

Limitations:
* CONNECT does not support *TEXT and *BLOB datatypes. Such data needs to be explicitly converted to VARCHAR.
** Note that unassigned @variables contain a LONGBLOB NULL.
* The table cannot be temporary (MariaDB/CONNECT bug).

Example:

```
MariaDB [test]> CALL _.materialize_sql('test', 'schemas', 'SHOW SCHEMAS LIKE \'h%\'');
Query OK, 0 rows affected (0.10 sec)
MariaDB [test]> SELECT * FROM test.schemas;
+---------------+
| Database (h%) |
+---------------+
| hangman       |
+---------------+
1 row in set (0.09 sec)
```

_.administrative_sql()
======================

As explained above, *TEXT and *BLOB datatypes are not supported by CONNECT. Hoever, administrative statements like CHECK, REPAIR, ANALYZE, OPTIMIZE return a BLOB column and do not support a syntax to convert it. _.administrative_sql() executes them and returns the results.

Parameters:
* p_sql: SQL statement to execute.

Example:

```
MariaDB [test]> CALL _.administrative_sql('CHECK TABLE mysql.tables_priv');
+-------------------+-------+----------+----------+
| Table             | Op    | Msg_type | Msg_text |
+-------------------+-------+----------+----------+
| mysql.tables_priv | check | status   | OK       |
+-------------------+-------+----------+----------+
1 row in set (0.04 sec)

Query OK, 0 rows affected (0.04 sec)
```

show_*
======

Some SHOW statements return results that can also be retreive by querying a system table in the information_schema database. Others, like SHOW MASTER STATUS, return results that cannot be obtained in other ways. For each statement of the second type, a table will be created in the _ database. For example:

```
MariaDB [test]> SELECT * FROM _.show_master_status;
+---------------+----------+--------------+------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+---------------+----------+--------------+------------------+
| binlog.000018 |     2231 |              |                  |
+---------------+----------+--------------+------------------+
1 row in set (0.02 sec)
```

