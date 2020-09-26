.. _queryguide_toplevel:

.. rst-class:: orm_core

===========================
SQLAlchemy Querying Guide
===========================

A complete guide to the SQLAlchemy Expression Language, covering all major
concepts, as well as Core and ORM usage patterns.

The color coding from the SQLAlchemy Tutorial is applied here as well.

TODO: it's possible this may need to be in multiple pages as it is going to
get very long.

..  Setup code, not for display

    >>> from sqlalchemy import create_engine
    >>> engine = create_engine("sqlite+pysqlite:///:memory:", echo=True, future=True)
    >>> from sqlalchemy import MetaData, Table, Column, Integer, String
    >>> metadata = MetaData()
    >>> user_table = Table(
    ...     "user_account",
    ...     metadata,
    ...     Column('id', Integer, primary_key=True),
    ...     Column('name', String(30)),
    ...     Column('fullname', String)
    ... )
    >>> from sqlalchemy import ForeignKey
    >>> address_table = Table(
    ...     "address",
    ...     metadata,
    ...     Column('id', Integer, primary_key=True),
    ...     Column('user_id', None, ForeignKey('user_account.id')),
    ...     Column('email_address', String, nullable=False)
    ... )
    >>> metadata.create_all(engine)
    BEGIN (implicit)
    ...
    >>> from sqlalchemy.orm import declarative_base
    >>> Base = declarative_base()
    >>> from sqlalchemy.orm import relationship
    >>> class User(Base):
    ...     __tablename__ = 'user_account'
    ...
    ...     id = Column(Integer, primary_key=True)
    ...     name = Column(String(30))
    ...     fullname = Column(String)
    ...
    ...     addresses = relationship("Address", back_populates="user")
    ...
    ...     def __repr__(self):
    ...        return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

    >>> class Address(Base):
    ...     __tablename__ = 'address'
    ...
    ...     id = Column(Integer, primary_key=True)
    ...     email_address = Column(String, nullable=False)
    ...     user_id = Column(Integer, ForeignKey('user_account.id'))
    ...
    ...     user = relationship("User", back_populates="addresses")
    ...
    ...     def __repr__(self):
    ...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"
    >>> conn = engine.connect()
    >>> from sqlalchemy.orm import Session
    >>> session = Session(conn)
    >>> session.add_all([
    ... User(name="spongebob", fullname="Spongebob Squarepants", addresses=[
    ...    Address(email_address="spongebob@sqlalchemy.org")
    ... ]),
    ... User(name="sandy", fullname="Sandy Cheeks", addresses=[
    ...    Address(email_address="sandy@sqlalchemy.org"),
    ...     Address(email_address="squirrel@squirrelpower.org")
    ...     ]),
    ...     User(name="patrick", fullname="Patrick Star", addresses=[
    ...         Address(email_address="pat999@aol.com")
    ...     ]),
    ...     User(name="squidward", fullname="Squidward Tentacles", addresses=[
    ...         Address(email_address="stentcl@sqlalchemy.org")
    ...     ]),
    ...     User(name="ehkrabs", fullname="Eugene H. Krabs"),
    ... ])
    >>> session.commit()
    BEGIN ...
    >>> conn.begin()
    BEGIN ...

.. _queryguide_select:

SELECT statements
=================

SELECT statements are produced by the :func:`_sql.select` function which
returns a :class:`_sql.Select` object::

    >>> from sqlalchemy import select
    >>> stmt = select(user_table).where(user_table.c.name == 'spongebob')

The sections below will detail most major patterns and constructs that are
meaningful with :class:`_sql.Select` constructs.

.. seealso::

    :ref:`tutorial_selecting_data` - in the SQLAlchemy Tutorial

.. rst-class:: core-header

.. _queryguide_select_core_columns:

Core COLUMNS clause specification
---------------------------------

:func:`_sql.select` accepts any number of :class:`_sql.ColumnElement`
or :class:`_sql.FromClause` elements in order to produce the COLUMNS clase
to be SELECTed from.  The FROM clause is then inferred from the selectables
present here, as well as from those in the WHERE clause:

.. sourcecode:: pycon+sql

    {sql}>>> result = conn.execute(select(user_table.c.name, user_table.c.fullname))
    SELECT user_account.name, user_account.fullname
    FROM user_account
    [...] (){stop}
    >>> result.first()
    ('spongebob', 'Spongebob Squarepants')


