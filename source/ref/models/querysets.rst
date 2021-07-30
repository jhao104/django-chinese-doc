========================
``QuerySet`` API 参考
========================

.. currentmodule:: django.db.models.query

本篇文档包含 ``QuerySet`` API的详细信息. 它建立在 :doc:`模型 </topics/db/models>` 和
:doc:`数据库查询 </topics/db/queries>` 指南之上, 所以在阅读本文档之前, 你需要先阅读和理解这两部分的文档.

本文档将通篇使用在 :doc:`数据库查询指南
</topics/db/queries>` 中用到的 :ref:`WebBlog模型例子
<queryset-model-example>`.

.. _when-querysets-are-evaluated:

``QuerySet`` 何时求值
======================

实际上, 当一个 ``QuerySet`` 被创建, 过滤, 切片和传递时并不会实际操作数据库.
在对查询集做求值之前, 不会产生任何实际的数据库操作.

``QuerySet`` 求值有以下几种方式:

* **迭代.** ``QuerySet`` 是可迭代的, 当它被首次迭代是会执行数据库查询. 例如,
  下面的语句会将数据库中所有Entry的headline打印出来::

      for e in Entry.objects.all():
          print(e.headline)

  主要: 不要使用上面语句来验证数据库中是否存在某条记录, 使用 :meth:`~QuerySet.exists` 方法更高效.

* **切片.** 正如 :ref:`limiting-querysets` 中描述一样, 可以使用Python的序列切片语法对 ``QuerySet`` 进行切片操作,
  对一个未求值的 ``QuerySet`` 进行切片操作会返回另一个未求值的 ``QuerySet``,
  但是如果使用了 "step" 参数, Django将执行数据库查询, 然后返回查询结果的列表.
  对已经求值过的 ``QuerySet`` 切片也是返回一个列表.

  需要注意的是, 对未求值的 ``QuerySet`` 切片返回的 ``QuerySet`` 不可以再进行修改操作(e.g.,
  新增过滤器, 或者修改排序). 因为这样不难再转化成SQL而且这样的需求没有实际意义.

* **Pickling/缓存.** 有关 `pickling QuerySets`_ 的细节, 请参阅 `pickling QuerySets`_ 部份.
  这里提到它的目的是强调序列化时会读取数据库.

* **repr().** 当对 ``QuerySet`` 调用 ``repr()`` 方法时会对其求值.
  这是为了在Python交互式解释器中使用方便, 这样就可以在交互式解释器中使用这个API立即看到结果.

* **len().** 当对 ``QuerySet`` 调用 ``len()`` 方法时会对其求值.
  正如猜想那样，会返回查询集的长度.

  注意: 如果只是想确认集合中的记录条数(而并不需要实际对象), 使用SQL的 ``SELECT COUNT(*)`` 来处理数据库级别的计数更有效.
  Django为此提供了 :meth:`~QuerySet.count` 方法.

* **list().** 对 ``QuerySet`` 调用 ``list()`` 可以对其进行强制求值, 例如::

      entry_list = list(Entry.objects.all())

* **bool().** 试探 ``QuerySet`` 的布尔值, 例如使用 ``bool()``, ``or``, ``and`` 或者 ``if`` 判断,
  都触发求值操作. 如果结果至少包含一条记录, 则 ``QuerySet`` 为 ``True``, 否则为 ``False``. 例如::

      if Entry.objects.filter(headline="Test"):
         print("Entry 中至少有一条记录的headline为Test")

  注意: 如果只是想确认结果中是否至少存在一条记录(并且不需要实际对象), 使用 :meth:`~QuerySet.exists` 方法更加高效.

.. _pickling QuerySets:

Pickling ``QuerySet``
-----------------------

如果对 ``QuerySet`` 进行 :mod:`pickle` 操作, 它将在Pickle之前强制将所有的结果加载到内存中.
Pickling 通常用缓存之前, 当下次重新加载缓存的查询集时, 其结果已经就是能够直接使用的了(免去了再次从数据库读取的耗时).
也就是说当unpickle ``QuerySet`` 时, 就是从数据库中查询的结果.

如果只是想序列化部分必要的信息, 以便后面可以从数据库中重建 ``Queryset``, 那只序列化 ``QuerySet`` 的 ``query`` 属性即可.
接下来就可以使用下面的代码重建原来的 ``QuerySet`` (这个过程没有数据库读取)::

    >>> import pickle
    >>> query = pickle.loads(s)     # Assuming 's' is the pickled string.
    >>> qs = MyModel.objects.all()
    >>> qs.query = query            # Restore the original 'query'.

``query`` 属性是一个不透明的对象. 它表示查询的内部结构, 不属于公开的API. 即便如此, 对于本节提到的序列化和反序列化来说, 它仍是安全和被完全支持的.

.. admonition:: 不同版本间不能共享Pickle结果

    ``QuerySets`` 的Pickle只能用于生成它们的Django版本中. 如果使用Django的版本N生成一个Pickle,
    不保证这个Pickle在Django 的版本N+1中可以读取. Pickle不可用于归档的长期策略.

    因为Pickle兼容性的错误很难诊断例如产生损坏的对象， 当试图Unpickle的查询集与Pickle时的Django 版本不同时，将引发一个 ``RuntimeWarning``.

.. _queryset-api:

``QuerySet`` API
================

下面是对 ``QuerySet`` 的正式定义:

.. class:: QuerySet(model=None, query=None, using=None)

    通常使用 ``QuerySet`` 时会以 :ref:`链式过滤 <chaining-filters>` 来使用. 因此大部分
    ``QuerySet`` 方法返回的是一个新的查询集. 本节将会详细介绍这些方法.

    ``QuerySet`` 类具有两个可用于自省的公共属性:

    .. attribute:: ordered

        如果 ``QuerySet`` 是有序的则为 ``True``  — 例如
        :meth:`order_by()` 子句或者模型默认的排序. 否则为 ``False`` .

    .. attribute:: db

        如果执行查询, 将使用该数据库.

    .. note::

        :class:`QuerySet` 存在 ``query`` 参数是为了让具有特殊查询用途的子类如
        :class:`~django.contrib.gis.db.models.GeoQuerySet` 可以重新构造内部查询状态.
        这个参数的值是查询状态的不透明表示， 不是一个公开的API.
        简而言之：如果你有疑问，其实你实际上不需要使用它.

.. currentmodule:: django.db.models.query.QuerySet

返回新 ``QuerySet`` 的方法
----------------------------

Django提供了一系列的 ``QuerySet`` 筛选方法，用于修改 ``QuerySet`` 返回的结果类型或者SQL的查询方式.

``filter()``
~~~~~~~~~~~~

.. method:: filter(**kwargs)

返回一个新的包含满足查询参数的 ``QuerySet`` 对象.

查询参数(``**kwargs``) 必须满足下文 `Field 查询`_ 的格式.

如果需要更复杂的查询 (例如 ``OR`` 语句), 可以使用 :class:`Q查询 <django.db.models.Q>`.

``exclude()``
~~~~~~~~~~~~~

.. method:: exclude(**kwargs)

返回一个新的不包含满足查询参数的 ``QuerySet`` 对象.

查询参数(``**kwargs``) 必须满足下文 `Field 查询`_ 的格式. 在底层SQL语句中, 多个参数通过 ``AND`` 连接.
然后所查的内容都会被放入 ``NOT()`` 句子中.

下面的示例排除所有 ``pub_date`` 大于2005-1-3 且 ``headline`` 为“Hello”的记录::

    Entry.objects.exclude(pub_date__gt=datetime.date(2005, 1, 3), headline='Hello')

用SQL语句表示, 它等同于::

    SELECT ...
    WHERE NOT (pub_date > '2005-1-3' AND headline = 'Hello')

下面示例排序所有 whose ``pub_date`` 大于 2005-1-3 或者
headline 为 "Hello"的记录::

    Entry.objects.exclude(pub_date__gt=datetime.date(2005, 1, 3)).exclude(headline='Hello')

用SQL语句表示, 它等同于::

    SELECT ...
    WHERE NOT pub_date > '2005-1-3'
    AND NOT headline = 'Hello'

第二个例子过滤更严格.

如果需要更复杂的查询 (例如 ``OR`` 语句), 可以使用 :class:`Q查询 <django.db.models.Q>`.

``annotate()``
~~~~~~~~~~~~~~

.. method:: annotate(*args, **kwargs)

使用 :doc:`查询表达式 </ref/models/expressions>` 注解 ``QuerySet`` 中的每个对象.
表达式可以是简单的值、模型或关联模型的字段引用,或者是对与 ``QuerySet`` 中对象相关的对象进行计算的聚合表达式(平均值、总和等).

``annotate()`` 的每个参数都是一个注解，将添加到返回的 ``QuerySet`` 中的每个对象中.

Django提供的聚合函数在下文的 `聚合函数`_ 文档中有详细介绍.

关键字参数指定的注解将使用关键字作为注解的别名. 匿名参数的别名将基于聚合函数的名称和模型的字段生成.
只有引用单个字段的聚合表达式才可以使用匿名参数. 其它所有形式都必须用关键字参数.

例如，如果操作一个Blog列表，如果想知道每个Blog有多少Entry::

    >>> from django.db.models import Count
    >>> q = Blog.objects.annotate(Count('entry'))
    # The name of the first blog
    >>> q[0].name
    'Blogasaurus'
    # The number of entries on the first blog
    >>> q[0].entry__count
    42

``Blog`` 模型本身并没有定义 ``entry__count`` 属性, 但是如果使用关键字参数来指定聚合函数. 就生成了相应注解名称的属性::

    >>> q = Blog.objects.annotate(number_of_entries=Count('entry'))
    # The number of entries on the first blog, using the name provided
    >>> q[0].number_of_entries
    42

有关聚合的深入讨论,参考 :doc:`聚合主题指南 </topics/db/aggregation>`.

``order_by()``
~~~~~~~~~~~~~~

.. method:: order_by(*fields)

默认情况下， ``QuerySet`` 返回的结果是根据模型 ``Meta`` 中的 ``ordering`` 选项给出的排序元组排序.
也可以使用 ``order_by`` 方法给每个 ``QuerySet`` 指定特定的排序.

例如::

    Entry.objects.filter(pub_date__year=2005).order_by('-pub_date', 'headline')

上面结果将根据 ``pub_date`` 降序, 按 ``headline`` 升序. ``"-pub_date"`` 前面的负号表示
*降序* 排列. 隐式形式是升序排列, 使用 ``"?"`` 表示随机排序, 例如::

    Entry.objects.order_by('?')

注意: ``order_by('?')`` 查询可能会耗费资源且很慢, 这也取决于使用的数据库.

若要按照另外一个模型中的字段排序, 可以使用查询关联模型时的语法. 即通过字段的名称后面跟上两个下划线(``__``),
再跟上新模型中的字段的名称,像这样::

    Entry.objects.order_by('blog__name', 'headline')

如果根据关联模型字段排序, Django将使用关联的模型的默认排序, 或者如果没有指定 :attr:`Meta.ordering
<django.db.models.Options.ordering>`  将通过关联的模型的主键排序. 例如, 因为 ``Blog`` 模型没有指定默认的排序::

    Entry.objects.order_by('blog')

...其等价于::

    Entry.objects.order_by('blog__id')

如果 ``Blog`` 设置了 ``ordering = ['name']``, 那么第一个查询等价于::

    Entry.objects.order_by('blog__name')

通过引用相关字段的 ``_id`` , 同样可以通过相关字段来排序查询集，而不会导致JOIN开销::

    # No Join
    Entry.objects.order_by('blog_id')

    # Join
    Entry.objects.order_by('blog__id')

你也可以通过 :doc:`查询表达式 </ref/models/expressions>` 调用
 ``asc()`` 或者 ``desc()`` 排序::

    Entry.objects.order_by(Coalesce('summary', 'headline').desc())

当使用关联模型排序还使用到了 :meth:`distinct()` 时需要注意,
:meth:`distinct` 中有说明关联模型的排序如何会对预期结果产生影响.

.. note::
    指定一个多值字段来排序结果(例如, 一个 :class:`~django.db.models.ManyToManyField` 字段,
    或者 :class:`~django.db.models.ForeignKey` 的反向关联字段)

    考虑下面这种情况::

         class Event(Model):
            parent = models.ForeignKey(
                'self',
                on_delete=models.CASCADE,
                related_name='children',
            )
            date = models.DateField()

         Event.objects.order_by('children__date')

    在这里，每个 ``Event`` 可能有多个排序数据；具有多个 ``children`` 的每个 ``Event`` 将被多次返回到 ``order_by()``
    创建的新的 ``QuerySet`` 中. 换句话说, 用 ``order_by()`` 方法对 ``QuerySet`` 对象进行操作会返回一个扩大版的新
    ``QuerySet`` 对象——新增的条目也许并没有什么用，你也用不着它们.

    因此，当使用多值字段对结果进行排序时要格外小心. **如果** 可以确保每个排序项只有一个排序数据,
    这种方法不会出现问题. 如果不确定，请确保结果是你期望的.


是没有方法指定排序是否对大小写敏感. 对于大小写的敏感性, Django将根据数据库中的排序方式给出排序结果.

你可以通过 :class:`~django.db.models.functions.Lower` 将字段转换为小写来排序, 这样就能达到大小写一致的排序::

    Entry.objects.order_by(Lower('headline').desc())

如果你不需要查询做任何排序,默认排序也不需要! 可以调用不带参数的 :meth:`order_by()` .

可以通过检查 :attr:`.QuerySet.ordered` 来判断查询结果是否有序. 不论 ``QuerySet`` 以任何方式排序，它将是 ``True``.

