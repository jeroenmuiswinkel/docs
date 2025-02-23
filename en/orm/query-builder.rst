Query Builder
#############


.. php:namespace:: Cake\ORM

.. php:class:: Query

The ORM's query builder provides a simple to use fluent interface for creating
and running queries. By composing queries together, you can create advanced
queries using unions and subqueries with ease.

Underneath the covers, the query builder uses PDO prepared statements which
protect against SQL injection attacks.

The Query Object
================

The easiest way to create a ``Query`` object is to use ``find()`` from a
``Table`` object. This method will return an incomplete query ready to be
modified. You can also use a table's connection object to access the lower level
Query builder that does not include ORM features, if necessary. See the
:ref:`database-queries` section for more information::

    use Cake\ORM\Locator\LocatorAwareTrait;

    $articles = $this->getTableLocator()->get('Articles');

    // Start a new query.
    $query = $articles->find();

When inside a controller, you can use the automatic table variable that is
created using the conventions system::

    // Inside ArticlesController.php

    $query = $this->Articles->find();

Selecting Rows From A Table
---------------------------

::

    use Cake\ORM\Locator\LocatorAwareTrait;

    $query = $this->getTableLocator()->get('Articles')->find();

    foreach ($query->all() as $article) {
        debug($article->title);
    }

For the remaining examples, assume that ``$articles`` is a
:php:class:`~Cake\\ORM\\Table`. When inside controllers, you can use
``$this->Articles`` instead of ``$articles``.

Almost every method in a ``Query`` object will return the same query, this means
that ``Query`` objects are lazy, and will not be executed unless you tell them
to::

    $query->where(['id' => 1]); // Return the same query object
    $query->order(['title' => 'DESC']); // Still same object, no SQL executed

You can of course chain the methods you call on Query objects::

    $query = $articles
        ->find()
        ->select(['id', 'name'])
        ->where(['id !=' => 1])
        ->order(['created' => 'DESC']);

    foreach ($query->all() as $article) {
        debug($article->created);
    }

If you try to call ``debug()`` on a Query object, you will see its internal
state and the SQL that will be executed in the database::

    debug($articles->find()->where(['id' => 1]));

    // Outputs
    // ...
    // 'sql' => 'SELECT * FROM articles where id = ?'
    // ...

You can execute a query directly without having to use ``foreach`` on it.
The easiest way is to either call the ``all()`` or ``toList()`` methods::

    $resultsIteratorObject = $articles
        ->find()
        ->where(['id >' => 1])
        ->all();

    foreach ($resultsIteratorObject as $article) {
        debug($article->id);
    }

    $resultsArray = $articles
        ->find()
        ->where(['id >' => 1])
        ->toList();

    foreach ($resultsArray as $article) {
        debug($article->id);
    }

    debug($resultsArray[0]->title);

In the above example, ``$resultsIteratorObject`` will be an instance of
``Cake\ORM\ResultSet``, an object you can iterate and apply several extracting
and traversing methods on.

Often, there is no need to call ``all()``, you can simply iterate the
Query object to get its results. Query objects can also be used directly as the
result object; trying to iterate the query, calling ``toList()`` or some of the
methods inherited from :doc:`Collection </core-libraries/collections>`, will
result in the query being executed and results returned to you.

Selecting A Single Row From A Table
-----------------------------------

You can use the ``first()`` method to get the first result in the query::

    $article = $articles
        ->find()
        ->where(['id' => 1])
        ->first();

    debug($article->title);

Getting A List Of Values From A Column
--------------------------------------

::

    // Use the extract() method from the collections library
    // This executes the query as well
    $allTitles = $articles->find()->all()->extract('title');

    foreach ($allTitles as $title) {
        echo $title;
    }

You can also get a key-value list out of a query result::

    $list = $articles->find('list')->all();
    foreach ($list as $id => $title) {
        echo "$id : $title"
    }

For more information on how to customize the fields used for populating the list
refer to :ref:`table-find-list` section.

Queries Are Collection Objects
------------------------------

Once you get familiar with the Query object methods, it is strongly encouraged
that you visit the :doc:`Collection </core-libraries/collections>` section to
improve your skills in efficiently traversing the results. Query results
implement the collection interface::

    // Use the combine() method from the collections library
    // This is equivalent to find('list')
    $keyValueList = $articles->find()->all()->combine('id', 'title');

    // An advanced example
    $results = $articles->find()
        ->where(['id >' => 1])
        ->order(['title' => 'DESC'])
        ->all()
        ->map(function ($row) {
            $row->trimmedTitle = trim($row->title);
            return $row;
        })
        ->combine('id', 'trimmedTitle') // combine() is another collection method
        ->toArray(); // Also a collections library method

    foreach ($results as $id => $trimmedTitle) {
        echo "$id : $trimmedTitle";
    }

Queries Are Lazily Evaluated
----------------------------

Query objects are lazily evaluated. This means a query is not executed until one
of the following things occur:

- The query is iterated with ``foreach``.
- The query's ``execute()`` method is called. This will return the underlying
  statement object, and is to be used with insert/update/delete queries.
- The query's ``first()`` method is called. This will return the first result in the set
  built by ``SELECT`` (it adds ``LIMIT 1`` to the query).
- The query's ``all()`` method is called. This will return the result set and
  can only be used with ``SELECT`` statements.
- The query's ``toList()`` or ``toArray()`` method is called.

Until one of these conditions are met, the query can be modified without additional
SQL being sent to the database. It also means that if a Query hasn't been
evaluated, no SQL is ever sent to the database. Once executed, modifying and
re-evaluating a query will result in additional SQL being run. Calling the same query without modification multiple times will return same reference.

If you want to take a look at what SQL CakePHP is generating, you can turn
database :ref:`query logging <database-query-logging>` on.

Selecting Data
==============

CakePHP makes building ``SELECT`` queries simple. To limit the fields fetched,
you can use the ``select()`` method::

    $query = $articles->find();
    $query->select(['id', 'title', 'body']);
    foreach ($query->all() as $row) {
        debug($row->title);
    }

You can set aliases for fields by providing fields as an associative array::

    // Results in SELECT id AS pk, title AS aliased_title, body ...
    $query = $articles->find();
    $query->select(['pk' => 'id', 'aliased_title' => 'title', 'body']);

