#easy-pg
[![Build Status](https://travis-ci.org/falsecz/easy-pg.png?branch=master)](https://travis-ci.org/falsecz/easy-pg)
[![Dependency Status](https://david-dm.org/falsecz/easy-pg.png)](https://david-dm.org/falsecz/easy-pg)

easy-pg is "easy to use" deferred PostgreSQL client for node.js with possibility of using native libpq bindings and providing some frequently used querying functions. It prevents queries from not being processed due to unexpected <a href="#acceptable-errors">minor errors</a> such as temporary loss of connection. Easy-pg stacks queries during transactions as well to revive whole transaction in the case of interrupted connection.

##Installation

    npm install easy-pg

##Examples

###Simple Connection
Simple example of connecting to postgres instance, running a query and disconnecting. Client is created as deferrer client thus it's not connected until the first query is requested. In this example number <b>7</b> is inserted into table called <b>numbers</b>, column <b>number</b>. Client is disconnected right after the query result is known.

```coffeescript
epg = require "easy-pg"
# epg = require("easy-pg").native -to use native libpq bindings

client = epg "pg://postgres@localhost/myapp_test"

client.on "ready", () -> console.log "Client is connected"
client.on "end", () -> console.log "Client is disconnected"

# not connected so far, it's deferred client!
# client creates connection only with the first query

#insert number into specified table and disconnect
client.insert "numbers", {number: 7}, (err, res) ->
	console.log err if err?
	console.log res if res?
	client.end()
```
Previous code results in:

    Client is connected
    {id: 1, number: 7}
    Client is disconnected

###Connection Parameters & Options
You can pass connection string or object into easy-pg constructor with connection options. These options are processed by client (if known) and transparently forwarded to postgres instance later.

```coffeescript
epg = require "easy-pg"

#connection string
conString = "pg://postgres@localhost/myapp_test?opt1=val1&opt2=val2&opt3=val3"

#the same connection object
conObject =
	protocol:	"pg:"
	user:		"postgres"
	host:		"localhost"
	db:			"myapp_test"

	options: {
		opt1: val1
		opt2: val2
		opt3: val3
	}

#both following is correct
client = epg conString
client = epg conObject
```

Following connection parameters and options can be used:

* Connection parameters
  * <b>protocol</b> <i>(required)</i>
  * <b>user</b>
  * <b>password</b>
  * <b>host</b> <i>(required)</i>
  * <b>port</b>
  * <b>db</b> <i>(required)</i>
* Connection options
  * <b>lazy</b> <i>-set to "no" or "false" to force the client to connect immediately</i>
  * <b>datestyle</b> <i>-instead of (in SQL) commonly used SET DATESTYLE</i>
  * <b>searchPath</b> <i>-instead of (in SQL) commonly used SET SEARCH_PATH</i>

Full connection string may look like this: <i>"pg://postgres:123456@localhost:5432/myapp_test?lazy=no&datestyle=iso, mdy&searchPath=public&poolSize=1"</i>, where <b>poolSize</b> is not handled by easy-pg, but postgres instance. Connection options are checked and applied every time the client is (re)connected, thus once you for example set <b>datestyle</b>, it is kept set until the client is disconnected and destroyed. Even if the connection is temporarily lost.

###Disconnection

Client creates connection to specified postgres instance automatically, however disconnection has to be done manually. Easy-pg provides two functions for client disconnection or termination. Function <b>end</b> can be used to disconnect client with the last processed query. This way of client disconnection should be used in common cases. In the case of stuck, <b>kill</b> can be used to terminate the client immediately, but there is a risk of unpredictable behavior. Both functions emit <b>end</b> event.

```coffeescript
epg = require "easy-pg"

client = epg "pg://postgres@localhost/myapp_test"

# not connected yet

client.queryAll "SELECT * FROM table"

# auto-connecting, client is going to send the query

client.end()

# connected, query is being processed
# end() is waiting until the query is finished

client.kill()
# connected, query is being processed, we don't want to wait anymore
# client is terminated immediately, error could occur

# not connected, end event is emitted
```

###Client Events

There are 3 events emitted by easy-pg client:

* <b>ready</b>
* <b>end</b>
* <b>error</b> <i>(client throws an ordinary Error if <b>error</b> listener is not registered, as shown in the following code)</i>

```coffeescript
epg = require "easy-pg"

client = epg "pg://postgres@localhost/myapp_test"

# an ordinary Error can be thrown here

client.on "ready", () -> console.log "Client is connected"
client.on "end", () -> console.log "Client is disconnected"

# an ordinary Error can still be thrown here

client.on "error", (err) ->
	console.log "Client error: " + err

# error event can be emitted here
```

<b>Error</b> event is emitted just in the case of fatal error (syntax error, etc.). For example, if postgres server is restarted while processing query and the query fails, client reconnects itself and tries to process this query again without emitting or throwing any error.

###Making Queries

Any kind of queries can be created and sent by easy-pg client, even with parameter binding. Queries can be easily made using following functions:

* <b>query</b>
* <b>queryAll</b>
* <b>queryOne</b>

These functions differ just in the data format of their results. Function <b>query</b> returns raw result of the query containing number of rows, table id, etc. <b>QueryAll</b> returns only array of all rows of the query result and <b>queryOne</b> returns only the first entry (row) of this array. They can be used as shown in the code:

```coffeescript
epg = require "easy-pg"

client = epg "pg://postgres@localhost/myapp_test"

client.query "SET DATESTYLE = iso"
client.query "SELECT * FROM table", (err, res) -> # do sth. in callback...
client.query "SELECT $1 FROM $2", ["*", "table"] # bind some parameters...
client.query "SELECT $1 FROM $2", ["*", "table"], (err, res) -> # do sth. in callback...

client.queryAll "SET DATESTYLE = iso"
client.queryAll "SELECT * FROM table", (err, res) -> # do sth. in callback...
client.queryAll "SELECT $1 FROM $2", ["*", "table"] # bind some parameters...
client.queryAll "SELECT $1 FROM $2", ["*", "table"], (err, res) -> # do sth. in callback...

client.queryOne "SET DATESTYLE = iso"
client.queryOne "SELECT * FROM table", (err, res) -> # do sth. in callback...
client.queryOne "SELECT $1 FROM $2", ["*", "table"] # bind some parameters...
client.queryOne "SELECT $1 FROM $2", ["*", "table"], (err, res) -> # do sth. in callback...
```

###Built-in Querying Functions

Easy-pg provides some well known querying functions as well to make your work easier and source code cleaner. Implemented querying functions are <b>insert</b>, <b>update</b>, <b>upsert</b>, <b>delete</b> and <b>paginate</b>.

```coffeescript
epg = require "easy-pg"

client = epg "pg://postgres@localhost/myapp_test"

# db contains table "numbers" with column "number"

# table, value
client.insert "numbers", {number: 0} # insert one row
client.insert "numbers", {number: 4}, (err, res) -> # do sth. in callback...
client.insert "numbers", [{number: 1}, {number: 2}, {number: 3}] # insert 3 rows
client.insert "numbers", [{number: 1}, {number: 2}, {number: 3}], (err, res) -> # do sth. in callback...

# table, value, where
client.update "numbers", {number: 99}, "number = 0" # replaces number 0 by 99
client.update "numbers", {number: 99}, "number = 0", (err, res) -> # do sth. in callback...
client.update "numbers", {number: 99}, "number = $1", [1] # replaces number 1 by 99
client.update "numbers", {number: 99}, "number = $1", [1], (err, res) -> # do sth. in callback...

# table, value, where
client.upsert "numbers", {number: 9}, "number = 9" # inserts number 9
client.upsert "numbers", {number: 9}, "number = 9", (err, res) -> # do sth. in callback...
client.upsert "numbers", {number: 9}, "number = $1", [9] # replaces number 9 by 9
client.upsert "numbers", {number: 9}, "number = $1", [9], (err, res) -> # do sth. in callback...

# table
client.delete "numbers" # deletes table "numbers"
client.delete "numbers", (err, res) -> # do sth. in callback...
client.delete "numbers", "number = 0" # deletes rows with 0
client.delete "numbers", "number = 0", (err, res) -> # do sth. in callback...
client.delete "numbers", "number = $1", [1] # deletes rows with 1
client.delete "numbers", "number = $1", [1], (err, res) -> # do sth. in callback...

# offset, limit, columns, query
client.paginate 0, 10, "number", "SELECT * FROM numbers" # lists first 10 rows of the given query
client.paginate 0, 10, "number", "SELECT * FROM numbers", (err, res) -> # do sth. in callback...
client.paginate 0, 10, "_id, number", "SELECT * FROM numbers WHERE _id > $1 ORDER BY _id", [9] # the same with ids > 9
client.paginate 0, 10, "_id, number", "SELECT * FROM numbers WHERE _id > $1 ORDER BY _id", [9], (err, res) -> # do sth. in callback...
```

###Transactions

Transactions are also carefully handled by easy-pg. Once the transaction is started, all queries are saved into transaction stack until the final <b>commit</b> or <b>rollback</b> is called. In the case of temporary connection loss or other <a href="#acceptable-errors">minor error</a>, whole transaction is revived and processed again. Following functions can be used to control the transaction flow:

* <b>begin</b>
* <b>savepoint</b>
* <b>commit</b>
* <b>rollback</b>

See the source code below to understand the use of these functions.

```coffeescript
epg = require "easy-pg"

client = epg "pg://postgres@localhost/myapp_test"

client.begin() # begins transaction, client.query "BEGIN" can be used instead
client.begin (err, res) -> # do sth. in callback...

client.savepoint "my_savepoint" # creates savepoint for rollback to savepoint
client.savepoint "my_savepoint", (err, res) -> # do sth. in callback...

client.commit() # commits changes in db
client.commit (err, res) -> # do sth. in callback...

client.rollback() # rolls back to closest begin
client.rollback (err, res) -> # do sth. in callback...
client.rollback "my_savepoint" # rolls back to "my_savepoint"
client.rollback "my_savepoint", (err, res) -> # do sth. in callback...
```

Stacks are used to allows the client proper handling of nested transactions!

    COMMANDS   STACK
    begin      B
    query      QB
    begin      BQB
    query      QBQB
    query      QQBQB
    commit     QB
    query      QQB
    commit   


###Acceptable Errors

Minor errors were mentioned in the text above. All messages sent by PostgreSQL server contain error codes to inform client about the state of the database. Most of these codes are not handled, just forwarded through callback-err, except of 3 error code classes:

* <b>00</b> Successful Completion
* <b>08</b> Connection Exception
* <b>57</b> Operator Intervention

These 3 types of errors only forces the client to restart current connection and continue in query queue processing later. More information about PostgreSQL error codes can be found <a href="http://www.postgresql.org/docs/9.2/static/errcodes-appendix.html">here</a>.