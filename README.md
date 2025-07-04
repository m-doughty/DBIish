# NAME

DBIish - a simple database interface for Raku

# SYNOPSIS

    use v6;
    use DBIish;

    my $dbh = DBIish.connect("SQLite", :database<example-db.sqlite3>);

    $dbh.execute(q:to/STATEMENT/);
        DROP TABLE IF EXISTS nom
        STATEMENT

    $dbh.execute(q:to/STATEMENT/);
        CREATE TABLE nom (
            name        varchar(4),
            description varchar(30),
            quantity    int,
            price       numeric(5,2)
        )
        STATEMENT

    $dbh.execute(q:to/STATEMENT/);
        INSERT INTO nom (name, description, quantity, price)
        VALUES ( 'BUBH', 'Hot beef burrito', 1, 4.95 )
        STATEMENT

    my $sth = $dbh.prepare(q:to/STATEMENT/);
        INSERT INTO nom (name, description, quantity, price)
        VALUES ( ?, ?, ?, ? )
        STATEMENT

    $sth.execute('TAFM', 'Mild fish taco', 1, 4.85);
    $sth.execute('BEOM', 'Medium size orange juice', 2, 1.20);


    # For one-off execution
    $sth = $dbh.execute(q:to/STATEMENT/);
        SELECT name, description, quantity, price, quantity*price AS amount
        FROM nom
        STATEMENT
    say $sth.rows; # 3

    for $sth.allrows() -> $row {
        say $row[0];  # BUBH␤TAFM␤BEOM
    }

    $sth.dispose;

    # For efficient multiple execution
    $sth = $dbh.prepare('SELECT description FROM nom WHERE name = ?');
    for <TAFM BEOM> -> $name {
        for $sth.execute($name).allrows(:array-of-hash) -> $row {
            say $row<description>;
        }
    }

    $dbh.dispose;

### Notes for macOS users
In order for DBIish to be able to load any of the required libraries, it needs:
- Know the name of the library file on the local platform.
- That the library it is in one of the places where the loader looks for it.
In macOS there can be problems because the rules have changed frequently and Homebrew can use not standard names or places. So some tests can help us:

To get the library file name that will be used for PostgreSQL try the following in the raku REPL:
```say $*VM.platform-library-name('pq'.IO, :version(Version.new(5))).Str;```
By default the loader will search for that file in ~/lib, /usr/local/lib, and /usr/lib or in any path in the environment variables PATH, LD_LIBRARY_PATH, DYLD_LIBRARY_PATH or DYLD_FALLBACK_LIBRARY_PATH.

Check that your library has the expected name and it is installed in any of those places or add its path to one of these variables.

# DESCRIPTION

The DBIish project provides a simple database interface for Raku.

It's not a port of the Perl 5 DBI and does not intend to become one.
It is, however, a simple and useful database interface for Raku that works
now. It looks like a DBI, and it talks like a DBI (although it only offers
a subset of the functionality).

## Connecting to, and disconnecting from, a database

You obtain a `DataBaseHandler` by calling the static `DBIish.connect` method, passing
as the only positional argument the driver name followed by any required named arguments.

Those named arguments are driver specific, but commonly required ones are:
`database`, `user` and `password`.

