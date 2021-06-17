==========================
执行原生SQL查询
==========================

.. currentmodule:: django.db.models

在 :doc:`模型查询APIs </topics/db/queries>` 不能满足使用时可以使用原生SQL查询.
Django提供了两种执行原生SQL查询的方式: 一是使用 :meth:`Manager.raw()`  `执行原生查询返回模型实例`__,
二是不使用模型层 `直接执行自定义SQL`__.

__ `执行原生查询`_
__ `直接执行自定义SQL`_

.. warning::

    使用原生SQL时要格外小心. 确保将 ``params`` 中用户传入的参数进行转义, 避免遭到SQL注入攻击.
    详见 :ref:`SQL注入保护  <sql-injection-protection>`.

.. _executing-raw-queries:

执行原生查询
======================

管理器方法 ``raw()`` 用于执行原生SQL查询返回模型实例:

.. method:: Manager.raw(raw_query, params=None, translations=None)

该方法接收原生SQL语句并执行, 返回一个
``django.db.models.query.RawQuerySet`` 实例. 该 ``RawQuerySet`` 实例可以像
:class:`~django.db.models.query.QuerySet` 那样通过迭代获取实例对象.

下面用具体例子来解释下. 假设有如下模型::

    class Person(models.Model):
        first_name = models.CharField(...)
        last_name = models.CharField(...)
        birth_date = models.DateField(...)

可以像这样执行自定义SQL语句::

    >>> for p in Person.objects.raw('SELECT * FROM myapp_person'):
    ...     print(p)
    John Smith
    Jane Jones

当然这个例子并不是很贴切, 因为它和 ``Person.objects.all()`` 功能一样. 但是 ``raw()`` 有很多额外选项使得它非常强大.

.. admonition:: 模型表名

    上面例子中 ``Person`` 表的名称从何而来?

    默认情况下, Django通过将模型的"app label"(即在 ``manage.py startapp`` 中使用的名称)和模型类名用下划线拼接来推算模型表名称.
    在本例中, 我们假定 ``Person`` 模型位于一个叫做 ``myapp`` 的应用中, 这样模型的表名就是 ``myapp_person``.

    更多信息详见 :attr:`~Options.db_table` 选项文档, 它可以自定义表名.

.. warning::

    传入 ``.raw()`` 的SQL语句不会被检查. Django默认语句会返回一组行, 但没有强制要求.
    但如果该查询没有返回这些数据, 则会导致一些(神秘)错误.

.. warning::

    在MySQL上执行查询时, 要小心在类型不一样时MySQL的静默强制类型可能导致意想不到的事情发生.
    比如在一个字符串类型的字段上查询一个整数类型的值. MySQL会在比较前强制把每个值的类型转成整数.
    例如, 字段包含值 ``'abc'`` 和 ``'def'``, 在查询 ``WHERE mycolumn=0`` 时这两行都会匹配上.
    要防止这种情况, 在查询中使用值之前要做好正确的类型转换.

.. warning::

    虽然 ``RawQuerySet`` 实例可以像 :class:`~django.db.models.query.QuerySet` 一样被迭代,
    但 ``RawQuerySet`` 并没有实现 ``QuerySet`` 上的所有方法. 比如,
    ``__bool__()`` 和 ``__len__()`` 方法 ``RawQuerySet`` 就没有实现,
    所以 ``RawQuerySet`` 实例的布尔值始终为 ``True``.  ``RawQuerySet``
    没有实现它们的原因是, 在没有内部缓存的情况下会导致性能下降, 而且增加内部缓存不向后兼容.

将查询字段映射为模型字段
------------------------------------

``raw()`` 方法会自动将查询字段映射到模型字段.

这不受查询字段顺序的影响. 换句话说, 下面两种查询结果是相同的::

    >>> Person.objects.raw('SELECT id, first_name, last_name, birth_date FROM myapp_person')
    ...
    >>> Person.objects.raw('SELECT last_name, birth_date, first_name, id FROM myapp_person')
    ...

