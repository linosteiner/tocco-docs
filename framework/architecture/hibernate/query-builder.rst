.. _query_builder:

Query Builder
=============

There are currently 4 different query builders for different query types:

    * The :nice:`EntityQueryBuilderImpl <ch/tocco/nice2/persist/hibernate/query/EntityQueryBuilderImpl>` builds queries that
      return :nice:`Entity <ch/tocco/nice2/persist/entity/Entity>` instances. It is convenient to use the :nice:`Entity <ch/tocco/nice2/persist/entity/Entity>`
      interface, but it might have a performance impact when too much data is loaded or too many queries are executed.
    * The :nice:`SinglePathQueryBuilderImpl <ch/tocco/nice2/persist/hibernate/query/SinglePathQueryBuilderImpl>` can be used to
      fetch exactly one property or path of an entity (for example to get all primary keys of a given query). This avoids
      unnecessarily loading the entire :nice:`Entity <ch/tocco/nice2/persist/entity/Entity>`.
    * The :nice:`PathQueryBuilderImpl <ch/tocco/nice2/persist/hibernate/query/PathQueryBuilderImpl>` is similar to the :nice:`SinglePathQueryBuilderImpl <ch/tocco/nice2/persist/hibernate/query/SinglePathQueryBuilderImpl>`
      but can select more than one path and always returns a container object (currently ``Object[]`` and ``Map`` are supported)  as a result.
    * The :nice:`CriteriaCountQueryBuilder <ch/tocco/nice2/persist/hibernate/query/CriteriaCountQueryBuilder>` can be
      used to execute ``COUNT`` queries.

In addition there is the :nice:`SubqueryBuilderImpl <ch/tocco/nice2/persist/hibernate/query/SubqueryBuilderImpl>` which is used
to create sub-queries. An instance of this can be acquired from the :nice:`SubqueryFactory <ch/tocco/nice2/persist/hibernate/query/PredicateBuilder.SubqueryFactory>`
of another query builder.

Internally, JPA Criteria Queries are used. The reason for the query builder
classes is that the user should not have access to the :java-hibernate:`Session <org/hibernate/Session>` object to make
sure that all query interceptors (security and more) are always applied.

Parts of the JPA Criteria API can still be used however, for example to specify conditions.
A tutorial can be found here: https://www.ibm.com/developerworks/java/library/j-typesafejpa/

.. note::
    The metamodel classes are currently not available, which means typesafe queries are not possible
    at the moment.

All query builders can be obtained from the :nice:`PersistenceService <ch/tocco/nice2/persist/hibernate/PersistenceService>`.
In addition the :nice:`QueryBuilderFactory <ch/tocco/nice2/persist/hibernate/query/builder/QueryBuilderFactory>` provides
several helper methods to create query builders.

Implementation
--------------

