=======================
条件表达式
=======================

.. currentmodule:: django.db.models.expressions

使用条件表达式可以在过滤, 注解, 聚合和更新中使用 :keyword:`if` ... :keyword:`elif` ...
:keyword:`else` 逻辑. 条件表达式为表中的每一行执行一系列的条件判断并返回匹配到的结果表达式.
条件表达式也可以像其它 :doc:`表达式 <expressions>` 一样组合和嵌套.

条件表达式类
==================================

我们会在后面的例子中使用下面的模型::

    from django.db import models

    class Client(models.Model):
        REGULAR = 'R'
        GOLD = 'G'
        PLATINUM = 'P'
        ACCOUNT_TYPE_CHOICES = (
            (REGULAR, 'Regular'),
            (GOLD, 'Gold'),
            (PLATINUM, 'Platinum'),
        )
        name = models.CharField(max_length=50)
        registered_on = models.DateField()
        account_type = models.CharField(
            max_length=1,
            choices=ACCOUNT_TYPE_CHOICES,
            default=REGULAR,
        )

``When``
--------

.. class:: When(condition=None, then=None, **lookups)

``When()`` 对象用来封装条件表达式中的条件和结果. 使用 ``When()`` 对象类似 :meth:`~django.db.models.query.QuerySet.filter` 方法.
可以使用 :ref:`字段查找 <field-lookups>` 和 :class:`~django.db.models.Q` 对象来指定条件. 使用 ``then`` 关键字参数指定结果.

一些例子::

    >>> from django.db.models import When, F, Q
    >>> # String arguments refer to fields; the following two examples are equivalent:
    >>> When(account_type=Client.GOLD, then='name')
    >>> When(account_type=Client.GOLD, then=F('name'))
    >>> # You can use field lookups in the condition
    >>> from datetime import date
    >>> When(registered_on__gt=date(2014, 1, 1),
    ...      registered_on__lt=date(2015, 1, 1),
    ...      then='account_type')
    >>> # Complex conditions can be created using Q objects
    >>> When(Q(name__startswith="John") | Q(name__startswith="Paul"),
    ...      then='name')

请记住, 每个值都可以是表达式.

.. note::

    因此 ``then`` 关键字参数是用作 ``When()`` 的结果, 所有如果
    :class:`~django.db.models.Model` 有个名为 ``then`` 的字段. 可以通过下面两种方法避免冲突::

        >>> When(then__exact=0, then=1)
        >>> When(Q(then=0), then=1)

``Case``
--------

.. class:: Case(*cases, **extra)

``Case()`` 表达式类似于 ``Python`` 中的 :keyword:`if` ... :keyword:`elif` ...
:keyword:`else` 语句. 每个 ``condition`` 中的 ``When()`` 对象都会被依次执行, 直到执行出一个真值.
从匹配的 ``When()`` 对象中返回 ``result`` 表达式.

一个例子::

    >>>
    >>> from datetime import date, timedelta
    >>> from django.db.models import CharField, Case, Value, When
    >>> Client.objects.create(
    ...     name='Jane Doe',
    ...     account_type=Client.REGULAR,
    ...     registered_on=date.today() - timedelta(days=36))
    >>> Client.objects.create(
    ...     name='James Smith',
    ...     account_type=Client.GOLD,
    ...     registered_on=date.today() - timedelta(days=5))
    >>> Client.objects.create(
    ...     name='Jack Black',
    ...     account_type=Client.PLATINUM,
    ...     registered_on=date.today() - timedelta(days=10 * 365))
    >>> # Get the discount for each Client based on the account type
    >>> Client.objects.annotate(
    ...     discount=Case(
    ...         When(account_type=Client.GOLD, then=Value('5%')),
    ...         When(account_type=Client.PLATINUM, then=Value('10%')),
    ...         default=Value('0%'),
    ...         output_field=CharField(),
    ...     ),
    ... ).values_list('name', 'discount')
    [('Jane Doe', '0%'), ('James Smith', '5%'), ('Jack Black', '10%')]

``Case()`` 接受任意数量的 ``When()`` 对象作为独立参数. 其它选项使用关键字参数提供.
如果没有一个条件的值为 ``TRUE``, 那么会返回 ``default`` 关键字参数提供表达式.
如果没有提供 ``default`` 参数则返回 ``None``.