To select distinct fields, you can use the ``distinct()`` method::

    // Results in SELECT DISTINCT country FROM ...
    $query = $articles->find();
    $query->select(['country'])
        ->distinct(['country']);

To set some basic conditions you can use the ``where()`` method::

    // Conditions are combined with AND
    $query = $articles->find();
    $query->where(['title' => 'First Post', 'published' => true]);

    // You can call where() multiple times
    $query = $articles->find();
    $query->where(['title' => 'First Post'])
        ->where(['published' => true]);

You can also pass an anonymous function to the ``where()`` method. The passed
anonymous function will receive an instance of
``\Cake\Database\Expression\QueryExpression`` as its first argument, and
``\Cake\ORM\Query`` as its second::

    $query = $articles->find();
    $query->where(function (QueryExpression $exp, Query $q) {
        return $exp->eq('published', true);
    });

See the :ref:`advanced-query-conditions` section to find out how to construct
more complex ``WHERE`` conditions.


Selecting Specific Fields
-------------------------

By default a query will select all fields from a table, the exception is when you
call the ``select()`` function yourself and pass certain fields::

    // Only select id and title from the articles table
    $articles->find()->select(['id', 'title']);

If you wish to still select all fields from a table after having called
``select($fields)``, you can pass the table instance to ``select()`` for this
purpose::

    // Only all fields from the articles table including
    // a calculated slug field.
    $query = $articlesTable->find();
    $query
        ->select(['slug' => $query->func()->concat(['title' => 'identifier', '-', 'id' => 'identifier'])])
        ->select($articlesTable); // Select all fields from articles

If you want to select all but a few fields on a table, you can use
``selectAllExcept()``::

    $query = $articlesTable->find();

    // Get all fields except the published field.
    $query->selectAllExcept($articlesTable, ['published']);

You can also pass an ``Association`` object when working with contained
associations.

.. _using-sql-functions:

Using SQL Functions
-------------------

CakePHP's ORM offers abstraction for some commonly used SQL functions. Using the
abstraction allows the ORM to select the platform specific implementation of the
function you want. For example, ``concat`` is implemented differently in MySQL,
PostgreSQL and SQL Server. Using the abstraction allows your code to be
portable::

    // Results in SELECT COUNT(*) count FROM ...
    $query = $articles->find();
    $query->select(['count' => $query->func()->count('*')]);

