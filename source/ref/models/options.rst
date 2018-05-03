======================
模型 ``Meta`` 选项
======================

本篇文档阐述所有可用的 :ref:`metadata 选项 <meta-options>`,  你可以在模型的 ``Meta 类`` 中设置它们.

``Meta`` 可用选项
==================

.. currentmodule:: django.db.models

``abstract``
------------

.. attribute:: Options.abstract

    如果设置 ``abstract = True``, 改模型将成为
    :ref:`抽象基类 <abstract-base-classes>`.

``app_label``
-------------

.. attribute:: Options.app_label

    如果模型定义在 :setting:`INSTALLED_APPS` 之外, 那么必须声明其属于哪个app::

        app_label = 'myapp'

    .. versionadded:: 1.9

    如果想要使用 ``app_label.object_name`` 或者 ``app_label.model_name`` 来表示模型, 可以分别使用 ``model._meta.label`` 和
    ``model._meta.label_lower``.

``base_manager_name``
---------------------

.. attribute:: Options.base_manager_name

    .. versionadded:: 1.10

    模型中 :attr:`~django.db.models.Model._base_manager` 所使用的manager名称.

``db_table``
------------

.. attribute:: Options.db_table

    该模型所使用的数据库表的名称::

        db_table = 'music_album'

.. _table-names:

数据库表名
~~~~~~~~~~~

为节省时间, Django自动使用模型的名称和包含此模型的app名称来生成表名. 模型在数据库中的表名由模型的"app label" --
即在 :djadmin:`manage.py startapp <startapp>` 中使用的名字和模型的类名通过下划线连接组成.

举个例子, 如果你有个名为 ``bookstore`` (通过
``manage.py startapp bookstore`` 创建的) 的应用, 定义在 ``class Book`` 中的模型的数据库中的表名为 ``bookstore_book``.

如果要重新数据库中的表名, 可以使用 ``class Meta`` 中的 ``db_table`` 参数.

如果你的数据库表名称是SQL保留字, 或包含Python变量名称中不允许的字符,特别是连字符 —-  这都不是问题. Django是在后台引用列和表名.

.. admonition:: 使用小写字符串作为 MySQL 表名

    当你使用 ``db_table`` 重写数据库中表名时， 强烈建议你使用小写字符作为表名. 尤其是在使用MySQL作为后台数据库时.
    详情参见 :ref:`MySQL注意事项 <mysql-notes>` .

.. admonition:: Oracle中表名称的引号处理

   为了遵从Oracle中30个字符的限制, 以及一些常见的约定, Django会缩短表的名称，而且会把它全部转为大写.
   如果你不想名称按照预定设置发生变化, 可以在 ``db_table`` 的值外面加上引号来避免这种情况::

       db_table = '"name_left_in_lowercase"'

   这种带引号的名称也可以与Django所支持的其它数据库后端一起使用；但Oracle除外, 引号没有效果.
   详情参见 :ref:`Oracle 注意事项 <oracle-notes>`.

``db_tablespace``
-----------------

.. attribute:: Options.db_tablespace

    当前模型所使用的 :doc:`database tablespace </topics/db/tablespaces>` 的名字.
    默认值是项目设置中的 :setting:`DEFAULT_TABLESPACE`,如果后端并不支持表空间，这个选项可以忽略.

``default_manager_name``
------------------------

.. attribute:: Options.default_manager_name

    .. versionadded:: 1.10

    模型中
    :attr:`~django.db.models.Model._default_manager` 用到的管理器名称.

``default_related_name``
------------------------