匹配是根据名字来的. 这意味着你可以通过SQL的 ``AS`` 子句来将查询字段映射到模型字段.
因此如果在其他表中存在 ``Person`` 数据, 可以通过该访问来将其转换成 ``Person`` 实例::

    >>> Person.objects.raw('''SELECT first AS first_name,
    ...                              last AS last_name,
    ...                              bd AS birth_date,
    ...                              pk AS id,
    ...                       FROM some_other_table''')

只要名字能对应上模型的实例就会被正确创建.

也可以使用 ``raw()`` 的 ``translations`` 参数来将查询的字段名字映射到模型字段名字, 该参数是一个字典.
例如, 上面的查询可以写成这样::

    >>> name_map = {'first': 'first_name', 'last': 'last_name', 'bd': 'birth_date', 'pk': 'id'}
    >>> Person.objects.raw('SELECT * FROM some_other_table', translations=name_map)

索引查询
-------------

``raw()`` 方法支持索引访问, 比如查询第一条记录可以这样写::

    >>> first_person = Person.objects.raw('SELECT * FROM myapp_person')[0]

然而, 索引和切片并不是在数据库层级实现的. 如果 ``Person`` 在数据库中的数据量非常大, 最好将此类操作放在SQL中::

    >>> first_person = Person.objects.raw('SELECT * FROM myapp_person LIMIT 1')[0]

延迟加载字段
----------------------

查询时也可以省略部分字段::

    >>> people = Person.objects.raw('SELECT id, first_name FROM myapp_person')

该查询返回的 ``Person`` 对象是一个延迟模型实例
(见 :meth:`~django.db.models.query.QuerySet.defer()`). 这样查询时被省略的字段会在使用时加载. 例如::

    >>> for p in Person.objects.raw('SELECT id, first_name FROM myapp_person'):
    ...     print(p.first_name, # 这是开始时查询得到的
    ...           p.last_name) # 这是被使用时延迟加载的
    ...
    John Smith
    Jane Jones

表面上看起来这个查询同时查出了first name和last name. 然而这个例子中实际上执行了3次查询.
只有first names是通过raw()查询出来的 --
last names 是在被print时延迟加载的.

除了主键字段其他字段都是可以省略的. 主键字段是Django用来识别模型实例的, 因此原生查询必须包含主键.
如果没有则会引发 ``InvalidQuery`` 异常.

添加注解
------------------

查询时也可以使用模型中没有的字段. 例如, 使用 `PostgreSQL的 age() 函数`__ 查询数据库中一组人并带有age字段::

    >>> people = Person.objects.raw('SELECT *, age(birth_date) AS age FROM myapp_person')
    >>> for p in people:
    ...     print("%s is %s." % (p.first_name, p.age))
    John is 37.
    Jane is 42.
    ...

__ https://www.postgresql.org/docs/current/static/functions-datetime.html

传递参数给 ``raw()``
---------------------------------

通过给 ``raw()`` 传递 ``params`` 参数, 执行参数化查询::

    >>> lname = 'Doe'
    >>> Person.objects.raw('SELECT * FROM myapp_person WHERE last_name = %s', [lname])

``params`` 是一组参数列表或字典. 列表参数被用于替换查询语句中的 ``%s`` 占位符,
字典参数被用于替换查询语句中的 ``%(key)s`` 占位符(``key`` 替换成字典中对应key的值),
不论使用哪个数据库引擎这些占位符会被  ``params`` 参数的值替换.

.. note::

   SQLite不支持字典参数; 使用SQLite只能使用列表参数.

.. warning::

    **不要在原生查询语句上使用字符串格式化!**

    它类似于这种样子::

        >>> query = 'SELECT * FROM myapp_person WHERE last_name = %s' % lname
        >>> Person.objects.raw(query)

    **千万不要这样做.**

    使用 ``params`` 参数可以完全防止 `SQL注入攻击`__, 这是一个攻击者常用的漏洞,
    它可以向你的数据库中注入任何SQL语句, 如果使用字符串格式化则不能避免SQL注入攻击,
    记住一定要使用 ``params`` 参数让你避免此风险.

__ https://en.wikipedia.org/wiki/SQL_injection

.. _executing-custom-sql:

直接执行自定义SQL
=============================

