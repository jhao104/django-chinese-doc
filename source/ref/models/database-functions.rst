==================
数据库函数
==================

.. module:: django.db.models.functions
    :synopsis: Database Functions

下面记录的类为用户提供了一种方法, 可以在Django中使用底层数据库提供的函数作为注解, 聚合和过滤等操作. 函数也是 :doc:`表达式 <expressions>`,
所以它们可以和其他表达式一起组合使用, 比如 :ref:`聚合函数<aggregation-functions>`.

我们将在函数的例子中使用以下模型::

    class Author(models.Model):
        name = models.CharField(max_length=50)
        age = models.PositiveIntegerField(null=True, blank=True)
        alias = models.CharField(max_length=50, null=True, blank=True)
        goes_by = models.CharField(max_length=50, null=True, blank=True)

不建议 ``CharField`` 设置 ``null=True``, 因为这导致该字段有两种"空值", 但下面的 ``Coalesce`` 示例需要这个设置.

``Cast``
========

.. class:: Cast(expression, output_field)

.. versionadded:: 1.10

强制将 ``expression`` 返回类型转换为 ``output_field``.

用例::

    >>> from django.db.models import FloatField
    >>> from django.db.models.functions import Cast
    >>> Value.objects.create(integer=4)
    >>> value = Value.objects.annotate(as_float=Cast('integer', FloatField())).get()
    >>> print(value.as_float)
    4.0

``Coalesce``
============

.. class:: Coalesce(*expressions, **extra)

接收一个至少包含两个字段名或表达式的列表, 返回第一个非null值(注意空字符串不会被认为null值).
所有参数必须是相似类型, 混合文本和数字类型会导致数据库错误.

用例::

    >>> # Get a screen name from least to most public
    >>> from django.db.models import Sum, Value as V
    >>> from django.db.models.functions import Coalesce
    >>> Author.objects.create(name='Margaret Smith', goes_by='Maggie')
    >>> author = Author.objects.annotate(
    ...    screen_name=Coalesce('alias', 'goes_by', 'name')).get()
    >>> print(author.screen_name)
    Maggie

    >>> # Prevent an aggregate Sum() from returning None
    >>> aggregated = Author.objects.aggregate(
    ...    combined_age=Coalesce(Sum('age'), V(0)),
    ...    combined_age_default=Sum('age'))
    >>> print(aggregated['combined_age'])
    0
    >>> print(aggregated['combined_age_default'])
    None

.. warning::

    在MySQL中传递给 ``Coalesce`` 的Python值可能会被转换为错误的类型, 除非显示地转换为正确的数据库类型:

    >>> from django.db.models import DateTimeField
    >>> from django.db.models.functions import Cast, Coalesce
    >>> from django.utils import timezone
    >>> now = timezone.now()
    >>> Coalesce('updated', Cast(now, DateTimeField()))

``Concat``
==========

.. class:: Concat(*expressions, **extra)

接收一个至少包含两个文本字段或者表达式的列表, 返回连接后的文本.
每个参数必须是文本或字符类型. 如果要连接 ``TextField()`` 和 ``CharField()``, 那么必须将 ``output_field`` 设置为 ``TextField()``.
连接 ``Value`` 时也需要做同样设置.

该函数绝不会返回null. 如果返回null会导致表达式变成null, Django会把参数中null的部分转换为空字符串.

用例::

    >>> # Get the display name as "name (goes_by)"
    >>> from django.db.models import CharField, Value as V
    >>> from django.db.models.functions import Concat
    >>> Author.objects.create(name='Margaret Smith', goes_by='Maggie')
    >>> author = Author.objects.annotate(
    ...    screen_name=Concat('name', V(' ('), 'goes_by', V(')'),
    ...    output_field=CharField())).get()
    >>> print(author.screen_name)
    Margaret Smith (Maggie)

``Greatest``
============

.. class:: Greatest(*expressions, **extra)

.. versionadded:: 1.9

接收一个至少包含两个字段或表达式的列表, 返回最大值. 每个参数必须是相似类型, 例如混合文本和数字导致数据库错误.

用例::

    class Blog(models.Model):
        body = models.TextField()
        modified = models.DateTimeField(auto_now=True)

    class Comment(models.Model):
        body = models.TextField()
        modified = models.DateTimeField(auto_now=True)
        blog = models.ForeignKey(Blog, on_delete=models.CASCADE)

    >>> from django.db.models.functions import Greatest
    >>> blog = Blog.objects.create(body='Greatest is the best.')
    >>> comment = Comment.objects.create(body='No, Least is better.', blog=blog)
    >>> comments = Comment.objects.annotate(last_updated=Greatest('modified', 'blog__modified'))
    >>> annotated_comment = comments.get()