.. attribute:: Options.default_related_name

    从关联的对象反向查找当前对象用到的默认名称. 默认为 ``<model_name>_set``.

    这个选项还设置 :attr:`~ForeignKey.related_query_name`.

    一个字段的反向名称应该是唯一的, 当子类化你的模型时, 要格外小心.
    为了规避名称冲突，名称应该含有 ``'%(app_label)s'`` 和 ``'%(model_name)s'``,
    它们会被该模型所在的应用标签的名称和模型的名称替换，二者都是小写的.
    详情请参考
    :ref:`抽象模型的relate names <abstract-related-name>`.

    .. deprecated:: 1.10

        该属性名现在只影响 ``related_query_name``. 旧的查询名已弃用::

            from django.db import models

            class Foo(models.Model):
                pass

            class Bar(models.Model):
                foo = models.ForeignKey(Foo)

                class Meta:
                    default_related_name = 'bars'

        ::

            >>> bar = Bar.objects.get(pk=1)
            >>> # Using model name "bar" as lookup string is deprecated.
            >>> Foo.objects.get(bar=bar)
            >>> # You should use default_related_name "bars".
            >>> Foo.objects.get(bars=bar)

``get_latest_by``
-----------------

.. attribute:: Options.get_latest_by

    模型中可以用来排序的字段, 比如 :class:`DateField`,
    :class:`DateTimeField`, 或者 :class:`IntegerField`. 它用来指定
    模型中 :class:`Manager` 的
    :meth:`~django.db.models.query.QuerySet.latest` 和
    :meth:`~django.db.models.query.QuerySet.earliest` 方法的模型字段.

    例如::

        get_latest_by = "order_date"

    详情请参考文档 :meth:`~django.db.models.query.QuerySet.latest` .

``managed``
-----------

.. attribute:: Options.managed

    默认为 ``True``, 意味着Django将在 :djadmin:`migrate` 或作为迁移的一部分中创建适当的数据库表，
    并将它们作为一个 :djadmin:`flush` management命令的一部分删除.
    也就是说，Django *管理* 数据库表的生命周期.

    如果为 ``False``, Django 就不会为当前模型创建和删除数据表.
    这种适用于,当前模型使用一个已经存在的通过其它方法创建的数据表或数据库视图.
    这是设置了 ``managed=False`` 的 *唯一* 不同之处.
    其它任何方面处理都和通常一样. 这包括:

    1. 如果没有声明，会自动向模型中添加一个自增主键.
       为了避免给后面的代码读者带来混乱，当你在使用未被管理的模型时，
       强烈推荐你指定(specify)数据表中所有的列.

    2. 如果模型设置了 ``managed=False`` 并且包含
       :class:`~django.db.models.ManyToManyField`, 且这个多对多关联关系也指向一个没被管理(unmanaged)的模型,
       那么这两个未被管理的模型的多对多关系的中间表也不会被创建. 但是,
       一个被管理模型和一个未被管理模型之间的中间表就 *会* 被创建.

       如果需要修改这一默认行为，请将中间表创建为显式模型(根据需要设置 ``managed``)，
       并使用 :attr:`ManyToManyField.through` . 通过属性使关系使用您的定制模型.

    对于涉及 ``managed=False`` 模型的测试, 在测试之前, 必须要确保在测试启动时已经创建了正确的数据表。

    如果你对在Python层面修改模型类行为感兴趣, 可以设置 ``managed=False``,
    并且为模型创建一个副本. 不过在这种场景下还有个更好的办法就是使用 :ref:`proxy-models`.

``order_with_respect_to``
-------------------------