每个 ``order_by()`` 都会清除它之前的所有排序. 例如, 下面查询将会按照
``pub_date`` 排序而不是 ``headline``::

    Entry.objects.order_by('headline').order_by('pub_date')

.. warning::

    排序不是没有开销的操作. 添加到排序中的每个字段都将带来数据库的开销. 添加的每个外键也都将隐式包含进它的默认排序.

    如果查询没有指定顺序，则会以未指定的顺序从数据库返回结果. 仅当通过唯一标识结果中的每个对象的一组字段排序时，
    才能保证特定的排序。 例如，如果 ``name`` 字段不唯一，由其排序则不会保证具有相同名称的对象总是以相同的顺序显示.

``reverse()``
~~~~~~~~~~~~~

.. method:: reverse()

 ``reverse()`` 方法用于反向排序QuerySet中的元素. 再次调用 ``reverse()`` 将恢复原有排序.

比如要获取QuerySet中的最后五个元素,可以这样::

    my_queryset.reverse()[:5]

注意, 这和Python中的在列表末尾切片不一样. 上面例子将先返回最后一个元素,然后是倒数第二个,依次类推.
如果在Python序列中调用 ``seq[-5:]``, 我们将先看到返回的倒数第五个元素. Django 并不支持这种模式访问(从末尾切片),
因此这不好在SQL中高效实现.

同时, ``reverse()`` 也只能在定义了ordering的 ``QuerySet`` 上调用
(e.g., 一个定义了默认排序的模型，或者使用了 :meth:`order_by()` 方法).
如果给定的 ``QuerySet`` 没有定义这样的ordering，那么调用 ``reverse()`` 就没有实际效果
(``reverse()`` 之前没有定义ordering, 之后也将保持未定义).

``distinct()``
~~~~~~~~~~~~~~

.. method:: distinct(*fields)

返回一个在SQL查询中使用 ``SELECT DISTINCT`` 句子的 ``QuerySet``.  它将去除查询结果中重复的行.

默认情况下, ``QuerySet`` 不会进行去重操作. 而在实际情况中, 这一般不会有问题, 因为像 ``Blog.objects.all()`` 这样简答的查询不会引入重复的行.
但是, 在跨多表查询时， ``QuerySet`` 可能就会包含重复的结果. 这时候就应该使用 ``distinct()``.

.. note ::
    :meth:`order_by` 调用中使用的任何字段都包含在SQL的 ``SELECT`` 列当中. 当和 ``distinct()`` 一起使用时，可能会导致意料之外的结果.
    如果根据关联模型的字段排序，那么这个字段将被添加到查询字段中，这样它们可能会使其他本来是重复的行看起来不同了.
    而由于这个额外的字段不会出现在返回的结果中(它们只用于排序)，所以这时看起来返回的结果并不正确.

    类似地，如果使用 :meth:`values()` 查询来限制所选的列，那么 :meth:`order_by()` (或默认的模型排序)中使用的列仍然会涉及，并可能影响结果的唯一性。

    上面的意思是，如果您使用的是 ``distinct()`` ，那么使用相关模型字段排序时一定得小心。同样，
    当将 ``distinct()`` 和 :meth:`values()` 一起使用时，请注意字段在不在 :meth:`values()` 中。

仅在PostgreSQL上, 可以传递位置参数(``*fields``), 用来指定 ``DISTINCT`` 应该应用到的字段的名称.
转换为SQL查询上的 ``SELECT DISTINCT ON``. 区别于其他的普通 ``distinct()`` 调用,
数据库在确定哪些行是不同的时候比较每一行中的每个字段. 对于具有指定字段名的 ``distinct()`` 调用, 数据库将只比较指定字段名.

.. note::

    当指定字段名时, *必须* 在 ``QuerySet`` 中使用 ``order_by()``, ``order_by()`` 中的字段必须和 ``distinct()`` 字段顺序相同.

    例如, ``SELECT DISTINCT ON (a)`` 为每个列 ``a`` 中提供第一行, 如果没有指定顺序就会返回随机的一行.


示例 (除第一个例子,其他仅在PostgreSQL上有效)::

    >>> Author.objects.distinct()
    [...]

    >>> Entry.objects.order_by('pub_date').distinct('pub_date')
    [...]

    >>> Entry.objects.order_by('blog').distinct('blog')
    [...]

    >>> Entry.objects.order_by('author', 'pub_date').distinct('author', 'pub_date')
    [...]

    >>> Entry.objects.order_by('blog__name', 'mod_date').distinct('blog__name', 'mod_date')
    [...]

    >>> Entry.objects.order_by('author', 'pub_date').distinct('author')
    [...]

.. note::
    注意, :meth:`order_by` 使用在定义了默认排序的关联模型中时，可能需要使用关联 ``_id`` 或者关联字段显式排序,
    以 ``DISTINCT ON`` 表达式与 ``ORDER BY`` 子句开头的表达式匹配. 例如，如果 ``Blog`` 模型按定义了一个
    按 ``name`` 的 :attr:`~django.db.models.Options.ordering`::

        Entry.objects.order_by('blog').distinct('blog')

    ...将不会生效, 因为查询时将会使用 ``blog_name`` 排序, 这与 ``DISTINCT ON`` 表达式不匹配. 这种情况下必须使用关联 `_id` 字段
     (该例中为 ``blog_id`` ) 或者引用的字段(``blog__pk``) 显式排序, 保证两个表达式匹配.

``values()``
~~~~~~~~~~~~

.. method:: values(*fields)

返回一个 ``QuerySet`` ，该 ``QuerySet`` 返回字典，而不是可迭代的模型实例.

每个字典都表示一个对象, 其键对应于模型对象的属性名.

这个例子比较了 ``values()`` 字典和普通模型对象::

    # This list contains a Blog object.
    >>> Blog.objects.filter(name__startswith='Beatles')
    <QuerySet [<Blog: Beatles Blog>]>

    # This list contains a dictionary.
    >>> Blog.objects.filter(name__startswith='Beatles').values()
    <QuerySet [{'id': 1, 'name': 'Beatles Blog', 'tagline': 'All the latest Beatles news.'}]>

``values()`` 方法接受可选位置参数 ``*fields``, 其作用是用于指定 ``SELECT`` 中限制的字段名.
如果设置了限制字段, 那个所有字典只会包含指定的 键/值. 如果没有指定字段, 那么所有字段将包含数据库表中所有字段的键值.

例子::

    >>> Blog.objects.values()
    <QuerySet [{'id': 1, 'name': 'Beatles Blog', 'tagline': 'All the latest Beatles news.'}]>
    >>> Blog.objects.values('id', 'name')
    <QuerySet [{'id': 1, 'name': 'Beatles Blog'}]>

值得注意的几点:

* 如果有一个名为 ``foo`` 的 :class:`~django.db.models.ForeignKey` 字段, 那么调用默认的 ``values()`` 返回的字典将
  包含一个 ``foo_id`` 的键, 因为这是存储实际值的隐藏模型属性名称(``foo`` 属性引用相关的模型). 当调用 ``values()`` 并传入字段名时,
  您可以传入 ``foo`` 或 ``foo_id``, 返回相同的内容(字典键会匹配传入的字段名).

  示例::

    >>> Entry.objects.values()
    <QuerySet [{'blog_id': 1, 'headline': 'First Entry', ...}, ...]>

    >>> Entry.objects.values('blog')
    <QuerySet [{'blog': 1}, ...]>

    >>> Entry.objects.values('blog_id')
    <QuerySet [{'blog_id': 1}, ...]>

* 当同时使用 ``values()`` 和 ``distinct()`` 时, 注意排序会影响结果. 详细信息请参阅 :meth:`distinct` 中的注释.

* 如果在 :meth:`extra()` 调用之后使用 ``values()`` 子句, 那么在 :meth:`extra()` 中的 ``select`` 参数定义的任何字段都必须显式包含
  在 ``values()`` 中. 在 ``values()`` 之后进行的任何 :meth:`extra()` 都将忽略其selected的额外字段.

* 在 ``values()`` 之后调用 :meth:`only()` 和 :meth:`defer()` 不太合理, 因此这么做会引发 ``NotImplementedError``.

当只需要少量可用字段的值, 并且不需要模型实例对象的功能时, 只选择需要使用的字段会更有效.

最后, 可以在 ``values()`` 调用之后调用 ``filter()`` 、 ``order_by()`` 等, 这意味着下面这两个调用是相同的::

    Blog.objects.values().order_by('id')
    Blog.objects.order_by('id').values()

Django的开发者喜欢将所有影响sql的方法放在前面(可选), 然后才是影响输出的方法(例如 ``values()`` ),
但是实际上无所谓, 这是卖弄你个性的好机会.

还可以通过 ``OneToOneField``, ``ForeignKey`` 和 ``ManyToManyField`` 属性来引用具有反向关系的相关模型的字段::

    >>> Blog.objects.values('name', 'entry__headline')
    <QuerySet [{'name': 'My blog', 'entry__headline': 'An entry'},
         {'name': 'My blog', 'entry__headline': 'Another entry'}, ...]>

.. warning::

   因为 :class:`~django.db.models.ManyToManyField` 字段和反向关联关系可以有多个关联的行,
   包括这些行可能会使结果集倍数放大.如果在 ``values()`` 查询中包含多个此类字段，这会特别明显，
   在这种情况下，将返回所有可能的组合.

``values_list()``
~~~~~~~~~~~~~~~~~

.. method:: values_list(*fields, flat=False)

它与 ``values()`` 非常类似, 只是在迭代时返回的是元组而不是字典.
每个元组包含传递到 ``values_list()`` 的相应字段的值——因此第一个项是第一个字段, etc. 例如::

    >>> Entry.objects.values_list('id', 'headline')
    [(1, 'First entry'), ...]

如果只传递了一个字段, 可以使用 ``flat`` 参数. 如何设置为 ``True``, 返回的结果将会是单个值而不是元组,
下面的例子更容易理解其作用::

    >>> Entry.objects.values_list('id').order_by('id')
    [(1,), (2,), (3,), ...]

    >>> Entry.objects.values_list('id', flat=True).order_by('id')
    [1, 2, 3, ...]

如果传入多个字段同时设置了 ``flat`` 时将产生错误.

如果没有向``values_list()`` 中传入字段, 那么它将会返回模型中所有字段, 顺序为模型在定义的顺序.

一个常见的需求是获取某个模型实例的特定字段值. 使用 ``values_list()`` 跟上 ``get()`` 调用来实现::

    >>> Entry.objects.values_list('headline', flat=True).get(pk=1)
    'First entry'

这个比喻在处理多对多和其他多值关系（例如反向外键的一对多关系）时分歧，因为“一行一对象”的假设不成立
``values()`` 和 ``values_list()`` 都是用于特定用例的优化: 检索数据子集而不需要创建模型实例.
但是在处理多对多和其他多值关系(比如反向外键的一对多关系)时不适用, 因为“一行，一个对象”的假设都成立.

例子，注意下面通过 :class:`~django.db.models.ManyToManyField` 进行查询时的行为::

    >>> Author.objects.values_list('name', 'entry__headline')
    [('Noam Chomsky', 'Impressions of Gaza'),
     ('George Orwell', 'Why Socialists Do Not Believe in Fun'),
     ('George Orwell', 'In Defence of English Cooking'),
     ('Don Quixote', None)]

具有多个entry的Author会多次出现，而没有任何entry的Author则是 ``None``.

类似地, 当查询反向外键时. 对于没有entry的Author仍然是 ``None`` ::

    >>> Entry.objects.values_list('authors')
    [('Noam Chomsky',), ('George Orwell',), (None,)]

``dates()``
~~~~~~~~~~~

.. method:: dates(field, kind, order='ASC')

返回一个计算结果为 :class:`datetime.date` 列表的 ``QuerySet``. 内容是 ``QuerySet`` 中某一特定类型的所有可用日期.

``field`` 是模型中 ``DateField`` 字段的名称.
``kind`` 接受 ``"year"`` 、 ``"month"`` 或者 ``"day"`` 参数. 结果中每个
``datetime.date`` 对象都会返回按指定的 ``type`` 截断的结果.

* ``"year"`` 返回字段的所有不同年份值的列表.
* ``"month"`` 返回字段的所有不同 year/month 的列表.
* ``"day"`` 返回字段的所有不同 year/month/day 的列表.

``order``, 默认为 ``'ASC'``, 接受 ``'ASC'`` 和 ``'DESC'`` 两种参数. 用于指定排序方式.

Examples::

    >>> Entry.objects.dates('pub_date', 'year')
    [datetime.date(2005, 1, 1)]
    >>> Entry.objects.dates('pub_date', 'month')
    [datetime.date(2005, 2, 1), datetime.date(2005, 3, 1)]
    >>> Entry.objects.dates('pub_date', 'day')
    [datetime.date(2005, 2, 20), datetime.date(2005, 3, 20)]
    >>> Entry.objects.dates('pub_date', 'day', order='DESC')
    [datetime.date(2005, 3, 20), datetime.date(2005, 2, 20)]
    >>> Entry.objects.filter(headline__contains='Lennon').dates('pub_date', 'day')
    [datetime.date(2005, 3, 20)]

``datetimes()``
~~~~~~~~~~~~~~~

.. method:: datetimes(field_name, kind, order='ASC', tzinfo=None)

返回一个计算结果为 :class:`datetime.datetime` 列表的 ``QuerySet``. 内容是 ``QuerySet`` 中某一特定类型的所有可用日期.


``field_name`` 是模型中 ``DateTimeField`` 字段的名称.

``kind`` 接受 ``"year"``, ``"month"``, ``"day"``, ``"hour"``,
``"minute"`` 和 ``"second"`` 参数. 结果中每个
``datetime.datetime`` 对象都会返回按指定的 ``type`` 截断的结果.