Passing Tables vs. Columns
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Passing any :class:`_sql.FromClause`, most typically a :class:`_schema.Table`
object but also for objects such as :class:`_sql.Subquery` and :class:`_sql.Alias`,
the effect is that all column expressions are extracted in the order in which
they were first stated when the :class:`_sql.FromClause` was constructed, and added
to the :class:`_schema.Select` construct individually:

.. sourcecode:: pycon+sql

    {sql}>>> result = conn.execute(select(user_table).order_by(user_table.c.id))
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account ORDER BY user_account.id
    [...] (){stop}
    >>> result.first()
    (1, 'spongebob', 'Spongebob Squarepants')

.. rst-class:: orm-addin

Selecting from Labeled SQL Expressions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :meth:`_sql.ColumnElement.label` method as well as the same-named method
available on ORM attributes provides a SQL label of a column or expression,
allowing it to have a specific name in a result set.  This can be helpful
when referring to arbitrary SQL expressions in a result row by name:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import func, cast
    >>> stmt = (
    ...     select(
    ...         ("Username: " + user_table.c.name).label("username"),
    ...         ("Num Addresses: " + cast(func.count(address_table.c.id), String)).label("num_addresses")
    ...     )
    ...     .outerjoin(address_table).group_by(user_table.c.name).order_by(user_table.c.name)
    ... )
    {sql}>>> for row in conn.execute(stmt):
    ...     print(f"{row.username}: {row.num_addresses}")
    SELECT
        ? || user_account.name AS username,
        ? || CAST(count(address.id) AS VARCHAR) AS num_addresses
    FROM user_account
    LEFT OUTER JOIN address ON user_account.id = address.user_id
    GROUP BY user_account.name
    ORDER BY user_account.name
    [...] ('Username: ', 'Num Addresses: '){stop}
    Username: ehkrabs: Num Addresses: 0
    Username: patrick: Num Addresses: 1
    Username: sandy: Num Addresses: 2
    Username: spongebob: Num Addresses: 1
    Username: squidward: Num Addresses: 1

.. rst-class:: orm-addin

Adding additional column clause expressions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :class:`_sql.Select` construct allows for additional column expressions
to be added after iniial consruction using the :meth:`_sql.Select.add_columns`
method.   This method works like :func:`_sql.select` and
accepts any :class:`_sql.ColumnElement` or :class:`_sql.FromClause`, as well
as ORM-enabled objects:


.. sourcecode:: pycon+sql

    >>> stmt = select(user_table.c.name).where(user_table.c.name == 'squidward')
    >>> stmt = stmt.join(address_table).add_columns(address_table.c.email_address)
    {sql}>>> conn.execute(stmt).first()
    SELECT user_account.name, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    WHERE user_account.name = ?
    [...] ('squidward',){stop}
    ('squidward', 'stentcl@sqlalchemy.org')

Selecting from Aliases
^^^^^^^^^^^^^^^^^^^^^^

An "alias" means we are applying a name to a table in a SELECT statement
such that it is now referred to by that new name, rather than by its original
name.  This allows us to do things like SELECT from the same table more than
once, and also allows us to refer to subqueries by name.

An aliased :class:`_schema.Table` in Core is produced using the
:meth:`_schema.Table.alias` method, with or without an explicit name; an
explicit name is not generally needed as an anonymous name is applied
if one is not passed:

.. sourcecode:: pycon+sql

    >>> user_alias_1 = user_table.alias()
    >>> user_alias_2 = user_table.alias()
    >>> stmt = (
    ...     select(user_alias_1.c.name, user_alias_2.c.name).
    ...     join_from(user_alias_1, user_alias_2, user_alias_1.c.id > user_alias_2.c.id).
    ...     order_by(user_alias_1.c.name, user_alias_2.c.name)
    ... )
    {sql}>>> result = conn.execute(stmt)
    SELECT user_account_1.name, user_account_2.name
    FROM user_account AS user_account_1
    JOIN user_account AS user_account_2 ON user_account_1.id > user_account_2.id
    ORDER BY user_account_1.name, user_account_2.name
    [...] (){stop}
    >>> result.all()
    [('ehkrabs', 'patrick'), ('ehkrabs', 'sandy'), ('ehkrabs', 'spongebob'),
     ('ehkrabs', 'squidward'), ('patrick', 'sandy'), ('patrick', 'spongebob'),
     ('sandy', 'spongebob'), ('squidward', 'patrick'),
     ('squidward', 'sandy'), ('squidward', 'spongebob')]