.. attribute:: Options.order_with_respect_to

    使此对象相对于给定字段可以排序, 通常为 ``ForeignKey``.
    这可以用于使关联的对象相对于父对象可排序. 比如, 如果 ``Answer`` 和 ``Question`` 关联,
    一个问题有至少一个答案, 并且答案的顺序非常重要，你可以这样做::

        from django.db import models

        class Question(models.Model):
            text = models.TextField()
            # ...

        class Answer(models.Model):
            question = models.ForeignKey(Question, on_delete=models.CASCADE)
            # ...

            class Meta:
                order_with_respect_to = 'question'

    如果设置了 ``order_with_respect_to`` , 模型会提供两个额外的用于设置和获取关联对象顺序的方法
    ``get_RELATED_order()`` 和 ``set_RELATED_order()``, ``RELATED`` 就行模型名称的小写字符串.
    例如, 假设一个 ``Question`` 对象有很多关联的 ``Answer`` 对象, 返回含有与之关联 ``Answer`` 对象主键的列表::

        >>> question = Question.objects.get(id=1)
        >>> question.get_answer_order()
        [1, 2, 3]

    ``Question`` 对象关联的 ``Answer`` 对象的顺序， 可以通过传入一个包含 ``Answer`` 主键的列表来设置::

        >>> question.set_answer_order([3, 1, 2])

    与之关联的对象也有两个方法, ``get_next_in_order()`` 和
    ``get_previous_in_order()``, 用于按照合适的顺序访问它们.
    假设 ``Answer`` 对象按照 ``id`` 排序 ::

        >>> answer = Answer.objects.get(id=2)
        >>> answer.get_next_in_order()
        <Answer: 3>
        >>> answer.get_previous_in_order()
        <Answer: 1>

.. admonition:: ``order_with_respect_to`` 隐式设置 ``ordering`` 选项

    在内部, ``order_with_respect_to`` 加了一个名为 ``_order`` 的附加字段/数据库列.
    并将该模型的 :attr:`~Options.ordering` 设置为此选项, 因此,
    ``order_with_respect_to`` 和 ``ordering`` 不能一起使用,
    每当你获得此模型的对象列表的时候，将使用 ``order_with_respect_to`` 的排序.

.. admonition:: 修改 ``order_with_respect_to`` 时

    因为 ``order_with_respect_to`` 添加了一个新的数据库列,
    所以在初始化 :djadmin:`migrate` 之后添加或更改 ``order_with_respect_to`` 时，请确保进行相应的迁移操作.

``ordering``
------------

.. attribute:: Options.ordering

    对象默认的顺序, 在获取对象的列表时使用::

        ordering = ['-order_date']

    它是一个字符串的列表或元组. 每个字符串是一个字段名, 前面带有可选的“-”前缀表示倒序.
    前面没有“-”的字段表示升序. 使用字符串“？”来表示随机排序.

    例如, 按照字段 ``pub_date`` 升序排列, 这样使用::

        ordering = ['pub_date']

    按照字段 ``pub_date`` 降序排列, 这样使用::

        ordering = ['-pub_date']

    按照字段 ``pub_date`` 降序, 然后按照字段 ``author`` 升序, 这样使用::

        ordering = ['-pub_date', 'author']

    默认排序也会影响 :ref:`聚合查询
    <aggregation-ordering-interaction>`.

.. warning::

    排序并不是没有任何代价的操作. 你向ordering属性添加的每个字段都会产生数据库额外的开销.
    添加的每个外键也会隐式包含它的默认顺序.

    如果查询没有指定顺序, 则会以未指定的顺序从数据库返回结果. 仅当排序的一组字段可以唯一标识结果中的每个对象时,
    才能保证稳定排序. 例如, 如果 ``name`` 字段不唯一, 则由其排序不会保证具有相同名称的对象总是以相同的顺序显示.

``permissions``
---------------

.. attribute:: Options.permissions

    创建对象时写入权限表中额外的权限. 每个模型将自动创建增加、删除和修改权限. 下面例子创建一个额外的权限 ``can_deliver_pizzas``::

        permissions = (("can_deliver_pizzas", "Can deliver pizzas"),)

    它是一个包含二元元组的列表或元组, 格式为 ``(权限编码, 书面的权限名称)``.

``default_permissions``
------------------------------

.. attribute:: Options.default_permissions

    默认为 ``('add', 'change', 'delete')``. 也可以自定义这个列表,
    例如, 如果你的应用不需要默认权限中的任何一项, 可以把它设置成空列表.
    这个属性必须在模型被 :djadmin:`migrate` 命令创建之前指定，避免一些遗漏的权限被创建.

``proxy``
---------

.. attribute:: Options.proxy

    如果设置为 ``proxy = True``,如果它是另一个模型的子类，将会视为一个
    :ref:`代理模型 <proxy-models>`.