Note that most of the functions accept an additional argument to specify the types
to bind to the arguments and/or the return type, for example::

    $query->select(['minDate' => $query->func()->min('date', ['date']);

For details, see the documentation for :php:class:`Cake\\Database\\FunctionsBuilder`.

You can access existing wrappers for several SQL functions through ``Query::func()``:

``rand()``
    Generate a random value between 0 and 1 via SQL.
``sum()``
    Calculate a sum. `Assumes arguments are literal values.`
``avg()``
    Calculate an average. `Assumes arguments are literal values.`
``min()``
    Calculate the min of a column. `Assumes arguments are literal values.`
``max()``
    Calculate the max of a column. `Assumes arguments are literal values.`
``count()``
    Calculate the count. `Assumes arguments are literal values.`
``concat()``
    Concatenate two values together. `Assumes arguments are bound parameters.`
``coalesce()``
    Coalesce values. `Assumes arguments are bound parameters.`
``dateDiff()``
    Get the difference between two dates/times. `Assumes arguments are bound parameters.`
``now()``
    Defaults to returning date and time, but accepts 'time' or 'date' to return only
    those values.
``extract()``
    Returns the specified date part from the SQL expression.
``dateAdd()``
    Add the time unit to the date expression.
``dayOfWeek()``
    Returns a FunctionExpression representing a call to SQL WEEKDAY function.

Window-Only Functions
^^^^^^^^^^^^^^^^^^^^^

These window-only functions contain a window expression by default:

``rowNumber()``
    Returns an Aggregate expression for the ``ROW_NUMBER()`` SQL function.
``lag()``
    Returns an Aggregate expression for the ``LAG()`` SQL function.
``lead()``
    Returns an Aggregate expression for the ``LEAD()`` SQL function.

.. versionadded:: 4.1.0
    Window functions were added in 4.1.0

When providing arguments for SQL functions, there are two kinds of parameters
you can use, literal arguments and bound parameters. Identifier/Literal parameters allow
you to reference columns or other SQL literals. Bound parameters can be used to
safely add user data to SQL functions. For example::

    $query = $articles->find()->innerJoinWith('Categories');
    $concat = $query->func()->concat([
        'Articles.title' => 'identifier',
        ' - CAT: ',
        'Categories.name' => 'identifier',
        ' - Age: ',
        $query->func()->dateDiff(
            'NOW()' => 'literal',
            'Articles.created' => 'identifier'
        )
    ]);
    $query->select(['link_title' => $concat]);

Both ``literal`` and ``identifier`` arguments allow you to reference other columns
and SQL literals while ``identifier`` will be appropriately quoted if auto-quoting
is enabled.  If not marked as literal or identifier, arguments will be bound
parameters allowing you to safely pass user data to the function.

The above example generates something like this in MYSQL.


.. code-block:: mysql

    SELECT CONCAT(
        Articles.title,
        :c0,
        Categories.name,
        :c1,
        (DATEDIFF(NOW(), Articles.created))
    ) FROM articles;

The ``:c0`` argument will have ``' - CAT:'`` text bound when the query is
executed. The ``dateDiff`` expression was translated to the appropriate SQL.

Custom Functions
^^^^^^^^^^^^^^^^

If ``func()`` does not already wrap the SQL function you need, you can call
it directly through ``func()`` and still safely pass arguments and user data
as described. Make sure you pass the appropriate argument type for custom
functions or they will be treated as bound parameters::

    $query = $articles->find();
    $year = $query->func()->year([
        'created' => 'identifier'
    ]);
    $time = $query->func()->date_format([
        'created' => 'identifier',
        "'%H:%i'" => 'literal'
    ]);
    $query->select([
        'yearCreated' => $year,
        'timeCreated' => $time
    ]);

These custom function would generate something like this in MYSQL:

.. code-block:: mysql

    SELECT YEAR(created) as yearCreated,
           DATE_FORMAT(created, '%H:%i') as timeCreated
    FROM articles;

.. note::
    Use ``func()`` to pass untrusted user data to any SQL function.

Ordering Results
----------------

To apply ordering, you can use the ``order`` method::

    $query = $articles->find()
        ->order(['title' => 'ASC', 'id' => 'ASC']);

When calling ``order()`` multiple times on a query, multiple clauses will be
appended.  However, when using finders you may sometimes need to overwrite the
``ORDER BY``.  Set the second parameter of ``order()`` (as well as
``orderAsc()`` or ``orderDesc()``) to ``Query::OVERWRITE`` or to ``true``::

    $query = $articles->find()
        ->order(['title' => 'ASC']);
    // Later, overwrite the ORDER BY clause instead of appending to it.
    $query = $articles->find()
        ->order(['created' => 'DESC'], Query::OVERWRITE);

The ``orderAsc`` and ``orderDesc`` methods can be used when you need to sort on
complex expressions::

    $query = $articles->find();
    $concat = $query->func()->concat([
        'title' => 'identifier',
        'synopsis' => 'identifier'
    ]);
    $query->orderAsc($concat);

To build complex order clauses, use a Closure to build order expressions::

    $query->orderAsc(function (QueryExpression $exp, Query $query) {
        return $exp->addCase(...);
    });


Limiting Results
----------------

To limit the number of rows or set the row offset you can use the ``limit()``
and ``page()`` methods::

    // Fetch rows 50 to 100
    $query = $articles->find()
        ->limit(50)
        ->page(2);

As you can see from the examples above, all the methods that modify the query
provide a fluent interface, allowing you to build a query through chained method
calls.

Aggregates - Group and Having
-----------------------------

When using aggregate functions like ``count`` and ``sum`` you may want to use
``group by`` and ``having`` clauses::

    $query = $articles->find();
    $query->select([
        'count' => $query->func()->count('view_count'),
        'published_date' => 'DATE(created)'
    ])
    ->group('published_date')
    ->having(['count >' => 3]);

Case Statements
---------------

The ORM also offers the SQL ``case`` expression. The ``case`` expression allows
for implementing ``if ... then ... else`` logic inside your SQL. This can be useful
for reporting on data where you need to conditionally sum or count data, or where you
need to specific data based on a condition.

If we wished to know how many published articles are in our database, we could use the following SQL:

.. code-block:: sql

    SELECT
    COUNT(CASE WHEN published = 'Y' THEN 1 END) AS number_published,
    COUNT(CASE WHEN published = 'N' THEN 1 END) AS number_unpublished
    FROM articles

To do this with the query builder, we'd use the following code::

    $query = $articles->find();
    $publishedCase = $query->newExpr()
        ->case()
        ->when(['published' => 'Y'])
        ->then(1);
    $unpublishedCase = $query->newExpr()
        ->case()
        ->when(['published' => 'N'])
        ->then(1);

    $query->select([
        'number_published' => $query->func()->count($publishedCase),
        'number_unpublished' => $query->func()->count($unpublishedCase)
    ]);

The ``when()`` method accepts SQL snippets, array conditions, and ``Closure``
for when you need additional logic to build the cases. If we wanted to classify
cities into SMALL, MEDIUM, or LARGE based on population size, we could do the
following::

    $query = $cities->find();
    $sizing = $query->newExpr()->case()
        ->when(['population <' => 100000])
        ->then('SMALL')
        ->when($q->between('population', 100000, 999000))
        ->then('MEDIUM')
        ->when(['population >=' => 999001])
        ->then('LARGE');
    $query = $query->select(['size' => $sizing]);
    # SELECT CASE
    #   WHEN population < 100000 THEN 'SMALL'
    #   WHEN population BETWEEN 100000 AND 999000 THEN 'MEDIUM'
    #   WHEN population >= 999001 THEN 'LARGE'
    #   END AS size

You need to be careful when including user provided data into case expressions
as it can create SQL injection vulnerabilities::

    // Unsafe do *not* use
    $case->when($requestData['published']);

    // Instead pass user data as values to array conditions
    $case->when(['published' => $requestData['published']]);

For more complex scenarios you can use ``QueryExpression`` objects and bound
values::

    $userValue = $query->newExpr()
        ->case()
        ->when($query->newExpr('population >= :userData'))
        ->then(123, 'integer');

    $query->select(['val' => $userValue])
        ->bind(':userData', $requestData['value'], 'integer');

By using bindings you can safely embed user data into complex raw SQL snippets.

``then()``, ``when()`` and ``else()`` will try to infer the
value type based on the parameter type. If you need to bind a value as
a different type you can declare the desired type::

    $case->when(['published' => true])->then('1', 'integer');

You can create ``if ... then ... else`` conditions by using ``else()``::

    $published = $query->newExpr()
        ->case()
        ->when(['published' => true])
        ->then('Y');
        ->else('N');

    # CASE WHEN published = true THEN 'Y' ELSE 'N' END;

Also, it's possible to create the simple variant by passing a value to ``case()``::

    $published = $query->newExpr()
        ->case($query->identifier('published'))
        ->when(true)
        ->then('Y');
        ->else('N');

    # CASE published WHEN true THEN 'Y' ELSE 'N' END;

.. versionchanged:: 4.3.0
    The fluent ``case()`` builder method was added.

Prior to 4.3.0, you would need to use::

    $query = $articles->find();
    $publishedCase = $query->newExpr()
        ->addCase(
            $query->newExpr()->add(['published' => 'Y']),
            1,
            'integer'
        );
    $unpublishedCase = $query->newExpr()
        ->addCase(
            $query->newExpr()->add(['published' => 'N']),
            1,
            'integer'
        );

    $query->select([
        'number_published' => $query->func()->count($publishedCase),
        'number_unpublished' => $query->func()->count($unpublishedCase)
    ]);

The ``addCase`` function can also chain together multiple statements to create
``if .. then .. [elseif .. then .. ] [ .. else ]`` logic inside your SQL.

If we wanted to classify cities into SMALL, MEDIUM, or LARGE based on population
size, we could do the following::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->addCase(
                [
                    $q->newExpr()->lt('population', 100000),
                    $q->newExpr()->between('population', 100000, 999000),
                    $q->newExpr()->gte('population', 999001),
                ],
                ['SMALL',  'MEDIUM', 'LARGE'], # values matching conditions
                ['string', 'string', 'string'] # type of each value
            );
        });
    # WHERE CASE
    #   WHEN population < 100000 THEN 'SMALL'
    #   WHEN population BETWEEN 100000 AND 999000 THEN 'MEDIUM'
    #   WHEN population >= 999001 THEN 'LARGE'
    #   END