``annotated_comment.last_updated`` 会返回最近的 ``blog.modified`` 或 ``comment.modified``.

.. warning::

    当有一个或多个表达式为null时, ``Greatest`` 在不同数据库间行为有区别:

    - PostgreSQL: ``Greatest`` 会返回最大的非null值, 如果所有表达式都为 ``null`` 则返回 ``null``.
    - SQLite, Oracle, and MySQL: 如果存在为 ``null`` 的表达式, ``Greatest`` 则会返回 ``null``.

    如果你想用一个合理的值作为返回值, 可以使用 ``Coalesce`` 来模仿PostgreSQL的行为.

``Least``
=========

.. class:: Least(*expressions, **extra)

.. versionadded:: 1.9

接收一个至少包含两个字段或表达式的列表, 返回最小值. 每个参数必须是相似类型, 例如混合文本和数字导致数据库错误.

.. warning::

    当有一个或多个表达式为null时, ``Least`` 在不同数据库间行为有区别:

    - PostgreSQL: ``Least`` 会返回最大的非null值, 如果所有表达式都为 ``null`` 则返回 ``null``.
    - SQLite, Oracle, and MySQL: 如果存在为 ``null`` 的表达式, ``Greatest`` 则会返回 ``null``.

    如果你想用一个合理的值作为返回值, 可以使用 ``Coalesce`` 来模仿PostgreSQL的行为.

``Length``
==========

.. class:: Length(expression, **extra)

接收一个文本字段或表达式, 返回其包含的字符数. 如果表达式为null则返回null.

用例::

    >>> # Get the length of the name and goes_by fields
    >>> from django.db.models.functions import Length
    >>> Author.objects.create(name='Margaret Smith')
    >>> author = Author.objects.annotate(
    ...    name_length=Length('name'),
    ...    goes_by_length=Length('goes_by')).get()
    >>> print(author.name_length, author.goes_by_length)
    (14, None)

也可以将其注册为转换. 例如::

    >>> from django.db.models import CharField
    >>> from django.db.models.functions import Length
    >>> CharField.register_lookup(Length, 'length')
    >>> # Get authors whose name is longer than 7 characters
    >>> authors = Author.objects.filter(name__length__gt=7)

.. versionchanged:: 1.9

    新增将函数注册为转换的功能.

``Lower``
=========

.. class:: Lower(expression, **extra)

接收一个文本字段或表达式, 返回其小写形式.

它也可以像 :class:`Length` 那样注册为转换.

用例::

    >>> from django.db.models.functions import Lower
    >>> Author.objects.create(name='Margaret Smith')
    >>> author = Author.objects.annotate(name_lower=Lower('name')).get()
    >>> print(author.name_lower)
    margaret smith

.. versionchanged:: 1.9

    新增将函数注册为转换的功能.

``Now``
=======

.. class:: Now()

.. versionadded:: 1.9

返回执行查询时数据库服务器的当前日期和时间, 通常使用SQL ``CURRENT_TIMESTAMP``.

用例::

    >>> from django.db.models.functions import Now
    >>> Article.objects.filter(published__lte=Now())
    <QuerySet [<Article: How to Django>]>

.. admonition:: PostgreSQL considerations

    在PostgreSQL中, SQL ``CURRENT_TIMESTAMP`` 返回的是当前事务的开始时间.
    因此为了跨数据库的兼容性, ``Now()`` 使用 ``STATEMENT_TIMESTAMP`` 作为代替.
    如果需要事务时间戳, 可以使用 :class:`django.contrib.postgres.functions.TransactionNow`.

``Substr``
==========

.. class:: Substr(expression, pos, length=None, **extra)

返回字段或表达式 ``pos`` 位置处开始长度为 ``length`` 的子串. 位置下标从1开始, 因此改参数必须大于0.
如果 ``length`` 为 ``None``, 那么会返回剩余的所有字符串.

用例::

    >>> # Set the alias to the first 5 characters of the name as lowercase
    >>> from django.db.models.functions import Substr, Lower
    >>> Author.objects.create(name='Margaret Smith')
    >>> Author.objects.update(alias=Lower(Substr('name', 1, 5)))
    1
    >>> print(Author.objects.get(name='Margaret Smith').alias)
    marga

``Upper``
=========

.. class:: Upper(expression, **extra)