For the different syntactic forms of
[named arguments](https://doc.perl6.org/language/functions#Arguments) see
the language documentation.

For example, for connect to a database 'hierarchy' on PostgreSQL, with the user in `$user`
and using the function `get-secret` to obtain you password, you can:

    my $dbh = DBIish.connect('Pg', :database<hierarchy>, :$user, password => get-secret());

See ahead more examples.

To disconnect from a database and free the allocated resources you should call the
`dispose` method:

    $dbh.dispose;

## Executing queries

### `execute`

For a single execution of a query you may use execute directly. This starts the query within the database and returns
a `StatementHandle` which will be used for [Retrieving Data](#retrieving-data).

    $dbh.execute(q:to/SQL/);
      CREATE TABLE tab (
        id serial PRIMARY KEY
        col text
      );
      SQL

Errors occurring within the database for an SQL statement will typically throw an exception within Raku. The level of
detail within the exception object depends on the database driver.

    $dbh.execute('CREATE TABLE failtab ( id );');

    CATCH {
       when X::DBDish::DBError {
          say .message;
       }
    }

In order to build a query dynamically without risk of SQL injection you need to use parameter binding. The `?`
will be replaced by an escaped copy of the parameter provided after the query.

    my $value-id = 19;
    $dbh.execute('INSERT INTO tab (id) VALUES (?)', $value-id);

    my $value-text = q{Complex text ' value " with quotes};
    $dbh.execute('INSERT INTO tab (id, col) VALUES (?, ?)', $value-id, $value-text);

    # Undefined or Nil values will be converted to NULL by parameter binding.
    my $value-nil;
    $dbh.execute('INSERT INTO tab (id, col) VALUES (?, ?), $value-id, $value-nil);

Parameter binding should be used where-ever possible, even if the number of parameters is dynamic. In this case the
number of elements for IN is dynamic. The number of `?`'s is scaled to fit the list of items, and the list
is provided to execute as individual items.

    my @value-list = 1 .. 6;
    my $parameter-bind-marks = @value-list.map({'?'}).join(',')  # ?,?,?,?,?,?
    my $query = 'SELECT id FROM tab WHERE id IN (%s)'.sprintf($parameter-bind-marks);
    $dbh.execute($query, |@value-list);

All database drivers support basic Raku types like Int, Rat, Str, and Buf; some databases may support additional
complex types such as an Array of Str or Array of Int which may simplify the above example significantly. Please see
database specific documentation for additional type support.

### `prepare`

Execute performs a couple of steps on the client side, and often the database side as well, which may be cached
if a query is going to be executed several times. For simple queries, prepare() may increase performance by up to 50%

This is an inefficient example of running a query multiple times:

    for 1 .. 100 -> $id {
      $dbh.execute('INSERT INTO tab (id) VALUES (?)', $id);
    }

This example is more efficient as it uses prepare to decrease overhead on the client side; often on the
server-side too as the database may only need to parse the SQL once.

    my $sth = $dbh.prepare('INSERT INTO tab (id) VALUES (?)');
    for 1 .. 100 -> $id {
      $sth.execute($id);
    }

## Retrieving data

DBIish provides the `row` and `allrows` methods to fetch values from a `StatementHandle` object returned by execute.
These functions provide you typed values; for example an int4 field in the database will be provided as an Int
 scalar in Raku.

### `row`

`row` take the `hash` adverb if you want to have the values in a Hash form instead of a plain Array

Example:

    my $sth = $dbh.execute('SELECT id, col FROM tab WHERE id = ?', $value-id);

    my @values = $sth.row();
    my %values = $sth.row(:hash);

### `allrows`

`allrows` lazily returns all the rows as a list of arrays.
If you want to fetch the values in a hash form, use one of the two adverbs `array-of-hash`
or `hash-of-array`

Example:

    my $sth = $dbh.execute('SELECT id, col FROM tab');

    my @data = $sth.allrows(); # [[1, 'val1'], [3, 'val2']]
    my @data = $sth.allrows(:array-of-hash); # [ ( id => 1, col => 'val1'), ( id => 3, col => 'val2') ]
    my %data = $sth.allrows(:hash-of-array); # id => [1, 3], col => ['val1', 'val2']

    for $sth.allrows(:array-of-hash) -> $row {
      say $row<id>;  # 1␤3
    }

    # Or as a shorter example:
    for $dbh.execute('SELECT id, col FROM tab').allrows(:array-of-hash) -> $row {
       say $row<id>  # 1␤3
    }

### `dispose`

After you have fetched all data using the statement handle, you can free its memory immediately using `dispose`.

    $sth.dispose;

### `server-version`

`server-version` returns a `Version` object for the version of the server you are connected to. Not all drivers support
this function (some may not connect to a server at all) so it's best to wrap in a `can`.

    my Version $version = $dbh.server-version() if $dbh.can('server-version');

## Statement Exceptions

All exceptions for a query result are thrown as or inherit `X::DBDish::DBError`. Additional functionality may
 be provided by the database driver.

- `driver-name`

    Database Driver name for the connection

- `native-message`

    Unmodified message received from the database server.

- `code`

    Int return code from the local client library for the call; typically -1. This is not an SQL state.

- `why`

    A Str indicating why the exception was thrown. Typically 'Error'.

- `message`

    Human friendly and more informative version of the database message.

- `is-temporary`

    A Boolean flag which when true indicates that the transaction may succeed if retried. Connectivity issues,
    serialization issues and other temporary items may set this as True.

## Advanced Query Building

In general you should use the `?` parameter for substitution whenever possible. The database driver
will ensure values are properly escaped prior to insertion into the database. However, if you need to create
a query string by hand then you can use `quote` to help prevent an SQL injection attack from being successful.

### `quote($literal)` and `quote($identifier, :as-id)`

Using parameter substitution is preferred:

    my $val = 'literal';
    $dbh.execute('INSERT INTO tab VALUES (?)', $val);

However, if you must build the query directly you can:

    my $val = 'literal';
    my $query = 'INSERT INTO tab VALUES (%s)'.sprintf( $dbh.quote($val) );
    $dbh.execute($query);

To build a query with a dynamic identifier:

    # Notice that C<?> is still used for the value being inserted; it is still recommended where possible.
    my $id = 'table';
    my $val = 'literal';
    my $query = 'INSERT INTO %s VALUES (?)'.sprintf( $dbh.quote($id, :as-id) );
    $dbh.execute($query, $val);

# INSTALLATION

    $ zef install DBIish

# DBDish CLASSES

Some DBDish drivers install together with DBIish.pm6 and are maintained as a single project.

Search the Raku ecosystem for additional [DBDish](https://modules.raku.org/search/?q=dbdish) drivers such
as [ODBC](https://github.com/salortiz/DBDish-ODBC).

Currently the following backends are included:

## Pg (PostgreSQL)

Supports basic CRUD operations and prepared statements with placeholders

    my $dbh = DBIish.connect('Pg', :host<db01.yourdomain.com>, :port(5432),
            :database<blerg>, :user<myuser>, password => get-secret());

Pg supports the following named arguments:
`host`, `hostaddr`, `port`, `database` (or its alias `dbname`), `user`, `password`,
`connect-timeout`, `client-encoding`, `options`, `application-name`, `keepalives`,
`keepalives-idle`, `keepalives-interval`, `sslmode`, `requiressl`, `sslcert`, `sslkey`,
`sslrootcert`, `sslcrl`, `requirepeer`, `krbsrvname`, `gsslib`, and `service`.

See your [PostgreSQL documentation](https://www.postgresql.org/docs/current/libpq-envars.html) for details.

### Parameter Substitution

In addition to the `?` style of parameter substitution supported by all drivers, PostgreSQL also supports numbered
parameter. The advantage is that a numbered parameter may be reused

    $dbh.execute('INSERT INTO tab VALUES ($1, $2, $2 - $1)', $var1, $var2);

This is equivalent to the below statement except the subtraction operation is performed by PostgreSQL:

    $dbh.execute('INSERT INTO tab VALUES (?, ?, ?)', $var1, $var2, $var2 - $var1);

### pg arrays

Pg arrays are supported for both writing via execute and retrieval via `row/allrows`.
You will get the properly typed array according to the field type.

Passing an array to `execute` is now implemented. But you can also use the
`pg-array-str` method on your Pg StatementHandle to convert an Array to a
string Pg can understand:

    # Insert an array via an execute statement
    my $sth = $dbh.execute('INSERT INTO tab (array_column) VALUES ($1);', @data);

    # Prepare an insertion of an array field
    my $sth = $dbh.prepare('INSERT INTO tab (array_column) VALUES ($1);');
    $sth.execute(@data1);   # or $sth.execute($sth.pg-array-str(@data1));
    $sth.execute(@data2);

    # Retrieve the array values back again.
    for $dbh.execute('SELECT array_column FROM tab').allrows() -> $row {
       my @array-column = $row[0];
    }

    # Check if "value" is in the dataset. This is similar to an IN statement.
    my $sth = $dbh.prepare('SELECT * FROM tab WHERE value = ANY($1)');
    $sth.execute(@data);

    # If a datatype is needed you can cast the placeholder with the PostgreSQL datatype.
    my $sth = $dbh.prepare('SELECT * FROM tab WHERE value = ANY($1::_cidr)');
    $sth.execute(['127.0.0.1', '10.0.0.1']);

### `pg-consume-input`

Consume available input from the server, buffering the read data if there is any.
This is only necessary if you are planning on calling `pg-notifies` without having
requested input by other means (such as an `execute`.)

### `pg-notifies`

    $ret = $dbh.pg-notifies;

Looks for any asynchronous notifications received and returns a pg-notify object that looks like this

        class pg-notify {
            has Str                           $.relname; # Channel Name
            has int32                         $.be_pid; # Backend pid
            has Str                           $.extra; # Payload
        }

or nothing if there are no pending notifications.

In order to receive the notifications you should execute the PostgreSQL command "LISTEN"
prior to calling `pg-notifies` the first time; if you have not executed any other
commands in the meantime you will also need to execute `pg-consume-input` first.

For example:

    $dbh.execute("LISTEN foo");

    loop {
        $dbh.pg-consume-input
        if $dbh.pg-notifies -> $not {
            say $not;
        }
    }

The payload is optional and will always be an empty string for PostgreSQL servers less than version 9.0.

### `ping`

Test to see if the connection is still considered live.

    $dbh.ping

### Statement Exceptions

Exceptions for a query result are thrown as `X::DBDish::DBError::Pg` objects (inherits `X::DBDish::DBError`)
and have the following additional attributes (described with a `PG_DIAG_*` source name) as provided by the
[PostgreSQL client library libpq](https://www.postgresql.org/docs/current/libpq-exec.html):

- `message`

    `PG_DIAG_MESSAGE_PRIMARY` -
    The primary human-readable error message (typically one line). Always present.

- `message-detail`

    `PG_DIAG_MESSAGE_DETAIL` -
    Detail: an optional secondary error message carrying more detail about the problem. Might run to multiple lines.

- `message-hint`

    `PG_DIAG_MESSAGE_HINT` -
    Hint: an optional suggestion what to do about the problem. This is intended to differ from detail in that it offers advice (potentially inappropriate) rather than hard facts. Might run to multiple lines.

- `context`

    `PG_DIAG_CONTEXT` -
    An indication of the context in which the error occurred. Presently this includes a call stack traceback of active procedural language functions and internally-generated queries. The trace is one entry per line, most recent first.

- `type`

    `PG_DIAG_SEVERITY_NONLOCALIZED` -
    The severity; the field contents are ERROR, FATAL, or PANIC (in an error message), or WARNING, NOTICE, DEBUG, INFO, or LOG (in a notice message). This is identical to the PG\_DIAG\_SEVERITY field except that the contents are never localized. This is present only in reports generated by PostgreSQL versions 9.6 and later.

- `type-localized`

    `PG_DIAG_SEVERITY` -
    The severity; the field contents are ERROR, FATAL, or PANIC (in an error message), or WARNING, NOTICE, DEBUG, INFO, or LOG (in a notice message), or a localized translation of one of these. Always present.

- `sqlstate`

    `PG_DIAG_SQLSTATE` -
    The SQLSTATE code for the error. The SQLSTATE code identifies the type of error that has occurred; it can be used by front-end applications to perform specific operations (such as error handling) in response to a particular database error. For a list of the possible SQLSTATE codes, see Appendix A. This field is not localizable, and is always present.

- `statement`

    Statement provided to prepare() or execute()

- `statement-name`

    Statement Name provided to prepare() or created internally

- `statement-position`

    `PG_DIAG_STATEMENT_POSITION` -
    A string containing a decimal integer indicating an error cursor position as an index into the original statement string. The first character has index 1, and positions are measured in characters not bytes.

- `internal-position`

    `PG_DIAG_INTERNAL_POSITION` -
    This is defined the same as the `PG_DIAG_STATEMENT_POSITION` field, but it is used when the cursor position refers to an internally generated command rather than the one submitted by the client. The `PG_DIAG_INTERNAL_QUERY` field will always appear when this field appears.

- `internal-query`

    `PG_DIAG_INTERNAL_QUERY` -
    The text of a failed internally-generated command. This could be, for example, a SQL query issued by a PL/pgSQL function.

- `dbname`

    Database Name from libpq `pg-db()`

- `host`

    Host from libpq `pg-host()`

- `user`

    User from libpq `pg-user()`

- `port`

    Port from libpq `pg-port()`

- `schema`

    `PG_DIAG_SCHEMA_NAME` -
    If the error was associated with a specific database object, the name of the schema containing that object, if any.

- `table`

    `PG_DIAG_TABLE_NAME` -
    If the error was associated with a specific table, the name of the table. (Refer to the schema name field for the name of the table's schema.)

- `column`

    `PG_DIAG_COLUMN_NAME` -
    If the error was associated with a specific table column, the name of the column. (Refer to the schema and table name fields to identify the table.)

- `datatype`

    `PG_DIAG_DATATYPE_NAME` -
    If the error was associated with a specific data type, the name of the data type. (Refer to the schema name field for the name of the data type's schema.)

- `constraint`

    `PG_DIAG_CONSTRAINT_NAME` -
    If the error was associated with a specific constraint, the name of the constraint. Refer to fields listed above for the associated table or domain. (For this purpose, indexes are treated as constraints, even if they weren't created with constraint syntax.)

- `source-file`

    `PG_DIAG_SOURCE_FILE` -
    The file name of the source-code location where the error was reported.

- `source-line`

    `PG_DIAG_SOURCE_LINE` -
    The line number of the source-code location where the error was reported.

- `source-function`

    `PG_DIAG_SOURCE_FUNCTION` -
    The name of the source-code function reporting the error.

Please see the [PostgreSQL documentation](https://www.postgresql.org/docs/current/static/libpq-exec.html#LIBPQ-PQRESULTERRORFIELD)
for additional information.

A special `is-temporary()` method returns True if an immediate retry of the full transaction should be attempted:

It is set to true when the [SQLState](https://www.postgresql.org/docs/current/static/errcodes-appendix.html)
is any of the following codes:

- SQLState Class 08XXX

    All connection exceptions (possible temporary network issues)

- SQLState 40001

    serialization\_failure - Two or more transactions conflicted in a manner which may succeed if executed later.

- SQLState 40P01

    deadlock\_detected - Two or more transactions had locking conflicts resulting in a deadlock and this transaction being rolled back.

- SQLState Class 57XXX

    Operator Intervention (early/forced connection termination).

- SQLState 72000

    snapshot\_too\_old - The transaction took too long to execute. It may succeed during a quieter period.

### `pg-socket`

        my Int $socket = $dbh.pg-socket;

Returns the file description number of the connection socket to the server.

## SQLite (and SQLCipher)

Supports basic CRUD operations and prepared statements with placeholders

    my $dbh = DBIish.connect('SQLite', :database<thefile>);

The `:database` parameter can be an absolute file path as well (or even an
`IO::Path` object):

    my $dbh = DBIish.connect('SQLite', database => '/path/to/sqlite.db' );

If the SQLite library was compiled to be threadsafe (which is usually the
case), then it is possible to use SQLite from multiple threads. This can be
introspected:

    say DBIish.install-driver('SQLite').threadsafe;

SQLite does support using one connection object concurrently, however other
databases may not; if portability is a concern, then only use a particular
connection object from one thread at a time (and so have multiple connection
objects).

When using a SQLite database concurrently (from multiple threads, or even
multiple processes), operations may not be able to happen immediately due to
the database being locked. DBIish sets a default timeout of 10000 milliseconds;
this can be changed by passing the `busy-timeout` option to `connect`.

    my $dbh = DBIish.connect('SQLite',    :database<thefile>, :60000busy-timeout);
    my $dbh = DBIish.connect('SQLCipher', :database<thefile>, :60000busy-timeout);

Passing a value less than or equal to zero will disable the timeout, resulting
in any operation that cannot take place immediately producing a database
locked error.

### Function rows()

Since SQLite may retrieve records in the background, the `rows()` method will not be accurate
until all records have been retrieved from the database. A warning is thrown when this may be the case.

This warning message may be suppressed using a `CONTROL` phaser:

    CONTROL {
        when CX::Warn {
            when .message.starts-with('SQLite rows()') { .resume }
            default { .rethrow }
        }
    }

Making `rows()` accurate for all calls would require the driver pre-retrieving and caching
all records with a large performance and memory penalty, then providing the records as requested.

For best performance you are recommended to use:

    while my $row = $sth.row {
        # Do something with all records as retrieved
    }

    if my $row = $sth.row {
        # Do something with a single record
    }

### Keys - SQLCipher only

To set the key for the current connection:

```raku
    $dbh.key('Tr0ub4dor&1');
```

To change the key for the current connection:

```raku
    $dbh.rekey('CorrectHorseBatteryStaple');
```

## MySQL

Supports basic CRUD operations and prepared statements with placeholders

    my $dbh = DBIish.connect('mysql', :host<db02.yourdomain.com>, :port(3306),
            :database<blerg>, :user<myuser>, :$password);

    # Or via socket:
    my $dbh = DBIish.connect('mysql', :socket<mysql.sock>,
            :database<blerg>, :user<myuser>, :$password);


MySQL driver supports the following named arguments:
`connection-timeout`, `read-timeout`, `write-timeout`

See your [MySQL documentation](https://dev.mysql.com/doc/c-api/5.6/en/mysql-options.html) for details.


Since MariaDB uses the same wire protocol as MySQL, the \`mysql\` backend
also works for MariaDB.

### Statement Exceptions

Exceptions for a query result are thrown as `X::DBDish::DBError::mysql` objects (inherits `X::DBDish::DBError`)
and have the following additional attributes as provided by the MySQL client libraries.

- `message`

  `mysql_error` -
  The primary human-readable error message (typically one line). Always present.

- `code`

  `mysql_errno` -
  Integer code. Always present.

- `sqlstate`

  `mysql_sqlstate` -
  The SQLSTATE code for the error. The SQLSTATE code identifies the type of error that has occurred; it can be used by front-end applications to perform specific operations (such as error handling) in response to a particular database error. For a list of the possible SQLSTATE codes, see Appendix A. This field is not localizable, and is always present.


### Required Client-C libraries

DBDish::mysql by default searches for 'mysql' (libmysql.ddl) on Windows, and
'mariadb' (libmariadb.so.xx where xx in 0 .. 4) then
'mysqlclient' (libmysqlclient.so.xx where xx in 16..21) on POSIX systems.

Remember that Windows uses `PATH` to locate the library. On POSIX,
unversionized `*.so` files installed by "dev" packages aren't needed nor used,
you need the run-time versionized library.

On POSIX you can use the `$DBIISH_MYSQL_LIB` environment variable to request another
client library to be searched and loaded.

Example using the unadorned name:

    DBIISH_MYSQL_LIB=mariadb rakudo t/25-mysql-common.t

Using the absolute path in uninstalled DBIish:

    DBIISH_MYSQL_LIB=/lib64/libmariadb.so.3 rakudo -t lib t/25-mysql-common.t

With MariaBD-Embedded:

    DBIISH_MYSQL_LIB=mariadbd rakudo -I lib t/01-basic.t

### `insert-id`

Returns the AUTO\_INCREMENT value of the most recently inserted record.

    my $sth = $dbh.execute( 'INSERT INTO tab (description) VALUES (?)', $description );

    my $id = $sth.insert-id;

    # or
    my $id = $dbh.insert-id;

## Oracle

Supports basic CRUD operations and prepared statements with placeholders

    my $dbh = DBIish.connect('Oracle', database => 'XE', :user<sysadm>, :password('secret'));

By default connections to Oracle will apply this session alteration in an attempt to
ensure the formatted "TIMESTAMP WITH TIME ZONE" field string will be compatible with DateTime
and returned to the user as DateTime.new($ts\_str).

    ALTER SESSION SET nls_timestamp_tz_format = 'YYYY-MM-DD"T"HH24:MI:SS.FFTZR'

WARNING: This alteration does not include support for these field types. Also until now these
types would have thrown an exception as an unknown TYPE.

    TIMESTAMP
    TIMESTAMP WITH LOCAL TIME ZONE

WARNING: Any form of TIMESTAMP(0) will produce a string not compatible with DateTime
due to the ".FF" and the lack of fractional seconds to fulfill it.

You can choose to use this session alteration in an attempt to simplify the use of
ISO-8601 timestamps; strictly speaking, formatted as "YYYY-MM-DDTHH:MI:SSZ"; no offsets
etc shown but Oracle outo converts to GMT(00:00).
This session management forces all client sessions to UTC and sets formats for all DATE
and TIMESTAMP types; it does however sacrifice any fraction seconds TIMESTAMPS may be
storing. It also insures TIMSTAMP(0) works without causing DateTime.new($ts) to fault.

    DBIish.connect( 'Oracle', :alter-session-iso8601, ... );

    ALTER SESSION SET time_zone               = '-00:00'
    ALTER SESSION SET nls_date_format         = 'YYYY-MM-DD"T"HH24:MI:SS"Z"'
    ALTER SESSION SET nls_timestamp_format    = 'YYYY-MM-DD"T"HH24:MI:SS"Z"'
    ALTER SESSION SET nls_timestamp_tz_format = 'YYYY-MM-DD"T"HH24:MI:SS"Z"'

WARNING: Preexisting databases that used time zones other than UTC/GMT/-00:00 may need
to convert current timestamps to -00:00 to ensure timestamp correctness. It will depend
on the type of TIMESTAMP used and how Oracle was configured.

NOTICE: By default DBIish lower-cases FIELD names. This is noticed when data is returned
as a hash.

For consumer purists that desire DBIish to leave session management alone, the above
behaviors can be disabled using these options in the connect method. These options will
allow DBIish to most closely behave like Perl5's DBI defaults. These are my personal
favorite settings.

    :no-alter-session      # don't alter session
    :no-datetime-container # return the date/timestamps as stings
    :no-lc-field-names     # return field names unaltered

## Threads

I have a long history of using Threads, Oracle and Perl5. I have yet to read any
useful online notes regarding successful usage of Raku, threads & DBIish; I thought
I'd share recent experience.

Since early 2021 I've successfully implemented my Raku solution for using as
many as Eight(8) threads all connected to Oracle performing simultaneous Reads and
writes. As with Perl-5 the number one requirement is to ensure each thread
creates is own connection handle with Oracle. In case you're interested; its
implemented as a layer on top of DBIish that when .enable(N) is used determines
the number of worker threads; each capable to handling reads or writes. The
primary application delegates writes to the workers and reads are asynchronously
delivered where requested. This solution allows the application to stay focused on
it's primary purpose while dedicated writers handle the DB updates.

Regards,
ancient-wizard

# TESTING

The `DBIish::CommonTesting` module, now with over 100 tests, provides a common unit
testing that allows a driver developer to test its driver capabilities and the
minimum expected compatibility.

Set environment variable `DBIISH_WRITE_TEST=YES` to run tests which may leave permanent state changes in the database.


# SEE ALSO

The Raku Pod in the [doc:DBIish](doc:DBIish) module and examples in the [examples](https://github.com/raku-community-modules/DBIish/tree/master/examples) directory.

This README and the documentation of the DBIish and the DBDish modules
are in the Pod6 format. It can be extracted by running

    rakudo --doc <filename>

Or, if [Pod::To::HTML](https://metacpan.org/pod/Pod%3A%3ATo%3A%3AHTML) is installed,

    rakudo --doc=html <filename>

Additional modules of interest may include:

- [DBIish::Transaction](https://modules.raku.org/dist/DBIish::Transaction:cpan:RBT)

    A wrapper for managing transactions, including automatic retry for temporary failures.

- [DBIish::Pool](https://modules.raku.org/dist/DBIish::Pool:cpan:RBT)

    Connection reuse for DBIish to reduce loads on high-volume or heavily encrypted database sessions.

## HISTORY

DBIish is based on Martin Berends' [MiniDBI](https://github.com/mberends/MiniDBI) project, but unlike MiniDBI, DBDish
aims to provide an interface that takes advantage of Raku idioms.

There is/was an intention to integrate with the [DBDI project](http://github.com/timbunce/DBDI)
once it has sufficient functionality.

So, while it is indirectly inspired by [Perl 5 DBI](https://metacpan.org/pod/DBI), there are also many differences.

# COPYRIGHT

Written by Moritz Lenz, based on the MiniDBI code by Martin Berends.

See the [CREDITS](https://github.com/raku-community-modules/DBIish/blob/master/CREDITS) file for a list of all contributors.

# LICENSE

Copyright © 2009-2020, the DBIish contributors
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

- Redistributions of source code must retain the above copyright notice,
this list of conditions and the following disclaimer.
- Redistributions in binary form must reproduce the above copyright
notice, this list of conditions and the following disclaimer in the
documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS
OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY
WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.
