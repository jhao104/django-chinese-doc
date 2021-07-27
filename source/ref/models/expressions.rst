=================
查询表达式
=================

.. currentmodule:: django.db.models

查询表达式是用作更新, 创建, 过滤, 排序, 注解和聚合的一部分值或计算. 这里(下文)有很多内置表达式可以帮助你完成查询. 表达式可以通过组合和嵌套来完成复杂的计算.

.. versionchanged:: 1.9

    添加了对在创建新模型实例时使用表达式的支持.

支持的算术运算
====================

Django支持在查询表达式中使用加, 减, 乘, 除, 求模, 幂运算, Python常量, 变量等.

一些例子
=============

.. code-block:: python

    from django.db.models import F, Count, Value
    from django.db.models.functions import Length, Upper

    # Find companies that have more employees than chairs.
    Company.objects.filter(num_employees__gt=F('num_chairs'))

    # Find companies that have at least twice as many employees
    # as chairs. Both the querysets below are equivalent.
    Company.objects.filter(num_employees__gt=F('num_chairs') * 2)
    Company.objects.filter(
        num_employees__gt=F('num_chairs') + F('num_chairs'))

    # How many chairs are needed for each company to seat all employees?
    >>> company = Company.objects.filter(
    ...    num_employees__gt=F('num_chairs')).annotate(
    ...    chairs_needed=F('num_employees') - F('num_chairs')).first()
    >>> company.num_employees
    120
    >>> company.num_chairs
    50
    >>> company.chairs_needed
    70

    # Create a new company using expressions.
    >>> company = Company.objects.create(name='Google', ticker=Upper(Value('goog')))
    # Be sure to refresh it if you need to access the field.
    >>> company.refresh_from_db()
    >>> company.ticker
    'GOOG'

    # Annotate models with an aggregated value. Both forms
    # below are equivalent.
    Company.objects.annotate(num_products=Count('products'))
    Company.objects.annotate(num_products=Count(F('products')))

    # Aggregates can contain complex computations also
    Company.objects.annotate(num_offerings=Count(F('products') + F('services')))

    # Expressions can also be used in order_by()
    Company.objects.order_by(Length('name').asc())
    Company.objects.order_by(Length('name').desc())


内置表达式
====================

.. note::

    这些表达式定义在 ``django.db.models.expressions`` 和 ``django.db.models.aggregates`` 中, 但是为了方便可以直接从 :mod:`django.db.models` 导入.

``F()`` 表达式
-------------------

.. class:: F

``F()`` 对象代表模型的字段或注释列的值. 它使得引用模型字段值并使用它们执行数据库操作成为可能, 而不用再把它们查询出来放到python内存中.

取而代之的是, Django使用 ``F()`` 对象生成描述数据库级所需操作的SQL表达式.

这可以通过一个例子很容易理解. 往常我们会这样做::

    # Tintin filed a news story!
    reporter = Reporters.objects.get(name='Tintin')
    reporter.stories_filed += 1
    reporter.save()

这里, 我们把 ``reporter.stories_filed`` 的值从数据库查询出来放到内存中, 然后再用python运算符操作它, 最后再把它保存到数据库. 但是我们还可以这样做::

    from django.db.models import F

    reporter = Reporters.objects.get(name='Tintin')
    reporter.stories_filed = F('stories_filed') + 1
    reporter.save()

看上去 ``reporter.stories_filed = F('stories_filed') + 1`` 是一个给实例属性赋值的普通Python操作, 事实上这是一个描述数据库操作的SQL结构.

当Django遇到一个 ``F()`` 的实例时, 它会重写标准的Python运算符来创建一个封装的SQL表达式; 在本例中, 它指示数据库增加 ``reporter.stories_filed`` 字段表示的数据库字段的值.

无论 ``reporter.stories_filed`` 的值是或曾是什么, Python不需要知道--它完全由数据库处理. 通过Django的 ``F()`` 类, Python所做的只是去创建SQL语句引用字段和描述操作.