接收一个文本字段或表达式, 返回其大写形式.

它也可以像 :class:`Length` 那样注册为转换.

用例::

    >>> from django.db.models.functions import Upper
    >>> Author.objects.create(name='Margaret Smith')
    >>> author = Author.objects.annotate(name_upper=Upper('name')).get()
    >>> print(author.name_upper)
    MARGARET SMITH

.. versionchanged:: 1.9

    新增将函数注册为转换的功能.

日期函数
==============

.. module:: django.db.models.functions.datetime

.. versionadded:: 1.10

我们会在下面函数的示例中使用下面的模型::

    class Experiment(models.Model):
        start_datetime = models.DateTimeField()
        start_date = models.DateField(null=True, blank=True)
        end_datetime = models.DateTimeField(null=True, blank=True)
        end_date = models.DateField(null=True, blank=True)

``Extract``
-----------

.. class:: Extract(expression, lookup_name=None, tzinfo=None, **extra)

提取日期组成部分的数值.

接收一个 ``DateField`` 或 ``DateTimeField`` 的 ``expression`` 和 ``lookup_name`` 参数,
返回 ``IntegerField`` 类型的日期的 ``lookup_name`` 部分的值.
Django使用数据库的extract函数, 所以可以使用所有数据库支持的 ``lookup_name``.
可以传递 ``pytz`` 模块提供的 ``tzinfo`` 子类来指定时区.

对于datetime ``2015-06-15 23:30:01.000321+00:00``, 内置的 ``lookup_name`` 会返回:

* "year": 2015
* "month": 6
* "day": 15
* "week_day": 2
* "hour": 23
* "minute": 30
* "second": 1

如果在Django中使用了不同时区, 例如 ``Australia/Melbourne``, 那么会在提取前将datetime转换为当前时区.
上述示例日期中墨尔本的时区偏移为+10:00. 使用此时区时, 返回的值将与上述相同除了:

* "day": 16
* "week_day": 3
* "hour": 9

.. admonition:: ``week_day`` 值

    ``lookup_type`` ``week_day`` 的计算与大多数数据库和Python标准函数不一样.
    该函数的返回中. 星期日为 ``1``, 星期一为 ``2``, 星期六为 ``7``.

    等价于Python中的::

        >>> from datetime import datetime
        >>> dt = datetime(2015, 6, 15)
        >>> (dt.isoweekday() % 7) + 1
        2

上面每个 ``lookup_name`` 都具有相应的 ``Extract`` 子类, 通常可以使用这个子类来代替原本冗长的用法,
例如. 使用 ``ExtractYear(...)`` 代替 ``Extract(..., lookup_name='year')``.

用例::

    >>> from datetime import datetime
    >>> from django.db.models.functions import Extract
    >>> start = datetime(2015, 6, 15)
    >>> end = datetime(2015, 7, 2)
    >>> Experiment.objects.create(
    ...    start_datetime=start, start_date=start.date(),
    ...    end_datetime=end, end_date=end.date())
    >>> # Add the experiment start year as a field in the QuerySet.
    >>> experiment = Experiment.objects.annotate(
    ...    start_year=Extract('start_datetime', 'year')).get()
    >>> experiment.start_year
    2015
    >>> # How many experiments completed in the same year in which they started?
    >>> Experiment.objects.filter(
    ...    start_datetime__year=Extract('end_datetime', 'year')).count()
    1

``DateField`` 提取
~~~~~~~~~~~~~~~~~~~~~~

.. class:: ExtractYear(expression, tzinfo=None, **extra)

    .. attribute:: lookup_name = 'year'

.. class:: ExtractMonth(expression, tzinfo=None, **extra)

    .. attribute:: lookup_name = 'month'

.. class:: ExtractDay(expression, tzinfo=None, **extra)

    .. attribute:: lookup_name = 'day'

.. class:: ExtractWeekDay(expression, tzinfo=None, **extra)

    .. attribute:: lookup_name = 'week_day'

这些逻辑等价于 ``Extract('date_field', lookup_name)``. 每个类也是在 ``DateField`` 和 ``DateTimeField`` 上注册为 ``__(lookup_name)`` 的 ``Transform``, 例如 ``__year``.

