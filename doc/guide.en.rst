.. encoding: utf-8

Developer's guide
=================

Basic concepts
--------------

Spaces
^^^^^^

Spaces is a collections of tuples.
Usually, tuples in one space represent objects of the same type,
although this is not necessary.

.. note:: The analogue of spaces is tables in traditional (SQL) databases.

Spaces have integer identifiers defined in the server configuration.
To access the space as a named object it is possible to use the method
:meth:`Connection.space() <tarantool.connection.Connection.space>`
and an instance of :class:`~tarantool.space.Space`.

Example::

    >>> customer = connection.space(0)
    >>> customer.insert(('FFFF', 'Foxtrot'))


Field types
^^^^^^^^^^^

Three field types are supported in Tarantool: ``STR``, ``NUM`` and ``NUM64``.
These types are used only for index configuration
but not saved in tuple's data and not transferred between the client and server.
Thus, from the client point of view, fields are raw byte arrays
without explicitly definde types.

It is much easier to use native types for python developer:
``int``, ``long``, ``unicode`` (``int`` and ``str`` for Python 3.x).
For raw binary data ``bytes`` should be used
(in this case the type casting is not performed).

Pass field types using argument ``field_types`` for automatic type casting::

    >>> demo = connection.space(0, field_types=(int, unicode))
    >>> demo.insert((0, u'this is unicode string'))
    >>> demo.select(0)
    [(0, u'this is unicode string')]

As you can see, a tuple of types is passed with ``field_types``.
Original "raw" fields will be casted to these native types.

Tarantool's tuple can contain any number of fields.

If the tuple contains more fields than types listed in ``field_types``,
the latest type from ``field_types`` will be applied to all the remaining fields of the tuple.

Example::

    >>> demo = connection.space(0, field_types=(int, unicode))
    >>> demo.insert((0, u'I am the Alpha (Α) and Omega (Ω)', b'AAAA', b'BBBB', b'CCCC', b'DDDD'))
    >>> demo.select(0)
    [(0, u'I am the Α and Ω', u'AAAA', u'BBBB', u'CCCC', u'DDDD')]

As you can see, all values was converted to unicode-strings.

To prevent such implicit type casting it is enough to add ``bytes``
as the last element of ``field_types`` tuple::

    >>> demo = connection.space(0, field_types=(int, unicode, bytes))
    >>> demo.insert((0, u'I am the Alpha (Α) and Omega (Ω)', b'AAAA', b'BBBB', b'CCCC', b'DDDD'))
    >>> demo.select(0)
    [(0, u'I am the Α and Ω', 'AAAA', 'BBBB', 'CCCC', 'DDDD')]


Raw byte fields should be used if the application uses binary data
(eg, images or python objects packed with ``picke``).


.. note::

   Python 2.6 adds :class:`bytes` as a synonym for the :class:`str` type, and it also supports the ``b''`` notation.


.. note:: **utf-8** allways used for type conversion between ``unicode`` and ``bytes``



Request response
^^^^^^^^^^^^^^^^

Requests (:meth:`insert() <tarantool.space.Space.insert>`,
:meth:`delete() <tarantool.space.Space.delete>`,
:meth:`update() <tarantool.space.Space.update>`,
:meth:`select() <tarantool.space.Space.select>`) return a
:class:`~tarantool.response.Response` instance.

Class :class:`~tarantool.response.Response` inherited from `list`,
so in fact response can be used as a list of a tuples.

In addition :class:`~tarantool.response.Response` instance has the ``rowcount`` attribute.
The value of ``rowcount`` equals to the number of records affected by the request.
For example for :meth:`delete() <tarantool.space.Space.delete>`
request ``rowcount`` is equals to ``1`` if record was deleted.



Connect to the server
---------------------

To connect to the server it is required to use :meth:`tarantool.connect` method.
It returns an :class:`~tarantool.connection.Connection` instance.

Example::

    >>> import tarantool
    >>> connection = tarantool.connect("localhost", 33013)
    >>> type(connection)
    <class 'tarantool.connection.Connection'>



Data manipulation
-----------------

There are four basic operations supported by Tarantool:
**insert**, **delete**, **update** and **select**.

.. Note:: НЕОБХОДИМО ОБЪЯСНИТЬ КАКИЕ ДАННЫЕ ИСПОЛЬЗУЮТСЯ ДЛЯ ПРИМЕРА


Inserting and replacing records
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To insert or replace records :meth:`Space.insert() <tarantool.space.Space.insert>`
method should be used::

    >>> user.insert((user_id, email, int(time.time())))

The first element of the tuple is always its unique primary key.

If an entry with the same key already exists, it will be replaced
without any warning or error message.

.. note:: In case of ``insert`` request ``Response.rowcount`` is always equals to ``1``