要访问以这种方式保存的新值, 必须重新加载该对象::

   reporter = Reporters.objects.get(pk=reporter.pk)
   # Or, more succinctly:
   reporter.refresh_from_db()

除了如上所述在单个实例的操作中使用外, ``F()`` 表达式还可以与 ``update()`` 一起用于对象实例的 ``QuerySets``.
这将我们在上面使用的两个查询 ``get()`` 和 :meth:`~Model.save()` 减少到一个::

    reporter = Reporters.objects.filter(name='Tintin')
    reporter.update(stories_filed=F('stories_filed') + 1)

我们还可以使用 :meth:`~django.db.models.query.QuerySet.update()` 来批量增加多个对象的字段值.
这比将所有对象从数据库中存入Python, 然后循环增加每个对象的字段值并将每个对象保存回数据库要快得多::

    Reporter.objects.all().update(stories_filed=F('stories_filed') + 1)

``F()`` 表达式的效率上的优点主要体现在:

* 让数据库处理, 而不是Python来完成
* 减少某些操作所需的查询次数

.. _avoiding-race-conditions-using-f:

使用 ``F()`` 避免竞争条件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``F()`` 的另一个好处是让数据库去操作——而不是用Python更新字段的值, 这避免了 **竞争条件**.

假设有两个Python线程同时执行上面第一个例子中的代码, 一个线程可能是在另一个线程刚从数据库中获取完字段值后检索, 增加, 保存该字段值. 第二个线程保存的值则是基于原始值的. 第一个线程的工作将会丢失.

如果让数据库负责更新字段, 那么这个过程就更加健壮: 它会在执行 :meth:`~Model.save()` 或 ``update()`` 时根据数据库中字段的值更新字段, 而不是根据检索实例时的值更新字段.

``F()`` 赋值在 ``Model.save()`` 后持续存在
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

模型实例被保存后, 分配给模型字段的 ``F()`` 对象仍然存在, 并将应用于每个 :meth:`~Model.save()`. 例如::

    reporter = Reporters.objects.get(name='Tintin')
    reporter.stories_filed = F('stories_filed') + 1
    reporter.save()

    reporter.name = 'Tintin Jr.'
    reporter.save()

``stories_filed`` 将会被更新两次. 如果初始值是 ``1``, 那么最终值将是 ``3``.

在filters中使用 ``F()``
~~~~~~~~~~~~~~~~~~~~~~~~

``F()`` 在 ``QuerySet`` 过滤器中也非常有用, 它使得可以通过字段值而不是Python值来过滤一组对象变得可能.

详见 :ref:`在查询中使用 F() 表达式 <using-f-expressions-in-filters>`.

.. _using-f-with-annotations:

在注解中使用 ``F()``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``F()`` 可用于通过将不同字段与算术组合, 在模型上创建动态字段::

    company = Company.objects.annotate(
        chairs_needed=F('num_employees') - F('num_chairs'))

如果组合的字段是不同类型的, 需要告诉Django返回的字段类型. 由于 ``F()`` 不支持 ``output_field`` 参数, 你需要用 :class:`ExpressionWrapper` 来封装表达式::

    from django.db.models import DateTimeField, ExpressionWrapper, F

    Ticket.objects.annotate(
        expires=ExpressionWrapper(
            F('active_at') + F('duration'), output_field=DateTimeField()))

当引用的是关联字段比如 ``ForeignKey`` 时, ``F()`` 返回的是主键值而不是关联模型实例::

    >> car = Company.objects.annotate(built_by=F('manufacturer'))[0]
    >> car.manufacturer
    <Manufacturer: Toyota>
    >> car.built_by
    3

.. _func-expressions:

``Func()`` 表达式
----------------------

``Func()`` expressions are the base type of all expressions that involve
database functions like ``COALESCE`` and ``LOWER``, or aggregates like ``SUM``.
They can be used directly::

    from django.db.models import Func, F

    queryset.annotate(field_lower=Func(F('field'), function='LOWER'))

