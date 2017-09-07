.. _asyncpg-examples:


asyncpg Usage
=============

The interaction with the database normally starts with a call to
:func:`connect() <asyncpg.connection.connect>`, which establishes
a new database session and returns a new
:class:`Connection <asyncpg.connection.Connection>` instance,
which provides methods to run queries and manage transactions.


.. code-block:: python

    import asyncio
    import asyncpg
    import datetime

    async def main():
        # Establish a connection to an existing database named "test"
        # as a "postgres" user.
        conn = await asyncpg.connect('postgresql://postgres@localhost/test')
        # Execute a statement to create a new table.
        await conn.execute('''
            CREATE TABLE users(
                id serial PRIMARY KEY,
                name text,
                dob date
            )
        ''')

        # Insert a record into the created table.
        await conn.execute('''
            INSERT INTO users(name, dob) VALUES($1, $2)
        ''', 'Bob', datetime.date(1984, 3, 1))

        # Select a row from the table.
        row = await conn.fetchrow(
            'SELECT * FROM users WHERE name = $1', 'Bob')
        # *row* now contains
        # asyncpg.Record(id=1, name='Bob', dob=datetime.date(1984, 3, 1))

        # Close the connection.
        await conn.close()

    asyncio.get_event_loop().run_until_complete(main())



.. note::

   asyncpg uses the native PostgreSQL syntax for query arguments: ``$n``.



Type Conversion
---------------

asyncpg automatically converts PostgreSQL types to the corresponding Python
types and vice versa.  All standard data types are supported out of the box,
including arrays, composite types, range types, enumerations and any
combination of them.  It is possible to supply codecs for non-standard
types or override standard codecs.  See :ref:`asyncpg-custom-codecs` for
more information.

The table below shows the correspondence between PostgreSQL and Python types.

+----------------------+-----------------------------------------------------+
| PostgreSQL Type      |  Python Type                                        |
+======================+=====================================================+
| ``anyarray``         | :class:`list <python:list>`                         |
+----------------------+-----------------------------------------------------+
| ``anyenum``          | :class:`str <python:str>`                           |
+----------------------+-----------------------------------------------------+
| ``anyrange``         | :class:`asyncpg.Range <asyncpg.types.Range>`        |
+----------------------+-----------------------------------------------------+
| ``record``           | :class:`asyncpg.Record`,                            |
|                      | :class:`tuple <python:tuple>`                       |
+----------------------+-----------------------------------------------------+
| ``bit``, ``varbit``  | :class:`asyncpg.BitString <asyncpg.types.BitString>`|
+----------------------+-----------------------------------------------------+
| ``bool``             | :class:`bool <python:bool>`                         |
+----------------------+-----------------------------------------------------+
| ``box``              | :class:`asyncpg.Box <asyncpg.types.Box>`            |
+----------------------+-----------------------------------------------------+
| ``bytea``            | :class:`bytes <python:bytes>`                       |
+----------------------+-----------------------------------------------------+
| ``char``, ``name``,  | :class:`str <python:str>`                           |
| ``varchar``,         |                                                     |
| ``text``,            |                                                     |
| ``xml``              |                                                     |
+----------------------+-----------------------------------------------------+
| ``cidr``             | :class:`ipaddress.IPv4Network\                      |
|                      | <python:ipaddress.IPv4Network>`,                    |
|                      | :class:`ipaddress.IPv6Network\                      |
|                      | <python:ipaddress.IPv6Network>`                     |
+----------------------+-----------------------------------------------------+
| ``inet``             | :class:`ipaddress.IPv4Network\                      |
|                      | <python:ipaddress.IPv4Network>`,                    |
|                      | :class:`ipaddress.IPv6Network\                      |
|                      | <python:ipaddress.IPv6Network>`,                    |
|                      | :class:`ipaddress.IPv4Address\                      |
|                      | <python:ipaddress.IPv4Address>`,                    |
|                      | :class:`ipaddress.IPv6Address\                      |
|                      | <python:ipaddress.IPv6Address>`                     |
+----------------------+-----------------------------------------------------+
| ``macaddr``          | :class:`str <python:str>`                           |
+----------------------+-----------------------------------------------------+
| ``circle``           | :class:`asyncpg.Circle <asyncpg.types.Circle>`      |
+----------------------+-----------------------------------------------------+
| ``date``             | :class:`datetime.date <python:datetime.date>`       |
+----------------------+-----------------------------------------------------+
| ``time``             | offset-naïve :class:`datetime.time \                |
|                      | <python:datetime.time>`                             |
+----------------------+-----------------------------------------------------+
| ``time with          | offset-aware :class:`datetime.time \                |
| timezone``           | <python:datetime.time>`                             |
+----------------------+-----------------------------------------------------+
| ``timestamp``        | offset-naïve :class:`datetime.datetime \            |
|                      | <python:datetime.datetime>`                         |
+----------------------+-----------------------------------------------------+
| ``timestamp with     | offset-aware :class:`datetime.datetime \            |
| timezone``           | <python:datetime.datetime>`                         |
+----------------------+-----------------------------------------------------+
| ``interval``         | :class:`datetime.timedelta \                        |
|                      | <python:datetime.timedelta>`                        |
+----------------------+-----------------------------------------------------+
| ``float``,           | :class:`float <python:float>`                       |
| ``double precision`` |                                                     |
+----------------------+-----------------------------------------------------+
| ``smallint``,        | :class:`int <python:int>`                           |
| ``integer``,         |                                                     |
| ``bigint``           |                                                     |
+----------------------+-----------------------------------------------------+
| ``numeric``          | :class:`Decimal <python:decimal.Decimal>`           |
+----------------------+-----------------------------------------------------+
| ``json``, ``jsonb``  | :class:`str <python:str>`                           |
+----------------------+-----------------------------------------------------+
| ``line``             | :class:`asyncpg.Line <asyncpg.types.Line>`          |
+----------------------+-----------------------------------------------------+
| ``lseg``             | :class:`asyncpg.LineSegment \                       |
|                      | <asyncpg.types.LineSegment>`                        |
+----------------------+-----------------------------------------------------+
| ``money``            | :class:`str <python:str>`                           |
+----------------------+-----------------------------------------------------+
| ``path``             | :class:`asyncpg.Path <asyncpg.types.Path>`          |
+----------------------+-----------------------------------------------------+
| ``point``            | :class:`asyncpg.Point <asyncpg.types.Point>`        |
+----------------------+-----------------------------------------------------+
| ``polygon``          | :class:`asyncpg.Polygon <asyncpg.types.Polygon>`    |
+----------------------+-----------------------------------------------------+
| ``uuid``             | :class:`uuid.UUID <python:uuid.UUID>`               |
+----------------------+-----------------------------------------------------+