Deleting Records
^^^^^^^^^^^^^^^^

To delete records :meth:`Space.delete() <tarantool.space.Space.delete>`
method should be used::

    >>> user.delete(primary_key)

.. note:: If the record was deleted ``Response.rowcount`` equals to ``1``.
          If the record was not found ``Response.rowcount`` equals to ``0``.


Updating Records
^^^^^^^^^^^^^^^^

*Update* request in Tarantool allows to simultaneous and atomic update multiple
fields of a tuple.

To update records :meth:`Space.update() <tarantool.space.Space.update>`
method should be used.

Example::

    >>> user.update(1001, [(1, '=', 'John'), (2, '=', 'Smith')])

In this example new values for fields ``1`` and ``2`` are assigned.

:meth:`Space.update() <tarantool.space.Space.update>` method allows to change
multiple fields of the tuple at a time.

The following update operations are supported by Tarantool:
    • ``'='`` – assign new value to the field
    • ``'+'`` – add argument to the field (*both arguments are treated as signed 32-bit ints*)
    • ``'^'`` – bitwise AND (*only for 32-bit integers*)
    • ``'|'`` – bitwise XOR (*only for 32-bit integers*)
    • ``'&'`` – bitwise OR  (*only for 32-bit integers*)
    • ``'splice'`` – implementation of `Perl splice <http://perldoc.perl.org/functions/splice.html>`_ function


.. note:: The zero (i.e. [0]) field of the tuple can not be updated,
           because it is the primary key

.. seealso:: See :meth:`Space.update() <tarantool.space.Space.update>` documentation for details

.. warning:: ``'splice'`` operation is not implemented yet


Selecting Records
^^^^^^^^^^^^^^^^^

To select records :meth:`Space.select() <tarantool.space.Space.select>`
method should be used.
*SELECT* query can return one or many records.


.. rubric:: Select by primary key

Select a record using its primary key ``3800``::

    >>> world.select(3800)
    [(3800, u'USA', u'Texas', u'Dallas', 1188580)]


.. rubric:: Select using secondary index

::

    >>> world.select('USA', index=1)
    [(3796, u'USA', u'Texas', u'Houston', 1953631),
     (3801, u'USA', u'Texas', u'Huston', 10000),
     (3802, u'USA', u'California', u'Los Angeles', 10000),
     (3805, u'USA', u'California', u'San Francisco', 776733),
     (3800, u'USA', u'Texas', u'Dallas', 1188580),
     (3794, u'USA', u'California', u'Los Angeles', 3694820)]


Argument ``index = 1`` indicates that secondary index (``1``) should be used.
The primary key (``index=0``) is used by default.

.. note:: Secondary indexes must be explicitly declared in the server configuration


.. rubric:: Select records using several keys

.. note:: This conforms to ``where key in (k1, k2, k3...)``

Select records with primary key values ``3800``, ``3805`` and ``3796``::

    >>> world.select([3800, 3805, 3796])
    [(3800, u'USA', u'Texas', u'Dallas', 1188580),
     (3805, u'USA', u'California', u'San Francisco', 776733),
     (3796, u'USA', u'Texas', u'Houston', 1953631)]


.. rubric:: Retrieve a record by using a composite index

Select data on cities in Texas::

    >>> world.select([('USA', 'Texas')], index=1)
    [(3800, u'USA', u'Texas', u'Dallas', 1188580), (3796, u'USA', u'Texas', u'Houston', 1953631)]


.. rubric:: Select records explicitly specifying field types

Tarantool has no strict schema so all fields are raw binary byte arrays.
You can specify field types in directly in
:meth:`Space.select() <tarantool.space.Space.select>` method
using ``field_types`` keyword argument::

    >>> world.select(3800, field_types=(bytes, str, str, str, bytes))
    [('\xd8\x0e\x00\x00', 'USA', 'Texas', 'Dallas', '\xe4"\x12\x00')]

As you can see, ``3800`` returned
as 4-byte array (string) instead of integer.


Call server-side functions
--------------------------

A server-side function written in Lua can select and modify data,
access configuration and perform administrative tasks.

To call stored function
:meth:`Connection.call() <tarantool.connection.Connection.call>`
method should be used.
Also, this method has an alias :meth:`Space.call() <tarantool.space.Space.call>`.

Example::

    >>> server.call("box.select_range", (1, 3, 2, 'AAAA'))
    [(3800, u'USA', u'Texas', u'Dallas', 1188580), (3794, u'USA', u'California', u'Los Angeles', 3694820)]

.. seealso::

    Tarantool/Box User Guide » `Writing stored procedures in Lua <http://tarantool.org/tarantool_user_guide.html#stored-programs>`_