Any time there are fewer case conditions than values, ``addCase`` will
automatically produce an ``if .. then .. else`` statement::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->addCase(
                [
                    $q->newExpr()->eq('population', 0),
                ],
                ['DESERTED', 'INHABITED'], # values matching conditions
                ['string', 'string'] # type of each value
            );
        });
    # WHERE CASE
    #   WHEN population = 0 THEN 'DESERTED' ELSE 'INHABITED' END

Fetching Arrays Instead of Entities
-----------------------------------

While ORMs and object result sets are powerful, creating entities is sometimes
unnecessary. For example, when accessing aggregated data, building an Entity may
not make sense. The process of converting the database results to entities is
called hydration. If you wish to disable this process you can do this::

    $query = $articles->find();
    $query->enableHydration(false); // Results as arrays instead of entities
    $result = $query->toList(); // Execute the query and return the array

After executing those lines, your result should look similar to this::

    [
        ['id' => 1, 'title' => 'First Article', 'body' => 'Article 1 body' ...],
        ['id' => 2, 'title' => 'Second Article', 'body' => 'Article 2 body' ...],
        ...
    ]

.. _format-results:

Adding Calculated Fields
------------------------

After your queries, you may need to do some post-processing. If you need to add
a few calculated fields or derived data, you can use the ``formatResults()``
method. This is a lightweight way to map over the result sets. If you need more
control over the process, or want to reduce results you should use
the :ref:`Map/Reduce <map-reduce>` feature instead. If you were querying a list
of people, you could calculate their age with a result formatter::

    // Assuming we have built the fields, conditions and containments.
    $query->formatResults(function (\Cake\Collection\CollectionInterface $results) {
        return $results->map(function ($row) {
            $row['age'] = $row['birth_date']->diff(new \DateTime)->y;
            return $row;
        });
    });

As you can see in the example above, formatting callbacks will get a
``ResultSetDecorator`` as their first argument. The second argument will be
the Query instance the formatter was attached to. The ``$results`` argument can
be traversed and modified as necessary.

Result formatters are required to return an iterator object, which will be used
as the return value for the query. Formatter functions are applied after all the
Map/Reduce routines have been executed. Result formatters can be applied from
within contained associations as well. CakePHP will ensure that your formatters
are properly scoped. For example, doing the following would work as you may
expect::

    // In a method in the Articles table
    $query->contain(['Authors' => function ($q) {
        return $q->formatResults(function (\Cake\Collection\CollectionInterface $authors) {
            return $authors->map(function ($author) {
                $author['age'] = $author['birth_date']->diff(new \DateTime)->y;
                return $author;
            });
        });
    }]);

    // Get results
    $results = $query->all();

    // Outputs 29
    echo $results->first()->author->age;

As seen above, the formatters attached to associated query builders are scoped
to operate only on the data in the association. CakePHP will ensure that
computed values are inserted into the correct entity.

.. _advanced-query-conditions:

Advanced Conditions
===================

The query builder makes it simple to build complex ``where`` clauses.
Grouped conditions can be expressed by providing combining ``where()`` and
expression objects. For simple queries, you can build conditions using
an array of conditions::

    $query = $articles->find()
        ->where([
            'author_id' => 3,
            'OR' => [['view_count' => 2], ['view_count' => 3]],
        ]);

The above would generate SQL like

.. code-block:: sql

    SELECT * FROM articles WHERE author_id = 3 AND (view_count = 2 OR view_count = 3)

If you'd prefer to avoid deeply nested arrays, you can use the callback form of
``where()`` to build your queries. The callback accepts a QueryExpression which allows
you to use the expression builder interface to build more complex conditions without arrays.
For example::

    $query = $articles->find()->where(function (QueryExpression $exp, Query $query) {
        // Use add() to add multiple conditions for the same field.
        $author = $query->newExpr()->or(['author_id' => 3])->add(['author_id' => 2]);
        $published = $query->newExpr()->and(['published' => true, 'view_count' => 10]);

        return $exp->or([
            'promoted' => true,
            $query->newExpr()->and([$author, $published])
        ]);
    });

The above generates SQL similar to:

.. code-block:: sql

    SELECT *
    FROM articles
    WHERE (
        (
            (author_id = 2 OR author_id = 3)
            AND
            (published = 1 AND view_count = 10)
        )
        OR promoted = 1
    )

The ``QueryExpression`` passed to the callback allows you to use both
**combinators** and **conditions** to build the full expression.

Combinators
    These create new ``QueryExpression`` objects and set how the conditions added
    to that expression are joined together.

    - ``and()`` creates new expression objects that joins all conditions with ``AND``.
    - ``or()``  creates new expression objects that joins all conditions with ``OR``.

Conditions
    These are added to the expression and automatically joined together
    depending on which combinator was used.

The ``QueryExpression`` passed to the callback function defaults to ``and()``::

    $query = $articles->find()
        ->where(function (QueryExpression $exp) {
            return $exp
                ->eq('author_id', 2)
                ->eq('published', true)
                ->notEq('spam', true)
                ->gt('view_count', 10);
        });

Since we started off using ``where()``, we don't need to call ``and()``, as
that happens implicitly. The above shows a few new condition
methods being combined with ``AND``. The resulting SQL would look like:

.. code-block:: sql

    SELECT *
    FROM articles
    WHERE (
    author_id = 2
    AND published = 1
    AND spam != 1
    AND view_count > 10)

However, if we wanted to use both ``AND`` & ``OR`` conditions we could do the
following::

    $query = $articles->find()
        ->where(function (QueryExpression $exp) {
            $orConditions = $exp->or(['author_id' => 2])
                ->eq('author_id', 5);
            return $exp
                ->add($orConditions)
                ->eq('published', true)
                ->gte('view_count', 10);
        });

Which would generate the SQL similar to:

.. code-block:: sql

    SELECT *
    FROM articles
    WHERE (
        (author_id = 2 OR author_id = 5)
        AND published = 1
        AND view_count >= 10
    )