All other types are encoded and decoded as text by default.


.. _asyncpg-custom-codecs:

Custom Type Conversions
-----------------------

asyncpg allows defining custom type conversion functions both for standard
and user-defined types using the :meth:`Connection.set_type_codec() \
<asyncpg.connection.Connection.set_type_codec>` and
:meth:`Connection.set_builtin_type_codec() \
<asyncpg.connection.Connection.set_builtin_type_codec>` methods.
The example below shows how to configure asyncpg to encode and decode
JSON values using the :mod:`json <python:json>` module.

.. code-block:: python

    import asyncio
    import asyncpg
    import json


    async def main():
        conn = await asyncpg.connect('postgresql://postgres@localhost/test')

        try:
            def _encoder(value):
                return json.dumps(value).encode('utf-8')

            def _decoder(value):
                return json.loads(value.decode('utf-8'))

            await conn.set_type_codec(
                'json', encoder=_encoder, decoder=_decoder,
                schema='pg_catalog', binary=True
            )

            data = {'foo': 'bar', 'spam': 1}
            res = await conn.fetchval('SELECT $1::json', data)

        finally:
            await conn.close()

    asyncio.get_event_loop().run_until_complete(main())


Transactions
------------

To create transactions, the
:meth:`Connection.transaction() <asyncpg.connection.Connection>` method
should be used.

The most common way to use transactions is through an ``async with`` statement:

.. code-block:: python

   async with connection.transaction():
       await connection.execute("INSERT INTO mytable VALUES(1, 2, 3)")

.. note::

   When not in an explicit transaction block, any changes to the database
   will be applied immediately.  This is also known as *auto-commit*.

See the :ref:`asyncpg-api-transaction` API documentation for more information.


Connection Pools
----------------

For server-type type applications, that handle frequent requests and need
the database connection for a short period time while handling a request,
the use of a connection pool is recommended.  asyncpg provides an advanced
pool implementation, which eliminates the need to use an external connection
pooler such as PgBouncer.

To create a connection pool, use the
:func:`asyncpg.create_pool <asyncpg.pool.create_pool>` function.
The resulting :class:`Pool <asyncpg.pool.Pool>` object can then be used
to borrow connections from the pool.

Below is an example of how **asyncpg** can be used to implement a simple
Web service that computes the requested power of two.


.. code-block:: python

    import asyncio
    import asyncpg
    from aiohttp import web


    async def handle(request):
        """Handle incoming requests."""
        pool = request.app['pool']
        power = int(request.match_info.get('power', 10))

        # Take a connection from the pool.
        async with pool.acquire() as connection:
            # Open a transaction.
            async with connection.transaction():
                # Run the query passing the request argument.
                result = await connection.fetchval('select 2 ^ $1', power)
                return web.Response(
                    text="2 ^ {} is {}".format(power, result))


    async def init_app():
        """Initialize the application server."""
        app = web.Application()
        # Create a database connection pool
        app['pool'] = await asyncpg.create_pool(database='postgres',
                                                user='postgres')
        # Configure service routes
        app.router.add_route('GET', '/{power:\d+}', handle)
        app.router.add_route('GET', '/', handle)
        return app


    loop = asyncio.get_event_loop()
    app = loop.run_until_complete(init_app())
    web.run_app(app)

See :ref:`asyncpg-api-pool` API documentation for more information.