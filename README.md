# Any-DB API

This repo specifies the API that must be implemented by an Any-DB adapter. The
API is described in this README using [jsig][] and prose, and the
[test suite][] can be used by adapter implementations to ensure they conform.

Because this documentation is primarily intended for end-users of Any-DB, it
begins by describing the objects an adapter must create. The final section
describes the exact API for creating these objects that an adapter must
implement.

## Interfaces

 - [Queryable][] - a common interface implemented by connections, pools, and
   transactions.
 - [Connection][] - the "transport" responsible for getting SQL queries to a
   database, and streaming results back through a `Query` instance.
 - [Query][] - a [Readable][] stream that emits row objects.

## Queryable

```ocaml
Queryable := EventEmitter & {
  adapter: Adapter
  query:   (text: String, params: Array?, Continuation<Results>?) => Query
  query:   (Query) => Query
}
```

Known implementations:
 - [Connection][Connection]
 - [ConnectionPool][ConnectionPool] (external)
 - [Transaction][Transaction] (external).

### Queryable.adapter

The [Adapter][] instance that will be used by this `Queryable` for creating
[Query][] instances and/or connections.

### Queryable.query

```ocaml
(text: String, params: Array?, Continuation<ResultSet>?) => Query
(Query) => Query
```

Execute a SQL statement using bound parameters (if they are provided) and
return a [Query][] object that is a [Readable][] stream of the resulting
rows. If a `Continuation<Results>` is provided the rows returned by the
database will be aggregated into a [ResultSet][] which will be passed to the
continuation after the query has completed.

The second form is not needed for normal use, but must be implemented by
adapters to work correctly with [ConnectionPool][] and [Transaction][]. See
[Adapter.createQuery](#adapter-createquery) for more details.

*Callback-style*
```javascript
queryable.query('SELECT * FROM my_table', function (err, res) {
  if (err) return console.error(err)
  res.rows.forEach(console.log)
  console.log('All done!')
})
```

*Stream-style*
```javascript
queryable.query('SELECT * FROM my_table')
  .on('error', console.error)
  .on('data', console.log)
  .on('end', function () { console.log('All done!') })
```

### Queryable events

 - `'query', query` - Emitted immediately before a query is executed. 

## Connection

```ocaml
Connection := Queryable & {
  end: (Continuation<void>?) => void
}
```

Known implementations:

 - [any-db-mysql][] (external)
 - [any-db-postgres][] (external)
 - [any-db-sqlite3][] (external)

Connection objects are obtained using [createConnection][] from [Any-DB][] or
[ConnectionPool.acquire][], both of which delegate to the
[createConnection](#adaptercreateconnection) implementation of the specified
adapter.

While all `Connection` objects implement the [Queryable][] interface, the
implementations in each adapter may add additional methods or emit additional
events. If you need to access a feature of your database that is not described
here (such as Postgres' server-side prepared statements), consult the
documentation for your adapter.

### Connection.end

`(Continuation<void>) => void`

Close the database connection. If a continuation is provided it will be
called after the connection has closed.

### Connection Events

 * `'error', err` - Emitted when there is a connection-level error.
 * `'close'` - Emitted when the connection has been closed.

## Query

```ocaml
Query := Readable<Object> & {
  text: String,
  values: Array
}
```

`Query` objects are returned by the [Queryable.query][Queryable.query] method,
available on [connections][Connection], [pools][ConnectionPool.query], and
[transactions][Transaction.query]. Queries are instances of [Readable][], and 
as such can be [piped][Readable.pipe] through transforms and support backpressure
for more efficient memory-usage on very large results sets. (Note: at this time
the `sqlite3` driver does not support backpressure)

Internally, `Query` instances are
[created by a database Adapter][Adapter.createQuery] and may have more methods,
properties, and events than are described here. Consult the documentation for
your specific adapter to find out about any extensions.

### Query.text

The SQL query as a string. If you are using MySQL this will contain
interpolated values *after* the query has been enqueued by a connection.

### Query.values

The array of parameter values.

### Query Events

 * `'error', error` - Emitted if the query results in an error.
 * `'fields', fields` - An array of [Field][ResultSet] objects emitted before
   any `'data'` events.
 * `'data', row` - Emitted for each row in the query result set.
 * `'end'` - Emitted when the query completes.

## ResultSet

```ocaml
ResultSet := {
  fields:       Array<Field>
  rows:         Array<Object<Any>>
  rowCount:     Integer
  lastInsertId: Any?
}

Field := {
  name: String
  {* other properties are driver specific *}
}
```

`ResultSet` objects are just plain data that collect results of a query when a 
continuation is provided to [Queryable.query][]. The `lastInsertId` is optional,
and currently supported by `sqlite3` and `mysql` but not `postgres`, because
it is not supported by Postgres itself.

## Adapter

```ocaml
Adapter: {
  name:             String
  createConnection: (Object, Continuation<Connection>?) => Connection,
  createQuery:      (String, Array?, Continuation<Results>?) => Query,
}
```

This section is mostly intended for adapter authors, other users should rarely
need to interact with this API directly.

### Adapter.name

The string name of the adapter, e.g. `'mysql'`, `'postgres'` or `'sqlite3'`.

### Adapter.createConnection

`(config: Object, Continuation<Connection>?) => Connection`

Create a new connection object. In common usage, `config` will be created by
[parse-db-url][] and passed to the adapter by [any-db][].

If a continuation is given, it **must** be called, either with an error or the
established connection.

See also: the [Connection API](#connection)

### Adapter.createQuery

```ocaml
(text: String, params: Array?, Continuation<ResultSet>?) => Query
(Query) => Query {* returns same Query *}
```

Create a [Query](#query) that *may* eventually be executed later on by a
[Connection][]. While this function is rarely needed by user code, it makes
it possible for [ConnectionPool.query][] and [Transaction.query][] to fulfill
the [Queryable.query][] contract by synchronously returning a [Query][] stream.

# License

2-clause BSD

[jsig]: https://github.com/jden/jsig
[once]: http://npm.im/once
[parse-db-url]: https://github.com/grncdr/parse-db-url#api
[any-db]: https://github.com/grncdr/node-any-db
[any-db-mysql]: https://github.com/grncdr/node-any-db-mysql
[any-db-postgres]: https://github.com/grncdr/node-any-db-postgres
[any-db-sqlite3]: https://github.com/grncdr/node-any-db-sqlite3
[createConnection]: https://github.com/grncdr/node-any-db#exportscreateconnection

[Readable]: http://nodejs.org/api/stream.html#stream_class_stream_readable
[Readable.pipe]: http://nodejs.org/api/stream.html#stream_readable_pipe_destination_options

[ConnectionPool.query]: https://github.com/grncdr/node-any-db-pool#connectionpoolquery
[ConnectionPool.acquire]: https://github.com/grncdr/node-any-db-pool#connectionpoolacquire
[ConnectionPool]: https://github.com/grncdr/node-any-db-pool#api
[Transaction]: https://github.com/grncdr/node-any-db-transaction#api
[any-db-transaction]: https://github.com/grncdr/node-any-db-transaction
[Transaction.query]: https://github.com/grncdr/node-any-db-transaction#transactionquery

[test suite]: tests
[Adapter]: #adapter
[Queryable]: #queryable
[Queryable.query]: #queryablequery
[Connection]: #connection
[Connection.query]: #connectionquery
[Query]: #query
[Adapter.createQuery]: #adaptercreatequery