.. rst-class:: orm-header

.. _queryguide_select_orm_columns:

ORM COLUMNS clause specification
---------------------------------

The :func:`_sql.select` construct also accepts ORM entities, including mapped
classes as well as class-level attributes representing mapped columns, which
are converted into ORM-annotated :class:`_sql.FromClause` and
:class:`_sql.ColumnElement` elements at construction time.

A :class:`_sql.Select` object that contains ORM-annotated entities is normally
executed using a :class:`_orm.Session` object, and not a :class:`_future.Connection`
object, so that ORM-related features may take effect.

Selecting ORM Entities
^^^^^^^^^^^^^^^^^^^^^^

Below we select from the ``User`` entity, producing a :class:`_sql.Select`
that selects from the mapped :class:`_schema.Table` to which ``User`` is mapped:

.. sourcecode:: pycon+sql

    {sql}>>> result = session.execute(select(User).order_by(User.id))
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account ORDER BY user_account.id
    [...] (){stop}

When selecting from ORM entities, the entity itself is returned in the result
as a single column value; for example above, the :class:`_engine.Result`
returns :class:`_engine.Row` objects that have just a single column, that column
holding onto a ``User`` object::

    >>> result.fetchone()
    (User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)

When selecting a list of single-column ORM entities, it is typical to skip
the generation of :class:`_engine.Row` objects and instead receive
ORM entities directly, which is achieved using the :meth:`_engine.Result.scalars`
method::

    >>> result.scalars().all()
    [User(id=2, name='sandy', fullname='Sandy Cheeks'),
     User(id=3, name='patrick', fullname='Patrick Star'),
     User(id=4, name='squidward', fullname='Squidward Tentacles'),
     User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')]

Selecting ORM Attributes
^^^^^^^^^^^^^^^^^^^^^^^^

The attributes on a mapped class, such as ``User.name`` and ``Address.email_address``,
have a similar behavior as that of the entity class itself such as ``User``
in that they are automatically converted into ORM-annotated Core objects
when passed to :func:`_sql.select`.   They may be used in the same way
as table columns are used:

.. sourcecode:: pycon+sql

    {sql}>>> result = session.execute(
    ...     select(User.name, Address.email_address).
    ...     join(User.addresses).
    ...     order_by(User.id, Address.id)
    ... )
    SELECT user_account.name, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    ORDER BY user_account.id, address.id
    [...] (){stop}

ORM attributes, themselves known as :class:`_orm.InstrumentedAttribute`
objects, can be used in the same way as any :class:`_sql.ColumnElement`,
and are delivered in result rows just the same way, such as below
where we refer to their values by column name within each row::

    >>> for row in result:
    ...     print(f"{row.name}  {row.email_address}")
    spongebob  spongebob@sqlalchemy.org
    sandy  sandy@sqlalchemy.org
    sandy  squirrel@squirrelpower.org
    patrick  pat999@aol.com
    squidward  stentcl@sqlalchemy.org


Selecting from Aliases
----------------------

(link to alias section below)



Selecting ORM Aliases
^^^^^^^^^^^^^^^^^^^^^

(link to alias section below)

ORM Select From Statement
^^^^^^^^^^^^^^^^^^^^^^^^^^

Selecting Bundles
^^^^^^^^^^^^^^^^^

Label Styles
^^^^^^^^^^^^



Getting data on columns
^^^^^^^^^^^^^^^^^^^^^^^

.selected_columns
~~~~~~~~~~~~~~~~~

.column_descriptions
~~~~~~~~~~~~~~~~~~~~~

WHERE Clause
------------

Mechanics of clause construction
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Operators
^^^^^^^^^

Using Conjunctions
^^^^^^^^^^^^^^^^^^

The most common conjunction, "AND", is automatically applied if we make repeated use of the :meth:`_sql.Select.where` method::

    >>> print(
    ...        select(address_table.c.email_address).
    ...        where(user_table.c.name == 'squidward').
    ...        where(address_table.c.user_id == user_table.c.id)
    ...    )
    SELECT address.email_address
    FROM address, user_account
    WHERE user_account.name = :name_1 AND address.user_id = user_account.id

:meth:`_sql.Select.where` also accepts multiple expressions with the same effect::

    >>> print(
    ...        select(address_table.c.email_address).
    ...        where(
    ...            user_table.c.name == 'squidward',
    ...            address_table.c.user_id == user_table.c.id
    ...        )
    ...    )
    SELECT address.email_address
    FROM address, user_account
    WHERE user_account.name = :name_1 AND address.user_id = user_account.id

The "AND" conjunction, as well as its partner "OR", are both available directly using the :func:`_sql.and_` and :func:`_sql.or_` functions::


    >>> from sqlalchemy import and_, or_
    >>> print(
    ...     select(address_table.c.email_address).
    ...     where(
    ...         and_(
    ...             or_(user_table.c.name == 'squidward', user_table.c.name == 'sandy'),
    ...             address_table.c.user_id == user_table.c.id
    ...         )
    ...     )
    ... )
    SELECT address.email_address
    FROM address, user_account
    WHERE (user_account.name = :name_1 OR user_account.name = :name_2)
    AND address.user_id = user_account.id



In Operations
^^^^^^^^^^^^^

list of values
~~~~~~~~~~~~~~~

IN select
~~~~~~~~~

tuple IN forms
~~~~~~~~~~~~~~


Datatype-specific Operators
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Operator Customization
^^^^^^^^^^^^^^^^^^^^^^

Conjunctions
^^^^^^^^^^^^


ORDER BY
--------

Ordering by label
^^^^^^^^^^^^^^^^^^

GROUP BY / HAVING
-----------------

Grouping by label
^^^^^^^^^^^^^^^^^^

Set Operations
---------------

union / union_all
^^^^^^^^^^^^^^^^^^

ordering unions
^^^^^^^^^^^^^^^^

exclude / intersection / etc
^^^^^^^^^^^^^^^^^^^^^^^^^^^^


LIMIT / OFFSET / FETCH
----------------------

Limit
^^^^^

Offset
^^^^^^

The FETCH clause
^^^^^^^^^^^^^^^^


Joins
-----

ORM Join forms
^^^^^^^^^^^^^^


Selecting with Aliases
-----------------------

ORM Alias forms
^^^^^^^^^^^^^^^

Selecting with Subuqeries
-------------------------

ORM Subquery forms (like from_Self equiv)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


Selecting with Scalar Subqueries
--------------------------------

Correlated Subqueries
^^^^^^^^^^^^^^^^^^^^^

LATERAL Correlation
^^^^^^^^^^^^^^^^^^^


Using Common Table Expressions (CTE)
------------------------------------

Explicit Bound Parameters
--------------------------

SQL Functions
-------------

Window Functions
----------------

Data Casts and Type Coercions
-----------------------------

Concurrency Control - WITH FOR UPDATE
-------------------------------------

Using Prefixes and Suffixes
----------------------------

Using Hints
------------

ORM Options
===========

Populate Existing
------------------

Autoflush
----------

Yield Per
----------

Loader Options
---------------

(a few lines, then link to relationship loading stuff)

.. _queryguide_insert:

INSERT Statements
===================

Sending params separately
--------------------------

Using values
------------

Using multi-values
-------------------

RETURNING
----------

INSERT from SELECT
------------------

INSERT w/ conflict resolution
------------------------------

PG ON Conflict
^^^^^^^^^^^^^^

brief one liner, link to PG docs

MySQL whatever they have
^^^^^^^^^^^^^^^^^^^^^^^^^

brief one liner, link to MySQL

SQLite whatever they have
^^^^^^^^^^^^^^^^^^^^^^^^^^^

brief one liner, link to SQLite docs


UPDATE statements
=================

SET clause
-----------

WHERE clause
--------------

link to SELECT where clause, it's the same

Matched Row Counts
------------------

Correlated Updates
---------------------

RETURNING
----------

Multi Table Updates
--------------------

Parameter Ordered Updates
-------------------------

DELETE statements
=================

Matched Row Counts
------------------

Multiple Table Deletes
-----------------------


Textual SQL
============

boudn parameter behaviors
-------------------------

result column behaviors
------------------------

text fragments in bigger statements
-----------------------------------