有时候 :meth:`Manager.raw` 也不能满足需求: 我们可能不需要将查询结果映射成模型,
或者我们需要执行
``UPDATE``  ``INSERT`` 和 ``DELETE`` 查询.

这种情况下, 可以直接访问数据库绕开模型层.

``django.db.connection`` 提供了默认数据库的连接. 要使用这个数据库连接, 先调用 ``connection.cursor()`` 方法来获取一个游标对象.
然后调用 ``cursor.execute(sql, [params])`` 来执行SQL, 调用 ``cursor.fetchone()`` 或者 ``cursor.fetchall()`` 来获取返回行.

例如::

    from django.db import connection

    def my_custom_sql(self):
        with connection.cursor() as cursor:
            cursor.execute("UPDATE bar SET foo = 1 WHERE baz = %s", [self.baz])
            cursor.execute("SELECT foo FROM bar WHERE baz = %s", [self.baz])
            row = cursor.fetchone()

        return row

注意, 如果查询中包含百分号, 你需要写成两个百分号::

     cursor.execute("SELECT foo FROM bar WHERE baz = '30%'")
     cursor.execute("SELECT foo FROM bar WHERE baz = '30%%' AND id = %s", [self.id])

使用了 :doc:`多个数据库 </topics/db/multi-db>` 时, 可以使用 ``django.db.connections`` 获取指定的数据库连接(或游标)对象.
``django.db.connections`` 是一个类似字典的对象, 它允许你通过数据库的别名获取指定的连接::

    from django.db import connections
    cursor = connections['my_db_alias'].cursor()
    # Your code here...

默认情况下, Python DB API返回的结果不包含字段名, 也就是说你拿到的是一组 ``list`` 的值而不是 ``dict``.
在追求较少的运算和内存消耗下, 可以通过以下代码来返回 ``dict`` 结果::

    def dictfetchall(cursor):
        "Return all rows from a cursor as a dict"
        columns = [col[0] for col in cursor.description]
        return [
            dict(zip(columns, row))
            for row in cursor.fetchall()
        ]

另一个方法是使用Python标准库中的 :func:`collections.namedtuple`.
``namedtuple`` 是一个类元组对象, 可以通过属性访问字段. 它是可索引和可迭代的.
结果是不可变的, 可以通过字段名称和索引访问, 这在某些场景下非常实用::

    from collections import namedtuple

    def namedtuplefetchall(cursor):
        "Return all rows from a cursor as a namedtuple"
        desc = cursor.description
        nt_result = namedtuple('Result', [col[0] for col in desc])
        return [nt_result(*row) for row in cursor.fetchall()]

下面例子演示了这三者的区别::

    >>> cursor.execute("SELECT id, parent_id FROM test LIMIT 2");
    >>> cursor.fetchall()
    ((54360982, None), (54360880, None))

    >>> cursor.execute("SELECT id, parent_id FROM test LIMIT 2");
    >>> dictfetchall(cursor)
    [{'parent_id': None, 'id': 54360982}, {'parent_id': None, 'id': 54360880}]

    >>> cursor.execute("SELECT id, parent_id FROM test LIMIT 2");
    >>> results = namedtuplefetchall(cursor)
    >>> results
    [Result(id=54360982, parent_id=None), Result(id=54360880, parent_id=None)]
    >>> results[0].id
    54360982
    >>> results[0][0]
    54360982

Connections和cursors
-----------------------

``connection`` 和 ``cursor`` 实现了 :pep:`249` 中大部分的 Python DB-API — 除非它涉及 :doc:`事务处理
</topics/db/transactions>`.

如果你不熟悉Python DB-API, 请注意 ``cursor.execute()`` 中SQL语句使用 ``"%s"`` 占位符,
而不是直接在SQL中添加参数. 如果你使用这种方法, 底层数据库的库会在必要时自动转义你的参数.

还需要注意Django中是 ``"%s"`` 占位符, **不是** ``"?"`` 占位符, 后者由 SQLite 和 Python 绑定使用. 这是为了一致性和正确性.

将cursor作为上下文管理器使用::

    with connection.cursor() as c:
        c.execute(...)

它等价于::

    c = connection.cursor()
    try:
        c.execute(...)
    finally:
        c.close()