or they can be used to build a library of database functions::

    class Lower(Func):
        function = 'LOWER'

    queryset.annotate(field_lower=Lower('field'))

But both cases will result in a queryset where each model is annotated with an
extra attribute ``field_lower`` produced, roughly, from the following SQL::

    SELECT
        ...
        LOWER("db_table"."field") as "field_lower"

See :doc:`database-functions` for a list of built-in database functions.

The ``Func`` API is as follows:

.. class:: Func(*expressions, **extra)

    .. attribute:: function

        A class attribute describing the function that will be generated.
        Specifically, the ``function`` will be interpolated as the ``function``
        placeholder within :attr:`template`. Defaults to ``None``.

    .. attribute:: template

        A class attribute, as a format string, that describes the SQL that is
        generated for this function. Defaults to
        ``'%(function)s(%(expressions)s)'``.

        If you're constructing SQL like ``strftime('%W', 'date')`` and need a
        literal ``%`` character in the query, quadruple it (``%%%%``) in the
        ``template`` attribute because the string is interpolated twice: once
        during the template interpolation in ``as_sql()`` and once in the SQL
        interpolation with the query parameters in the database cursor.

    .. attribute:: arg_joiner

        A class attribute that denotes the character used to join the list of
        ``expressions`` together. Defaults to ``', '``.

    .. attribute:: arity

        .. versionadded:: 1.10

        A class attribute that denotes the number of arguments the function
        accepts. If this attribute is set and the function is called with a
        different number of expressions, ``TypeError`` will be raised. Defaults
        to ``None``.

    .. method:: as_sql(compiler, connection, function=None, template=None, arg_joiner=None, **extra_context)

        Generates the SQL for the database function.

        The ``as_vendor()`` methods should use the ``function``, ``template``,
        ``arg_joiner``, and any other ``**extra_context`` parameters to
        customize the SQL as needed. For example:

        .. snippet::
            :filename: django/db/models/functions.py

            class ConcatPair(Func):
                ...
                function = 'CONCAT'
                ...

                def as_mysql(self, compiler, connection):
                    return super(ConcatPair, self).as_sql(
                        compiler, connection,
                        function='CONCAT_WS',
                        template="%(function)s('', %(expressions)s)",
                    )

        .. versionchanged:: 1.10

            Support for the ``arg_joiner`` and ``**extra_context`` parameters
            was added.

The ``*expressions`` argument is a list of positional expressions that the
function will be applied to. The expressions will be converted to strings,
joined together with ``arg_joiner``, and then interpolated into the ``template``
as the ``expressions`` placeholder.

Positional arguments can be expressions or Python values. Strings are
assumed to be column references and will be wrapped in ``F()`` expressions
while other values will be wrapped in ``Value()`` expressions.

The ``**extra`` kwargs are ``key=value`` pairs that can be interpolated
into the ``template`` attribute. The ``function``, ``template``, and
``arg_joiner`` keywords can be used to replace the attributes of the same name
without having to define your own class. ``output_field`` can be used to define
the expected return type.

``Aggregate()`` expressions
---------------------------

An aggregate expression is a special case of a :ref:`Func() expression
<func-expressions>` that informs the query that a ``GROUP BY`` clause
is required. All of the :ref:`聚合函数 <aggregation-functions>`,
like ``Sum()`` and ``Count()``, inherit from ``Aggregate()``.

Since ``Aggregate``\s are expressions and wrap expressions, you can represent
some complex computations::

    from django.db.models import Count

    Company.objects.annotate(
        managers_required=(Count('num_employees') / 4) + Count('num_managers'))

The ``Aggregate`` API is as follows:

.. class:: Aggregate(expression, output_field=None, **extra)

    .. attribute:: template

        A class attribute, as a format string, that describes the SQL that is
        generated for this aggregate. Defaults to
        ``'%(function)s( %(expressions)s )'``.

    .. attribute:: function

        A class attribute describing the aggregate function that will be
        generated. Specifically, the ``function`` will be interpolated as the
        ``function`` placeholder within :attr:`template`. Defaults to ``None``.