由于 ``DateField`` 没有time部分, 因此只有处理date部分的 ``Extract`` 子类才能在 ``DateField`` 上使用::

    >>> from datetime import datetime
    >>> from django.utils import timezone
    >>> from django.db.models.functions import (
    ...    ExtractYear, ExtractMonth, ExtractDay, ExtractWeekDay
    ... )
    >>> start_2015 = datetime(2015, 6, 15, 23, 30, 1, tzinfo=timezone.utc)
    >>> end_2015 = datetime(2015, 6, 16, 13, 11, 27, tzinfo=timezone.utc)
    >>> Experiment.objects.create(
    ...    start_datetime=start_2015, start_date=start_2015.date(),
    ...    end_datetime=end_2015, end_date=end_2015.date())
    >>> Experiment.objects.annotate(
    ...     year=ExtractYear('start_date'),
    ...     month=ExtractMonth('start_date'),
    ...     day=ExtractDay('start_date'),
    ...     weekday=ExtractWeekDay('start_date'),
    ... ).values('year', 'month', 'day', 'weekday').get(
    ...     end_date__year=ExtractYear('start_date'),
    ... )
    {'year': 2015, 'month': 6, 'day': 15, 'weekday': 2}

``DateTimeField`` 提取
~~~~~~~~~~~~~~~~~~~~~~~~~~

除了以下内容外, 上面所有的 ``DateField`` extracts也可以应用于 ``DateTimeField``.

.. class:: ExtractHour(expression, tzinfo=None, **extra)

    .. attribute:: lookup_name = 'hour'

.. class:: ExtractMinute(expression, tzinfo=None, **extra)

    .. attribute:: lookup_name = 'minute'

.. class:: ExtractSecond(expression, tzinfo=None, **extra)

    .. attribute:: lookup_name = 'second'

这些逻辑等价于 ``Extract('datetime_field', lookup_name)``. 每个类也是在 ``DateTimeField`` 上注册为 ``__(lookup_name)`` 的 ``Transform``, 例如 ``__minute``.

``DateTimeField`` 例子::

    >>> from datetime import datetime
    >>> from django.utils import timezone
    >>> from django.db.models.functions import (
    ...    ExtractYear, ExtractMonth, ExtractDay, ExtractWeekDay,
    ...    ExtractHour, ExtractMinute, ExtractSecond,
    ... )
    >>> start_2015 = datetime(2015, 6, 15, 23, 30, 1, tzinfo=timezone.utc)
    >>> end_2015 = datetime(2015, 6, 16, 13, 11, 27, tzinfo=timezone.utc)
    >>> Experiment.objects.create(
    ...    start_datetime=start_2015, start_date=start_2015.date(),
    ...    end_datetime=end_2015, end_date=end_2015.date())
    >>> Experiment.objects.annotate(
    ...     year=ExtractYear('start_datetime'),
    ...     month=ExtractMonth('start_datetime'),
    ...     day=ExtractDay('start_datetime'),
    ...     weekday=ExtractWeekDay('start_datetime'),
    ...     hour=ExtractHour('start_datetime'),
    ...     minute=ExtractMinute('start_datetime'),
    ...     second=ExtractSecond('start_datetime'),
    ... ).values(
    ...     'year', 'month', 'day', 'weekday', 'hour', 'minute', 'second',
    ... ).get(end_datetime__year=ExtractYear('start_datetime'))
    {'year': 2015, 'month': 6, 'day': 15, 'weekday': 2, 'hour': 23, 'minute': 30, 'second': 1}

当 :setting:`USE_TZ` 设置为 ``True`` 时数据库中的日期时间是以UTC储存. 如果在 Django 中使用了不同的时区, 那么在提取前会先转换为当前时区.
下面的例子将转换为墨尔本时区(UTC +10:00), 返回的日期、星期和小时值会有改变::

    >>> import pytz
    >>> tzinfo = pytz.timezone('Australia/Melbourne')  # UTC+10:00
    >>> with timezone.override(tzinfo):
    ...    Experiment.objects.annotate(
    ...        day=ExtractDay('start_datetime'),
    ...        weekday=ExtractWeekDay('start_datetime'),
    ...        hour=ExtractHour('start_datetime'),
    ...    ).values('day', 'weekday', 'hour').get(
    ...        end_datetime__year=ExtractYear('start_datetime'),
    ...    )
    {'day': 16, 'weekday': 3, 'hour': 9}

显式地传递时区给 ``Extract``, 将优先于当前时区::

    >>> import pytz
    >>> tzinfo = pytz.timezone('Australia/Melbourne')
    >>> Experiment.objects.annotate(
    ...     day=ExtractDay('start_datetime', tzinfo=melb),
    ...     weekday=ExtractWeekDay('start_datetime', tzinfo=melb),
    ...     hour=ExtractHour('start_datetime', tzinfo=melb),
    ... ).values('day', 'weekday', 'hour').get(
    ...     end_datetime__year=ExtractYear('start_datetime'),
    ... )
    {'day': 16, 'weekday': 3, 'hour': 9}