The **combinators**  also allow you pass in a callback which takes
the new expression object as a parameter if you want to separate
the method chaining::

    $query = $articles->find()
        ->where(function (QueryExpression $exp) {
            $orConditions = $exp->or(function (QueryExpression $or) {
                return $or->eq('author_id', 2)
                    ->eq('author_id', 5);
            });
            return $exp
                ->not($orConditions)
                ->lte('view_count', 10);
        });

You can negate sub-expressions using ``not()``::

    $query = $articles->find()
        ->where(function (QueryExpression $exp) {
            $orConditions = $exp->or(['author_id' => 2])
                ->eq('author_id', 5);
            return $exp
                ->not($orConditions)
                ->lte('view_count', 10);
        });

Which will generate the following SQL looking like:

.. code-block:: sql

    SELECT *
    FROM articles
    WHERE (
        NOT (author_id = 2 OR author_id = 5)
        AND view_count <= 10
    )

It is also possible to build expressions using SQL functions::

    $query = $articles->find()
        ->where(function (QueryExpression $exp, Query $q) {
            $year = $q->func()->year([
                'created' => 'identifier'
            ]);
            return $exp
                ->gte($year, 2014)
                ->eq('published', true);
        });

Which will generate the following SQL looking like:

.. code-block:: sql

    SELECT *
    FROM articles
    WHERE (
        YEAR(created) >= 2014
        AND published = 1
    )

When using the expression objects you can use the following methods to create
conditions:

- ``eq()`` Creates an equality condition::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->eq('population', '10000');
        });
    # WHERE population = 10000

- ``notEq()`` Creates an inequality condition::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->notEq('population', '10000');
        });
    # WHERE population != 10000

- ``like()`` Creates a condition using the ``LIKE`` operator::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->like('name', '%A%');
        });
    # WHERE name LIKE "%A%"

- ``notLike()`` Creates a negated ``LIKE`` condition::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->notLike('name', '%A%');
        });
    # WHERE name NOT LIKE "%A%"

- ``in()`` Create a condition using ``IN``::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->in('country_id', ['AFG', 'USA', 'EST']);
        });
    # WHERE country_id IN ('AFG', 'USA', 'EST')

- ``notIn()`` Create a negated condition using ``IN``::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->notIn('country_id', ['AFG', 'USA', 'EST']);
        });
    # WHERE country_id NOT IN ('AFG', 'USA', 'EST')

- ``gt()`` Create a ``>`` condition::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->gt('population', '10000');
        });
    # WHERE population > 10000

- ``gte()`` Create a ``>=`` condition::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->gte('population', '10000');
        });
    # WHERE population >= 10000

- ``lt()`` Create a ``<`` condition::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->lt('population', '10000');
        });
    # WHERE population < 10000

- ``lte()`` Create a ``<=`` condition::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->lte('population', '10000');
        });
    # WHERE population <= 10000

- ``isNull()`` Create an ``IS NULL`` condition::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->isNull('population');
        });
    # WHERE (population) IS NULL

- ``isNotNull()`` Create a negated ``IS NULL`` condition::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->isNotNull('population');
        });
    # WHERE (population) IS NOT NULL

- ``between()`` Create a ``BETWEEN`` condition::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->between('population', 999, 5000000);
        });
    # WHERE population BETWEEN 999 AND 5000000,

- ``exists()`` Create a condition using ``EXISTS``::

    $subquery = $cities->find()
        ->select(['id'])
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->equalFields('countries.id', 'cities.country_id');
        })
        ->andWhere(['population >' => 5000000]);

    $query = $countries->find()
        ->where(function (QueryExpression $exp, Query $q) use ($subquery) {
            return $exp->exists($subquery);
        });
    # WHERE EXISTS (SELECT id FROM cities WHERE countries.id = cities.country_id AND population > 5000000)

- ``notExists()`` Create a negated condition using ``EXISTS``::

    $subquery = $cities->find()
        ->select(['id'])
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->equalFields('countries.id', 'cities.country_id');
        })
        ->andWhere(['population >' => 5000000]);

    $query = $countries->find()
        ->where(function (QueryExpression $exp, Query $q) use ($subquery) {
            return $exp->notExists($subquery);
        });
    # WHERE NOT EXISTS (SELECT id FROM cities WHERE countries.id = cities.country_id AND population > 5000000)

Expression objects should cover many commonly used functions and expressions. If
you find yourself unable to create the required conditions with expressions you
can may be able to use ``bind()`` to manually bind parameters into conditions::

    $query = $cities->find()
        ->where([
            'start_date BETWEEN :start AND :end'
        ])
        ->bind(':start', '2014-01-01', 'date')
        ->bind(':end',   '2014-12-31', 'date');

In situations when you can't get, or don't want to use the builder methods to
create the conditions you want you can also use snippets of SQL in where
clauses::

    // Compare two fields to each other
    $query->where(['Categories.parent_id != Parents.id']);

.. warning::

    The field names used in expressions, and SQL snippets should **never**
    contain untrusted content as you will create SQL Injection vectors. See the
    :ref:`using-sql-functions` section for how to safely include unsafe data
    into function calls.

Using Identifiers in Expressions
--------------------------------

When you need to reference a column or SQL identifier in your queries you can
use the ``identifier()`` method::

    $query = $countries->find();
    $query->select([
            'year' => $query->func()->year([$query->identifier('created')])
        ])
        ->where(function ($exp, $query) {
            return $exp->gt('population', 100000);
        });

.. warning::

    To prevent SQL injections, Identifier expressions should never have
    untrusted data passed into them.

Automatically Creating IN Clauses
---------------------------------

When building queries using the ORM, you will generally not have to indicate the
data types of the columns you are interacting with, as CakePHP can infer the
types based on the schema data. If in your queries you'd like CakePHP to
automatically convert equality to ``IN`` comparisons, you'll need to indicate
the column data type::

    $query = $articles->find()
        ->where(['id' => $ids], ['id' => 'integer[]']);

    // Or include IN to automatically cast to an array.
    $query = $articles->find()
        ->where(['id IN' => $ids]);

The above will automatically create ``id IN (...)`` instead of ``id = ?``. This
can be useful when you do not know whether you will get a scalar or array of
parameters. The ``[]`` suffix on any data type name indicates to the query
builder that you want the data handled as an array. If the data is not an array,
it will first be cast to an array. After that, each value in the array will
be cast using the :ref:`type system <database-data-types>`. This works with
complex types as well. For example, you could take a list of DateTime objects
using::

    $query = $articles->find()
        ->where(['post_date' => $dates], ['post_date' => 'date[]']);