ch.tocco.nice2.persist.hibernate.query.QueryBuilderBaseImpl
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :nice:`QueryBuilderBaseImpl <ch/tocco/nice2/persist/hibernate/query/QueryBuilderBaseImpl>` is the base class for all query
builders.
It contains a list of :java-javax:`Predicate <javax/persistence/criteria/Predicate>` instances and provides several ways to add a
condition to the query:

    * Use ``QueryBuilderBase#where(Predicate...)`` to add a JPA :java-javax:`Predicate <javax/persistence/criteria/Predicate>` instance
    * The :nice:`PredicateBuilder <ch/tocco/nice2/persist/hibernate/query/PredicateBuilder>` is a functional interface that
      can be used to create :java-javax:`Predicate <javax/persistence/criteria/Predicate>` instances using lambda expressions
      that can be passed to ``QueryBuilderBase#where(PredicateBuilder)``. The :java-javax:`CriteriaBuilder <javax/persistence/criteria/CriteriaBuilder>`,
      :java-javax:`Root <javax/persistence/criteria/Root>`, :abbr:`FieldAccessor (ch.tocco.nice2.persist.hibernate.query.ch.tocco.nice2.persist.hibernate.query.FieldAccessor)` :nice:`SubqueryFactory <ch/tocco/nice2/persist/hibernate/query/PredicateBuilder.SubqueryFactory>`
      and the query hints are passed as parameters into the lambda expression.
    * :nice:`Node <ch/tocco/nice2/conditionals/tree/Node>` or :nice:`Condition <ch/tocco/nice2/persist/qb2/Condition>` instances (created by the :nice:`Conditions <ch/tocco/nice2/persist/qb2/Conditions>` API)
      can also be passed to ``QueryBuilderBase#where(Condition...)``. This API is also used by the security conditions.
      A :nice:`Condition <ch/tocco/nice2/persist/qb2/Condition>` is first converted into a :nice:`Node <ch/tocco/nice2/conditionals/tree/Node>`
      instance using the :nice:`ConditionFactory <ch/tocco/nice2/persist/query/ConditionFactory>` and then transformed into a
      :java-javax:`Predicate <javax/persistence/criteria/Predicate>` using the :nice:`PredicateFactory <ch/tocco/nice2/persist/hibernate/PredicateFactory>`.
    * Conditions added through the ``whereInsecure()`` methods are added in ``insecure`` mode (the ``isInsecure`` flag passed to ``QueryBuilderInterceptor#buildConditionFor()``
      and ``QueryBuilderInterceptor#fieldUsedInQueryCondition()`` is set to true) - this means that no ACL conditions will be added to any joins or subqueries that are present in the condition.
      The separate ``whereInsecure()`` method is necessary for security reasons to control where insecure conditions may be used, otherwise
      any user could execute insecure queries, for example through the REST API.
      The ``secure`` and ``insecure`` TQL keywords are no longer supported and will be ignored. This was necessary with the introduction of the
      query builder interceptors for joins because there was no way to mark a join as insecure (which caused huge ACL and constriction conditions).

It also invokes the ``QueryBuilderInterceptor#buildConditionFor()`` method of all interceptors when
the query initialization has been completed and adds the created conditions to the list of predicates.

.. note::
    The ``QueryBuilderInterceptor#buildConditionFor()`` method should be called when the query builder is created; not when it is executed. For example it is expected
    that if a query that is created in privileged mode, it should remain privileged even if the privileged mode is no longer active
    when the query is executed.

The method ``QueryBuilderBase#build()`` should be called by the user when the query builder configuration is completed
and returns an object that allows to access the results. The returned object depends on the subclass and is defined by
generic parameter ``QW``.

ch.tocco.nice2.persist.hibernate.query.AbstractCriteriaBuilder
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :nice:`AbstractCriteriaBuilder <ch/tocco/nice2/persist/hibernate/query/AbstractCriteriaBuilder>` is the base class
for all query builders that depend on a :java-javax:`CriteriaQuery <javax/persistence/criteria/CriteriaQuery>`.

It initializes a :java-javax:`CriteriaQuery <javax/persistence/criteria/CriteriaQuery>`, :java-javax:`CriteriaBuilder <javax/persistence/criteria/CriteriaBuilder>`,
:java-javax:`Root <javax/persistence/criteria/Root>` and :nice:`SubqueryFactory <ch/tocco/nice2/persist/hibernate/query/PredicateBuilder.SubqueryFactory>`
using the ``entityClass`` (the entity that should be queried) and ``queryType`` (the result type of the query) constructor parameters.

This class also contains a map of parameters that are manually added to the query by the user and provides a helper method
to apply the parameters to the query.

Parameter handling
~~~~~~~~~~~~~~~~~~

A condition like ``field("name").is(value)`` might be mapped with a :java-javax:`ParameterExpression <javax/persistence/criteria/ParameterExpression>`
even though the user specified the value directly. These parameters are collected and added to the query by the :nice:`ParameterCollector <ch/tocco/nice2/persist/impl/qb2/ParameterCollector>`.

The parameter collector is a visitor for :nice:`Node <ch/tocco/nice2/conditionals/tree/Node>` objects. It sets an unique
name to all parameter nodes and collects their values.

The :nice:`ParameterCollector <ch/tocco/nice2/persist/impl/qb2/ParameterCollector>` is contained by the :nice:`QueryBuilderBaseImpl <ch/tocco/nice2/persist/hibernate/query/QueryBuilderBaseImpl>`
base class, because it is needed to create conditions.

.. warning::
    It is important that only one parameter collector is used per query. Otherwise the parameter names are not unique and
    the parameter values get overwritten. This means that all :nice:`Node <ch/tocco/nice2/conditionals/tree/Node>` instances
    passed to ``QueryBuilderBase#addCondition()`` must not have been already been processed by a parameter collector.

Before the query is executed the parameters collected by the :nice:`ParameterCollector <ch/tocco/nice2/persist/impl/qb2/ParameterCollector>`
as well as parameters that are manually passed to ``AbstractCriteriaBuilder#addParameter#addParameter()`` are applied to the
:java-hibernate:`Query <org/hibernate/query/Query>` instance (see ``AbstractCriteriaBuilder#applyParametersToQuery()``).

If the parameter value does not match the parameter type it is attempted to convert the value using ``TypeManager#convert()``.
If a :java:`Collection <java/util/Collection>` is used as a parameter value ``Query#setParameterList()`` is used which can be
substantially faster for large parameter lists.

There are also global parameters that are applied to every query if a parameter with a certain name exists.
These are provided by the :nice:`ParameterProvider <ch/tocco/nice2/persist/hibernate/query/ParameterProvider>` interface.
An example would be the parameter ``currentUser`` (see :nice:`PrincipalNameFactory <ch/tocco/nice2/userbase/impl/ArgumentFactories.PrincipalNameFactory>`).

Subqueries
~~~~~~~~~~

The :nice:`AbstractCriteriaBuilder <ch/tocco/nice2/persist/hibernate/query/AbstractCriteriaBuilder>` also contains the
only implementation of the :nice:`SubqueryFactory <ch/tocco/nice2/persist/hibernate/query/PredicateBuilder.SubqueryFactory>`
which can be used to create subqueries.

There are two different options:

    * ``createSubquery()`` creates a subquery that is correlated to main query (based on a given association). This can for example be used
      to create ``EXISTS`` subqueries.
    * ``createUncorrelatedSubquery()`` can be used to create any other subquery that is not correlated to the main query. The selection and
      target entity can be freely chosen.

Both methods return an instance of :nice:`SubqueryBuilderImpl <ch/tocco/nice2/persist/hibernate/query/SubqueryBuilderImpl>` which supports
similar functionality as the standard query builder.

ch.tocco.nice2.persist.hibernate.query.CriteriaQueryBuilderImpl
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :nice:`CriteriaQueryBuilderImpl <ch/tocco/nice2/persist/hibernate/query/CriteriaQueryBuilderImpl>` is a base class for
'standard' query builders that expect multiple result rows and adds support for offset, limit and ordering.

Ordering
~~~~~~~~
The ordering can be defined through ``CriteriaQueryBuilderImpl#addOrder()``. Both the JPA :java-javax:`Order <javax/persistence/criteria/Order>`
(can be created by the :java-javax:`CriteriaBuilder <javax/persistence/criteria/CriteriaBuilder>`)
and the :nice:`Ordering <ch/tocco/nice2/persist/query/Ordering>` class of the persist API are accepted.

There is a special ordering expression that can order the results by a given list of keys.
This is created using ``OrderingUtils#orderByKeys()`` and results in a ``ORDER BY CASE WHEN ...`` clause.

.. note::

    ``OrderingUtils#orderByKeys()`` is only supported for non-distinct queries. However this should not be a problem
    as this ordering is usually combined with a ``primaryKeyIn()`` condition.

Query Wrappers
~~~~~~~~~~~~~~
The :nice:`CriteriaQueryBuilderImpl <ch/tocco/nice2/persist/hibernate/query/CriteriaQueryBuilderImpl>` defines that all
subclasses must return an implementation of :abbr:`CriteriaQueryWrapper (ch.tocco.nice2.persist.hibernate.query.CriteriaQueryWrapper)`
from their ``build()`` method and provides a base implementation (``AbstractCriteriaQueryWrapper``).

It also defines the ``QT`` type parameter of its superclass to ``Object[]``. That means that the hibernate queries always
return ``Object[]`` instances. This is necessary because sometime we need to expand the user selection (see below).

The :abbr:`CriteriaQueryWrapper (ch.tocco.nice2.persist.hibernate.query.CriteriaQueryWrapper)` interface defines the
following methods:

    * ``getResultList()`` returns a list of results
    * ``firstResult()`` returns the first result that was found
    * ``uniqueResult()`` returns exactly one result or null. If the query returns multiple rows, an exception will be thrown.
      Optionally a :java-javax:`LockModeType <javax/persistence/LockModeType>` can be passed to this method, which allows
      pessimistic locking of an entity.

``firstResult()`` and  ``uniqueResult()`` will throw an exception if no result was found. However there are
``firstResultOptional()`` and  ``uniqueResultOptional()`` methods for the case when a result is not required.

    * ``distinct()`` to configure if the query should be executed with the ``DISTINCT`` keyword. The default is true.

.. note::
    Because a join in TQL is always a ``LEFT JOIN`` all standard queries need to be executed ``DISTINCT``
    to avoid duplicate results.
    However some :java-javax:`LockModeType <javax/persistence/LockModeType>` cause a ``SELECT FOR UPDATE`` which does not support
    distinct queries. In that case, distinct queries need to be manually disabled by calling ``distinct(false)``.

AbstractCriteriaQueryWrapper
````````````````````````````

The :nice:`AbstractCriteriaQueryWrapper <ch/tocco/nice2/persist/hibernate/query/CriteriaQueryBuilderImpl.AbstractCriteriaQueryWrapper>`
is the base implementation of :abbr:`CriteriaQueryWrapper (ch.tocco.nice2.persist.hibernate.query.CriteriaQueryWrapper)` and provides
the following functionality:

It requires a transformation :java:`Function <java/util/function/Function>` which converts a result row (which is always
an ``Object[]``) into the desired target type (subclasses must override ``createMapperFunction()``).

When ``getResultList()`` is called, the following steps are taken:

    * The final ordering clause is created: If no explicit ordering is defined for the query, the default ordering defined in the entity model is used.
      In addition, the primary key is always added as the last sorting parameter (unless it already is part of the sorting clause).
      This is necessary to guarantee a consistent ordering when ``LIMIT`` or ``OFFSET`` is used (otherwise the order might be
      partially random if there are many rows with same value in the order column).
    * The final :java-javax:`Selection <javax/persistence/criteria/Selection>` of the query is determined: The user defined selection
      is provided by the subclass (abstract method ``getSelection()``), however it might have to be expanded:

      According to the SQL Standard all columns that are part of the ``ORDER BY`` clause must also be part of the select clause
      if it is a ``DISTINCT`` query.
      The missing columns are automatically added to the selection (``expandSelection(List<Order> order)``)
      and are removed again before the results are processed (``unwrapResults(List<Object[]> results)``).

      If a ``SELECT CASE`` expression is used in the ordering clause, it also needs to be added to the selection. However in this case
      the ``ORDER BY`` expression needs to be replaced with a literal reference to the selection (``ORDER BY 1`` for example),
      otherwise PostgreSQL does not recognize that both of these expressions are the same. Since by default all literals
      will be rendered as parameters we need to explicitly use ``CriteriaBuilderWrapper#inlineLiteral()`` that uses an
      :nice:`InlineLiteralExpression <ch/tocco/nice2/persist/hibernate/InlineLiteralExpression>` which overrides the
      default :java-hibernate:`LiteralHandlingMode <org/hibernate/query/criteria/LiteralHandlingMode>` to ``AUTO`` (we do
      not use ``INLINE`` to make sure that strings are never inlined, as this would be an SQL injection risk).

      Due to a bug in hibernate an array selection of size 1 is not returned as array. As this breaks our code we
      add a dummy selection (the literal '1') if the the selection size is 1.

    * The :java-javax:`CriteriaQuery <javax/persistence/criteria/CriteriaQuery>` is then converted into a :java-hibernate:`Query <org/hibernate/query/Query>` and
      selection, conditions, ordering and parameters are applied.
    * The query is then executed and the results returned after they have been processed by the transformation function (see above).

``uniqueResult()`` works similarly, but as we expect only one result, we do not have to worry about the ordering clause.

ch.tocco.nice2.persist.hibernate.query.EntityQueryBuilderImpl
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :nice:`EntityQueryBuilderImpl <ch/tocco/nice2/persist/hibernate/query/EntityQueryBuilderImpl>` is an implementation
that queries for :nice:`Entity <ch/tocco/nice2/persist/entity/Entity>` instances.

It defines the :java-javax:`Root <javax/persistence/criteria/Root>` as the selection of the query and the mapping function
simply casts the first element of the result array into an :nice:`Entity <ch/tocco/nice2/persist/entity/Entity>`.

ch.tocco.nice2.persist.hibernate.query.AbstractPathQueryBuilder
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :nice:`AbstractPathQueryBuilder <ch/tocco/nice2/persist/hibernate/query/AbstractPathQueryBuilder>` is a base class
for query builders that use a :nice:`CustomSelection <ch/tocco/nice2/persist/hibernate/query/selection/CustomSelection>`.
This means that they do not return entity instances, but only certain paths.

It provides a method called ``clearSelection()`` that re-initializes the selection. However this method cannot remove joins that
were created by the previous selection and is used internally only.

This class also provides the :abbr:`CriteriaQueryWrapper (ch.tocco.nice2.persist.hibernate.query.CriteriaQueryWrapper)` implementation
for its subclasses: :nice:`CustomSelectionCriteriaQueryWrapper <ch/tocco/nice2/persist/hibernate/query/AbstractPathQueryBuilder.CustomSelectionCriteriaQueryWrapper>`.
``getSelection()`` returns the selection created by ``CustomSelection#toJpaSelection()``.

It provides a protected method ``mapResults()`` that initializes the result structure and processes the query results using ``CustomSelection#mapResults()``.
This is necessary because the :nice:`CustomSelection <ch/tocco/nice2/persist/hibernate/query/selection/CustomSelection>`
may add additional paths (for internal processing) and some paths need to evaluated in an additional query (to-many paths for example).

ch.tocco.nice2.persist.hibernate.query.SinglePathQueryBuilderImpl
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :nice:`SinglePathQueryBuilderImpl <ch/tocco/nice2/persist/hibernate/query/SinglePathQueryBuilderImpl>` can be used to
query for exactly one path of an entity. The constructor takes a ``Class<T>`` parameter which defines the return type
of the query.

The ``setPath(String)`` method needs to be called to define which path should be selected.
It is verified if the selected path matches the return type, otherwise an exception will be thrown.

An exception is also thrown if ``setPath(String)`` is never called.

It returns a :nice:`CustomSelectionCriteriaQueryWrapper <ch/tocco/nice2/persist/hibernate/query/AbstractPathQueryBuilder.CustomSelectionCriteriaQueryWrapper>`
from its ``build()`` method with a mapping function that returns the first element of the result array.

It also provides a simple implementation of :nice:`ResultRowMapper <ch/tocco/nice2/persist/hibernate/query/mapper/ResultRowMapper>`.
Because the result is always the selected path of type ``T`` the ``mapToOnePath()`` and ``mapToManyPath()`` methods can simply return
the values provided by the given :nice:`ValueProvider <ch/tocco/nice2/persist/hibernate/query/mapper/ResultRowMapper.ValueProvider>`.

See :ref:`custom_selection` for more information about the :nice:`ResultRowMapper <ch/tocco/nice2/persist/hibernate/query/mapper/ResultRowMapper>`
class.

ch.tocco.nice2.persist.hibernate.query.PathQueryBuilderImpl
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :nice:`PathQueryBuilderImpl <ch/tocco/nice2/persist/hibernate/query/PathQueryBuilderImpl>` can be used to
query for multiple paths of an entity and always returns a container type like ``Object[]`` or ``Map``.

The constructor of this class requires an instance of :nice:`ResultRowMapper <ch/tocco/nice2/persist/hibernate/query/mapper/ResultRowMapper>`
that supports the return type ``T``.

There currently are two different implementations available:

    * :abbr:`ArrayResultRowMapper (ch.tocco.nice2.persist.hibernate.query.mapper.ArrayResultRowMapperFactory.ArrayResultRowMapper)` converts
      query results into a flat structure using an ``Object[]``. The order in the array depends on the order the paths were given
      to ``addPathToSelection()``.
    * :abbr:`MapResultRowMapper (ch.tocco.nice2.persist.hibernate.query.mapper.MapResultRowMapperFactory.MapResultRowMapper)` converts each row
      into a :java:`Map <java/util/Map>`. This creates a nested structure and is useful to group fields by their relation paths.
    * :nice:`CustomResultRowMapperFactory <ch/tocco/nice2/persist/hibernate/query/mapper/CustomResultRowMapperFactory>` supports custom beans.
      Any java class that is annotated with :nice:`QueryBuilderResult <ch/tocco/nice2/persist/hibernate/query/builder/QueryBuilderResult>`
      is supported. If the field name of the bean matches the entity field name, it will be mapped automatically, otherwise the
      :nice:`ResultPath <ch/tocco/nice2/persist/hibernate/query/builder/ResultPath>` annotation must be used to specify
      the mapping. It is also possible to map a sub-path of the result to a nested java bean using the :nice:`NestedResultPath <ch/tocco/nice2/persist/hibernate/query/builder/NestedResultPath>`.
      The nested bean supports the same features as the main bean (but the class level annotation is not necessary).
      To-many paths are supported using a :java:`List <java/util/List>` or :java:`Set <java/util/Set>`.

The ``PersistenceService#createPathQueryBuilder()`` methods builds an instance of :nice:`ResultRowMapper <ch/tocco/nice2/persist/hibernate/query/mapper/ResultRowMapper>`
using contributed :nice:`ResultRowMapperFactory <ch/tocco/nice2/persist/hibernate/query/mapper/ResultRowMapperFactory>` instances, based on the
requested result type.

The method ``addPathToSelection()`` can be called multiple times to add paths to the selection.
At least one path needs to be added otherwise an exception will be thrown.

ch.tocco.nice2.persist.hibernate.query.CriteriaCountQueryBuilder
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :nice:`CriteriaCountQueryBuilder <ch/tocco/nice2/persist/hibernate/query/CriteriaCountQueryBuilder>`
executes ``COUNT`` queries and always returns a :java:`Long <java/lang/Long>`.

It inherits directly from :nice:`AbstractCriteriaBuilder <ch/tocco/nice2/persist/hibernate/query/AbstractCriteriaBuilder>`
because it does not return an ``Object[]`` and also returns a different object from its ``build()`` method.

.. _custom_selection:

Custom Selection
----------------

The :nice:`CustomSelection <ch/tocco/nice2/persist/hibernate/query/selection/CustomSelection>` is used by some query builders
that select only certain paths (not entire entities).

It is not sufficient to simply add all requested paths to the JPA selection due to the following reasons:

    * Security: It must be possible to intercept field selection. The query only adds the security conditions of
      the target entity by default. But it does not check field permissions and also a path may point to a different entity
      that needs to be checked as well.
    * Paths pointing to a to-many property would return multiple rows per target entity. Even if the data would be
      merged later, it would make ``LIMIT/OFFSET`` options useless.

A custom selection contains a :nice:`SelectionRegistry <ch/tocco/nice2/persist/hibernate/query/selection/SelectionRegistry>`.
The selection registry keeps track of all 'requested paths' (paths that should be included in the final ``Object[]``
returned from the query builder) and all 'query paths' (paths that are included in the query).
Not all 'requested paths' will generate a 'query path' (for example to-many paths are evaluated in a separate query) and
the 'query paths' may contain additional paths that are required for internal processing, but won't be returned from the
query builder.
The selection registry maintains maps that keep track which query/requested path is at which position in the result arrays.
It also makes sure that there are no duplicated 'query paths' (for example when the same internal path is required by
multiple paths).
All the query paths can be converted into a JPA :java-javax:`Selection <javax/persistence/criteria/Selection>` by the
method ``toSelection()``.

The :nice:`CustomSelection <ch/tocco/nice2/persist/hibernate/query/selection/CustomSelection>` also contains multiple
:nice:`SelectionPathHandler <ch/tocco/nice2/persist/hibernate/query/selection/SelectionPathHandler>`.
A :nice:`SelectionPathHandler <ch/tocco/nice2/persist/hibernate/query/selection/SelectionPathHandler>` is responsible
for handling a certain type of path.

``SelectionPathHandler#processSelection()`` is called just before the JPA :java-javax:`Selection <javax/persistence/criteria/Selection>`
is created. The :nice:`SelectionRegistry <ch/tocco/nice2/persist/hibernate/query/selection/SelectionRegistry>` is passed
as an argument and can be used to add all necessary query paths to the query.

``SelectionPathHandler#processResults()`` is called after the query has been executed. Both the list of results of the query
and the target (that will be returned from the query builder) are passed as arguments. The task of the handler is to
copy the query results into the target array. The :nice:`SelectionRegistry <ch/tocco/nice2/persist/hibernate/query/selection/SelectionRegistry>`
contains the source and target indices of the paths. In addition an instance of :nice:`ResultRowMapper <ch/tocco/nice2/persist/hibernate/query/mapper/ResultRowMapper>`
is passed to this method as well.

The :nice:`ResultRowMapper <ch/tocco/nice2/persist/hibernate/query/mapper/ResultRowMapper>`
does the actual mapping to the final result structure and has the following methods:

    * ``createInstanceOfResultType()`` creates an instance of the result container (like ``Object[]``, ``Map``). May also
      be null if there is only a single value and no container.
    * ``mapToOnePath()`` maps to-one paths to the result container. It has the following parameters:

        * ``paths`` all the paths that should be mapped
        * ``queryResultProvider`` an instance of :nice:`ValueProvider <ch/tocco/nice2/persist/hibernate/query/mapper/ResultRowMapper.ValueProvider>`
          that allows to access the result of the current row for a given path
        * ``result`` an instance of the result container. The results should be mapped to this object.
        * ``rootSelectionRegistry`` can be used to access the index of a given path to be able to insert it in the correct
          position of the result container

    *   ``mapToManyPath()`` maps to-many paths to the result container. It has the same parameters as ``mapToOnePath()``, except
        that it receives a list of :nice:`ValueProvider <ch/tocco/nice2/persist/hibernate/query/mapper/ResultRowMapper.ValueProvider>`

The :nice:`SelectionPathHandler <ch/tocco/nice2/persist/hibernate/query/selection/SelectionPathHandler>` are also
responsible for calling the :nice:`QueryBuilderInterceptor <ch/tocco/nice2/persist/hibernate/query/QueryBuilderInterceptor>`
selection builder methods.

    * The :nice:`ToOneSelectionPathHandler <ch/tocco/nice2/persist/hibernate/query/selection/ToOneSelectionPathHandler>`
      is responsible for all 'to-one' paths. It is relatively straight-forward: the paths can be included in the query
      and after the query execution the paths can simply mapped to the target array.

    * The :nice:`ToManySelectionPathHandler <ch/tocco/nice2/persist/hibernate/query/selection/ToManySelectionPathHandler>`
      handles all 'to-many' paths. These paths cannot be selected directly in the query. For each base path a separate
      query is generated that retrieves the values of these paths for *all* rows. The rows are then mapped to the target array
      using the primary key of the root entity, that is selected by both queries.

    * There are special implementations for ``binary`` fields, because the ``_nice_binary`` table is not mapped by
      hibernate at the moment and cannot be queried directly. They use the :nice:`BinaryDataAccessor <ch/tocco/nice2/persist/hibernate/binary/BinaryDataAccessor>`
      to efficiently load :nice:`BinaryData <ch/tocco/nice2/persist/hibernate/binary/BinaryData>` instances, which are then merged
      into the target array.

Query Builder Interceptor
-------------------------
The :nice:`QueryBuilderInterceptor <ch/tocco/nice2/persist/hibernate/query/QueryBuilderInterceptor>` participates
in the query building process.

``buildConditionFor()``
^^^^^^^^^^^^^^^^^^^^^^^

This method is called for every query root and for every subquery and can add additional conditions to the query.

    - ``BusinessUnitQueryBuilderInterceptor`` makes sure that only entities belonging to the current business unit are returned
    - ``SecureQueryInterceptor`` adds additional conditions based on the security policy

The method takes an instance of :nice:`QueryBuilderType <ch/tocco/nice2/persist/hibernate/query/QueryBuilderInterceptor.QueryBuilderType>`
which signifies by what kind of query builder it is called. Currently ``READ`` and ``DELETE`` are supported. The
``SecureQueryInterceptor`` uses this information to apply the correct security conditions depending on the query type.

The argument :nice:`QueryBuilderSituation <ch/tocco/nice2/persist/hibernate/query/QueryBuilderInterceptor.QueryBuilderSituation>`
indicates whether the returned conditions will be applied to a (sub)query or a join.

``fieldUsedInQueryCondition()``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method will be called whenever a field is used in a query condition, for example ``where username == 'user'``.
The ``SecureQueryInterceptor`` will return conditions based on ``entityPath`` rules and will throw
an exception when a field is used that is marked as ``privileged-only`` in the field model.

``createSelectionInterceptor()``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method is only used when a :nice:`CustomSelection <ch/tocco/nice2/persist/hibernate/query/selection/CustomSelection>`
is used. It is called once for each 'base path' (a path without field) of the query.
So for example when the paths ``relUser.name``, ``relUser.lastname``, ``relAddress.address``, ``relAddress.city`` are selected,
the method is called once for ``relUser`` and ``relAddress``.

The method may return an :nice:`SelectionInterceptor <ch/tocco/nice2/persist/hibernate/query/QueryBuilderInterceptor.SelectionInterceptor>`,
which allows modification of the selection and inspection & replacement of the query results.

SelectionInterceptor
~~~~~~~~~~~~~~~~~~~~

``beforeQueryExecution(SelectionData)`` is called before the relevant query is executed and allows adding additional
selection paths.
One use case is to add the primary key of a 'base path' to the selection in order to be able to check access permissions.

``handleQueryResults()`` gives access to the query results and also allows overriding the query results.
The use case of the ``SecureQueryInterceptor`` is to find all primary keys of a base path using ``QueryResult#getValuesForPath()``
then check access permissions and overwrite the value with null if access is denied (using ``QueryResult#findRowsWithValueAtPath()``
and ``Row#setValueForPath()``.

Interceptors for Joins
----------------------

The :nice:`QueryBuilderInterceptor <ch/tocco/nice2/persist/hibernate/query/QueryBuilderInterceptor>` is also called for
joins that are used in conditions (in addition to subqueries and the root entity) to make sure
that the conditions cannot be used to bypass ACL rules.

For example the query ``find User where relUser_status.unique_id == "active"`` should not return any results
if the principal does not have access to the related ``User_status`` entity or the ``relUser_status`` field of the ``User``
entity.

Unlike additional conditions for the root entity, additional conditions for joins cannot just be added to the query builder:

``(relUser_status.unique_id == "active" or username is not null)`` would become
``(relUser_status.unique_id == "active" or username is not null) and <interceptor-condition>``.
This would never return any results if the condition added by the interceptor evaluates to false, even if the second part of the OR
clause is true.
Therefore the condition needs to be combined only with the clause that contains the join:
``(relUser_status.unique_id == "active" and <interceptor-condition>) or username is not null``.

.. note::

    Due to this, large ``OR`` clauses should be replaced with an ``IN`` clause, as the ``OR`` clause can become very inefficient:
    ``where value = 1 AND <interceptor-condition> OR value = 2 AND <interceptor-condition> ...`` versus
    ``where value IN (1,2,...) AND <interceptor-condition>``.

To achieve this we use an extended :java-javax:`CriteriaBuilder <javax/persistence/criteria/CriteriaBuilder>` that
intercepts the creation of all predicates and wraps them with the conditions from the interceptors if necessary
(:nice:`CriteriaBuilderWrapper <ch/tocco/nice2/persist/hibernate/query/CriteriaBuilderWrapper>`).

The wrapper overrides methods like ``equal()`` and ``notEqual()``:

    * The creation of the actual predicate is delegated to the 'real' criteria builder
    * All expressions that are passed to the criteria builder (see below) are then processed by
      the interceptors and the resulting :nice:`Node <ch/tocco/nice2/conditionals/tree/Node>` instances
      will be converted to :java-javax:`Predicate <javax/persistence/criteria/Predicate>` instances using
      a derived :nice:`PredicateFactory <ch/tocco/nice2/persist/hibernate/PredicateFactory>`. The predicate
      factory needs to be derived to use the current join as the query root (as the conditions are based on this
      entity, not the query root) and to use the real criteria builder to avoid endless recursion.
    * The actual predicate is then combined with the interceptor predicates and an AND predicate is returned from the call
      (only if there are any interceptor predicates, otherwise just the actual predicate is returned directly).

Conditions are collected from the following expressions:

:java-javax:`Path <javax/persistence/criteria/Path>`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A path might for example look like ``relEntity.relEntity2.field``. The :java-javax:`Path <javax/persistence/criteria/Path>` instance always references the last
path element. If it is an instance of :java-javax:`From <javax/persistence/criteria/From>`, the last path element is
a relation, otherwise it is a field.

For the example path ``relUser.relAddress.city`` the conditions of the following interceptor calls
are collected:

    * ``fieldUsedInQueryCondition("Address", "city")`` (this call only applies when the path points to a field)
    * ``buildConditionFor("Address")``
    * ``fieldUsedInQueryCondition("User", "relAddress")``
    * ``buildConditionFor("User")``
    * ``fieldUsedInQueryCondition(ROOT, "relUser")``

:java-hibernate:`ParameterizedFunctionExpression <org/hibernate/query/criteria/internal/expression/function/ParameterizedFunctionExpression>`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

All parameter expressions of the function call are recursively evaluated (see above how :java-javax:`Path <javax/persistence/criteria/Path>`
expression are evaluated).

:java-javax:`Subquery <javax/persistence/criteria/Subquery>`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A (correlated) subquery might be created for example from the following condition ``exists(relUser.relAddress.relStatus where ... )``.

In this example the ``relStatus`` join is the 'root' of the subquery: conditions of the ``Status`` entity do not need to be added to the join,
they will already be added to the subquery. However it is necessary to check the field of that join (``Address#relStatus``).

The ``relAddress`` join is the 'correlated' join. Conditions up to this join will be collected (see above how :java-javax:`Path <javax/persistence/criteria/Path>`
expression are evaluated).

So for the above example the following interceptor calls are made:

    * ``fieldUsedInQueryCondition("Address", "relStatus")``
    * ``buildConditionFor("Address")``
    * ``fieldUsedInQueryCondition("User", "relAddress")``
    * ``buildConditionFor("User")``
    * ``fieldUsedInQueryCondition(ROOT, "relUser")``

Joins and fields in the ORDER BY clause
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

It is also necessary to secure the ``ORDER BY`` clauses, it should not be possible to order by a field or relation
that is not accessible.
For that purpose the :nice:`CriteriaBuilderWrapper <ch/tocco/nice2/persist/hibernate/query/CriteriaBuilderWrapper>`
also overrides the ``asc`` and ``desc`` methods and returns a modified order by clause that uses a ``SELECT CASE ... WHEN ...`` expression.

Conditions are collected for the ``ORDER BY`` expression in the same way as described for conditions above.
The collected conditions are then wrapped in the following way:

``ORDER BY name`` becomes ``ORDER BY SELECT CASE <interceptor-condition> THEN name OTHERWISE null`` which means that
rows where the ``ORDER BY`` clause is not accessible will be ordered like if the ``ORDER BY`` clause would evaluate to NULL.

Custom JDBC Functions
---------------------
Custom query functions can be implemented using the :nice:`JdbcFunction <ch/tocco/nice2/persist/hibernate/query/JdbcFunction>` interface.
The contributions are registered with the :java-hibernate:`SessionFactoryBuilder <org/hibernate/boot/SessionFactoryBuilder>` by the
:nice:`HibernateCoreBootstrapContribution <ch/tocco/nice2/persist/hibernate/bootstrap/HibernateCoreBootstrapContribution>`.

In addition to the contributed functions, the :nice:`GlobSqlFunction <ch/tocco/nice2/persist/hibernate/dialect/GlobSqlFunction>`
is registered as well. It implements the ``glob`` function, which is internally used when the ``Operator#LIKE`` is specified.
It uses ``LIKE`` internally but is also replacing ``*`` with ``%`` and ``?`` with ``_`` so that both placeholders are supported.

Each function must provide a :java-hibernate:`SQLFunction <org/hibernate/dialect/function/SQLFunction>` which contains the SQL template.
Typically the :java-hibernate:`SQLFunctionTemplate <org/hibernate/dialect/function/SQLFunctionTemplate>` can be used for this.
An instance of :nice:`SqlWriter <ch/tocco/nice2/persist/query/SqlWriter>` is provided to facilitate writing the SQL query. The
sql writer is obtained from ``Context#createSqlWriter()`` and is automatically configured based on the current :java-hibernate:`Dialect <org/hibernate/dialect/Dialect>`.

The abstract base class :nice:`AbstractJdbcFunction <ch/tocco/nice2/persist/hibernate/query/AbstractJdbcFunction>` provides support
to create the sql function templates:

    * Find the correct hibernate :java-hibernate:`Type <org/hibernate/type/Type>` based on the nice :nice:`Type <ch/tocco/nice2/types/Type>`
    * The ``writeArgument()`` method can be used to write a parameter placeholder into the sql string

.. warning::

    The arguments of the :nice:`Condition <ch/tocco/nice2/persist/qb2/Condition>` are passed to the criteria builder in the same order.
    If the order of arguments is different in the sql template or a parameter is used multiple times, the ``argumentOrder()`` method
    needs to be overwritten by the :nice:`JdbcFunction <ch/tocco/nice2/persist/hibernate/query/JdbcFunction>`. The arguments
    are then reordered and/or duplicated by the :abbr:`FuncallArgumentProcessor (ch.tocco.nice2.persist.hibernate.pojo.CriteriaQueryCompiler.FuncallArgumentProcessor)`
    before the query is processed.

.. note::
    The :nice:`JdbcFunction <ch/tocco/nice2/persist/hibernate/query/JdbcFunction>` operates directly on the SQL level
    and can be used to access database specific functions.
    An example is the :nice:`BirthdayQueryFunction <ch/tocco/nice2/persist/backend/jdbc/impl/functions/BirthdayQueryFunction>`
    that uses the ``extract`` PostgreSQL function.

.. note::
    Each JDBC Function must implement the ``validateArguments()`` function which should check whether the given arguments (paths in particular)
    are compatible with the function. If an incompatible path is given to the function, the content of that path might be visible in
    the log file, which is a security issue.

Query Functions
---------------
A :nice:`QueryFunction <ch/tocco/nice2/persist/spi/query/ql/QueryFunction>` can be used to implement a custom function that
can be used in the query language.
The query functions are applied by the :nice:`ConditionFactory <ch/tocco/nice2/persist/query/ConditionFactory>` when
the :nice:`Node <ch/tocco/nice2/conditionals/tree/Node>` tree is processed and can manipulate its nodes.

.. note::
    An example would be the :nice:`FulltextSearchFunction <ch/tocco/nice2/enterprisesearch/impl/queryfunction/FulltextSearchFunction>`:
    It executes the fulltext search when the query is compiled and replaces the query function node with an ``IN`` condition
    that includes the primary keys of the results of the search.

Query Compiler
--------------
The :nice:`CriteriaQueryCompiler <ch/tocco/nice2/persist/hibernate/pojo/CriteriaQueryCompiler>` is responsible for creating a
:nice:`Query <ch/tocco/nice2/persist/query/Query>` instance based on a :nice:`Node <ch/tocco/nice2/conditionals/tree/Node>`.

The :abbr:`QueryVisitor (ch.tocco.nice2.persist.hibernate.pojo.CriteriaQueryCompiler.QueryVisitor)` visits the node tree
and collects the entity model, condition and ordering data, which in turn will be
wrapped in a :nice:`HibernateQueryAdapter <ch/tocco/nice2/persist/hibernate/pojo/HibernateQueryAdapter>` that is returned
to the user.

QueryVisitor
^^^^^^^^^^^^
The query visitor handles the following funcall nodes:

    - ``Keywords.FIND``: The entity model that should be queried
    - ``Keywords.ORDER``: Each child node represents an order path and direction
    - ``Keywords.WHERE``: The condition of the query.

The condition (the WHERE part of the query) is processed by the :nice:`ConditionFactory <ch/tocco/nice2/persist/query/ConditionFactory>`
before it is added to the conditions list.
The condition factory applies the following visitors:

    - ``TypeSettingVisitor``: Sets the :nice:`Type <ch/tocco/nice2/types/Type>` of a field to the corresponding path node
    - ``QueryFunctionCompiler``: Applies all :nice:`QueryFunction <ch/tocco/nice2/persist/spi/query/ql/QueryFunction>` to the conditions

Predicate Factory
-----------------
The :nice:`PredicateFactory <ch/tocco/nice2/persist/hibernate/PredicateFactory>` converts :nice:`Node <ch/tocco/nice2/conditionals/tree/Node>` instances
representing conditions into a :java-javax:`Predicate <javax/persistence/criteria/Predicate>`.
These conditions are created by the :nice:`QueryBuilderFactory <ch/tocco/nice2/persist/qb2/QueryBuilderFactory>`
as well as the ACL parser.

The node tree is parsed using different :nice:`NodeVisitor <ch/tocco/nice2/conditionals/tree/processing/NodeVisitor>`
implementations, that all extend from :abbr:`AbstractNodeVisitor (ch.tocco.nice2.persist.hibernate.PredicateFactory.AbstractNodeVisitor)`.

AbstractNodeVisitor
^^^^^^^^^^^^^^^^^^^
This is the base class that all visitor implementations use. It defines an abstract method (``getPredicate()``) which
should return a :java-javax:`Predicate <javax/persistence/criteria/Predicate>` instance for the current node.
For example the :abbr:`LogicalNodeVisitor (ch.tocco.nice2.persist.hibernate.PredicateFactory.LogicalNodeVisitor)` converts
an :nice:`AndNode <ch/tocco/nice2/conditionals/tree/AndNode>`, :nice:`OrNode <ch/tocco/nice2/conditionals/tree/OrNode>` or
:nice:`NotNode <ch/tocco/nice2/conditionals/tree/NotNode>` into a :java-hibernate:`CompoundPredicate <org/hibernate/query/criteria/internal/predicate/CompoundPredicate>`.

Additionally the base class provides helper methods to handle child nodes (``handle[...]Node()``).
These helper methods create a new visitor for the given node and pass it to ``processVisitor()``, which processes the node
with the new visitor. It also calls ``Cursor#next()`` to make sure that nested calls are only handled by the newly created visitor.
Each child node is processed in isolation by its own visitor instance and its results are then aggregated by the parent visitor.

A :nice:`FuncallNode <ch/tocco/nice2/conditionals/tree/FuncallNode>` may be a placeholder for different types of nodes:

    - ``EXISTS`` subquery
    - ``IN`` condition
    - ``COUNT`` subquery
    - a :nice:`JdbcFunction <ch/tocco/nice2/persist/hibernate/query/JdbcFunction>` call

AbstractJoiningVisitor
^^^^^^^^^^^^^^^^^^^^^^
An abstract base class that handles a :nice:`PathNode <ch/tocco/nice2/conditionals/tree/PathNode>` and converts
the path into a :java-javax:`Path <javax/persistence/criteria/Path>` performing joins if necessary.

The actual work is done in :nice:`QueryBuilderJoinHelper <ch/tocco/nice2/persist/hibernate/QueryBuilderJoinHelper>`:

    - Iteration over all path parts (``relUser.relAddress.value`` would be three different parts)
    - If the part is an association a join to the target entity is performed
    - If it is a field, the path to that field is returned

If the path points to a primary key that is referenced in a many to one association, the foreign key field is returned
instead of performing an unnecessary join (which results in ``address.fk_user = ?`` instead of ``INNER JOIN user ON user.pk = address.fk_user WHERE user.pk = ?``
for performance reasons.
This shortcut can only be used when the :nice:`QueryBuilderInterceptors <ch/tocco/nice2/persist/hibernate/query/QueryBuilderInterceptor>`
do not need to add any conditions to that join. This is checked through the :nice:`JoinInfo <ch/tocco/nice2/persist/hibernate/JoinInfo>`
class which internally uses the ``CriteriaBuilderWrapper#hasQueryRestrictions()`` method.

When a join is created it corresponds to an actual JOIN in the SQL. Therefore it should be tried to reuse the join instances
if the same entity is going to be joined multiple times.

RootNodeVisitor
^^^^^^^^^^^^^^^
The :abbr:`RootNodeVisitor (ch.tocco.nice2.persist.hibernate.PredicateFactory.RootNodeVisitor)` is the entry point which handles the
root node. It simply delegates to the visitor that can handle the root node and returns the predicate of that visitor.

LogicalNodeVisitor
^^^^^^^^^^^^^^^^^^
The :abbr:`LogicalNodeVisitor (ch.tocco.nice2.persist.hibernate.PredicateFactory.LogicalNodeVisitor)` is responsible for
handling :nice:`AndNode <ch/tocco/nice2/conditionals/tree/AndNode>`, :nice:`OrNode <ch/tocco/nice2/conditionals/tree/OrNode>`
and :nice:`NotNode <ch/tocco/nice2/conditionals/tree/NotNode>`.

This visitor collects all predicates of its child nodes (including other logical nodes) and nests them into an ``And``, ``Or`` or ``Not`` predicate.

ExistsNodeVisitor
^^^^^^^^^^^^^^^^^
The :abbr:`ExistsNodeVisitor (ch.tocco.nice2.persist.hibernate.PredicateFactory.ExistsNodeVisitor)` handles
a :nice:`FuncallNode <ch/tocco/nice2/conditionals/tree/FuncallNode>` with the ``EXISTS`` keyword.
These nodes represent an ``EXISTS`` subquery.

The first child node is always a :nice:`PathNode <ch/tocco/nice2/conditionals/tree/PathNode>` that references the
relation path which is queried by the subquery. Thus the ``visitPath()`` method first creates an instance of
:java-javax:`Subquery <javax/persistence/criteria/Subquery>` through the :nice:`SubqueryFactory <ch/tocco/nice2/persist/hibernate/query/PredicateBuilder.SubqueryFactory>`.

The path node might contain multiple relation paths which leads to nested ``EXISTS`` subqueries.
All exists predicates are collected on a stack until the path is parsed completely. The (optional)
condition is added to the top element of the stack (the one that was added last). While the predicates are removed
from the stack an exists condition is added (referencing the predicate that was removed before itself).
The last element removed from the stack is returned from the visitor.

InNodeVisitor
^^^^^^^^^^^^^
The :abbr:`InNodeVisitor (ch.tocco.nice2.persist.hibernate.PredicateFactory.InNodeVisitor)` is used for handling
``IN`` clauses.

The values of the ``IN`` clause can either be specified as literals or parameters. The parameter names or literal values
are collected, converted to :java-javax:`Expression <javax/persistence/criteria/Expression>` and then passed as parameters
to an :java-hibernate:`InPredicate <org/hibernate/query/criteria/internal/predicate/InPredicate>`.

IsTrueNodeVisitor
^^^^^^^^^^^^^^^^^
The :abbr:`IsTrueNodeVisitor (ch.tocco.nice2.persist.hibernate.PredicateFactory.IsTrueNodeVisitor)` creates a boolean
:java-javax:`Expression <javax/persistence/criteria/Expression>`.
Either based on a :java-javax:`Path <javax/persistence/criteria/Path>` that points to a boolean or a literal expression.
The latter may be used by the security framework to deny any access (``AND false``).

JpaIntegrationNodeVisitor
^^^^^^^^^^^^^^^^^^^^^^^^^
The :nice:`JpaIntegrationNode <ch/tocco/nice2/persist/hibernate/query/JpaIntegrationNode>` contains a
:nice:`PredicateBuilder <ch/tocco/nice2/persist/hibernate/query/PredicateBuilder>` which allows to create a
condition using the new query builder features (for example uncorrelated subqueries).

This makes it possible to integrate the new features with the old query builder (this was primarily created for the
:nice:`PermissionMatrixEvaluationService <ch/tocco/nice2/dms/security/policyprocessor/PermissionMatrixEvaluationService>`).

EquationNodeHandler
^^^^^^^^^^^^^^^^^^^
The :abbr:`EquationNodeHandler (ch.tocco.nice2.persist.hibernate.EquationNodeHandler)` converts an
:nice:`EquationNode <ch/tocco/nice2/conditionals/tree/EquationNode>` into a :java-javax:`Predicate <javax/persistence/criteria/Predicate>`.
An equation node consists of two nodes and an operator that defines how the two nodes can be compared.

Currently the following nodes are supported:

    * ``PathNode`` represents a path to a certain field
    * A count expression represented by a ``FuncallNode``
    * ``LiteralNode`` represents an explicit literal expression
    * ``ParameterNode`` represents a parameter expression
    * ``FuncallNode`` represents any sql function call

Obviously both nodes need to be of the same type, otherwise hibernate will throw an exception.
Since both the ``ParameterNode`` and the ``LiteralNode`` can be converted to a different type (if a suitable converter
exists), the 'other side' of the equation is evaluated first and then it is attempted to convert the literal or parameter
using the :nice:`TypeManager <ch/tocco/nice2/types/TypeManager>` to the type of the 'other side' (if necessary).

The ``LIKE`` operator is handled specially as it is not translated into a SQL ``LIKE`` but mapped to our custom ``glob``
:java-hibernate:`SQLFunction <org/hibernate/dialect/function/SQLFunction>` (:nice:`GlobSqlFunction <ch/tocco/nice2/persist/hibernate/dialect/GlobSqlFunction>`).
Both sides of the equation are
converted to lower case to simulate ``ILIKE`` behaviour.

Localized fields
^^^^^^^^^^^^^^^^
If a localized field is part of a query it needs to be resolved for the current locale before the query is parsed.
This is achieved by the :nice:`EntityInterceptorVisitor <ch/tocco/nice2/persist/hibernate/pojo/EntityInterceptorVisitor>`
which is executed before the query is parsed by the predicate factory.

All path nodes are processed by the :nice:`FieldResolver <ch/tocco/nice2/persist/hibernate/interceptor/FieldResolver>`
and all virtual fields are replaced.

Delete query builder
--------------------
The :nice:`CriteriaDeleteBuilderImpl <ch/tocco/nice2/persist/hibernate/query/CriteriaDeleteBuilderImpl>` is a special query builder
implementation that can be used to delete multiple entities by query without the need to load every single entity.

The query selects the primary keys of all entities that may be deleted (the correct security conditions are added by the
``SecureQueryInterceptor``).
For each result a proxy is created, marked as deleted and the ``entityDeleting()`` event is fired. The reason for the proxy is
to avoid loading the entire entity unless it is absolutely necessary (for example when the entity data is accessed by a listener).

Note that ``Entity#markDeleted()`` is used. This is an internal method that can be invoked without initializing the proxy
(as opposed to ``delete()``) and causes ``getState()`` to correctly return ``PHANTOM``.

After the invocation of the listeners the proxy instances are scheduled for deletion with the :nice:`EntityTransactionContext <ch/tocco/nice2/persist/hibernate/cascade/EntityTransactionContext>`.
Note that the ``addDeletedEntityBatch()`` method is used that deletes the entire batch with one delete statement (as opposed to
the normal behaviour which fires a delete statement for every deleted entity).

QueryDefinition / QueryConfigurator
-----------------------------------

The :nice:`QueryDefinition <ch/tocco/nice2/persist/query/QueryDefinition>` contains all necessary information
to build a query. It is used as a bridge between the legacy :nice:`Query <ch/tocco/nice2/persist/query/Query>`
and the new query builders.

An instance can be obtained from the method ``Query#toQueryDefinition()`` which can then be converted to a
:nice:`QueryConfigurator <ch/tocco/nice2/persist/hibernate/query/builder/QueryConfigurator>` which can be
applied to the new query builder using ``CriteriaQueryBuilder#applyConfiguration()``.

This was primarily developed to be able to combine the :nice:`EntityExplorerActionSelectionService <ch/tocco/nice2/netui/actions/entityoperation/EntityExplorerActionSelectionService>`
with the new query builders.

Query hints
-----------

When a query builder instance is created using the :nice:`PersistenceService <ch/tocco/nice2/persist/hibernate/PersistenceService>`
it is possible to pass query hints in the form of a ``Map<String, ?>``.
:nice:`QueryHints <ch/tocco/nice2/persist/hibernate/query/QueryHints>` are additional information for the query builder which can lead to an optimized query.

Currently there is only one supported hint: ``QUERY_BY_KEYS``.

``QUERY_BY_KEYS`` defines all primary keys which might possibly be returned from the query. It is usually combined with a
``primaryKeyIn()`` condition.

The hints are passed to the :nice:`PredicateBuilder <ch/tocco/nice2/persist/hibernate/query/PredicateBuilder>`
which can use it build an optimized condition.