The ``expression`` argument can be the name of a field on the model, or another
expression. It will be converted to a string and used as the ``expressions``
placeholder within the ``template``.

The ``output_field`` argument requires a model field instance, like
``IntegerField()`` or ``BooleanField()``, into which Django will load the value
after it's retrieved from the database. Usually no arguments are needed when
instantiating the model field as any arguments relating to data validation
(``max_length``, ``max_digits``, etc.) will not be enforced on the expression's
output value.

Note that ``output_field`` is only required when Django is unable to determine
what field type the result should be. Complex expressions that mix field types
should define the desired ``output_field``. For example, adding an
``IntegerField()`` and a ``FloatField()`` together should probably have
``output_field=FloatField()`` defined.

The ``**extra`` kwargs are ``key=value`` pairs that can be interpolated
into the ``template`` attribute.

Creating your own Aggregate Functions
-------------------------------------

Creating your own aggregate is extremely easy. At a minimum, you need
to define ``function``, but you can also completely customize the
SQL that is generated. Here's a brief example::

    from django.db.models import Aggregate

    class Count(Aggregate):
        # supports COUNT(distinct field)
        function = 'COUNT'
        template = '%(function)s(%(distinct)s%(expressions)s)'

        def __init__(self, expression, distinct=False, **extra):
            super(Count, self).__init__(
                expression,
                distinct='DISTINCT ' if distinct else '',
                output_field=IntegerField(),
                **extra
            )


``Value()`` expressions
-----------------------

.. class:: Value(value, output_field=None)


A ``Value()`` object represents the smallest possible component of an
expression: a simple value. When you need to represent the value of an integer,
boolean, or string within an expression, you can wrap that value within a
``Value()``.

You will rarely need to use ``Value()`` directly. When you write the expression
``F('field') + 1``, Django implicitly wraps the ``1`` in a ``Value()``,
allowing simple values to be used in more complex expressions. You will need to
use ``Value()`` when you want to pass a string to an expression. Most
expressions interpret a string argument as the name of a field, like
``Lower('name')``.

The ``value`` argument describes the value to be included in the expression,
such as ``1``, ``True``, or ``None``. Django knows how to convert these Python
values into their corresponding database type.

The ``output_field`` argument should be a model field instance, like
``IntegerField()`` or ``BooleanField()``, into which Django will load the value
after it's retrieved from the database. Usually no arguments are needed when
instantiating the model field as any arguments relating to data validation
(``max_length``, ``max_digits``, etc.) will not be enforced on the expression's
output value.

``ExpressionWrapper()`` expressions
-----------------------------------

.. class:: ExpressionWrapper(expression, output_field)

``ExpressionWrapper`` simply surrounds another expression and provides access
to properties, such as ``output_field``, that may not be available on other
expressions. ``ExpressionWrapper`` is necessary when using arithmetic on
``F()`` expressions with different types as described in
:ref:`using-f-with-annotations`.

Conditional expressions
-----------------------

Conditional expressions allow you to use :keyword:`if` ... :keyword:`elif` ...
:keyword:`else` logic in queries. Django natively supports SQL ``CASE``
expressions. For more details see :doc:`conditional-expressions`.

Raw SQL expressions
-------------------

.. currentmodule:: django.db.models.expressions

.. class:: RawSQL(sql, params, output_field=None)

Sometimes database expressions can't easily express a complex ``WHERE`` clause.
In these edge cases, use the ``RawSQL`` expression. For example::

    >>> from django.db.models.expressions import RawSQL
    >>> queryset.annotate(val=RawSQL("select col from sometable where othercol = %s", (someparam,)))

