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

``Func()`` 表达式是所有涉及数据库函数的表达式基类, 例如 ``COALESCE`` 和 ``LOWER``, 还包括聚合函数如 ``SUM``. 可以通过下面方式直接使用::

    from django.db.models import Func, F

    queryset.annotate(field_lower=Func(F('field'), function='LOWER'))

也可以用来构建数据库函数库::

    class Lower(Func):
        function = 'LOWER'

    queryset.annotate(field_lower=Lower('field'))

这两种写法都会产生一个查询集, 其中的每个模型对象都会用大致类似下面的SQL生成一个额外属性 ``field_lower``::

    SELECT
        ...
        LOWER("db_table"."field") as "field_lower"

参见 :doc:`database-functions` 查看内置的数据库函数列表.

``Func`` API如下:

.. class:: Func(*expressions, **extra)

    .. attribute:: function

        描述将要生成的函数的类属性. 具体来说, ``function`` 将被插入 :attr:`template` 中的 ``function`` 占位符. 默认为 ``None``.

    .. attribute:: template

        类属性, 格式化字符串, 描述该函数要生成的SQL. 默认值为 ``'%(function)s(%(expressions)s)'``.

        如果你想构造类似 ``strftime('%W', 'date')`` 这样的SQL. 需要在查询中使用 ``%`` 字符, 则需要在 ``template`` 属性中使用四个(``%%%%``),
        因为这个字符串会被插值两次, 一次是在模板 ``as_sql()`` 时插值, 另一次是在数据库查询SQL中参数的插值.

    .. attribute:: arg_joiner

        一个表示join ``expressions`` 列表的字符. 默认为 ``', '``.

    .. attribute:: arity

        .. versionadded:: 1.10

        类属性, 表示函数接受的参数数量. 如果设置了该属性, 并且调用时使用了不同数量的参数将引发 ``TypeError`` 异常. 默认值为 ``None``.

    .. method:: as_sql(compiler, connection, function=None, template=None, arg_joiner=None, **extra_context)

        生成数据库函数SQL.

        ``as_vendor()`` 方法会使用 ``function``, ``template``, ``arg_joiner``, 和 ``**extra_context`` 参数根据需要定制SQL. 例如:

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

            新增参数 ``arg_joiner`` 和 ``**extra_context``.

``*expressions`` 参数是将要应用到函数的表达式列表的位置参数. 表达式将被 ``arg_joiner`` 连接成字符串, 然后插入 ``template`` 中的 ``expressions`` 占位符.

位置参数可以是表达式和Python值. 字符串会被认为是列引用, 并将其封装在 ``F()`` 表达式中. 其他值将被封装在 ``Value()`` 表达式中.

``**extra`` 是插入 ``template`` 的 ``key=value`` 对. ``function``, ``template`` 和 ``arg_joiner`` 用于替换相同名称的属性, 而无需定义自己的类. ``output_field`` 用于定义期望的返回类型.

``Aggregate()`` 表达式
---------------------------

聚合表达式是一种特殊的 :ref:`Func() 表达式<func-expressions>`,  它告知查询需要一个 ``GROUP BY`` 子句.
所有 :ref:`聚合函数 <aggregation-functions>` 例如 ``Sum()`` 和 ``Count()`` 都继承至 ``Aggregate()``.

由于 ``Aggregate`` 是表达式或封装的表达式, 因此可以做一些复杂的计算::

    from django.db.models import Count

    Company.objects.annotate(
        managers_required=(Count('num_employees') / 4) + Count('num_managers'))

``Aggregate`` API如下:

.. class:: Aggregate(expression, output_field=None, **extra)

    .. attribute:: template

        类属性, 格式化字符串, 描述要生成的聚合SQL. 默认为 ``'%(function)s( %(expressions)s )'``.

    .. attribute:: function

        类属性, 描述要生成的聚合函数. 具体来说, ``function`` 将被插入到 :attr:`template` 中的 ``function`` 占位符. 默认为 ``None``.

``expression`` 参数可以是模型上的字段名称或其他表达式. 它将会被转换成字符串插入到 ``template`` 中的 ``expressions`` 占位符.

``output_field`` 参数接收一个模型字段实例, 如 ``output_field()`` 和 ``BooleanField()``.
Django将从数据库中检索的值装载到其中. 通常在实例化模型字段时不需要任何参数. 因为所有数据验证有关的参数(``max_length``, ``max_digits``, etc.)都不会在表达式的输出值上执行.

注意, 只有当Django无法确定结果应该是什么字段类型时才需要 ``output_field``. 混合字段类型的复杂表达式应定义 ``output_field``.
例如, 将一个 ``IntegerField()`` 和 ``FloatField()`` 相加, 就可以定义 ``output_field=FloatField()``.