``order``, 默认为 ``'ASC'``, 接受 ``'ASC'`` 和 ``'DESC'`` 两种参数. 用于指定排序方式.

``tzinfo`` 定义在截断之前将数据时间转换到的时区. 这取决于使用的时区. 此参数必须是 :class:`datetime.tzinfo` 对象.
如果传入为 ``None``, Django 会使用 :ref:`当前时区 <default-current-time-zone>`. 当 :setting:`USE_TZ` 设置为 ``False`` 时该项无效.

.. _database-time-zone-definitions:

.. note::

    此函数直接在数据库中执行时区转换。因此，您的数据库必须能够解析 ``tzinfo.tzname(None)`` 的值。这意味着以下要求:

    - SQLite: 安装 pytz_ — 转换过程其实是在Python中完成.
    - PostgreSQL: 没有要求 (参考 `Time Zones`_).
    - Oracle: 没有要求 (参考 `Choosing a Time Zone File`_).
    - MySQL: 安装 pytz_ 并且时使用 `mysql_tzinfo_to_sql`_ 加载时区表.

    .. _pytz: http://pytz.sourceforge.net/
    .. _Time Zones: https://www.postgresql.org/docs/current/static/datatype-datetime.html#DATATYPE-TIMEZONES
    .. _Choosing a Time Zone File: https://docs.oracle.com/cd/E11882_01/server.112/e10729/ch4datetime.htm#NLSPG258
    .. _mysql_tzinfo_to_sql: https://dev.mysql.com/doc/refman/en/mysql-tzinfo-to-sql.html

``none()``
~~~~~~~~~~

.. method:: none()

调用 ``none()`` 会返回一个不反悔任何对象的查询集,并且当访问该查询集时也不会执行任何查询. 比如 qs.none() 的查询集其实就是``EmptyQuerySet`` 的一个实例.

例如::

    >>> Entry.objects.none()
    <QuerySet []>
    >>> from django.db.models.query import EmptyQuerySet
    >>> isinstance(Entry.objects.none(), EmptyQuerySet)
    True

``all()``
~~~~~~~~~

.. method:: all()

返回当前 ``QuerySet`` 的一个 *copy*  (或者 ``QuerySet`` 子类). 它可以用于当你需要传入模型管理器或者对结果做进一步过滤.
无论以哪种方式调用 ``all()`` , 都可以获得一个可以正常工作的 ``QuerySet``.

当 ``QuerySet`` 被 :ref:`求值 <when-querysets-are-evaluated>`, Django会缓存其结果.
如果在此之后数据库中的值发生了改变. 可以通过调用求值前调用的 ``all()`` 来获取更新后的数据.

``select_related()``
~~~~~~~~~~~~~~~~~~~~

.. method:: select_related(*fields)

返回一个“遵循”外键关系的 ``QuerySet``, 在执行查询时可以选择其他关联对象的数据.
这是一个性能增强, 这样可以做更复杂的查询，而且意味着以后使用外键关系不需要再次做数据库查询.

下面的例子解释了普通查询和 ``select_related()`` 查询的区别, 下面是一个标准查询::

    # 访问数据库
    e = Entry.objects.get(id=5)

    # 再次访问数据库以得到关联的Blog对象.
    b = e.blog

下面是 ``select_related`` 查询::

    # 访问数据库
    e = Entry.objects.select_related('blog').get(id=5)

    # 不会访问数据库，因为e.blog已经
    # 在前面的查询中填写好.
    b = e.blog

``select_related()`` 可以用于objects的所有查询集::

    from django.utils import timezone

    # 找到所有pub_date在当前时间之后的blog.
    blogs = set()

    for e in Entry.objects.filter(pub_date__gt=timezone.now()).select_related('blog'):
        # 如果没有使用select_related(), 每次循环都将执行数据库查询来获得每个entry关联的blog
        blogs.add(e.blog)

``filter()`` 和 ``select_related()`` 的使用顺序并无影响.
下两个两个查询集时等同的::

    Entry.objects.filter(pub_date__gt=timezone.now()).select_related('blog')
    Entry.objects.select_related('blog').filter(pub_date__gt=timezone.now())

同样也可以按照类似的方式来查询外键. 例如下面的模型::

    from django.db import models

    class City(models.Model):
        # ...
        pass

    class Person(models.Model):
        # ...
        hometown = models.ForeignKey(
            City,
            on_delete=models.SET_NULL,
            blank=True,
            null=True,
        )

    class Book(models.Model):
        # ...
        author = models.ForeignKey(Person, on_delete=models.CASCADE)

... 然后调用 ``Book.objects.select_related('author__hometown').get(id=4)``
将缓存相关的 ``Person`` *和* 与之相关的 ``City``::

    b = Book.objects.select_related('author__hometown').get(id=4)
    p = b.author         # 不会访问数据库
    c = p.hometown       # 不会访问数据库

    b = Book.objects.get(id=4) # 本例不使用 select_related()
    p = b.author         # 访问数据库
    c = p.hometown       # 访问数据库

``select_related()`` 的字段接受任何 :class:`~django.db.models.ForeignKey` 和
:class:`~django.db.models.OneToOneField` 关联字段.

还可以给 ``select_related`` 传递
:class:`~django.db.models.OneToOneField` 的反向字段 — 这样可以回溯到定义 :class:`~django.db.models.OneToOneField` 字段的对象.
这样可以使用关联字段对象的 :attr:`related_name
<django.db.models.ForeignKey.related_name>` 而不用指定字段名称.

``select_related()`` 的使用场景是当有很多关联对象，或者你不知道所有的关联关系时.
这时可以使用不带参数的 ``select_related()`` . 它可以找到所有不能为空的外键，可以为空的外键必须明确指定.
但是多数情况下不建议这样做, 因为它会使底层的查询变得非常复杂并且返回的数据并不都是真正需要的.

如果需要清除 ``QuerySet`` 上以前的 ``select_related`` 调用添加的关联字段，可以传递一个 ``None`` 作为参数::

   >>> without_relations = queryset.select_related(None)

链式调用 ``select_related`` 的工作方式与其它方法类似 — 也就是说,
``select_related('foo', 'bar')`` 等同于 ``select_related('foo').select_related('bar')``.

``prefetch_related()``
~~~~~~~~~~~~~~~~~~~~~~

.. method:: prefetch_related(*lookups)

返回一个 ``QuerySet`` ，它将在单个批处理中为每个指定查询自动检索相关对象.

它和 ``select_related`` 具体相同的目的, 两者都是用来防止查询关联对象时导致过多的数据库查询,
但是两者的做法是完全不同的.

``select_related`` 通过在 ``SELECT`` 中声明关联字段,使用Join语句关联多个表实现.
因此, ``select_related`` 使用一次数据库查询得到关联对象.
但是, 为了避免关联'多个'关联关系导致查询集太多庞大,
``select_related`` 仅限使用于单值关系 - 外键和一对一.

``prefetch_related`` 则不同, 它为每个关联关系做一次查询, 然后在Python中做'joining'.
这使其支持除 ``select_related`` 支持的外键、一对一关系外, 还支持多对多和多对一关系.
它还支持 prefetching
:class:`~django.contrib.contenttypes.fields.GenericRelation` 和
:class:`~django.contrib.contenttypes.fields.GenericForeignKey`,
但是, 它必须限于一组均匀的结果. 例如, 仅当查询被限制为一个 ``ContentType`` 时,
才支持预取由 ``GenericForeignKey`` 引用的对象.

例子, 假设有如下模型::

    from django.db import models

    class Topping(models.Model):
        name = models.CharField(max_length=30)

    class Pizza(models.Model):
        name = models.CharField(max_length=50)
        toppings = models.ManyToManyField(Topping)

        def __str__(self):              # __unicode__ on Python 2
            return "%s (%s)" % (
                self.name,
                ", ".join(topping.name for topping in self.toppings.all()),
            )