These extra lookups may not be portable to different database engines (because
you're explicitly writing SQL code) and violate the DRY principle, so you
should avoid them if possible.

.. warning::

    You should be very careful to escape any parameters that the user can
    control by using ``params`` in order to protect against :ref:`SQL injection
    attacks <sql-injection-protection>`. ``params`` is a required argument to
    force you to acknowledge that you're not interpolating your SQL with user
    provided data.

.. currentmodule:: django.db.models

Technical Information
=====================

Below you'll find technical implementation details that may be useful to
library authors. The technical API and examples below will help with
creating generic query expressions that can extend the built-in functionality
that Django provides.

Expression API
--------------

Query expressions implement the :ref:`query expression API <query-expression>`,
but also expose a number of extra methods and attributes listed below. All
query expressions must inherit from ``Expression()`` or a relevant
subclass.

When a query expression wraps another expression, it is responsible for
calling the appropriate methods on the wrapped expression.

.. class:: Expression

    .. attribute:: contains_aggregate

        Tells Django that this expression contains an aggregate and that a
        ``GROUP BY`` clause needs to be added to the query.

    .. method:: resolve_expression(query=None, allow_joins=True, reuse=None, summarize=False, for_save=False)

        Provides the chance to do any pre-processing or validation of
        the expression before it's added to the query. ``resolve_expression()``
        must also be called on any nested expressions. A ``copy()`` of ``self``
        should be returned with any necessary transformations.

        ``query`` is the backend query implementation.

        ``allow_joins`` is a boolean that allows or denies the use of
        joins in the query.

        ``reuse`` is a set of reusable joins for multi-join scenarios.

        ``summarize`` is a boolean that, when ``True``, signals that the
        query being computed is a terminal aggregate query.

    .. method:: get_source_expressions()

        Returns an ordered list of inner expressions. For example::

          >>> Sum(F('foo')).get_source_expressions()
          [F('foo')]

    .. method:: set_source_expressions(expressions)

        Takes a list of expressions and stores them such that
        ``get_source_expressions()`` can return them.

    .. method:: relabeled_clone(change_map)

        Returns a clone (copy) of ``self``, with any column aliases relabeled.
        Column aliases are renamed when subqueries are created.
        ``relabeled_clone()`` should also be called on any nested expressions
        and assigned to the clone.

        ``change_map`` is a dictionary mapping old aliases to new aliases.

        Example::

          def relabeled_clone(self, change_map):
              clone = copy.copy(self)
              clone.expression = self.expression.relabeled_clone(change_map)
              return clone

    .. method:: convert_value(self, value, expression, connection, context)

        A hook allowing the expression to coerce ``value`` into a more
        appropriate type.

    .. method:: get_group_by_cols()

        Responsible for returning the list of columns references by
        this expression. ``get_group_by_cols()`` should be called on any
        nested expressions. ``F()`` objects, in particular, hold a reference
        to a column.

    .. method:: asc()

        Returns the expression ready to be sorted in ascending order.

    .. method:: desc()

        Returns the expression ready to be sorted in descending order.

    .. method:: reverse_ordering()

        Returns ``self`` with any modifications required to reverse the sort
        order within an ``order_by`` call. As an example, an expression
        implementing ``NULLS LAST`` would change its value to be
        ``NULLS FIRST``. Modifications are only required for expressions that
        implement sort order like ``OrderBy``. This method is called when
        :meth:`~django.db.models.query.QuerySet.reverse()` is called on a
        queryset.

Writing your own Query Expressions
----------------------------------

You can write your own query expression classes that use, and can integrate
with, other query expressions. Let's step through an example by writing an
implementation of the ``COALESCE`` SQL function, without using the built-in
:ref:`Func() expressions <func-expressions>`.

The ``COALESCE`` SQL function is defined as taking a list of columns or
values. It will return the first column or value that isn't ``NULL``.

We'll start by defining the template to be used for SQL generation and
an ``__init__()`` method to set some attributes::

  import copy
  from django.db.models import Expression

  class Coalesce(Expression):
      template = 'COALESCE( %(expressions)s )'

      def __init__(self, expressions, output_field):
        super(Coalesce, self).__init__(output_field=output_field)
        if len(expressions) < 2:
            raise ValueError('expressions must have at least 2 elements')
        for expression in expressions:
            if not hasattr(expression, 'resolve_expression'):
                raise TypeError('%r is not an Expression' % expression)
        self.expressions = expressions

We do some basic validation on the parameters, including requiring at least
2 columns or values, and ensuring they are expressions. We are requiring
``output_field`` here so that Django knows what kind of model field to assign
the eventual result to.

Now we implement the pre-processing and validation. Since we do not have
any of our own validation at this point, we just delegate to the nested
expressions::

    def resolve_expression(self, query=None, allow_joins=True, reuse=None, summarize=False, for_save=False):
        c = self.copy()
        c.is_summary = summarize
        for pos, expression in enumerate(self.expressions):
            c.expressions[pos] = expression.resolve_expression(query, allow_joins, reuse, summarize, for_save)
        return c

Next, we write the method responsible for generating the SQL::

    def as_sql(self, compiler, connection, template=None):
        sql_expressions, sql_params = [], []
        for expression in self.expressions:
            sql, params = compiler.compile(expression)
            sql_expressions.append(sql)
            sql_params.extend(params)
        template = template or self.template
        data = {'expressions': ','.join(sql_expressions)}
        return template % data, params

    def as_oracle(self, compiler, connection):
        """
        Example of vendor specific handling (Oracle in this case).
        Let's make the function name lowercase.
        """
        return self.as_sql(compiler, connection, template='coalesce( %(expressions)s )')

``as_sql()`` methods can support custom keyword arguments, allowing
``as_vendorname()`` methods to override data used to generate the SQL string.
Using ``as_sql()`` keyword arguments for customization is preferable to
mutating ``self`` within ``as_vendorname()`` methods as the latter can lead to
errors when running on different database backends. If your class relies on
class attributes to define data, consider allowing overrides in your
``as_sql()`` method.

We generate the SQL for each of the ``expressions`` by using the
``compiler.compile()`` method, and join the result together with commas.
Then the template is filled out with our data and the SQL and parameters
are returned.

We've also defined a custom implementation that is specific to the Oracle
backend. The ``as_oracle()`` function will be called instead of ``as_sql()``
if the Oracle backend is in use.

Finally, we implement the rest of the methods that allow our query expression
to play nice with other query expressions::

    def get_source_expressions(self):
        return self.expressions

    def set_source_expressions(self, expressions):
        self.expressions = expressions

Let's see how it works::

    >>> from django.db.models import F, Value, CharField
    >>> qs = Company.objects.annotate(
    ...    tagline=Coalesce([
    ...        F('motto'),
    ...        F('ticker_name'),
    ...        F('description'),
    ...        Value('No Tagline')
    ...        ], output_field=CharField()))
    >>> for c in qs:
    ...     print("%s: %s" % (c.name, c.tagline))
    ...
    Google: Do No Evil
    Apple: AAPL
    Yahoo: Internet Company
    Django Software Foundation: No Tagline

Adding support in third-party database backends
-----------------------------------------------

If you're using a database backend that uses a different SQL syntax for a
certain function, you can add support for it by monkey patching a new method
onto the function's class.

Let's say we're writing a backend for Microsoft's SQL Server which uses the SQL
``LEN`` instead of ``LENGTH`` for the :class:`~functions.Length` function.
We'll monkey patch a new method called ``as_sqlserver()`` onto the ``Length``
class::

    from django.db.models.functions import Length

    def sqlserver_length(self, compiler, connection):
        return self.as_sql(compiler, connection, function='LEN')

    Length.as_sqlserver = sqlserver_length

You can also customize the SQL using the ``template`` parameter of ``as_sql()``.

We use ``as_sqlserver()`` because ``django.db.connection.vendor`` returns
``sqlserver`` for the backend.

Third-party backends can register their functions in the top level
``__init__.py`` file of the backend package or in a top level ``expressions.py``
file (or package) that is imported from the top level ``__init__.py``.

For user projects wishing to patch the backend that they're using, this code
should live in an :meth:`AppConfig.ready()<django.apps.AppConfig.ready>` method.
