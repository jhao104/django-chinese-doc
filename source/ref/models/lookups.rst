====================
Lookup API
====================

.. module:: django.db.models.lookups
   :synopsis: Lookups API

.. currentmodule:: django.db.models

Django的Lookup API用于构建数据库查询的 ``WHERE`` 子句. 如何 *使用* Lookup请参见 :doc:`/topics/db/queries`;
自定义Lookup请参见 :doc:`/howto/custom-lookups`.

Lookup API 由两部分组成: 一是 :class:`~lookups.RegisterLookupMixin` 类用于注册lookup,
二是 :ref:`查找表达式接口 <查询表达式接口>`, 是普通类要注册为lookup必须要要实现的一组方法.

Django有两个查找表达式的基类, Django所有的内置查找都继承自它们:

* :class:`Lookup`: 查找字段 (例如. ``field_name__exact`` 中的 ``exact``)
* :class:`Transform`: 转换字段

一个查找表达式由三部分组成:

* 查找的字段名 (例如. ``Book.objects.filter(author__best_friends__first_name...``);
* 字段转换 (可以没有) (例如. ``__lower__first3chars__reversed``);
* 查找 (例如. ``__icontains``) 如果没有则默认为 ``__exact``.

.. _lookup-registration-api:

注册API
================

Django通过 :class:`~lookups.RegisterLookupMixin` 来给类提供注册查找的接口. 两个明显的例子:
:class:`~django.db.models.Field` 所有模型字段的基类 和
``Aggregate`` 所有聚合函数的基类.

.. class:: lookups.RegisterLookupMixin

    用于给类"混入"查找API.

    .. classmethod:: register_lookup(lookup, lookup_name=None)

        给类注册一个查找. 例如
        ``DateField.register_lookup(YearExact)`` 会将 ``YearExact`` 查找注册到 ``DateField``.
        如果已存在相同名字查找则会覆盖原查找. 如果传入了 ``lookup_name`` 将会使用其作为查找名,
        否则使用 ``lookup.lookup_name``.

        .. versionchanged:: 1.9

            新增 ``lookup_name`` 参数.

    .. method:: get_lookup(lookup_name)

        返回被注册名 ``lookup_name`` 的 :class:`Lookup`.
        默认会递归查找所有父类中名为 ``lookup_name`` 的查找, 然后返回第一个.

    .. method:: get_transform(transform_name)

        返回被注册名 ``transform_name`` 的 :class:`Transform`.
        默认会递归查找所有父类中名为 ``transform_name`` 的查找, 然后返回第一个.


要注册为查找的类必须遵循 :ref:`查询表达式接口
<查询表达式接口>`. :class:`~Lookup` 和 :class:`~Transform` 已遵循这个接口.

.. _查询表达式接口:
.. _query-expression:

查询表达式接口
========================

查询表达式接口是用于将查找转换成SQL查询的一组公共方法.
直接的字段引用, 聚合, 以及 ``Transform`` 类都遵循了这个接口, 遵循查询表达式接口需要实现以下接口:

.. method:: as_sql(self, compiler, connection)

    负责生成查询字符串和参数.
    ``compiler`` 是 ``SQLCompiler`` 对象, 它具有一个 ``compile()`` 方法可以编译其他的表达式.
    ``connection`` 是用于执行查询的连接.

    直接调用 ``expression.as_sql()`` 是错误的, 应该使用 ``compiler.compile(expression)``,
    ``compiler.compile()`` 才会去调用对应的数据库方法.

    如果 ``as_vendorname()`` 方法或子类传入数据来覆盖SQL字符串的生成,
    可以在这个方法上传入自定义关键字参数. 详见 :meth:`Func.as_sql`.

.. method:: as_vendorname(self, compiler, connection)

    作用类似 ``as_sql()``. 当表达式被
    ``compiler.compile()`` 编译后, Django会首先调用 ``as_vendorname()``,
    ``vendorname`` 是执行查询的后台数据库名称. Django内置的后台数据库有 ``postgresql``, ``oracle``,
    ``sqlite``, 和 ``mysql``.

.. method:: get_lookup(lookup_name)

    必须返回名为 ``lookup_name`` 的查找. 例如, ``self.output_field.get_lookup(lookup_name)``.

.. method:: get_transform(transform_name)

    必须返回名为 ``transform_name`` 的查找. 例如, ``self.output_field.get_transform(transform_name)``.

.. attribute:: output_field

    定义 ``get_lookup()`` 方法返回类的类型. 必须是 :class:`~django.db.models.Field` 实例.

``Transform`` 参考
=======================

.. class:: Transform

    ``Transform`` 是实现字段转换的通用类. 一个常用的例子 ``__year`` 将 ``DateField`` 转换成 ``IntegerField``.

    在查找表达式中 ``Transform`` 的用法是
    ``<expression>__<transformation>`` (例如. ``date__year``).

    该类遵循 :ref:`查询表达式接口 <查询表达式接口>`, 因此可以使用 ``<expression>__<transform1>__<transform2>``.
    它是一个特殊的 :ref:`Func()表达式 <func-expressions>` , 只接收一个参数.
    它可以用于右侧过滤或者直接作为注解使用.

    .. versionchanged:: 1.9

        ``Transform`` 作为 ``Func`` 子类.

    .. attribute:: bilateral

        布尔值, 表示该转换是否适用于 ``lhs`` 和 ``rhs``. 双边转换将按照查找表达式中出现的顺序应用于 ``rhs`` .
        默认值为 ``False``. 使用方法见: :doc:`/howto/custom-lookups`.

    .. attribute:: lhs

        左侧转换内容. 必须遵循
        :ref:`查询表达式接口 <查询表达式接口>`.

    .. attribute:: lookup_name

        查找的名字, 用于在解析查询表达式时识别. 不包含 ``"__"``.

    .. attribute:: output_field

        定义转换输出的类. 必须是 :class:`~django.db.models.Field` 实例. 默认与 ``lhs.output_field`` 相同.

``Lookup`` 参考
====================

.. class:: Lookup

    ``Lookup`` 是实现查找的通用类. 一个查询表达式是由左侧 :attr:`lhs`, 右侧,
    :attr:`rhs` 和 ``lookup_name`` 组成, 用于在 ``lhs`` 和 ``rhs`` 之间比较返回布尔值, 例如 ``lhs in rhs`` 和
    ``lhs > rhs``.

    在表达式中的使用方式是: ``<lhs>__<lookup_name>=<rhs>``.

    该类并不遵循 :ref:`查询表达式接口 <查询表达式接口>`,
    因此像上面的 ``=<rhs>`` 必须在查询表达式最后.

    .. attribute:: lhs

        查找的左侧. 它必须遵循 :ref:`查询表达式接口 <查询表达式接口>`.

    .. attribute:: rhs

        查找的右侧 - ``lhs`` 的比较对象. 它可以是一个普通值或者是可以被编译成SQL的东西, 通常是一个
        ``F()`` 对象或者 ``QuerySet``.

    .. attribute:: lookup_name

        查找的名称, 用于在解析查询表达式时识别. 不包含 ``"__"``.

    .. method:: process_lhs(compiler, connection, lhs=None)

        返回由 ``compiler.compile(lhs)`` 返回的数组 ``(lhs_string, lhs_params)``, 这个方法用来被重写以修改 ``lhs`` 的处理.

        ``compiler`` 是一个 ``SQLCompiler`` 对象, 用来 ``compiler.compile(lhs)`` 编译 ``lhs``.
        ``connection`` 用来编译特定后台数据库的SQL. 如果 ``lhs`` 不为
        ``None`` 则使用其处理后 ``lhs`` 替代 ``self.lhs``.

    .. method:: process_rhs(compiler, connection)

        以和 :meth:`process_lhs` 同样的方式处理右侧.