然后运行::

    >>> Pizza.objects.all()
    ["Hawaiian (ham, pineapple)", "Seafood (prawns, smoked salmon)"...

现在的问题是每次调用 ``Pizza.__str__()`` 都会触发
``self.toppings.all()``, 它一定会查询数据库, 因此
``Pizza.objects.all()`` 将会为Pizza ``QuerySet`` 中的
**每个** 元素到Toppings表进行查询.

可以使用 ``prefetch_related`` 将其减少为两次查询:

    >>> Pizza.objects.all().prefetch_related('toppings')

这意味着每个 ``Pizza`` 都做了 ``self.toppings.all()``; 但是现在每次调用
``self.toppings.all()`` 时, 它不必去数据库查询内容, 它会在一个预取的 ``QuerySet`` 缓存中找到,这个缓存一次查询就会生成.

也就是说，所有关联的topping都会在一个查询中获取, 作为 ``QuerySets`` 关联结果的预填充缓存;
然后这些 ``QuerySets`` 用于 ``self.toppings.all()`` 调用.

``prefetch_related()`` 的附加查询是在
``QuerySet`` 开始计算且主查询已执行完毕后执行.

如果有一个可迭代的模型实例，则可以使用 :func:`~django.db.models.prefetch_related_objects`
函数在这些实例上预取相关属性.

注意，主要 ``QuerySet`` 的结果缓存和所有指定的相关对象将被完全加载到内存中.
这不同于 ``QuerySets`` 的典型行为, 通常会尽量避免在需要之前将所有对象加载到内存中, 即使在数据库中执行了查询后.

.. note::

    请记住，与 ``QuerySets`` 一样, 任何后续的连接方法隐含不同的数据库查询时将忽略之前缓存的结果,
    并使用新的数据库查询检索数据. 所以像下面的代码:

        >>> pizzas = Pizza.objects.prefetch_related('toppings')
        >>> [list(pizza.toppings.filter(spicy=True)) for pizza in pizzas]

    ...那么事实上预取的 ``pizza.toppings.all()`` 不会有任何帮助.
    ``prefetch_related('toppings')`` 隐含了
    ``pizza.toppings.all()``, 但是 ``pizza.toppings.filter()`` 是一个完全不同的新查询.
    预取的缓存对它毫无帮助; 实际上它会影响性能, 因为做了一个毫无用处的数据库查询. 所有使用这个功能的时候一定小心!

您可以使用普通join语法来执行相关字段的相关字段. 假设上面例子中还有一个额外的模型::

    class Restaurant(models.Model):
        pizzas = models.ManyToManyField(Pizza, related_name='restaurants')
        best_pizza = models.ForeignKey(Pizza, related_name='championed_by')

下面都是合法的:

    >>> Restaurant.objects.prefetch_related('pizzas__toppings')

这将预取所有属于restaurant的pizza,所有属于pizza的topping.
这将产生总共3个数据库查询 - 一个用于restaurant，一个用于pizza，一个用于topping.

    >>> Restaurant.objects.prefetch_related('best_pizza__toppings')

这将预取所有best pizza,每个restaurant的 best pizza的topping.
这将产生总共3个数据库查询 - 一个用于restaurant，一个用于best pizza，一个用于topping.

当然, ``best_pizza`` 关系可以使用
``select_related`` 来获取，这样可以将数据库查询减少为2:

    >>> Restaurant.objects.select_related('best_pizza').prefetch_related('best_pizza__toppings')

因为预取在主查询(那个包括 ``select_related`` 所需的join）之后执行,
因此它能够检测到 ``best_pizza`` 对象已经被提取, 并且它会跳过不在获取它们.

链式的 ``prefetch_related`` 调用将会累计预取的查询. 如果要清除 ``prefetch_related`` 的行为,
可以通过传入参数 ``None`` 的方式实现:

   >>> non_prefetched = qs.prefetch_related(None)

使用 ``prefetch_related`` 时需要注意的一点是，查询创建的对象可以在它们相关的不同对象之间共享，
即在返回的对象树中，单个Python模型实例可以出现在多个点上.
它通常会在与外键的关系中出现. 这种行为不是一个缺点，实际上会节省内存和CPU时间.

虽然 ``prefetch_related`` 支持预取 ``GenericForeignKey`` 关系,但查询的数量将取决于数据.
由于 ``GenericForeignKey`` 可以引用多个表中的数据, 因此需要对每个表引用一个查询,而不是对所有项进行查询.
如果相关的行还没有被获取，那么ContentType表上可能会有额外的查询.

``prefetch_related`` 在多数情况下会使用“IN”运算符的SQL查询来实现.
这意味着对于大型 ``QuerySet`` 可能生成一个较大的“IN”子句,
在解析或执行SQL查询时是否会有性能性能问题这取决于数据库.一定为您的用例配置文件!

请注意，如果您使用 ``iterator()`` 来运行查询, 则会忽略 ``prefetch_related()`` 调用,
因为这两个优化一起使用并没有意义.

:class:`~django.db.models.Prefetch` 对象可以进一步控制预取操作.

在其最简单的形式中, ``Prefetch`` 等效于传统的基于字符串的查找:

    >>> from django.db.models import Prefetch
    >>> Restaurant.objects.prefetch_related(Prefetch('pizzas__toppings'))

可以使用可选的 ``queryset`` 参数提供自定义查询集. 下面代码可以用于更改查询集的默认顺序:

    >>> Restaurant.objects.prefetch_related(
    ...     Prefetch('pizzas__toppings', queryset=Toppings.objects.order_by('name')))

或者在适当时调用 :meth:`~django.db.models.query.QuerySet.select_related()`
以减少查询数量:

    >>> Pizza.objects.prefetch_related(
    ...     Prefetch('restaurants', queryset=Restaurant.objects.select_related('best_pizza')))

还可以使用可选的 ``to_attr`` 参数将预取结果分配给自定义属性.结果将直接存储在列表中.

下面代码允许使用不同的 ``QuerySet`` 多次预取相同的关系;例如:

    >>> vegetarian_pizzas = Pizza.objects.filter(vegetarian=True)
    >>> Restaurant.objects.prefetch_related(
    ...     Prefetch('pizzas', to_attr='menu'),
    ...     Prefetch('pizzas', queryset=vegetarian_pizzas, to_attr='vegetarian_menu'))

使用自定义 ``to_attr`` 创建的查找仍然可以像同样一样被其他查找遍历:

    >>> vegetarian_pizzas = Pizza.objects.filter(vegetarian=True)
    >>> Restaurant.objects.prefetch_related(
    ...     Prefetch('pizzas', queryset=vegetarian_pizzas, to_attr='vegetarian_menu'),
    ...     'vegetarian_menu__toppings')

在过滤预取结果时, 建议使用 ``to_attr``, 因为它比在相关管理器的缓存中存储经过过滤的结果更模糊:

    >>> queryset = Pizza.objects.filter(vegetarian=True)
    >>>
    >>> # Recommended:
    >>> restaurants = Restaurant.objects.prefetch_related(
    ...     Prefetch('pizzas', queryset=queryset, to_attr='vegetarian_pizzas'))
    >>> vegetarian_pizzas = restaurants[0].vegetarian_pizzas
    >>>
    >>> # Not recommended:
    >>> restaurants = Restaurant.objects.prefetch_related(
    ...     Prefetch('pizzas', queryset=queryset))
    >>> vegetarian_pizzas = restaurants[0].pizzas.all()

自定义预取也适用于单个相关关系, 例如前向 ``ForeignKey`` 和 ``OneToOneField``.
一般是使用 :meth:`select_related()` 来获取这些关系,
但是在下面情况下使用自定义的 ``QuerySet`` 预取更加有用:

* 想要在相关模型上执行进一步预取的 ``QuerySet``.

* 希望仅预取相关对象的子集.

* 希望使用性能优化,比如
  :meth:`deferred fields <defer()>`:

    >>> queryset = Pizza.objects.only('name')
    >>>
    >>> restaurants = Restaurant.objects.prefetch_related(
    ...     Prefetch('best_pizza', queryset=queryset))

.. note::

    查找的顺序很重要.

    请看下面的例子:

       >>> prefetch_related('pizzas__toppings', 'pizzas')

    即使它无序也可以正常工作, 因为 ``'pizzas__toppings'`` 已经包含了所有需要的信息，
    第二个参数 ``'pizzas'`` 实际上是多余的.

        >>> prefetch_related('pizzas__toppings', Prefetch('pizzas', queryset=Pizza.objects.all()))

    这将引发 ``ValueError``, 因为它试图重新定义先前看到的查询的查询集.
    注意,创建隐式queryset是为了遍历 ``“pizzas”``,
    作为 ``“pizzas__toppings”`` 查找的一部分.

        >>> prefetch_related('pizza_list__toppings', Prefetch('pizzas', to_attr='pizza_list'))

    这将触发一个 ``AttributeError`` 错误, 因为在处理 ``pizza_list__toppings`` 时,
    ``pizza_list`` 还不存在.

    这些考虑不限于 ``Prefetch``. 一些高级特性可能要求以特定的顺序执行查找, 来避免创建额外的查询;
    因此, 建议总是慎重地安排 ``prefetch_related`` 参数.

``extra()``
~~~~~~~~~~~

.. method:: extra(select=None, where=None, params=None, tables=None, order_by=None, select_params=None)

有时候, Django查询语法不能很好地表达复杂的 ``WHERE`` 子句.
对于这些边缘情况, Django提供了 ``QuerySet`` ``extra()`` 修饰符——
一个将特定的子句注入 ``QuerySet`` 生成的SQL的钩子.

.. admonition:: 这是实在没有办法的情况下使用的方法

    这是一个旧的API, 我们的目标是在将来的某个时候弃用.
    仅当您无法使用其他查询方法表达您的查询时才使用它.
    如果您确实需要使用它，请 `file a ticket
    <https://code.djangoproject.com/newticket>`_ 您用例中的 `QuerySet.extra
    keyword <https://code.djangoproject.com/query?status=assigned&status=new&keywords=~QuerySet.extra>`_
    (请先检查现有ticket列表是否已存在), 以便我们可以增强QuerySet API，最终移除 ``extra()`` 方法.
    我们不再为这个方法改进或修复bug.

    例如这样是使用 ``extra()``::

        >>> qs.extra(
        ...     select={'val': "select col from sometable where othercol = %s"},
        ...     select_params=(someparam,),
        ... )

    相等于::

        >>> qs.annotate(val=RawSQL("select col from sometable where othercol = %s", (someparam,)))

    使用 :class:`~django.db.models.expressions.RawSQL` 的好处在于可以根据需要设置
    ``output_field``.  主要的缺点是, 如果您在原始SQL中引用了查询器的某些表别名,
    那么Django可能会更改该别名(例如，当查询集用作另一个查询中的子查询)时.

.. warning::

    无论何时都需要非常谨慎的使用 ``extra()``. 每次使用它时,
    都应该转义用户可以使用 ``params`` 控制的任何参数,
    以防止SQL注入攻击. 请详细了解 :ref:`SQL注入保护 <sql-injection-protection>`.

由于产品差异的原因,这些自定义的查询难以保障在不同的数据库之间兼容(因为你手写SQL代码的原因),
而且违背了DRY原则,所以如非必要,还是尽量避免写 ``extra``.


``extra`` 可以指定一个或多个 ``where``,``select``,``params`` 或者 ``tables``.
这些参数都不是必须的，但是至少要使用一个.

* ``select``

  ``select`` 参数可以让你在 ``SELECT`` 子句中添加其他字段信息, 它是一个字典,
  存放着属性名到SQL子句的映射.

  例如::

      Entry.objects.extra(select={'is_recent': "pub_date > '2006-01-01'"})

  这样结果集中每个 ``Entry`` 都带有一个额外属性 ``is_recent``,
  它是一个布尔值,表示 ``pub_date`` 是在 Jan. 1. 2006 之后.

  Django 会直接在 ``SELECT`` 中加入对应的SQL片段, 所以上面例子的SQL应该类似这样::

      SELECT blog_entry.*, (pub_date > '2006-01-01') AS is_recent
      FROM blog_entry;


  下面是一个高级的用法例子; 它会执行一个子查询, 为每个
  ``Blog`` 对象提供一个 ``entry_count`` 属性, 一个关联的 ``Entry`` 对象的个数::

      Blog.objects.extra(
          select={
              'entry_count': 'SELECT COUNT(*) FROM blog_entry WHERE blog_entry.blog_id = blog_blog.id'
          },
      )

  在这个特例中, 需要了解一个事实， 就是 ``blog_blog`` 表已经存在于 ``FROM`` 从句中.

  上面例子的执行SQL将是::

      SELECT blog_blog.*, (SELECT COUNT(*) FROM blog_entry WHERE blog_entry.blog_id = blog_blog.id) AS entry_count
      FROM blog_blog;

  要注意的是，大多数数据库需要在子句两端添加括号,
  而在 Django 的 ``select`` 子句中却无须这样.
  另请注意，某些数据库后台(如某些MySQL版本)不支持子查询.

  在少数情况下，您可能希望将参数传递到 ``extra(select=...)`` 中的SQL片段.
  为此,可以使用 ``select_params`` 参数. 由于 ``select_params`` 是一个序列.
  并且 ``select`` 属性是字典,因此需要注意使参数与额外的选择片段正确匹配.
  在这种情况下, 需要 :class:`collections.OrderedDict`  作为 ``select`` 值,
  而不仅仅是普通的Python字典.

  比如下面例子::

      Blog.objects.extra(
          select=OrderedDict([('a', '%s'), ('b', '%s')]),
          select_params=('one', 'two'))

  如果需要在select字符串中使用文本 ``%s``, 请使用 ``%%s``.

* ``where`` / ``tables``

  可以使用 ``WHERE`` 显式定义SQL ``where`` 子句.
  您可以通过 ``FROM`` 手动将表添加到SQL ``tables`` 子句.

  ``where`` 和 ``tables`` 都接受字符串列表.
  所有 ``where`` 参数均通过“AND”连接其他条件.

  例如::

      Entry.objects.extra(where=["foo='a' OR bar = 'a'", "baz = 'a'"])

  ...(大致)相当于成以下SQL::

      SELECT * FROM blog_entry WHERE (foo='a' OR bar='a') AND (baz='a')

  如果要指定已在查询中使用的表, 请在谨慎使用 ``tables`` 参数.
  通过 ``tables`` 参数添加额外的表时, Django会假定您希望该表包含额外的时间(如果已包括).
  这会产生一个问题, 因为表名将会被赋予一个别名. 如果表在SQL语句中多次出现,
  则第二次和后续出现必须使用别名,以便数据库可以区分它们.
  如果在 ``where`` 参数中指定了添加的额外表,这将导致错误.

  通常,您只需添加尚未显示在查询中的额外表. 然而,如果发生上述情况,则有几种解决方案.
  首先,看看你是否可以不包括额外的表,并使用已经在查询中的一个.
  如果不可能, 请将 ``extra()`` 调用放在查询集结构的前面, 以便您的表是该表的第一次使用.
  最后, 如果所有失败,请查看生成的查询并重写 ``where`` 添加以使用给您的额外表的别名.
  每次以相同的方式构造查询集时,别名将是相同的,因此您可以依靠别名不更改.

* ``order_by``

  如果需要使用通过 ``extra()`` 包含的新字段或表来对结果查询进行排序,
  请使用 ``extra()`` 的 ``order_by`` 参数并传入一个字符串序列.
  这些字符串应该是模型字段(和查询集的普通 :meth:`order_by()` 方法一样),
  形式为 ``extra()`` 带 ``table_name.column_name`` 或 ``select`` 中的别名参数.

  例如::

      q = Entry.objects.extra(select={'is_recent': "pub_date > '2006-01-01'"})
      q = q.extra(order_by = ['-is_recent'])

  这会将所有 ``is_recent`` 为true的项排到最前面(在降序排列中 ``True`` 位于 ``False`` 前面).

  顺便说一句, 你可以多次调用 ``extra()``,它会按照期望(每次添加新的约束)运行.

* ``params``

  上述 ``where`` 参数可以使用标准Python数据库字符串占位符 - ``'%s'`` 来指示数据库引擎应自动引用的参数.
  ``params`` 参数是要替换的任何额外参数的列表.

  例如::

      Entry.objects.extra(where=['headline=%s'], params=['Lennon'])

  一定要使用 ``params`` 而不是将直接值嵌入 ``where``,
  因为 ``params`` 会确保根据数据库后台正确引用值. 比如, 引号会被正确转义.

  错误用法::

      Entry.objects.extra(where=["headline='Lennon'"])

  正确用法::

      Entry.objects.extra(where=['headline=%s'], params=['Lennon'])

.. warning::

    如果您在MySQL上执行查询,请注意MySQL的静默类型强制可能会在混合类型时导致意外结果.
    如果查询字符串类型列,但使用整数值,MySQL将强制表中所有值的类型为整数,然后再执行比较.
    例如,如果表中包含值 ``“abc”`` 、 ``“def”``,并且查询 ``WHERE mycolumn=0`` ,
    那么这两行都将匹配上.为了防止这种情况,在使用查询中的值之前执行正确的类型转换.

``defer()``
~~~~~~~~~~~

.. method:: defer(*fields)

在一些复杂的数据建模情况下,模型中可能包含大量字段,
其中一些可能包含大量数据(例如文本字段),或者将它们转换为Python对象的处理比较耗时.
初次获取数据时不知道是否需要这些特定字段的情况下,
使用查询集的结果时,可以告诉Django不要从数据库中检索它们.

它通过传递字段名称到 ``defer()`` 实现不加载::

    Entry.objects.defer("headline", "body")

查询集中 ``deferred`` 字段仍会返回在模型实例中.
当访问该字段时才会从数据库中检索.(每次只检索一个, 而不是一次性检索所有 ``deferred`` 字段).

可以多次调用 ``defer()``. 每次调用都会添加新的字段的 ``deferred`` 集::

    # Defers both the body and headline fields.
    Entry.objects.defer("body").filter(rating=5).defer("headline")

字段添加到  ``deferred`` 集的顺序无关紧要.
对已经在 ``deferred`` 集中的字段再次调用 ``defer()`` 也没有影响
(该字段仍然是 ``deferred``).

可以 ``延迟`` 加载关联模型中的字段(如果关联模型是通过 :meth:`select_related()` 加载的),
方法是使用标准的双下划线表示法来分离关联字段::

    Blog.objects.select_related().defer("entry__headline", "entry__body")

如果要清除 ``deferred`` 字段, 调用 ``defer()`` 传入一个 ``None`` 参数即可 ::

    # Load all fields immediately.
    my_queryset.defer(None)

模型中有些字段即使设置了延迟加载也不会延迟, 比如永远不能延迟加载主键.
如果使用 :meth:`select_related()` 检索关联模型, 则不能延迟加载从主模型连接到关联模型的字段,
否则会抛出异常.

.. note::

    ``defer()`` 方法(及其表亲, 下文中的 :meth:`only()`)仅适用于高级用例.
    它们用于提供一种优化, 当你仔细分析查询并且完全了解需要什么信息,
    知道返回需要的字段与返回模型的全部字段之间的区别非常重要.

    即使你认为你是在这种情况下,  只有当你在查询集加载时不能确定是否需要额外的字段时使用 ``defer()``.
    如果你经常加载和使用特定的数据子集, 最好的选择是规范你的模型, 将不加载的数据放入单独的模型(或数据库表).
    如果列由于某种原因必须保留在一个表中, 请创建一个具有 ``Meta.managed = False``
    (请参阅 :attr:`managed attribute <django.db.models.Options.managed>` 文档)的模型,
    只包含你通常需要加载和使用的, 否则就调用 ``defer()`` 的字段.
    这可以使你的代码对读者更加清晰, 并且在Python进程中消耗更少的内存,加载稍微更快一些.

    例如，这两个模型使用相同的底层数据库表::

        class CommonlyUsedModel(models.Model):
            f1 = models.CharField(max_length=10)

            class Meta:
                managed = False
                db_table = 'app_largetable'

        class ManagedModel(models.Model):
            f1 = models.CharField(max_length=10)
            f2 = models.CharField(max_length=10)

            class Meta:
                db_table = 'app_largetable'

        # 两个查询等价:
        CommonlyUsedModel.objects.all()
        ManagedModel.objects.all().defer('f2')

    如果需要在非托管(`unmanaged`)模型中复制多个字段,
    最好使用共享字段创建一个抽象模型,
    然后让非托管模型和托管模型从抽象模型继承.

.. note::

    当对具有延迟(`deferred`)字段的实例调用 :meth:`~django.db.models.Model.save()` 时,
    仅保存加载的字段. 有关详细信息，请参见 :meth:`~django.db.models.Model.save()`.

``only()``
~~~~~~~~~~

.. method:: only(*fields)

``only()`` 方法或多或少与 :meth:`defer()` 相反. 你可以在检索模型时在 *不* 延迟加载的字段使用过它.
如果你的模型几乎所有的字段都需要延迟加载, 那么使用 ``only()`` 来指定加载字段会使代码变得简单.

假设你有一个包含 ``name``, ``age`` 和 ``biography`` 三个字段的模型. 就延迟加载来说,
下面两个querysets等价::

    Person.objects.defer("age", "biography")
    Person.objects.only("name")

只要调用 ``only()``,它就会立即替换要加载的字段集.
该方法的名称可以帮助记忆: **只有** 那些字段被立即加载;其余的延迟.
因此, 对 ``only()`` 的连续调用只会考虑最终字段::

    # 这会延迟加载除了headline的所有字段.
    Entry.objects.only("body", "rating").only("headline")

由于 ``defer()`` 以递增方式运行(向延迟列表中添加字段),
因此你可以组合 ``only()`` 和 ``defer()`` 调用,
使它们合乎逻辑地工作.::

    # 最终除了 "headline" 都被延迟加载
    Entry.objects.only("headline", "body").defer("body")

    # 最终结果立即加载标题和正文
    # (only()替换字段集中任何字段).
    Entry.objects.defer("body").only("headline", "body")

:meth:`defer` 文档注释中的所有注意事项也适用于 ``only()``. 请谨慎使用它,
只有在没有其它选择时才使用.

仅使用 :meth:`only` 时用 :meth:`select_related` 省略字段也会导致错误.

.. note::

    当对具有延迟(`deferred`)字段的实例调用 :meth:`~django.db.models.Model.save()` 时,
    仅保存加载的字段. 有关详细信息，请参见 :meth:`~django.db.models.Model.save()`.

``using()``
~~~~~~~~~~~

.. method:: using(alias)

如果使用多个数据库，该方法用于控制 ``QuerySet`` 在哪个数据库上求值.
该方法的唯一参数是数据库的别名，定义在 :setting:`DATABASES` 中.

例如::

    # 使用别名为 'default' 的数据库.
    >>> Entry.objects.all()

    # 使用别名为 'backup' 的数据库
    >>> Entry.objects.using('backup')

``select_for_update()``
~~~~~~~~~~~~~~~~~~~~~~~

.. method:: select_for_update(nowait=False)

返回一个锁住行直到事务结束的查询集,
如果数据库支持，它将生成一个 ``SELECT ... FOR UPDATE`` 语句.

例如::

    entries = Entry.objects.select_for_update().filter(author=request.user)

所有匹配的行将被锁定,直到事务结束.这样可以通过锁防止数据被其它事务修改.

通常,如果所选的行已经被另一个事务锁住,那么查询将被阻塞,直到释放锁为止.
如果你不希望这样,那么可以调用 ``select_for_update(nowait=True)``.
这将使调用非阻塞方式.如果冲突锁已经被另一个事务获取, 则在计算queryset时将引发 :exc:`~django.db.DatabaseError`.

目前, ``postgresql`` 、 ``oracle`` 和 ``mysql`` 数据库后端都支持
``select_for_update()``. 但是, MySQL 不支持 ``nowait`` 参数.
所以, 使用其他第三方数据库后端的用户应该查看下他们的文档以了解这一细节.

使用像MySQL这种不支持 ``nowait`` 参数的数据库, 在调用 ``select_for_update()`` 时传入 ``nowait=True``
会导致 :exc:`~django.db.DatabaseError` . 这是为了防止代码被意外阻塞.

如果在自动提交模式下在支持 ``SELECT ... FOR UPDATE`` 的数据库后端使用  ``select_for_update()`` 会导致
:exc:`~django.db.transaction.TransactionManagementError` 错误.
因为这种情况下行不会被锁定. 如果允许这种调用可能会造成数据损坏, 而且这也很有可能在事务外被调用.

``select_for_update()`` 使用在不支持
``SELECT ... FOR UPDATE`` 的数据库后端(比如 SQLite) 将没有效果.
``SELECT ... FOR UPDATE`` 不会被添加到查询中, 并且在自动给提交模式下使用
``select_for_update()`` 也不会报错.

.. warning::

    虽然 ``select_for_update()`` 在自动提交模式下通常会失败, 因为
    :class:`~django.test.TestCase` 会自动将每个测试包装在一个事务中,
    即使在 :func:`~django.db.transaction.atomic()` 块之外调用 ``TestCase`` 的
    ``select_for_update()`` 也会在(可能会有意外)不引发 ``TransactionManagementError``
    的情况下通过. 但是要正确地测试 ``select_for_update()``, 请务必使用
    :class:`~django.test.TransactionTestCase`.

``raw()``
~~~~~~~~~

.. method:: raw(raw_query, params=None, translations=None)

提供原始SQL查询, 执行并返回一个
``django.db.models.query.RawQuerySet`` 实例.
这个 ``RawQuerySet`` 实例可以迭代获取实例对象，就像普通的 ``QuerySet`` 实例一样.

更多信息参见 :doc:`/topics/db/sql` .

.. warning::

  ``raw()`` 永远都是触发一个新的查询,和之前的filter无关.
  因此通常应该是从``Manager`` 或者新的``QuerySet`` 实例调用.

不返回 ``QuerySet`` 的方法
-----------------------------------------

下面的 ``QuerySet`` 方法计算 ``QuerySet`` 但返回的
*不是* ``QuerySet``.

这些方法不会使用 (see :ref:`caching-and-querysets`). 当然,
这些方法每次被调用都会查询数据库.

``get()``
~~~~~~~~~

.. method:: get(**kwargs)

返回根据查询参数匹配到的对象, 参数格式应该符合 `Field 查询`_ 要求.

如果 ``get()`` 匹配到多个对象将会抛出 :exc:`~django.core.exceptions.MultipleObjectsReturned`.
:exc:`~django.core.exceptions.MultipleObjectsReturned` 异常是模型类的属性.

如果 ``get()`` 根据查询参数没有匹配到对象将会抛出 :exc:`~django.db.models.Model.DoesNotExist` 异常.
这个异常也是模型类的属性.

例子::

    Entry.objects.get(id='foo') # raises Entry.DoesNotExist

:exc:`~django.db.models.Model.DoesNotExist` 异常继承自
:exc:`django.core.exceptions.ObjectDoesNotExist`, 因此可以同时捕获多个so you can target multiple
:exc:`~django.db.models.Model.DoesNotExist` 异常. 例如::

    from django.core.exceptions import ObjectDoesNotExist
    try:
        e = Entry.objects.get(id=3)
        b = Blog.objects.get(id=1)
    except ObjectDoesNotExist:
        print("Either the entry or blog doesn't exist.")

如果希望queryset返回一行, 则可以使用 ``get()`` 而不使用任何参数来返回该行对象::

    entry = Entry.objects.filter(...).exclude(...).get()

``create()``
~~~~~~~~~~~~

.. method:: create(**kwargs)

一个一次性创建对象并保存的快捷方法.  例如::

    p = Person.objects.create(first_name="Bruce", last_name="Springsteen")

和::

    p = Person(first_name="Bruce", last_name="Springsteen")
    p.save(force_insert=True)

是一样的.

参数 :ref:`force_insert <ref-models-force-insert>` 在其他的文档中有介绍,
它意味着一个新的对象一定会被创建. 正常情况下,你不必担心这点. 然而,
如果你的model中有一个手动设置主键,并且这个值已经存在于数据库中,
调用 ``create()`` 将会失败并且触发 :exc:`~django.db.IntegrityError`,
因为主键必须是唯一的. 如果你手动设置了主键,做好异常处理的准备.

``get_or_create()``
~~~~~~~~~~~~~~~~~~~

.. method:: get_or_create(defaults=None, **kwargs)

一个通过给定 ``kwargs`` 参数(如果模型中所有字段都有默认值,则可以为空)查询对象的快捷方法, 没有的话则创建一个.

返回一个元组 ``(object, created)``, 元组中的 ``object`` 是查到的或者被创建的对象,
``created`` 则是表示是否是新创建的布尔值.

这主要用作样板代码的一种快捷方式. 像这样::

    try:
        obj = Person.objects.get(first_name='John', last_name='Lennon')
    except Person.DoesNotExist:
        obj = Person(first_name='John', last_name='Lennon', birthday=date(1940, 10, 9))
        obj.save()

如果模型的字段数量较多的话,这种模式就不好用了.上面的例子可以用
``get_or_create()`` 重写::

    obj, created = Person.objects.get_or_create(
        first_name='John',
        last_name='Lennon',
        defaults={'birthday': date(1940, 10, 9)},
    )

所有传入 ``get_or_create()`` 的参数 — *除了* 一个可选参数
``defaults`` 都会用于 :meth:`get()` 调用. 如果能查到对象,
``get_or_create()`` 将返回包含这个对象和 ``False`` 的元组.
如果查到了多个对象, ``get_or_create`` 将引发
:exc:`~django.core.exceptions.MultipleObjectsReturned` 异常.
如果 *没有* 查到对象, ``get_or_create()`` 将实例化一个新对象并保存,
返回包含这个新对象和 ``True`` 的元组. 新对象按如下逻辑创建::

    params = {k: v for k, v in kwargs.items() if '__' not in k}
    params.update(defaults)
    obj = self.model(**params)
    obj.save()

用文字描述就是, 以任何不包含双下划线的非 ``'defaults'`` 关键字参数开始(双下划线表示非准确的查找).
然后添加 ``defaults`` 的内容，必要时会覆盖原键值, 将结果用作模型类的关键字参数.
如上所述，这是对所使用算法的简单描述, 但是它包含了所有相关的细节.
只是其内部实现有更多的错误检查,和处理一些额外的边缘条件;
如果您感兴趣，请阅读代码.

如果你有一个名为 ``defaults`` 的字段,想用 ``get_or_create()`` 对其精准查找,
可以使用 ``'defaults__exact'``, 像这样::

    Foo.objects.get_or_create(defaults__exact='bar', defaults={'defaults': 'baz'})

``get_or_create()`` 方法和 :meth:`create()` 有相同的错误行为,
如果你手动指定了主键, 并且查询的对象需要创建且数据库中主键已经重复,
则会导致 :exc:`~django.db.IntegrityError` 异常.

这种方法是原子性的, 假设正确使用底层数据库、数据库配置没有问题和其他行为都正确.
但是, 如果在调用 ``get_or_create`` 中使用的 ``kwargs`` 在数据库级别上没有强制唯一性
(请参阅 :attr:`~django.db.models.Field.unique` 和
:attr:`~django.db.models.Options.unique_together` ),
那么该方法很容易出现竞争条件, 导致同时插入具有相同参数的多行.

如果你使用的是Mysql数据库, 请使用 ``READ COMMITTED`` 隔离级别而不是 ``REPEATABLE READ`` (默认的),
否则可能会出现 ``get_or_create`` 抛出 :exc:`~django.db.IntegrityError` 异常, 但使用
:meth:`~django.db.models.query.QuerySet.get` 调用却没有对象.

最后在讲一点,在Django视图中使用 ``get_or_create()`` 时.
请一定只在 ``POST`` 请求中使用, 除非你有很充分的理由.
``GET`` 请求不应该去修改数据. 而 ``POST`` 则用于修改数据.有关信息请参考HTTP规范中的
:rfc:`Safe methods <7231#section-4.2.1>`.

.. warning::

  你可以通过 :class:`~django.db.models.ManyToManyField` 属性和反向关系来使用 ``get_or_create()``.
  但在这种情况下, 需要在该关系的上下文中限制查询. 如果不经常使用它，可能会导致一些完整性问题.

  根据下面的模型::

      class Chapter(models.Model):
          title = models.CharField(max_length=255, unique=True)

      class Book(models.Model):
          title = models.CharField(max_length=256)
          chapters = models.ManyToManyField(Chapter)

  您可以通过Book 的 chapter字段使用 ``get_or_create()``,
  但是它只会获取该Book 内部的上下文::

      >>> book = Book.objects.create(title="Ulysses")
      >>> book.chapters.get_or_create(title="Telemachus")
      (<Chapter: Telemachus>, True)
      >>> book.chapters.get_or_create(title="Telemachus")
      (<Chapter: Telemachus>, False)
      >>> Chapter.objects.create(title="Chapter 1")
      <Chapter: Chapter 1>
      >>> book.chapters.get_or_create(title="Chapter 1")
      # Raises IntegrityError

  发生这个错误是因为它尝试通过Book “Ulysses” 获取或者创建“Chapter 1”,
  但它是不可以的: 关联关系不能获取这个chapter, 因为它与这个book不关联,
  但因为 ``title`` 字段是唯一的,所以它也不能创建.

``update_or_create()``
~~~~~~~~~~~~~~~~~~~~~~

.. method:: update_or_create(defaults=None, **kwargs)

一个通过给定 ``kwargs`` 参数来更新对象的快捷方法, ``defaults`` 是由(键/值)组成的字典,
用于查找要更新对象, 如果对象不存在则新建一个。

返回一个元组 ``(object, created)``, 其中 ``object`` 是被创建或更新的对象, ``created`` 是一个
布尔值, 表示是否是新创建的对象.


``update_or_create`` 方式尝试根据传入的 ``kwargs`` 参数到数据库中获取对象,
如果成功匹配则会根据 ``defaults`` 字典中的数据更新字段.

这是用于某种情况的快捷方式. 例如::

    defaults = {'first_name': 'Bob'}
    try:
        obj = Person.objects.get(first_name='John', last_name='Lennon')
        for key, value in defaults.items():
            setattr(obj, key, value)
        obj.save()
    except Person.DoesNotExist:
        new_values = {'first_name': 'John', 'last_name': 'Lennon'}
        new_values.update(defaults)
        obj = Person(**new_values)
        obj.save()

但是随着模型中字段数量的增加，这种模式将会变得越来越笨拙.
上面的例子可以通过 ``update_or_create()`` 来重写, 如下::

    obj, created = Person.objects.update_or_create(
        first_name='John', last_name='Lennon',
        defaults={'first_name': 'Bob'},
    )

关于传入的 ``kwargs`` 中的名称是如何被解析, 请参考 :meth:`get_or_create`.

和上文 :meth:`get_or_create` 描述的一样, 该方法容易导致竞争条件,
如果在数据库级别没有设置强制唯一性, 则会导致同时插入多个行.

``bulk_create()``
~~~~~~~~~~~~~~~~~

.. method:: bulk_create(objs, batch_size=None)

该方法提供了一个批量操作用于同时向数据库中插入一组对象(无论有多少对象, 通常只做一次查询)::

    >>> Entry.objects.bulk_create([
    ...     Entry(headline='This is a test'),
    ...     Entry(headline='This is only a test'),
    ... ])

不过, 有几点需要注意:

* 模型的 ``save()`` 方法不会被调用, 并且 ``pre_save`` 和
  ``post_save`` 信号也不会被发送.
* 它不适用于多表继承场景中的子模型.
* 如果模型的主键是 :class:`~django.db.models.AutoField` , 那么它不会像 ``save()`` 方法那样自动检索并设置主键, 除非使用的数据库后端支持(当前只有 PostgreSQL).
* 它不适用于多对多关系.

.. versionchanged:: 1.9

    新增 ``bulk_create()`` 对代理模型(proxy models)支持.

.. versionchanged:: 1.10

    新增PostgreSQL数据库使用 ``bulk_create()`` 插入数据时设置主键.

参数 ``batch_size`` 用于控制单次创建的对象数. 默认情况下, 除SQLite外一次性创建所有对象, SQLite单次最多支持999个.

``count()``
~~~~~~~~~~~

.. method:: count()

返回数据查询结果集 ``QuerySet`` 中的对象数量. ``count()`` 不会抛出异常.

例子::

    # 返回数据库中所有条目数量.
    Entry.objects.count()

    # 返回数据库中headline包含'Lennon'的条目数量
    Entry.objects.filter(headline__contains='Lennon').count()

``count()`` 在后台执行的是 ``SELECT COUNT(*)`` , 因此请务必使用 ``count()`` 而不是将所有记录加载成Python对象然后调用 ``len()`` 函数
(除非你有其他需要必须要将其加载到内存中, 这种情况 ``len()`` 会比较快).

根据使用的数据库不同 (e.g. PostgreSQL vs. MySQL),
``count()`` 方法可能返回的是一个长整型而不是Python整数.

注意,如果你想计算 ``QuerySet`` 中的项目个数并且希望遍历每个数据对象(例如, 通过迭代它), 使用 ``len(queryset)`` 将会更好, 它不会像 ``count()`` 产生额外的数据库查询.

``in_bulk()``
~~~~~~~~~~~~~

.. method:: in_bulk(id_list=None)

接收一个主键组成的列表, 返回一个字典, 字典的key为传入的主键ID, value为匹配的对象实例. 如果不传入id_list, 那么将返回整个查询集的内容.

例子::

    >>> Blog.objects.in_bulk([1])
    {1: <Blog: Beatles Blog>}
    >>> Blog.objects.in_bulk([1, 2])
    {1: <Blog: Beatles Blog>, 2: <Blog: Cheddar Talk>}
    >>> Blog.objects.in_bulk([])
    {}
    >>> Blog.objects.in_bulk()
    {1: <Blog: Beatles Blog>, 2: <Blog: Cheddar Talk>, 3: <Blog: Django Weblog>}

如果传入空列表到 ``in_bulk()`` , 将返回一个空字典.

.. versionchanged:: 1.10

    在此之前版本, ``id_list`` 为必传参数.

``iterator()``
~~~~~~~~~~~~~~

.. method:: iterator()

计算 ``QuerySet`` (执行数据库查询) 返回一个迭代器
(see :pep:`234`). 通常 ``QuerySet`` 会在其内部缓存结果来防止重复查询. 相反 ``iterator()`` 会直接读取结果不会在
``QuerySet`` 级别执行缓存操作. 对于返回大量对象且只查询一次的 ``QuerySet``, 这可以带来更好的性能并显着降低内存.

注意对已经求值过的 ``QuerySet`` 调用 ``iterator()`` 会使其强制再计算, 导致重复查询.

另外, ``iterator()`` 会导致已调用的 ``prefetch_related()`` 方法被忽略.

.. warning::

    一些Python的数据库驱动比如 ``psycopg2`` 使用客户端游标执行缓存(实例化 ``connection.cursor()`` 配合
    Django's ORM使用). 使用 ``iterator()`` 不会影响到数据库层级的缓存. 如果要禁用此缓存, 请查看 `服务端游标`_.

.. _服务端游标: http://initd.org/psycopg/docs/usage.html#server-side-cursors

``latest()``
~~~~~~~~~~~~

.. method:: latest(field_name=None)

接收一个 ``field_name`` 日期字段, 返回表中最新的一个对象.

下面例子返回表中 ``pub_date`` 最近的一个 ``Entry`` ::

    Entry.objects.latest('pub_date')

如果在模型的 :ref:`Meta <meta-options>` 中指定了
:attr:`~django.db.models.Options.get_latest_by`, 则可以调用 ``earliest()`` 和 ``latest()`` 不传入 ``field_name`` .
Django 将默认使用 :attr:`~django.db.models.Options.get_latest_by` 指定的字段.

和 :meth:`get()` 方法一样, 如果查不到有效对象 ``earliest()`` 和 ``latest()`` 也会抛出
:exc:`~django.db.models.Model.DoesNotExist` 异常.

注意, ``earliest()`` 和 ``latest()`` 的存仅是为了方便和可读性.

.. admonition:: ``earliest()`` 和 ``latest()`` 可以返回日期为null的实例.

    因为排序是在数据库中执行的, 如果使用了不同的数据库返回的null值顺序可能会不同.
    比如PostgreSQL和MySQL中的null值的顺序高于非null值,而在SQLite中则相反.

    可以像这样过滤掉非空值::

        Entry.objects.filter(pub_date__isnull=False).latest('pub_date')

``earliest()``
~~~~~~~~~~~~~~

.. method:: earliest(field_name=None)

和 :meth:`~django.db.models.query.QuerySet.latest` 一样, 除了方向相反.

``first()``
~~~~~~~~~~~

.. method:: first()

返回结果集中的第一个对象, 如果没有查询到内容则返回 ``None``.
如果 ``QuerySet`` 没有设置排序, 则默认按照主键排序.

例子::

    p = Article.objects.order_by('title', 'pub_date').first()

注意 ``first()`` 是提供的一个快捷方法, 它的功能和下面例子作用一样::

    try:
        p = Article.objects.order_by('title', 'pub_date')[0]
    except IndexError:
        p = None

``last()``
~~~~~~~~~~

.. method:: last()

功能类似  :meth:`first()`, 只是它返回结果集的最后一个对象.

``aggregate()``
~~~~~~~~~~~~~~~

.. method:: aggregate(*args, **kwargs)

从 ``QuerySet`` 中计算聚合值 (均值, 求和等)并以字典形式返回. ``aggregate()`` 中一个参数对应字典的一组值.

Django提供的所有聚合函数在下文的
`聚合函数`_ 中查看. 因为聚合也是 :doc:`查询表达式 </ref/models/expressions>`, 因此可以结合多个聚合创建复杂的聚合.

聚合时使用了关键字参数将以关键字为名称返回. 如果是匿名参数将以聚合函数的名称和聚合字段的名称组合为名字返回.
复杂聚合不支持匿名参数必须指定关键字参数.

例如, 在博客的例子中查询作者名下的博文数量::

    >>> from django.db.models import Count
    >>> q = Blog.objects.aggregate(Count('entry'))
    {'entry__count': 16}

如果使用关键字参数就可以指定聚合返回的值::

    >>> q = Blog.objects.aggregate(number_of_entries=Count('entry'))
    {'number_of_entries': 16}

更加详细的介绍请查看 :doc:`聚合 </topics/db/aggregation>`.

``exists()``
~~~~~~~~~~~~

.. method:: exists()

如果 :class:`.QuerySet` 包含数据则返回 ``True``, 否则返回 ``False``.
该方式使用最简单也最快的方式完成查询, 且它执行的 *查询* 和一般的
:class:`.QuerySet` 查询几乎相同.

:meth:`~.QuerySet.exists` 对搜索 :class:`.QuerySet` 以及其关联对象是否存在相当有用, 特别是对于体量比较大的 :class:`.QuerySet`.

查找一个具有唯一字段(e.g. ``primary_key``)模型的 :class:`.QuerySet` 中是否具有指定成员的最高效方式::

    entry = Entry.objects.get(pk=123)
    if some_queryset.filter(pk=entry.pk).exists():
        print("Entry contained in queryset")

它会比下面这种求值再遍历的方式快很多::

    if entry in some_queryset:
       print("Entry contained in QuerySet")

查询queryset是否有值::

    if some_queryset.exists():
        print("There is at least one object in some_queryset")

将快于::

    if some_queryset:
        print("There is at least one object in some_queryset")

... 但效果不是很明显 (因此在很大的查询集中才需要这样做来提高效率).

另外, 如果 ``some_queryset`` 还没有被求值, 但你知道它将来会被求值,
那么使用 ``some_queryset.exists()`` 会比直接使用 ``bool(some_queryset)`` 做多余的工作. 后者会求值并检查是否有结果.

``update()``
~~~~~~~~~~~~

.. method:: update(**kwargs)

对指定字段执行更新语句返回受影响的行数(如果某些行已具备新值, 则可能不等于更新的行数).

例如, 对2010年发布的博客启用评论::

    >>> Entry.objects.filter(pub_date__year=2010).update(comments_on=False)

(假设 ``Entry`` 模型具有 ``pub_date`` 和 ``comments_on`` 字段.)

update没有数量限制可以同时更新多个字段.
例如, 同时更新 ``comments_on`` 和 ``headline`` 字段::

    >>> Entry.objects.filter(pub_date__year=2010).update(comments_on=False, headline='This is old')

``update()`` 方法是立即执行的, :class:`.QuerySet` update的唯一限制是它只可以更新模型主表中的字段, 不可以更新关联模型.
例如下面这种::

    >>> Entry.objects.update(blog__name='foo') # Won't work!

可以通过关联模型进行过滤, 例如::

    >>> Entry.objects.filter(blog__id=1).update(comments_on=True)

无法对已切片的或者无法进行过滤的 :class:`.QuerySet` 调用 ``update()`` 方法.

``update()`` 会返回受影响的行数::

    >>> Entry.objects.filter(id=64).update(comments_on=True)
    1

    >>> Entry.objects.filter(slug='nonexistent-slug').update(comments_on=True)
    0

    >>> Entry.objects.filter(pub_date__year=2010).update(comments_on=False)
    132

如果你仅仅是想更新记录而不需要做其他操作, 那么调用 ``update()`` 是最高效的方法, 而不是将数据加载到内存, 例如下面这种是不建议的::

    e = Entry.objects.get(id=10)
    e.comments_on = False
    e.save()

...正确做法::

    Entry.objects.filter(id=10).update(comments_on=False)

使用 ``update()`` 还可以防止在加载对象和调用 ``save()`` 这时间段内数据库某些内容发生更改导致的竞争条件.

最后, 需要知道 ``update()`` 是在SQL级执行更新, 因此它不会调用模型的 ``save()`` 方法, 也不会触发
:attr:`~django.db.models.signals.pre_save` 和
:attr:`~django.db.models.signals.post_save` 信号. 如果业务需要调用模型自己的
:meth:`~django.db.models.Model.save()` 方法, 那么请遍历结果集调用
:meth:`~django.db.models.Model.save()`, 例如::

    for e in Entry.objects.filter(pub_date__year=2010):
        e.comments_on = False
        e.save()

``delete()``
~~~~~~~~~~~~

.. method:: delete()

执行SQL删除语句, 删除 :class:`.QuerySet` 所有行, 返回删除数量和每个删除的对象与其数量组成的字典.

``delete()`` 是立即生效的. 不能对已切片和不能过滤的 :class:`.QuerySet` 调用t ``delete()`` 方法.

例如, 删除指定博客的所有条目::

    >>> b = Blog.objects.get(pk=1)

    # Delete all the entries belonging to this Blog.
    >>> Entry.objects.filter(blog=b).delete()
    (4, {'weblog.Entry': 2, 'weblog.Entry_authors': 2})

.. versionchanged:: 1.9

    新增返回删除对象及数量情况.

Django的 :class:`~django.db.models.ForeignKey` 仿效了SQL的 ``ON DELETE CASCADE`` 约束, — 换句话讲, 默认情况下任何对象别删除时与其关联的外键对象也会被删除.
例如::

    >>> blogs = Blog.objects.all()

    # This will delete all Blogs and all of their Entry objects.
    >>> blogs.delete()
    (5, {'weblog.Blog': 1, 'weblog.Entry': 2, 'weblog.Entry_authors': 2})

这个行为可以通过 :class:`~django.db.models.ForeignKey` 的
:attr:`~django.db.models.ForeignKey.on_delete` 参数进行设置.

``delete()`` 方法是批量删除, 它不会调用模型的 ``delete()`` 方法, 但是会为每一个删除的对象触发
:data:`~django.db.models.signals.pre_delete` 和
:data:`~django.db.models.signals.post_delete` 信号
(包含级联删除).

Django需要将对象加载到内存才可以处理级联和发送信号.
但是, 如果没有级联处理和信号操作, Django会采取快速方式删除对象而不需要加载到内存.
对于大型的查询这可以减少内存消耗和执行查询的量.

ForeignKeys :attr:`~django.db.models.ForeignKey.on_delete` 设置为
``DO_NOTHING`` 时不会阻止快速删除.

注意，在对象删除中生成的查询是具体实现, 可能会有更改.

``as_manager()``
~~~~~~~~~~~~~~~~

.. classmethod:: as_manager()

类方法, 返回一个带有 ``QuerySet`` 方法的 :class:`~django.db.models.Manager` 实例. 有关详细信息请参考
:ref:`create-manager-with-queryset-methods`.

.. _field-lookups:

``Field`` 查询
-----------------

Field查询就是SQL中的 ``WHERE`` 语句. 它们以 ``QuerySet`` 的 :meth:`filter()`
:meth:`exclude()`  :meth:`get()` 方法的关键字参数实现.

详细介绍参见 :ref:`模型和数据库查询文档
<field-lookups-intro>`.

Django内置查询如下. 同时也支持为模型字段
:doc:`自定义查询 </howto/custom-lookups>`.

为了方便起见, 当没有提供查询类型时(例如
``Entry.objects.get(id=14)``), 查询类型会被假定为 :lookup:`exact`.

.. fieldlookup:: exact

``exact``
~~~~~~~~~

精确匹配. 如果提供的查询值为 ``None`` 将会被解释为SQL的 ``NULL`` (详见 :lookup:`isnull`).

例如::

    Entry.objects.get(id__exact=14)
    Entry.objects.get(id__exact=None)

等价于SQL::

    SELECT ... WHERE id = 14;
    SELECT ... WHERE id IS NULL;

.. admonition:: MySQL查询

    在Mysql中, 数据表的 "COLLATE" 设置项会影响到
    ``exact`` 查询时大小写敏感. 这属于数据库的设置, *不是* Django的设置.
    这会影响到查询时的大小写敏感, 但也有解决方案. 参考 :doc:`数据库 </ref/databases>` 文档中的 :ref:`collation section <mysql-collation>`.

.. fieldlookup:: iexact

``iexact``
~~~~~~~~~~

大小写不敏感的精确匹配. 如果提供的查询值为 ``None`` 将会被解释为SQL的 ``NULL`` (详见 :lookup:`isnull`).

例如::

    Blog.objects.get(name__iexact='beatles blog')
    Blog.objects.get(name__iexact=None)

等价于SQL::

    SELECT ... WHERE name ILIKE 'beatles blog';
    SELECT ... WHERE name IS NULL;

注意第一个查询可以匹配到 ``'Beatles Blog'``, ``'beatles blog'``,
``'BeAtLes BLoG'`` 等等.

.. admonition:: SQLite用户

    当使用SQLite数据库和Unicode(非ASCII)字符时, 请注意 :ref:`数据库备注 <sqlite-string-matching>` 中的字符串比较. SQLite在匹配Unicode字符时大小写不敏感.

.. fieldlookup:: contains

``contains``
~~~~~~~~~~~~

大小写敏感的包含查询.

例如::

    Entry.objects.get(headline__contains='Lennon')

等价于SQL::

    SELECT ... WHERE headline LIKE '%Lennon%';

注意这可以匹配 ``'Lennon honored today'`` 但匹配不到 ``'lennon
honored today'``.

.. admonition:: SQLite用户

    SQLite 不支持大小写敏感的 ``LIKE`` 语句; ``contains``
    在SQLite中作用和 ``icontains`` 一样. 详见 :ref:`数据库备注
    <sqlite-string-matching>` .


.. fieldlookup:: icontains

``icontains``
~~~~~~~~~~~~~

大小写不敏感的包含查询.

例如::

    Entry.objects.get(headline__icontains='Lennon')

等价于SQL::

    SELECT ... WHERE headline ILIKE '%Lennon%';

.. admonition:: SQLite用户

    当使用SQLite数据库和Unicode(非ASCII)字符时, 请注意 :ref:`数据库备注 <sqlite-string-matching>` 中的字符串比较.

.. fieldlookup:: in

``in``
~~~~~~

存在于给定列表中.

例如::

    Entry.objects.filter(id__in=[1, 3, 4])

等价于SQL::

    SELECT ... WHERE id IN (1, 3, 4);

这里并不是一定要传入一个具有明确值的列表, 也可以嵌套传入一个查询集来查询::

    inner_qs = Blog.objects.filter(name__contains='Cheddar')
    entries = Entry.objects.filter(blog__in=inner_qs)

该查询将被视为一个子查询::

    SELECT ... WHERE blog.id IN (SELECT id FROM ... WHERE NAME LIKE '%Cheddar%')

如果传入的 ``QuerySet`` 调用了 ``values()`` 或者 ``values_list()`` 作为 ``__in`` 查询的参数, 那么你需要确保只提取了一个字段
例如, 下面这种做法是正确的::

    inner_qs = Blog.objects.filter(name__contains='Ch').values('name')
    entries = Entry.objects.filter(blog__name__in=inner_qs)

下面这种会抛出一个异常, 因为 ``__in`` 查询只需要一个字段确提取了两个::

    # Bad code! Will raise a TypeError.
    inner_qs = Blog.objects.filter(name__contains='Ch').values('name', 'id')
    entries = Entry.objects.filter(blog__name__in=inner_qs)

.. _nested-queries-performance:

.. admonition:: 性能考量

    请谨慎使用嵌套查询除非你非常了解数据库服务性能(如果不是,请做好基准测试!). 有些数据库, 尤其是MySQL并不能很好的优化嵌套查询.
    这种情况下, 先提取一组列表值, 然后再将其传递到第二个查询中会更有效. 也就是说分两次查询而不是一次查询::

        values = Blog.objects.filter(
                name__contains='Cheddar').values_list('pk', flat=True)
        entries = Entry.objects.filter(blog__in=list(values))

    注意第一个查询中Blog ``QuerySet`` 调用 ``list()`` 会强制执行查询.
    如果不调用它一样会导致嵌套查询, 因为
    :ref:`querysets-are-lazy`.

.. fieldlookup:: gt

``gt``
~~~~~~

大于查询.

例如::

    Entry.objects.filter(id__gt=4)

等价于SQL::

    SELECT ... WHERE id > 4;

.. fieldlookup:: gte

``gte``
~~~~~~~

大于等于查询.

.. fieldlookup:: lt

``lt``
~~~~~~

小于查询.

.. fieldlookup:: lte

``lte``
~~~~~~~

小于等于查询.

.. fieldlookup:: startswith

``startswith``
~~~~~~~~~~~~~~

大小写敏感的前缀查询.

例如::

    Entry.objects.filter(headline__startswith='Will')

等价于SQL::

    SELECT ... WHERE headline LIKE 'Will%';

SQLite不支持大小写敏感的 ``LIKE`` 语句; SQLite下 ``startswith`` 被作为
``istartswith`` 执行.

.. fieldlookup:: istartswith

``istartswith``
~~~~~~~~~~~~~~~

大小写不敏感的前缀查询.

例如::

    Entry.objects.filter(headline__istartswith='will')

等价于SQL::

    SELECT ... WHERE headline ILIKE 'Will%';

.. admonition:: SQLite用户

    当使用SQLite数据库和Unicode(非ASCII)字符时, 请注意 :ref:`数据库备注 <sqlite-string-matching>` 中的字符串比较.

.. fieldlookup:: endswith

``endswith``
~~~~~~~~~~~~

大小写敏感的后缀查询.

例如::

    Entry.objects.filter(headline__endswith='cats')

等价于SQL::

    SELECT ... WHERE headline LIKE '%cats';

.. admonition:: SQLite用户

    SQLite不支持大小写敏感的 ``LIKE`` 语句; SQLite下 ``endswith`` 被作为 ``iendswith`` 执行.
    请注意 :ref:`数据库备注 <sqlite-string-matching>` 中的字符串比较.

.. fieldlookup:: iendswith

``iendswith``
~~~~~~~~~~~~~

大小写不敏感的后缀查询.

例如::

    Entry.objects.filter(headline__iendswith='will')

等价于SQL::

    SELECT ... WHERE headline ILIKE '%will'

.. admonition:: SQLite用户

    当使用SQLite数据库和Unicode(非ASCII)字符时, 请注意 :ref:`数据库备注 <sqlite-string-matching>` 中的字符串比较.

.. fieldlookup:: range

``range``
~~~~~~~~~

范围查询(包含边界值).

例如::

    import datetime
    start_date = datetime.date(2005, 1, 1)
    end_date = datetime.date(2005, 3, 31)
    Entry.objects.filter(pub_date__range=(start_date, end_date))

等价于SQL::

    SELECT ... WHERE pub_date BETWEEN '2005-01-01' and '2005-03-31';

SQL中支持 ``BETWEEN`` 的地方都支持 ``range``  — 比如日期,数字,甚至字符串.

.. warning::

    用日期过滤 ``DateTimeField`` 不会包含最后一天的数据, 因为查询范围的边界值会被解释成"给定日期的零点".
    比如如果 ``pub_date`` 的类型为 ``DateTimeField``, 那么上面的查询就会变成这样的SQL::

        SELECT ... WHERE pub_date BETWEEN '2005-01-01 00:00:00' and '2005-03-31 00:00:00';

    一般来讲, 不要把date和datetime混在一起使用.

.. fieldlookup:: date

``date``
~~~~~~~~

.. versionadded:: 1.9

接收日期值. datetime字段会被转换为date. 可以再跟上额外查询.

例如::

    Entry.objects.filter(pub_date__date=datetime.date(2005, 1, 1))
    Entry.objects.filter(pub_date__date__gt=datetime.date(2005, 1, 1))

(本次查询没有给出等价的SQL片段, 因为不同的数据库引擎对此的实现不尽相同.)

:setting:`USE_TZ` 设置为 ``True`` 时, 执行过滤前字段会被转换为当前的时区.

.. fieldlookup:: year

``year``
~~~~~~~~

接收整数类型年份值,匹配精确年份. 可以再跟上额外查询.

例如::

    Entry.objects.filter(pub_date__year=2005)
    Entry.objects.filter(pub_date__year__gte=2005)

等价于SQL::

    SELECT ... WHERE pub_date BETWEEN '2005-01-01' AND '2005-12-31';
    SELECT ... WHERE pub_date >= '2005-01-01';

(确切的SQL语句因数据库而异.)

:setting:`USE_TZ` 设置为 ``True`` 时, datetime字段会在过滤前转换到当前时区.

.. versionchanged:: 1.9

    允许跟上额外查询.

.. fieldlookup:: month

``month``
~~~~~~~~~

接收1 (一月)至12 (十二月) 的整数. 用于date和datetime字段精确匹配月份. 允许跟上额外查询

例如::

    Entry.objects.filter(pub_date__month=12)
    Entry.objects.filter(pub_date__month__gte=6)

等价于SQL::

    SELECT ... WHERE EXTRACT('month' FROM pub_date) = '12';
    SELECT ... WHERE EXTRACT('month' FROM pub_date) >= '6';

(确切的SQL语句因数据库而异.)

:setting:`USE_TZ` 设置为 ``True`` 时, datetime字段会在过滤前转换到当前时区.
这需要 :ref:`数据库时区设置 <database-time-zone-definitions>`.

.. versionchanged:: 1.9

    允许跟上额外查询.

.. fieldlookup:: day

``day``
~~~~~~~

接收整型天数, 用于date和datetime字段精确匹配天数. 允许跟上额外查询.

例如::

    Entry.objects.filter(pub_date__day=3)
    Entry.objects.filter(pub_date__day__gte=3)

等价于SQL::

    SELECT ... WHERE EXTRACT('day' FROM pub_date) = '3';
    SELECT ... WHERE EXTRACT('day' FROM pub_date) >= '3';

(确切的SQL语句因数据库而异.)

注意这会匹配出所有月份3号的记录, 比如1月3号,7月3号等.

:setting:`USE_TZ` 设置为 ``True`` 时, datetime字段会在过滤前转换到当前时区.
这需要 :ref:`数据库时区设置 <database-time-zone-definitions>`.

.. versionchanged:: 1.9

    允许跟上额外查询.

.. fieldlookup:: week_day

``week_day``
~~~~~~~~~~~~

接收一周的天数1(星期一)到7(星期天),整型, 用于date和datetime字段匹配'一周中的第几天'. 允许跟上额外查询.

例如::

    Entry.objects.filter(pub_date__week_day=2)
    Entry.objects.filter(pub_date__week_day__gte=2)

(本次查询没有给出等价的SQL片段, 因为不同的数据库引擎对此的实现不尽相同.)

注意这会匹配出 ``pub_date`` 为星期一(一周的第二天), 不管是哪一月或是哪一年. 星期天的数值为1,星期六为7.

:setting:`USE_TZ` 设置为 ``True`` 时, datetime字段会在过滤前转换到当前时区.
这需要 :ref:`数据库时区设置 <database-time-zone-definitions>`.

.. versionchanged:: 1.9

    允许跟上额外查询.

.. fieldlookup:: hour

``hour``
~~~~~~~~

接收0到23的整数, 用于datetime和time字段精确匹配小时数. 允许跟上额外查询.

例如::

    Event.objects.filter(timestamp__hour=23)
    Event.objects.filter(time__hour=5)
    Event.objects.filter(timestamp__hour__gte=12)

等价于SQL::

    SELECT ... WHERE EXTRACT('hour' FROM timestamp) = '23';
    SELECT ... WHERE EXTRACT('hour' FROM time) = '5';
    SELECT ... WHERE EXTRACT('hour' FROM timestamp) >= '12';

(确切的SQL语句因数据库而异.)

对于datetime字段, :setting:`USE_TZ` 设置为 ``True`` 时, 字段值在过滤前将会被转换到当前时区.

.. versionchanged:: 1.9

    新增对SQLite :class:`~django.db.models.TimeField` 支持(其他数据库从1.7开始支持).

.. versionchanged:: 1.9

    允许跟上额外查询.

.. fieldlookup:: minute

``minute``
~~~~~~~~~~

接收0到59的整数, 用于datetime和time字段精确匹配分钟数. 允许跟上额外查询.

例如::

    Event.objects.filter(timestamp__minute=29)
    Event.objects.filter(time__minute=46)
    Event.objects.filter(timestamp__minute__gte=29)

等于SQL::

    SELECT ... WHERE EXTRACT('minute' FROM timestamp) = '29';
    SELECT ... WHERE EXTRACT('minute' FROM time) = '46';
    SELECT ... WHERE EXTRACT('minute' FROM timestamp) >= '29';

(确切的SQL语句因数据库而异.)

对于datetime字段, :setting:`USE_TZ` 设置为 ``True`` 时, 字段值在过滤前将会被转换到当前时区.

.. versionchanged:: 1.9

    新增对SQLite :class:`~django.db.models.TimeField` 支持(其他数据库从1.7开始支持).

.. versionchanged:: 1.9

    允许跟上额外查询.

.. fieldlookup:: second

``second``
~~~~~~~~~~

接收0到59的整数, 用于datetime和time字段精确匹配秒数. 允许跟上额外查询.

例如::

    Event.objects.filter(timestamp__second=31)
    Event.objects.filter(time__second=2)
    Event.objects.filter(timestamp__second__gte=31)

等价于SQL::

    SELECT ... WHERE EXTRACT('second' FROM timestamp) = '31';
    SELECT ... WHERE EXTRACT('second' FROM time) = '2';
    SELECT ... WHERE EXTRACT('second' FROM timestamp) >= '31';

(确切的SQL语句因数据库而异.)

对于datetime字段, :setting:`USE_TZ` 设置为 ``True`` 时, 字段值在过滤前将会被转换到当前时区.

.. versionchanged:: 1.9

    新增对SQLite :class:`~django.db.models.TimeField` 支持(其他数据库从1.7开始支持).

.. versionchanged:: 1.9

    允许跟上额外查询.

.. fieldlookup:: isnull

``isnull``
~~~~~~~~~~

接收 ``True`` 或者 ``False``, 它们分别对应了SQL查询的
``IS NULL`` 和 ``IS NOT NULL``.

例如::

    Entry.objects.filter(pub_date__isnull=True)

等价于SQL::

    SELECT ... WHERE pub_date IS NULL;

.. fieldlookup:: search

``search``
~~~~~~~~~~

.. deprecated:: 1.10

    查看 :ref:`the 1.10 release notes <search-lookup-replacement>` 如何代替它.

利用全文索引的布尔型的全文搜索, 它有一点像 :lookup:`contains` , 但得益于全文索引它更快.

例如::

    Entry.objects.filter(headline__search="+Django -jazz Python")

等价于SQL::

    SELECT ... WHERE MATCH(tablename, headline) AGAINST (+Django -jazz Python IN BOOLEAN MODE);

注意这仅在Mysql中生效, 而且还要求在数据库中添加有全文索引.
默认情况下, Django使用BOOLEAN MODE进行全文搜索. 详见 `MySQL 文档`_ .

.. _MySQL 文档: https://dev.mysql.com/doc/refman/en/fulltext-boolean.html

.. fieldlookup:: regex

``regex``
~~~~~~~~~

大小写敏感的正则匹配.

正则表达式必须是数据库使用的语法. SQLite没有内置正则表达式支持, 这一功能是有Python的UDF(user-defined function)的REGEXP提供,
因此正则表达式语法应参照Python的 ``re`` 模块的语法.

例如::

    Entry.objects.get(title__regex=r'^(An?|The) +')

等价于SQL::

    SELECT ... WHERE title REGEXP BINARY '^(An?|The) +'; -- MySQL

    SELECT ... WHERE REGEXP_LIKE(title, '^(An?|The) +', 'c'); -- Oracle

    SELECT ... WHERE title ~ '^(An?|The) +'; -- PostgreSQL

    SELECT ... WHERE title REGEXP '^(An?|The) +'; -- SQLite

建议使用原始字符串 (例如. 使用 ``r'foo'`` 代替 ``'foo'``) 传入正则表达式.

.. fieldlookup:: iregex

``iregex``
~~~~~~~~~~

大小写不敏感的正则匹配.

例如::

    Entry.objects.get(title__iregex=r'^(an?|the) +')

等价于SQL::

    SELECT ... WHERE title REGEXP '^(an?|the) +'; -- MySQL

    SELECT ... WHERE REGEXP_LIKE(title, '^(an?|the) +', 'i'); -- Oracle

    SELECT ... WHERE title ~* '^(an?|the) +'; -- PostgreSQL

    SELECT ... WHERE title REGEXP '(?i)^(an?|the) +'; -- SQLite

.. _聚合函数:
.. _aggregation-functions:

聚合函数
----------

.. currentmodule:: django.db.models

Django的 ``django.db.models`` 提供了一下聚合函数. 如何使用聚合函数请参考: :doc:`专题指南-聚合
</topics/db/aggregation>`. 如何创建聚合函数请参考: :class:`~django.db.models.Aggregate` .

.. warning::

    SQLite无法在date/time类型字段使用聚合函数.
    这是因为SQLite没有原生的date/time类型, Django是使用text类型还模拟这一功能.
    如果在SQLite上对date/time类型字段使用聚合查询将会抛出 ``NotImplementedError`` 异常.

.. admonition:: Note

    对空的 ``QuerySet`` 使用聚合函数将会返回 ``None``.
    例如, 当 ``QuerySet`` 为空时, 聚合函数 ``Sum`` 将会返回 ``None`` 而不是 ``0``.
    但 ``Count`` 函数例外, ``QuerySet`` 为空时会返回  ``0`` .

所有聚合函数都有以下共同参数:

``expression``
~~~~~~~~~~~~~~

模型字段的字符串表示, 或者是 :doc:`查询表达式
</ref/models/expressions>`.

``output_field``
~~~~~~~~~~~~~~~~

可选参数, 表示返回的 :doc:`模型字段 </ref/models/fields>` 类型.


.. note::

    聚合多个字段时, Django只能在所有字段类型相同的情况下确定
    ``output_field``, 否则必须传入 ``output_field`` .

``**extra``
~~~~~~~~~~~

关键字参数, 可以为聚合生成的SQL提供额外的上下文.

``Avg``
~~~~~~~

.. class:: Avg(expression, output_field=FloatField(), **extra)

    返回指定表达式的均值, 如果没有指定其他类型的 ``output_field`` 其必须为数值.

    * 默认别名: ``<field>__avg``
    * 返回类型: ``float`` (或者是指定的 ``output_field``)

    .. versionchanged:: 1.9

        ``output_field`` 参数支持非数值字段, 例如 ``DurationField``.

``Count``
~~~~~~~~~

.. class:: Count(expression, distinct=False, **extra)

    返回表达式关联对象的数量.

    * 默认别名: ``<field>__count``
    * 返回类型: ``int``

    包含一个可选参数:

    .. attribute:: distinct

        如果 ``distinct=True``, 返回数量仅包含非重复的实例.
        这等价于SQL中的 ``COUNT(DISTINCT <field>)``. 该参数默认值为 ``False``.

``Max``
~~~~~~~

.. class:: Max(expression, output_field=None, **extra)

    返回给定表达式的最大值.

    * 默认别名: ``<field>__max``
    * 返回类型: 和输入字段一致, 或者是传入的 ``output_field`` .

``Min``
~~~~~~~

.. class:: Min(expression, output_field=None, **extra)

    返回给定表达式的最小值.

    * 默认别名: ``<field>__min``
    * 返回类型: 和输入字段一致, 或者是传入的 ``output_field`` .

``StdDev``
~~~~~~~~~~

.. class:: StdDev(expression, sample=False, **extra)

    返回给定表达式中数据的标准差.

    * 默认别名: ``<field>__stddev``
    * 返回类型: ``float``

    包含一个可选参数:

    .. attribute:: sample

        默认情况下, ``StdDev`` 返回的是总体标准差. 如果设置``sample=True``, 则返回样本标准差.

    .. admonition:: SQLite

        SQLite没有直接提供 ``Variance``. 有一个SQLite扩展模块实现了该功能. 详见 `SQLite 文档`_ .

``Sum``
~~~~~~~

.. class:: Sum(expression, output_field=None, **extra)

    计算给定表达式的和.

    * 默认别名: ``<field>__sum``
    * 返回类型: 和输入字段一致, 或者是传入的 ``output_field`` .

``Variance``
~~~~~~~~~~~~

.. class:: Variance(expression, sample=False, **extra)

    返回给定表达式中数据的方差.

    * 默认别名: ``<field>__variance``
    * 返回类型: ``float``

    包含一个可选参数:

    .. attribute:: sample

        默认情况下, ``Variance`` 返回总体方差, 如果设置 ``sample=True``, 则返回样本方差.

    .. admonition:: SQLite

        SQLite没有直接提供 ``Variance``. 有一个SQLite扩展模块实现了该功能. 详见 `SQLite 文档`_ .

.. _SQLite 文档: https://www.sqlite.org/contrib

相关查询工具
===================

本节提供了一些其他地方没有记载的相关查询的工具和参考.

``Q()`` 对象
---------------

.. class:: Q

``Q()`` 和 :class:`~django.db.models.F` 类似,
将SQL表达式封装在表示数据库操作的Python对象中.

通常, ``Q() objects`` 常用于查询条件复用的情况.
然后使用 ``|`` (``OR``) 和 ``&`` (``AND``)运算符来构建复杂的查询.
详见 :ref:`构建复杂查询 <complex-lookups-with-q>` .

``Prefetch()`` objects
----------------------

.. class:: Prefetch(lookup, queryset=None, to_attr=None)

``Prefetch()`` 对象可以用来控制
:meth:`~django.db.models.query.QuerySet.prefetch_related()` 行为.

``lookup`` 参数描述了要遵循的关系, 其工作原理与传递给 :meth:`~django.db.models.query.QuerySet.prefetch_related()` 的基于字符串的查找相同.
例如:

    >>> from django.db.models import Prefetch
    >>> Question.objects.prefetch_related(Prefetch('choice_set')).get().choice_set.all()
    <QuerySet [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]>
    # This will only execute two queries regardless of the number of Question
    # and Choice objects.
    >>> Question.objects.prefetch_related(Prefetch('choice_set')).all()
    <QuerySet [<Question: Question object>]>

``queryset`` 参数为给定的查询提供基本的 ``QuerySet`` .
对于进一步过滤预取操作或从预取关系中调用
:meth:`~django.db.models.query.QuerySet.select_related()` 很有用, 从而进一步减少了查询数量:

    >>> voted_choices = Choice.objects.filter(votes__gt=0)
    >>> voted_choices
    <QuerySet [<Choice: The sky>]>
    >>> prefetch = Prefetch('choice_set', queryset=voted_choices)
    >>> Question.objects.prefetch_related(prefetch).get().choice_set.all()
    <QuerySet [<Choice: The sky>]>

``to_attr`` 参数将预取操作的结果设置为自定义属性:

    >>> prefetch = Prefetch('choice_set', queryset=voted_choices, to_attr='voted_choices')
    >>> Question.objects.prefetch_related(prefetch).get().voted_choices
    <QuerySet [<Choice: The sky>]>
    >>> Question.objects.prefetch_related(prefetch).get().choice_set.all()
    <QuerySet [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]>

.. note::

    当使用 ``to_attr`` 时, 预取结果存储在列表中.  与传统的 ``prefetch_related`` 调用相比，这可以显着提高速度,
    传统的 ``prefetch_related`` 调用将缓存的结果存储在 ``QuerySet`` 实例中.

``prefetch_related_objects()``
------------------------------

.. function:: prefetch_related_objects(model_instances, *related_lookups)

.. versionadded:: 1.10

在作为模型实例的可迭代对象上预取给定的查找.
这在接收模型实例列表而不是 ``QuerySet`` 的代码中很有用;
例如,从缓存中获取模型或手动实例化它们.

传递一个作为模型实例的可迭代对象(必须是同一个类)和你想预取的查找或 :class:`Prefetch` 对象.
例如::

    >>> from django.db.models import prefetch_related_objects
    >>> restaurants = fetch_top_restaurants_from_cache()  # A list of Restaurants
    >>> prefetch_related_objects(restaurants, 'pizzas__toppings')