``**extra`` 关键字是 ``key=value`` 对, 用来插入到 ``template`` 属性中.

自定义聚合函数
-------------------------------------

自定义聚合函数非常容易. 你可以定义 ``function`` 也可以完全自定义生成SQL. 下面是一个简单例子::

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


``Value()`` 表达式
-----------------------

.. class:: Value(value, output_field=None)

``Value()`` 对象是表达式中最小的组件: 普通值. 当需要在表达式中表示整数, 布尔或字符串的值时, 可以使用 ``Value()`` 封装该值.

很少需要直接使用 ``Value()``. 在表达式 ``F('field') + 1`` 中, Django会隐式地将1用 ``Value()`` 封装起来, 这样可以将普通值使用在更复杂的表达式中.
但是当你想要将字符串传递给表达式时需要使用 ``Value()``. 因为大多数表达式会将字符串参数解释为字段的名称, 如 ``Lower('name')``.

``value`` 参数表示包含在表达式中的值, 例如 ``1``, ``True`` 或 ``None``. Django会自动将Python值转换为对应的数据库类型.

``output_field`` 参数接收一个模型字段实例, 如 ``output_field()`` 和 ``BooleanField()``.
Django将从数据库中检索的值装载到其中. 通常在实例化模型字段时不需要任何参数. 因为所有数据验证有关的参数(``max_length``, ``max_digits``, etc.)都不会在表达式的输出值上执行.

``ExpressionWrapper()`` 表达式
-----------------------------------

.. class:: ExpressionWrapper(expression, output_field)

``ExpressionWrapper`` 用来封装表达式并提供 ``output_field`` 等可访问属性, 这些属性可能在其他表达式上无法使用.
当使用 ``F()`` 对不同类型表达式进行算术运算时 ``ExpressionWrapper`` 是必要的, 详见 :ref:`using-f-with-annotations`.

条件表达式
-----------------------

条件表达式使你可以在查询中可以使用 :keyword:`if` ... :keyword:`elif` ...
:keyword:`else` 逻辑. Django原生支持SQL的 ``CASE`` 表达式. 详见 :doc:`conditional-expressions`.

原生SQL表达式
-------------------

.. currentmodule:: django.db.models.expressions

.. class:: RawSQL(sql, params, output_field=None)

有些时候数据库表达式不太容易表达式比较复杂的 ``WHERE`` 子句. 在这些边缘情况下, 可以使用 ``RawSQL`` 表达式. 例如::

    >>> from django.db.models.expressions import RawSQL
    >>> queryset.annotate(val=RawSQL("select col from sometable where othercol = %s", (someparam,)))

这些额外的查询可能无法移植到不同的数据库引擎中(因为存在硬编码的SQL代码), 并且违反DRY原则, 因此应该尽可能避免使用它们.

.. warning::

    为了防止 :ref:`SQL注入攻击 <sql-injection-protection>`, 你应该小心地避免使用用户可以通过 ``params`` 控制的任何参数. ``params`` 是一个必传的参数, 用于强制确认没有使用用户提供的数据插入SQL.

.. currentmodule:: django.db.models

技术信息
=====================

在下面是可能对库作者有用的技术细节. 下面的技术API和示例将有助于创建可以扩展Django内置功能的通用查询表达式.

表达式API
--------------

查询表达式实现了 :ref:`查询表达式API <query-expression>`, 同时也提供了下面列出的一些额外方法和属性. 所有查询表达式都必须继承于 ``Expression()`` 或相关子类.

如果查询表达式封装另一个表达式, 它负责在被封装的表达式上调用相应的方法.