``Trunc``
---------

.. class:: Trunc(expression, kind, output_field=None, tzinfo=None, **extra)

日期截取方法.

当你只关心年份, 小时或者天数, 不需要确切的秒数, 那么 ``Trunc`` (及其子类)可以用于过滤或聚合数据.
例如, 可以使用 ``Trunc`` 计算每天的销售额.

``Trunc`` 接收一个 ``DateField`` 或 ``DateTimeField`` 的 ``expression``, ``kind`` 表示日期部分,
``output_field`` 可以是 ``DateTimeField()`` 或 ``DateField()``. 它根据 ``output_field`` 返回datetime或date, ``kind`` 以下的部分被设置为最小值.
如果 ``output_field`` 缺省, 将默认为 ``expression`` 的 ``output_field``.
可以传递 ``pytz`` 模块提供的 ``tzinfo`` 子类来指定时区.

给定datetime ``2015-06-15 14:30:50.000321+00:00``, 内置的 ``kind`` 返回:

* "year": 2015-01-01 00:00:00+00:00
* "month": 2015-06-01 00:00:00+00:00
* "day": 2015-06-15 00:00:00+00:00
* "hour": 2015-06-15 14:00:00+00:00
* "minute": 2015-06-15 14:30:00+00:00
* "second": 2015-06-15 14:30:50+00:00

如果Django中使用了不同的时区如 ``Australia/Melbourne``, 那么会在截取前将datetime转换为当前时区.
上述示例日期中墨尔本的时区偏移为+10:00. 使用此时区时, 返回的值变成:

* "year": 2015-01-01 00:00:00+11:00
* "month": 2015-06-01 00:00:00+10:00
* "day": 2015-06-16 00:00:00+10:00
* "hour": 2015-06-16 00:00:00+10:00
* "minute": 2015-06-16 00:30:00+10:00
* "second": 2015-06-16 00:30:50+10:00

该年度的偏移量为+11：00, 因为结果已转换为夏令时.

以上每个 ``kind`` 都有相应的 ``Trunc`` 子类, 通常应该使用这个子类来代替上面比较冗长的用法.
例如. 使用 ``TruncYear(...)`` 代替 ``Trunc(..., kind='year')``.

子类都被定义为变换, 但它们没有注册任何字段, 因为 ``Extract`` 子类已经保留了显示的查找名称.

用例::

    >>> from datetime import datetime
    >>> from django.db.models import Count, DateTimeField
    >>> from django.db.models.functions import Trunc
    >>> Experiment.objects.create(start_datetime=datetime(2015, 6, 15, 14, 30, 50, 321))
    >>> Experiment.objects.create(start_datetime=datetime(2015, 6, 15, 14, 40, 2, 123))
    >>> Experiment.objects.create(start_datetime=datetime(2015, 12, 25, 10, 5, 27, 999))
    >>> experiments_per_day = Experiment.objects.annotate(
    ...    start_day=Trunc('start_datetime', 'day', output_field=DateTimeField())
    ... ).values('start_day').annotate(experiments=Count('id'))
    >>> for exp in experiments_per_day:
    ...     print(exp['start_day'], exp['experiments'])
    ...
    2015-06-15 00:00:00 2
    2015-12-25 00:00:00 1
    >>> experiments = Experiment.objects.annotate(
    ...    start_day=Trunc('start_datetime', 'day', output_field=DateTimeField())
    ... ).filter(start_day=datetime(2015, 6, 15))
    >>> for exp in experiments:
    ...     print(exp.start_datetime)
    ...
    2015-06-15 14:30:50.000321
    2015-06-15 14:40:02.000123

``DateField`` 截取
~~~~~~~~~~~~~~~~~~~~~~~~

.. class:: TruncYear(expression, output_field=None, tzinfo=None, **extra)

    .. attribute:: kind = 'year'

.. class:: TruncMonth(expression, output_field=None, tzinfo=None, **extra)

    .. attribute:: kind = 'month'

它们逻辑上等价于 ``Trunc('date_field', kind)``. 它们截取日期的所有部分直至 ``kind``,
允许以较低的精度对日期进行分组或过滤. ``expression`` 的 ``output_field`` 可以是 ``DateField`` 或 ``DateTimeField``.