``required_db_features``
------------------------

.. attribute:: Options.required_db_features

    .. versionadded:: 1.9

    当前连接的数据库应该具有的功能列表, 以便在迁移阶段考虑该模型.
    例如,如果将此列表设置为 ``['gis_enabled']``, 则模型将仅在启用GIS的数据库上同步.
    在使用多个数据库后端进行测试时, 用于跳过某些模型也很有用. 避免与ORM无关的模型之间的关系.

``required_db_vendor``
----------------------

.. attribute:: Options.required_db_vendor

    .. versionadded:: 1.9

    该模型所特有的受支持的数据库种类的名称. 当前的内置数据库名称有: ``sqlite``, ``postgresql``, ``mysql``, ``oracle``.
    如果该属性不是空, 且当前连接的数据库不匹配, 则模型将不会同步.

``select_on_save``
------------------

.. attribute:: Options.select_on_save

    该选项决定Django是否采用1.6之前的 :meth:`django.db.models.Model.save()` 算法.
    旧的算法先使用 ``SELECT`` 来判断是否存在需要更新的行. 而新的算法直接尝试使用 ``UPDATE``.
    在某些少见的情况下， Django对已存在行的 ``UPDATE`` 操作不可访问. 比如,
    PostgreSQL的返回 ``NULL`` 的 ``ON UPDATE`` 触发器.
    这种情况下，新式的算法最终会执行 ``INSERT`` 操作，即使这一行已经在数据库中存在.

    通常这个属性不需要设置. 默认为 ``False``.

    关于旧式和新式两种保存算法, 请参见 :meth:`django.db.models.Model.save()`.

``unique_together``
-------------------

.. attribute:: Options.unique_together

    用来设置的不重复的字段组合::

        unique_together = (("driver", "restaurant"),)

    它是一个元组的元组, 组合起来的时候必须是唯一的. 它在Django admin层面使用,
    在数据库层上进行数据约束(比如，在 ``CREATE TABLE`` 语句中使用 ``UNIQUE`` 语句).

    为了方便起见, 处理单一字段的集合时, ``unique_together`` 可以是一维的元组::

        unique_together = ("driver", "restaurant")

    :class:`~django.db.models.ManyToManyField` 不能用在
    unique_together中. (不清楚这样的含义是什么!), 如果需要验证
    :class:`~django.db.models.ManyToManyField` 关联的唯一性, 可以尝试使用signal或者显式的
    :attr:`through <ManyToManyField.through>` 模型.

    当违反 ``unique_together`` 约束时, 模型校验期间会抛出 ``ValidationError`` 异常.

``index_together``
------------------

.. attribute:: Options.index_together

    用来设置带有索引的字段组合::

        index_together = [
            ["pub_date", "deadline"],
        ]

    将会为列表中的字段建立索引 (i.e. 会在 ``CREATE INDEX`` 语句中使用.)

    为了方便起见, 处理单一字段的集合时, ``index_together`` 可以是一维的列表::

        index_together = ["pub_date", "deadline"]

``verbose_name``
----------------

.. attribute:: Options.verbose_name

    对象的一个书面描述名字, 单数形式::

        verbose_name = "pizza"

    如果此项没有设置, Django会把类名拆分开来作为描述名, 比如 ``CamelCase`` 会变成 ``camel case``.

``verbose_name_plural``
-----------------------

.. attribute:: Options.verbose_name_plural

    该对象复数形式的名称::

        verbose_name_plural = "stories"

    如果没有设置, Django 将使用 :attr:`~Options.verbose_name` + ``"s"``.

``Meta`` 只读属性
==================

``label``
---------

.. attribute:: Options.label

    .. versionadded:: 1.9

    对象的表示, 返回 ``app_label.object_name``, e.g.
    ``'polls.Question'``.

``label_lower``
---------------

.. attribute:: Options.label_lower

    .. versionadded:: 1.9

    模型的表示, 返回 ``app_label.model_name``, e.g.
    ``'polls.question'``.