.. class:: Expression

    .. attribute:: contains_aggregate

        告诉Django该表达式包含聚合, 需要将 ``GROUP BY`` 子句添加到查询中.

    .. method:: resolve_expression(query=None, allow_joins=True, reuse=None, summarize=False, for_save=False)

        提供在将表达式添加到查询之前对其进行预处理和验证的机会. 所有嵌套表达式都必须调用 ``resolve_expression()``. 返回一个 ``self`` 的 ``copy()``, 并进行必要的转换.

        ``query`` 是后端查询实现.

        ``allow_joins`` 是一个允许或拒绝在查询中使用join的布尔值.

        ``reuse`` 是用于多join场景的一组可重用join.

        ``summarize`` 是一个布尔值, 当 ``True`` 时表示正在计算的查询是终端聚合查询.

    .. method:: get_source_expressions()

        返回内部表达式的有序列表. 例如::

          >>> Sum(F('foo')).get_source_expressions()
          [F('foo')]

    .. method:: set_source_expressions(expressions)

        接收一个表达式列表将其存储, 以便 ``get_source_expressions()`` 能够返回他们.

    .. method:: relabeled_clone(change_map)

        返回 ``self`` 的clone(副本), 并重新标记所有列别名. 创建子查询时, 列别名会被重新命名. ``relabeled_clone()`` 也应该对所有嵌套的表达式进行调用并分配给副本.

        ``change_map`` 是将旧别名映射到新别名的字典.

        例如::

          def relabeled_clone(self, change_map):
              clone = copy.copy(self)
              clone.expression = self.expression.relabeled_clone(change_map)
              return clone

    .. method:: convert_value(self, value, expression, connection, context)

        允许表达式将 ``value`` 强制转换为更适当类型的钩子.

    .. method:: get_group_by_cols()

        负责返回该表达式的列引用列表. ``get_group_by_cols()`` 应在所有嵌套的表达式上调用. 尤其是持有列的引用的 ``F()`` 对象.

    .. method:: asc()

        返回准备按升序排序的表达式.

    .. method:: desc()

        返回准备按降序排序的表达式.

    .. method:: reverse_ordering()

        返回 ``self`` 并进行包括以 ``order_by`` 调用中相反顺序排序的所有修改. 例如, 实现 ``NULLS LAST`` 的表达式将其值更改为 ``NULLS FIRST``.
        只有实现排序顺序如 ``OrderBy`` 的表达式才需要修改. 在queryset上调用 :meth:`~django.db.models.query.QuerySet.reverse()` 时调用此方法.

自定义查询表达式
----------------------------------

可以编写自己的查询表达式类, 它们可以使用和集成其他查询表达式.
下面例子演示不使用 :ref:`Func()表达式 <func-expressions>` 来实现SQL的 ``COALESCE``  函数.

SQL的 ``COALESCE`` 函数接收一组列或值. 它返回不为 ``NULL`` 的第一个列或值.

首先定义用于生成SQL的模板和 ``__init__()`` 方法来设置一些属性::

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

我们对参数进行一些基本验证, 包括至少需要两个列或值, 并确保它们是表达式.
我们在这里要求 ``output_field``, 以便Django知道要将最终结果分配给什么样的模型字段.

下面实现预处理和验证. 由于此时没有任何自己的验证, 所以这里委托给嵌套表达式::

    def resolve_expression(self, query=None, allow_joins=True, reuse=None, summarize=False, for_save=False):
        c = self.copy()
        c.is_summary = summarize
        for pos, expression in enumerate(self.expressions):
            c.expressions[pos] = expression.resolve_expression(query, allow_joins, reuse, summarize, for_save)
        return c

接下来, 编写负责生成SQL的方法::

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

``as_sql()`` 方法可以支持自定义关键字参数, 允许 ``as_vendorname()`` 方法覆盖用于生成SQL字符串的数据.
使用 ``as_sql()`` 定制的关键字参数比在 ``as_vendorname()`` 方法中突变 ``self`` 更好,
因为后者在不同的数据库后端发生错误. 如果您的类依赖于类属性来定义数据, 可以考虑在 ``as_sql()`` 方法中允许覆盖.

我们使用 ``compiler.compile()`` 方法为每个 ``expressions`` 生成SQL, 并用逗号拼接. 然后在模板中填入数据, 并返回SQL和参数.

我们还针对特定数据后端Oracle做了的自定义实现. 如果使用Oracle后端则将调用 ``as_oracle()`` 函数, 而不是 ``as_sql()``.

最后, 我们实现允许使查询表达式能够与其他查询表达式很好地配合方法::

    def get_source_expressions(self):
        return self.expressions

    def set_source_expressions(self, expressions):
        self.expressions = expressions

让我们看看它是如何工作的::

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

添加对三方数据库后端支持
-----------------------------------------------

如果你使用的数据库后端对某个函数使用了不同的SQL语法, 你可以通过在函数的类上添加一个新的方法来增加对它的支持.

比如说, 我们为微软的SQLServer编写一个后端, 它使用SQL的 ``LEN`` 而不是 ``LENGTH`` 来实现 :class:`~functions.Length`  函数.
我们将把一个名为 ``as_sqlserver()`` 的新方法添加到 ``Length`` 类上::

    from django.db.models.functions import Length

    def sqlserver_length(self, compiler, connection):
        return self.as_sql(compiler, connection, function='LEN')

    Length.as_sqlserver = sqlserver_length

也可以使用 ``as_sql()`` 的 ``template`` 参数自定义SQL.

我们使用 ``as_sqlserver()`` 是因为 ``django.db.connection.vendor`` 返回 ``sqlserver`` 作为后端.

第三方后端可以在后端程序包的顶级 ``__init__.py`` 文件或从顶级 ``__init__.py`` 导入的顶级 ``expressions.py`` 文件(或程序包)中注册其函数.

对于希望给自己正在使用的后端打补丁的用户项目来说, 这段代码应该存在于 :meth:`AppConfig.ready()<django.apps.AppConfig.ready>` 方法中.