由于 ``DateField`` 没有time部分, 只有处理date部分的 ``Trunc`` 子类才能与 ``DateField`` 使用::

    >>> from datetime import datetime
    >>> from django.db.models import Count
    >>> from django.db.models.functions import TruncMonth, TruncYear
    >>> from django.utils import timezone
    >>> start1 = datetime(2014, 6, 15, 14, 30, 50, 321, tzinfo=timezone.utc)
    >>> start2 = datetime(2015, 6, 15, 14, 40, 2, 123, tzinfo=timezone.utc)
    >>> start3 = datetime(2015, 12, 31, 17, 5, 27, 999, tzinfo=timezone.utc)
    >>> Experiment.objects.create(start_datetime=start1, start_date=start1.date())
    >>> Experiment.objects.create(start_datetime=start2, start_date=start2.date())
    >>> Experiment.objects.create(start_datetime=start3, start_date=start3.date())
    >>> experiments_per_year = Experiment.objects.annotate(
    ...    year=TruncYear('start_date')).values('year').annotate(
    ...    experiments=Count('id'))
    >>> for exp in experiments_per_year:
    ...     print(exp['year'], exp['experiments'])
    ...
    2014-01-01 1
    2015-01-01 2

    >>> import pytz
    >>> melb = pytz.timezone('Australia/Melbourne')
    >>> experiments_per_month = Experiment.objects.annotate(
    ...    month=TruncMonth('start_datetime', tzinfo=melb)).values('month').annotate(
    ...    experiments=Count('id'))
    >>> for exp in experiments_per_month:
    ...     print(exp['month'], exp['experiments'])
    ...
    2015-06-01 00:00:00+10:00 1
    2016-01-01 00:00:00+11:00 1
    2014-06-01 00:00:00+10:00 1

``DateTimeField`` 截取
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. class:: TruncDate(expression, **extra)

    .. attribute:: lookup_name = 'date'
    .. attribute:: output_field = DateField()

``TruncDate`` 将 ``expression`` 转换成date, 而不是使用内置的SQL truncate函数. 它也在  ``DateTimeField`` 上注册为 ``__date`` 的转换.

.. class:: TruncDay(expression, output_field=None, tzinfo=None, **extra)

    .. attribute:: kind = 'day'

.. class:: TruncHour(expression, output_field=None, tzinfo=None, **extra)

    .. attribute:: kind = 'hour'

.. class:: TruncMinute(expression, output_field=None, tzinfo=None, **extra)

    .. attribute:: kind = 'minute'

.. class:: TruncSecond(expression, output_field=None, tzinfo=None, **extra)

    .. attribute:: kind = 'second'

它们在逻辑上等价于 ``Trunc('datetime_field', kind)``. 它们截取日期的所有部分直至 ``kind``,
允许以较低的精度对日期进行分组或过滤. ``expression`` 的 ``output_field`` 必须是 ``DateTimeField``.

用例::

    >>> from datetime import date, datetime
    >>> from django.db.models import Count
    >>> from django.db.models.functions import (
    ...     TruncDate, TruncDay, TruncHour, TruncMinute, TruncSecond,
    ... )
    >>> from django.utils import timezone
    >>> import pytz
    >>> start1 = datetime(2014, 6, 15, 14, 30, 50, 321, tzinfo=timezone.utc)
    >>> Experiment.objects.create(start_datetime=start1, start_date=start1.date())
    >>> melb = pytz.timezone('Australia/Melbourne')
    >>> Experiment.objects.annotate(
    ...     date=TruncDate('start_datetime'),
    ...     day=TruncDay('start_datetime', tzinfo=melb),
    ...     hour=TruncHour('start_datetime', tzinfo=melb),
    ...     minute=TruncMinute('start_datetime'),
    ...     second=TruncSecond('start_datetime'),
    ... ).values('date', 'day', 'hour', 'minute', 'second').get()
    {'date': datetime.date(2014, 6, 15),
     'day': datetime.datetime(2014, 6, 16, 0, 0, tzinfo=<DstTzInfo 'Australia/Melbourne' AEST+10:00:00 STD>),
     'hour': datetime.datetime(2014, 6, 16, 0, 0, tzinfo=<DstTzInfo 'Australia/Melbourne' AEST+10:00:00 STD>),
     'minute': 'minute': datetime.datetime(2014, 6, 15, 14, 30, tzinfo=<UTC>),
     'second': datetime.datetime(2014, 6, 15, 14, 30, 50, tzinfo=<UTC>)
    }