Automatic IS NULL Creation
--------------------------

When a condition value is expected to be ``null`` or any other value, you can
use the ``IS`` operator to automatically create the correct expression::

    $query = $categories->find()
        ->where(['parent_id IS' => $parentId]);

The above will create ``parent_id` = :c1`` or ``parent_id IS NULL`` depending on
the type of ``$parentId``

Automatic IS NOT NULL Creation
------------------------------

When a condition value is expected not to be ``null`` or any other value, you
can use the ``IS NOT`` operator to automatically create the correct expression::

    $query = $categories->find()
        ->where(['parent_id IS NOT' => $parentId]);

The above will create ``parent_id` != :c1`` or ``parent_id IS NOT NULL``
depending on the type of ``$parentId``


Raw Expressions
---------------

When you cannot construct the SQL you need using the query builder, you can use
expression objects to add snippets of SQL to your queries::

    $query = $articles->find();
    $expr = $query->newExpr()->add('1 + 1');
    $query->select(['two' => $expr]);

``Expression`` objects can be used with any query builder methods like
``where()``, ``limit()``, ``group()``, ``select()`` and many other methods.

.. warning::

    Using expression objects leaves you vulnerable to SQL injection. You should
    never use untrusted data into expressions.

Getting Results
===============

Once you've made your query, you'll want to retrieve rows from it. There are
a few ways of doing this::

    // Iterate the query
    foreach ($query as $row) {
        // Do stuff.
    }

    // Get the results
    $results = $query->all();

You can use :doc:`any of the collection </core-libraries/collections>` methods
on your query objects to pre-process or transform the results::

    // Use one of the collection methods.
    $ids = $query->map(function ($row) {
        return $row->id;
    });

    $maxAge = $query->max(function ($max) {
        return $max->age;
    });

You can use ``first`` or ``firstOrFail`` to retrieve a single record. These
methods will alter the query adding a ``LIMIT 1`` clause::

    // Get just the first row
    $row = $query->first();

    // Get the first row or an exception.
    $row = $query->firstOrFail();

.. _query-count:

Returning the Total Count of Records
------------------------------------

Using a single query object, it is possible to obtain the total number of rows
found for a set of conditions::

    $total = $articles->find()->where(['is_active' => true])->count();

The ``count()`` method will ignore the ``limit``, ``offset`` and ``page``
clauses, thus the following will return the same result::

    $total = $articles->find()->where(['is_active' => true])->limit(10)->count();

This is useful when you need to know the total result set size in advance,
without having to construct another ``Query`` object. Likewise, all result
formatting and map-reduce routines are ignored when using the ``count()``
method.

Moreover, it is possible to return the total count for a query containing group
by clauses without having to rewrite the query in any way. For example, consider
this query for retrieving article ids and their comments count::

    $query = $articles->find();
    $query->select(['Articles.id', $query->func()->count('Comments.id')])
        ->matching('Comments')
        ->group(['Articles.id']);
    $total = $query->count();

After counting, the query can still be used for fetching the associated
records::

    $list = $query->all();

Sometimes, you may want to provide an alternate method for counting the total
records of a query. One common use case for this is providing
a cached value or an estimate of the total rows, or to alter the query to remove
expensive unneeded parts such as left joins. This becomes particularly handy
when using the CakePHP built-in pagination system which calls the ``count()``
method::

    $query = $query->where(['is_active' => true])->counter(function ($query) {
        return 100000;
    });
    $query->count(); // Returns 100000

In the example above, when the pagination component calls the count method, it
will receive the estimated hard-coded number of rows.

.. _caching-query-results:

Caching Loaded Results
----------------------

When fetching entities that don't change often you may want to cache the
results. The ``Query`` class makes this simple::

    $query->cache('recent_articles');

Will enable caching on the query's result set. If only one argument is provided
to ``cache()`` then the 'default' cache configuration will be used. You can
control which caching configuration is used with the second parameter::

    // String config name.
    $query->cache('recent_articles', 'dbResults');

    // Instance of CacheEngine
    $query->cache('recent_articles', $memcache);

In addition to supporting static keys, the ``cache()`` method accepts a function
to generate the key. The function you give it will receive the query as an
argument. You can then read aspects of the query to dynamically generate the
cache key::

    // Generate a key based on a simple checksum
    // of the query's where clause
    $query->cache(function ($q) {
        return 'articles-' . md5(serialize($q->clause('where')));
    });

The cache method makes it simple to add cached results to your custom finders or
through event listeners.

When the results for a cached query are fetched the following happens:

1. If the query has results set, those will be returned.
2. The cache key will be resolved and cache data will be read. If the cache data
   is not empty, those results will be returned.
3. If the cache misses, the query will be executed, the ``Model.beforeFind`` event
   will be triggered, and a new ``ResultSet`` will be created. This
   ``ResultSet`` will be written to the cache and returned.

.. note::

    You cannot cache a streaming query result.

Loading Associations
====================

The builder can help you retrieve data from multiple tables at the same time
with the minimum amount of queries possible. To be able to fetch associated
data, you first need to setup associations between the tables as described in
the :doc:`/orm/associations` section. This technique of combining queries
to fetch associated data from other tables is called **eager loading**.

.. include:: ./retrieving-data-and-resultsets.rst
    :start-after: start-contain
    :end-before: end-contain

Filtering by Associated Data
----------------------------

.. include:: ./retrieving-data-and-resultsets.rst
    :start-after: start-filtering
    :end-before: end-filtering

.. _adding-joins:

Adding Joins
------------

In addition to loading related data with ``contain()``, you can also add
additional joins with the query builder::

    $query = $articles->find()
        ->join([
            'table' => 'comments',
            'alias' => 'c',
            'type' => 'LEFT',
            'conditions' => 'c.article_id = articles.id',
        ]);

You can append multiple joins at the same time by passing an associative array
with multiple joins::

    $query = $articles->find()
        ->join([
            'c' => [
                'table' => 'comments',
                'type' => 'LEFT',
                'conditions' => 'c.article_id = articles.id',
            ],
            'u' => [
                'table' => 'users',
                'type' => 'INNER',
                'conditions' => 'u.id = articles.user_id',
            ]
        ]);