如果想修改之前的查询, 根据 ``Client`` 的 registered_on 时间来计算discount, 可以这样修改::

    >>> a_month_ago = date.today() - timedelta(days=30)
    >>> a_year_ago = date.today() - timedelta(days=365)
    >>> # Get the discount for each Client based on the registration date
    >>> Client.objects.annotate(
    ...     discount=Case(
    ...         When(registered_on__lte=a_year_ago, then=Value('10%')),
    ...         When(registered_on__lte=a_month_ago, then=Value('5%')),
    ...         default=Value('0%'),
    ...         output_field=CharField(),
    ...     )
    ... ).values_list('name', 'discount')
    [('Jane Doe', '5%'), ('James Smith', '0%'), ('Jack Black', '10%')]

.. note::

    请记住条件是按顺序计算的. 所以在上面的例子中, 尽管第二个条件同时符合Jane Doe和Jack Black, 最后还是得到了正确的结果.
    这就像在 ``Python`` 中的 :keyword:`if` ... :keyword:`elif` ... :keyword:`else` 语句一样.

``Case()`` 也可以在 ``filter()`` 中使用. 例如, 查询注册时间超过一月的gold客户和注册时间超过一年的platinum客户::

    >>> a_month_ago = date.today() - timedelta(days=30)
    >>> a_year_ago = date.today() - timedelta(days=365)
    >>> Client.objects.filter(
    ...     registered_on__lte=Case(
    ...         When(account_type=Client.GOLD, then=a_month_ago),
    ...         When(account_type=Client.PLATINUM, then=a_year_ago),
    ...     ),
    ... ).values_list('name', 'account_type')
    [('Jack Black', 'P')]

高级查询
================

条件表达式可用于注解, 聚合, 过滤, 查找和更新中. 它们还可以与其他表达式组合和嵌套. 这使你可以进行强大的条件查询.

条件更新
------------------

假设我们想根据客户的注册日期来设置 ``account_type``. 可以使用条件表达式和
:meth:`~django.db.models.query.QuerySet.update` 方法来实现::

    >>> a_month_ago = date.today() - timedelta(days=30)
    >>> a_year_ago = date.today() - timedelta(days=365)
    >>> # Update the account_type for each Client from the registration date
    >>> Client.objects.update(
    ...     account_type=Case(
    ...         When(registered_on__lte=a_year_ago,
    ...              then=Value(Client.PLATINUM)),
    ...         When(registered_on__lte=a_month_ago,
    ...              then=Value(Client.GOLD)),
    ...         default=Value(Client.REGULAR)
    ...     ),
    ... )
    >>> Client.objects.values_list('name', 'account_type')
    [('Jane Doe', 'G'), ('James Smith', 'R'), ('Jack Black', 'P')]

条件聚合
-----------------------

如我们想知道每个 ``account_type`` 有多少客户, 可以在
:ref:`聚合函数 <aggregation-functions>` 中嵌套条件表达式来实现::

    >>> # Create some more Clients first so we can have something to count
    >>> Client.objects.create(
    ...     name='Jean Grey',
    ...     account_type=Client.REGULAR,
    ...     registered_on=date.today())
    >>> Client.objects.create(
    ...     name='James Bond',
    ...     account_type=Client.PLATINUM,
    ...     registered_on=date.today())
    >>> Client.objects.create(
    ...     name='Jane Porter',
    ...     account_type=Client.PLATINUM,
    ...     registered_on=date.today())
    >>> # Get counts for each value of account_type
    >>> from django.db.models import IntegerField, Sum
    >>> Client.objects.aggregate(
    ...     regular=Sum(
    ...         Case(When(account_type=Client.REGULAR, then=1),
    ...              output_field=IntegerField())
    ...     ),
    ...     gold=Sum(
    ...         Case(When(account_type=Client.GOLD, then=1),
    ...              output_field=IntegerField())
    ...     ),
    ...     platinum=Sum(
    ...         Case(When(account_type=Client.PLATINUM, then=1),
    ...              output_field=IntegerField())
    ...     )
    ... )
    {'regular': 2, 'gold': 1, 'platinum': 3}