As seen above, when adding joins the alias can be the outer array key. Join
conditions can also be expressed as an array of conditions::

    $query = $articles->find()
        ->join([
            'c' => [
                'table' => 'comments',
                'type' => 'LEFT',
                'conditions' => [
                    'c.created >' => new DateTime('-5 days'),
                    'c.moderated' => true,
                    'c.article_id = articles.id'
                ]
            ],
        ], ['c.created' => 'datetime', 'c.moderated' => 'boolean']);

When creating joins by hand and using array based conditions, you need to
provide the datatypes for each column in the join conditions. By providing
datatypes for the join conditions, the ORM can correctly convert data types into
SQL. In addition to ``join()`` you can use ``rightJoin()``, ``leftJoin()`` and
``innerJoin()`` to create joins::

    // Join with an alias and string conditions
    $query = $articles->find();
    $query->leftJoin(
        ['Authors' => 'authors'],
        ['Authors.id = Articles.author_id']);

    // Join with an alias, array conditions, and types
    $query = $articles->find();
    $query->innerJoin(
        ['Authors' => 'authors'],
        [
        'Authors.promoted' => true,
        'Authors.created' => new DateTime('-5 days'),
        'Authors.id = Articles.author_id'
        ],
        ['Authors.promoted' => 'boolean', 'Authors.created' => 'datetime']);

It should be noted that if you set the ``quoteIdentifiers`` option to ``true`` when
defining your ``Connection``, join conditions between table fields should be set as follow::

    $query = $articles->find()
        ->join([
            'c' => [
                'table' => 'comments',
                'type' => 'LEFT',
                'conditions' => [
                    'c.article_id' => new \Cake\Database\Expression\IdentifierExpression('articles.id')
                ]
            ],
        ]);

This ensures that all of your identifiers will be quoted across the Query, avoiding errors with
some database Drivers (PostgreSQL notably)

Inserting Data
==============

Unlike earlier examples, you should not use ``find()`` to create insert queries.
Instead, create a new ``Query`` object using ``query()``::

    $query = $articles->query();
    $query->insert(['title', 'body'])
        ->values([
            'title' => 'First post',
            'body' => 'Some body text'
        ])
        ->execute();

To insert multiple rows with only one query, you can chain the ``values()``
method as many times as you need::

    $query = $articles->query();
    $query->insert(['title', 'body'])
        ->values([
            'title' => 'First post',
            'body' => 'Some body text'
        ])
        ->values([
            'title' => 'Second post',
            'body' => 'Another body text'
        ])
        ->execute();

Generally, it is easier to insert data using entities and
:php:meth:`~Cake\\ORM\\Table::save()`. By composing a ``SELECT`` and
``INSERT`` query together, you can create ``INSERT INTO ... SELECT`` style
queries::

    $select = $articles->find()
        ->select(['title', 'body', 'published'])
        ->where(['id' => 3]);

    $query = $articles->query()
        ->insert(['title', 'body', 'published'])
        ->values($select)
        ->execute();

.. note::
    Inserting records with the query builder will not trigger events such as
    ``Model.afterSave``. Instead you should use the :doc:`ORM to save
    data </orm/saving-data>`.

.. _query-builder-updating-data:

Updating Data
=============

As with insert queries, you should not use ``find()`` to create update queries.
Instead, create new a ``Query`` object using ``query()``::

    $query = $articles->query();
    $query->update()
        ->set(['published' => true])
        ->where(['id' => $id])
        ->execute();

Generally, it is easier to update data using entities and
:php:meth:`~Cake\\ORM\\Table::patchEntity()`.

.. note::
    Updating records with the query builder will not trigger events such as
    ``Model.afterSave``. Instead you should use the :doc:`ORM to save
    data </orm/saving-data>`.

Deleting Data
=============

As with insert queries, you should not use ``find()`` to create delete queries.
Instead, create new a query object using ``query()``::

    $query = $articles->query();
    $query->delete()
        ->where(['id' => $id])
        ->execute();

Generally, it is easier to delete data using entities and
:php:meth:`~Cake\\ORM\\Table::delete()`.

SQL Injection Prevention
========================

While the ORM and database abstraction layers prevent most SQL injections
issues, it is still possible to leave yourself vulnerable through improper use.

When using condition arrays, the key/left-hand side as well as single value
entries must not contain user data::

    $query->where([
        // Data on the key/left-hand side is unsafe, as it will be
        // inserted into the generated query as-is
        $userData => $value,

        // The same applies to single value entries, they are not
        // safe to use with user data in any form
        $userData,
        "MATCH (comment) AGAINST ($userData)",
        'created < NOW() - ' . $userData
    ]);

When using the expression builder, column names must not contain user data::

    $query->where(function (QueryExpression $exp) use ($userData, $values) {
        // Column names in all expressions are not safe.
        return $exp->in($userData, $values);
    });

When building function expressions, function names should never contain user
data::

    // Not safe.
    $query->func()->{$userData}($arg1);

    // Also not safe to use an array of
    // user data in a function expression
    $query->func()->coalesce($userData);

Raw expressions are never safe::

    $expr = $query->newExpr()->add($userData);
    $query->select(['two' => $expr]);

Binding values
--------------

It is possible to protect against many unsafe situations by using bindings.
Similar to :ref:`binding values to prepared statements <database-basics-binding-values>`,
values can be bound to queries using the :php:meth:`Cake\\Database\\Query::bind()`
method.

The following example would be a safe variant of the unsafe, SQL injection prone
example given above::

    $query
        ->where([
            'MATCH (comment) AGAINST (:userData)',
            'created < NOW() - :moreUserData'
        ])
        ->bind(':userData', $userData, 'string')
        ->bind(':moreUserData', $moreUserData, 'datetime');

.. note::

    Unlike :php:meth:`Cake\\Database\\StatementInterface::bindValue()`,
    ``Query::bind()`` requires to pass the named placeholders including the
    colon!

More Complex Queries
====================

If your application requires using more complex queries, you can express many
complex queries using the ORM query builder.

Unions
------

Unions are created by composing one or more select queries together::

    $inReview = $articles->find()
        ->where(['need_review' => true]);

    $unpublished = $articles->find()
        ->where(['published' => false]);

    $unpublished->union($inReview);

You can create ``UNION ALL`` queries using the ``unionAll()`` method::

    $inReview = $articles->find()
        ->where(['need_review' => true]);

    $unpublished = $articles->find()
        ->where(['published' => false]);

    $unpublished->unionAll($inReview);

Subqueries
----------

Subqueries enable you to compose queries together and build conditions and
results based on the results of other queries::

    $matchingComment = $articles->getAssociation('Comments')->find()
        ->select(['article_id'])
        ->distinct()
        ->where(['comment LIKE' => '%CakePHP%']);

    $query = $articles->find()
        ->where(['id IN' => $matchingComment]);

Subqueries are accepted anywhere a query expression can be used. For example, in
the ``select()`` and ``join()`` methods. The above example uses a standard
``Orm\Query`` object that will generate aliases, these aliases can make
referencing results in the outer query more complex. As of 4.2.0 you can use
``Table::subquery()`` to create a specialized query instance that will not
generate aliases::

    $comments = $articles->getAssociation('Comments')->getTarget();

    $matchingComment = $comments->subquery()
        ->select(['article_id'])
        ->distinct()
        ->where(['comment LIKE' => '%CakePHP%']);

    $query = $articles->find()
        ->where(['id IN' => $matchingComment]);

.. versionadded:: 4.2.0
    ``Table::subquery()`` and ``Query::subquery()`` were added.

Adding Locking Statements
-------------------------

Most relational database vendors support taking out locks when doing select
operations. You can use the ``epilog()`` method for this::

    // In MySQL
    $query->epilog('FOR UPDATE');

The ``epilog()`` method allows you to append raw SQL to the end of queries. You
should never put raw user data into ``epilog()``.

Window Functions
----------------

Window functions allow you to perform calculations using rows related to the
current row. They are commonly used to calculate totals or offsets on partial sets of rows
in the query. For example if we wanted to find the date of the earliest and latest comment on
each article we could use window functions::

    $query = $articles->find();
    $query->select([
        'Articles.id',
        'Articles.title',
        'Articles.user_id'
        'oldest_comment' => $query->func()
            ->min('Comments.created')
            ->partition('Comments.article_id'),
        'latest_comment' => $query->func()
            ->max('Comments.created')
            ->partition('Comments.article_id'),
    ])
    ->innerJoinWith('Comments');

The above would generate SQL similar to:

.. code-block:: sql

    SELECT
        Articles.id,
        Articles.title,
        Articles.user_id
        MIN(Comments.created) OVER (PARTITION BY Comments.article_id) AS oldest_comment,
        MAX(Comments.created) OVER (PARTITION BY Comments.article_id) AS latest_comment,
    FROM articles AS Articles
    INNER JOIN comments AS Comments

Window expressions can be applied to most aggregate functions. Any aggregate function
that cake abstracts with a wrapper in ``FunctionsBuilder`` will return an ``AggregateExpression``
which lets you attach window expressions. You can create custom aggregate functions
through ``FunctionsBuilder::aggregate()``.

These are the most commonly supported window features. Most features are provided
by ``AggregateExpresion``, but make sure you follow your database documentation on use and restrictions.

- ``order($fields)`` Order the aggregate group the same as a query ORDER BY.
- ``partition($expressions)`` Add one or more partitions to the window based on column
  names.
- ``rows($start, $end)`` Define a offset of rows that precede and/or follow the
  current row that should be included in the aggregate function.
- ``range($start, $end)`` Define a range of row values that precede and/or follow
  the current row that should be included in the aggregate function. This
  evaluates values based on the ``order()`` field.

If you need to re-use the same window expression multiple times you can create
named windows using the ``window()`` method::

    $query = $articles->find();

    // Define a named window
    $query->window('related_article', function ($window, $query) {
        $window->partition('Comments.article_id');

        return $window;
    });

    $query->select([
        'Articles.id',
        'Articles.title',
        'Articles.user_id'
        'oldest_comment' => $query->func()
            ->min('Comments.created')
            ->over('related_article'),
        'latest_comment' => $query->func()
            ->max('Comments.created')
            ->over('related_article'),
    ]);

.. versionadded:: 4.1.0
    Window function support was added.

Common Table Expressions
------------------------

Common Table Expressions or CTE are useful when building reporting queries where
you need to compose the results of several smaller query results together. They
can serve a similar purpose to database views or subquery results. Common Table
Expressions differ from derived tables and views in a couple ways:

#. Unlike views, you don't have to maintain schema for common table expressions.
   The schema is implicitly based on the result set of the table expression.
#. You can reference the results of a common table expression multiple times
   without incurring performance penalties unlike subquery joins.

As an example lets fetch a list of customers and the number of orders each of
them has made. In SQL we would use:

.. code-block:: sql

    WITH orders_per_customer AS (
        SELECT COUNT(*) AS order_count, customer_id FROM orders GROUP BY customer_id
    )
    SELECT name, orders_per_customer.order_count
    FROM customers
    INNER JOIN orders_per_customer ON orders_per_customer.customer_id = customers.id

To build that query with the ORM query builder we would use::

    // Start the final query
    $query = $this->Customers->find();

    // Attach a common table expression
    $query->with(function ($cte) {
        // Create a subquery to use in our table expression
        $q = $this->Orders->subquery();
        $q->select([
            'order_count' => $q->func()->count('*'),
            'customer_id'
        ])
        ->group('customer_id');

        // Attach the new query to the table expression
        return $cte
            ->name('orders_per_customer')
            ->query($q);
    });

    // Finish building the final query
    $query->select([
        'name',
        'order_count' => 'orders_per_customer.order_count',
    ])
    ->join([
        // Define the join with our table expression
        'orders_per_customer' => [
            'table' => 'orders_per_customer',
            'conditions' => 'orders_per_customer.customer_id = Customers.id'
        ]
    ]);

.. versionadded:: 4.1.0
    Common table expression support was added.

Executing Complex Queries
-------------------------

While the query builder makes most queries possible through builder methods,
very complex queries can be tedious and complicated to build. You may want to
:ref:`execute the desired SQL directly <running-select-statements>`.

Executing SQL directly allows you to fine tune the query that will be run.
However, doing so doesn't let you use ``contain`` or other higher level ORM
features.
