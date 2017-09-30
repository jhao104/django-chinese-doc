=========
执行查询
=========

.. currentmodule:: django.db.models

只要你建好 :doc:`数据 </topics/db/models>`, Django 会自动为你生成一套数据库抽象的API，
可以让你创建、检索、更新和删除对象。这篇文档阐述如何使用这些API。
关于模型查询所有选项的完整细节，请见 :doc:`数据模型参考 </ref/models/index>` 。

在整个文档（以及参考）中，都将引用下面的模型，它是一个博客应用：

.. _queryset-model-example:

.. code-block:: python

    from django.db import models

    class Blog(models.Model):
        name = models.CharField(max_length=100)
        tagline = models.TextField()

        def __str__(self):              # __unicode__ on Python 2
            return self.name

    class Author(models.Model):
        name = models.CharField(max_length=200)
        email = models.EmailField()

        def __str__(self):              # __unicode__ on Python 2
            return self.name

    class Entry(models.Model):
        blog = models.ForeignKey(Blog)
        headline = models.CharField(max_length=255)
        body_text = models.TextField()
        pub_date = models.DateField()
        mod_date = models.DateField()
        authors = models.ManyToManyField(Author)
        n_comments = models.IntegerField()
        n_pingbacks = models.IntegerField()
        rating = models.IntegerField()

        def __str__(self):              # __unicode__ on Python 2
            return self.headline

创建对象
=========

Django 使用一种直观的方式把数据库表中的数据表示成Python对象：
一个模型类代表数据库中的一个表，一个模型类的实例代表这个数据库表中的一条记录。

使用关键字参数实例化模型实例来创建一个对象，然后调用 :meth:`~django.db.models.Model.save` 把它保存到数据库中。

假设模型位于文件 ``mysite/blog/models.py`` 中，下面是一个例子::

    >>> from blog.models import Blog
    >>> b = Blog(name='Beatles Blog', tagline='All the latest Beatles news.')
    >>> b.save()

上面的代码其实是执行了SQL 的 ``INSERT`` 语句。在你调用 :meth:`~django.db.models.Model.save`
之前，Django 不会访问数据库。

:meth:`~django.db.models.Model.save` 方法没有返回值。

.. seealso::

    :meth:`~django.db.models.Model.save` 方法还有一些高级选项，完整的细节参见
    :meth:`~django.db.models.Model.save` 文档。

    如果你想只用一条语句创建并保存一个对象，可以使用
    :meth:`~django.db.models.query.QuerySet.create()` 方法。

修改对象
==========

要保存一个数据库已存在对象的修改也是使用
:meth:`~django.db.models.Model.save` 方法。

假设 ``Blog`` 的一个实例 ``b5`` 已经被保存在数据库中，下面这个例子将更改它的 name 并且更新数据库中的记录::

    >>> b5.name = 'New name'
    >>> b5.save()

上面的代码其实是执行了SQL 的 ``UPDATE`` 语句。在你调用 :meth:`~django.db.models.Model.save` 之前，Django 不会访问数据库。

保存 ``ForeignKey`` 和 ``ManyToManyField`` 字段
--------------------------------------------------

更新 :class:`~django.db.models.ForeignKey 字段的方式和保存普通字段相同 —— 只要把一个正确类型的对象赋值给该字段即可。
下面的例子更新了 ``Entry`` 类的实例 ``entry`` 的 ``blog`` 属性，假设 ``Entry`` 和 ``Blog`` 分别已经有一个实例保存在数据库中
（所以我们才能像下面这样获取它们）::

    >>> from blog.models import Entry
    >>> entry = Entry.objects.get(pk=1)
    >>> cheese_blog = Blog.objects.get(name="Cheddar Talk")
    >>> entry.blog = cheese_blog
    >>> entry.save()

更新 :class:`~django.db.models.ManyToManyField` 的方式有一些不同 —— 需要使用字段的
:meth:`~django.db.models.fields.related.RelatedManager.add` 方法来增加关联关系的一条记录。
下面这个例子向 ``entry`` 对象添加 ``Author`` 类的实例 ``joe`` ::

    >>> from blog.models import Author
    >>> joe = Author.objects.create(name="Joe")
    >>> entry.authors.add(joe)

可以在调用 :meth:`~django.db.models.fields.related.RelatedManager.add` 方法时传入多参数，一次性向
:class:`~django.db.models.ManyToManyField` 添加多条记录::

    >>> john = Author.objects.create(name="John")
    >>> paul = Author.objects.create(name="Paul")
    >>> george = Author.objects.create(name="George")
    >>> ringo = Author.objects.create(name="Ringo")
    >>> entry.authors.add(john, paul, george, ringo)

Django将会在你赋值或添加错误类型的对象时报错。

.. _retrieving-objects:

检索对象
==========

通过使用模型中的 :class:`~django.db.models.Manager` 构造一个
:class:`~django.db.models.query.QuerySet` 来从数据库中获取对象。

:class:`~django.db.models.query.QuerySet` 表示从数据库中取出来的对象的集合。
它可以包含零个、一个或者多个 *过滤器* 。 过滤器的功能是基于所给的参数过滤查询的结果。
从SQL角度看，
:class:`~django.db.models.query.QuerySet` 等价于 ``SELECT`` 语句,
过滤器就相当于 ``WHERE`` 或者 ``LIMIT`` 这样的子句。

:class:`~django.db.models.query.QuerySet` 通过模型的
:class:`~django.db.models.Manager` 获取。每个模型至少包含一个
:class:`~django.db.models.Manager`, 它们默认叫做
:attr:`~django.db.models.Model.objects` 。 可以通过模型类直接访问， 例如::

    >>> Blog.objects
    <django.db.models.manager.Manager object at ...>
    >>> b = Blog(name='Foo', tagline='Bar')
    >>> b.objects
    Traceback:
        ...
    AttributeError: "Manager isn't accessible via Blog instances."

.. note::

    ``Managers`` 只能通过模型类访问，而不是模型实例。
    目的是为了强制区分“表级别”的操作和“记录级别”的操作。

:class:`~django.db.models.Manager` 是 ``QuerySets`` 的主要来源。
比如， ``Blog.objects.all()`` 返回一个数据库中所有 ``Blog`` 对象的
:class:`~django.db.models.query.QuerySet` 。

获取所有对象
-------------

表中检索对象的最简单方法是获取所有对象。要做到这一点，在 :class:`~django.db.models.Manager` 上使用 :meth:`~django.db.models.query.QuerySet.all`
方法::

    >>> all_entries = Entry.objects.all()

:meth:`~django.db.models.query.QuerySet.all` 返回一个数据库中所有对象的
:class:`~django.db.models.query.QuerySet` 。

使用过滤器检索
---------------

:meth:`~django.db.models.query.QuerySet.all` 方法返回包含数据库所有记录的 :class:`~django.db.models.query.QuerySet`。
但是往往只需要获取其中的一个子集。

要创建这样一个子集，你需要在原始的的 :class:`~django.db.models.query.QuerySet` 上增加一些过滤条件。
有两种方式可以实现：

``filter(**kwargs)``
    返回一个新的 :class:`~django.db.models.query.QuerySet` ，它包含满足查询参数的对象。

``exclude(**kwargs)``
    返回一个新的 :class:`~django.db.models.query.QuerySet` 。它包含 *不* 满足查询参数的对象。

查询参数 (上面函数中的 ``**kwargs`` ) 需要满足特定的格式，下面 `字段查询`_ 一节会提到。

例如, 使用 :meth:`~django.db.models.query.QuerySet.filter`
方法获取年份为2006的所有文章的 :class:`~django.db.models.query.QuerySet` ::

    Entry.objects.filter(pub_date__year=2006)

利用默认的管理器，它相当于::

    Entry.objects.all().filter(pub_date__year=2006)

.. _chaining-filters:

链式过滤
~~~~~~~~~~

:class:`~django.db.models.query.QuerySet` 的筛选结果本身还是
:class:`~django.db.models.query.QuerySet`, 所以可以将筛选语句链接在一起::

    >>> Entry.objects.filter(
    ...     headline__startswith='What'
    ... ).exclude(
    ...     pub_date__gte=datetime.date.today()
    ... ).filter(
    ...     pub_date__gte=datetime(2005, 1, 30)
    ... )

这个例子最开始获取数据库中所有对象的一个 :class:`~django.db.models.query.QuerySet`, 之后增加一个过滤器，然后是一个排除器，再之后又是一个过滤器。
但是最终结果还是 :class:`~django.db.models.query.QuerySet` 。它包含标题以”What“开头、发布日期在2005年1月30日至当天之间的所有记录。

.. _filtered-querysets-are-unique:

过滤后的 ``QuerySet`` 是独立的
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

每次你筛选一个 :class:`~django.db.models.query.QuerySet`, 得到的都是全新的另一个
:class:`~django.db.models.query.QuerySet` ， 它和之前的 :class:`~django.db.models.query.QuerySet`
之间没有任何绑定关系。每次筛选都会创建一个独立的
:class:`~django.db.models.query.QuerySet` ，它可以被存储及反复使用。

例如::

    >>> q1 = Entry.objects.filter(headline__startswith="What")
    >>> q2 = q1.exclude(pub_date__gte=datetime.date.today())
    >>> q3 = q1.filter(pub_date__gte=datetime.date.today())

这三个 ``QuerySets`` 都是独立的。第一个是包含所有标题以“What”开头的
:class:`~django.db.models.query.QuerySet` 。第二个是第一个的子集，
增加了限制条件，排除了 ``pub_date`` 大于等于今天的记录。
第三个也是第一个的子集，限制条件是：只要 ``pub_date`` 大于等于今天的记录。
而原来的 :class:`~django.db.models.query.QuerySet` (``q1``) 不会受到筛选的影响。

.. _querysets-are-lazy:

``QuerySet`` 是惰性的
~~~~~~~~~~~~~~~~~~~~~~~

``QuerySets`` 是惰性执行的 —— 创建
:class:`~django.db.models.query.QuerySet` 不会立即执行任何数据库的访问。
你可以将过滤器保持一整天，直到 :class:`~django.db.models.query.QuerySet` 被
*求值* 时，Django 才会真正运行这个查询。看下这个例子::

    >>> q = Entry.objects.filter(headline__startswith="What")
    >>> q = q.filter(pub_date__lte=datetime.date.today())
    >>> q = q.exclude(body_text__icontains="food")
    >>> print(q)

虽然它看上去有三次数据库访问, 但事实上只有在最后一行 (``print(q)``) 时才访问一次数据库。
一般来说，只有在“请求”
:class:`~django.db.models.query.QuerySet` 的结果时才会到数据库中去获取它们。
当你确实需要结果时，
:class:`~django.db.models.query.QuerySet` 通过访问数据库来求值。 关于求值发生的准确时间，参见
:ref:`when-querysets-are-evaluated`.

.. _retrieving-single-object-with-get:

使用 ``get()`` 获取单个对象
-----------------------------------------

:meth:`~django.db.models.query.QuerySet.filter` 始终返回一个
:class:`~django.db.models.query.QuerySet` ，即使只有一个对象满足查询条件。
—— 这种情况下，
:class:`~django.db.models.query.QuerySet` 将只包含一个元素。

如果你知道只有一个对象满足你的查询，你可以使用 :class:`~django.db.models.Manager` 的
:meth:`~django.db.models.query.QuerySet.get` 方法，它直接返回该对象::

    >>> one_entry = Entry.objects.get(pk=1)

你可以对
:meth:`~django.db.models.query.QuerySet.get` 使用任何查询表达式， 就和
:meth:`~django.db.models.query.QuerySet.filter` 一样， 参考 `字段查询`_ 。

但是
:meth:`~django.db.models.query.QuerySet.get` 和
:meth:`~django.db.models.query.QuerySet.filter` 有一点区别，如果没有符合条件的查询结果
:meth:`~django.db.models.query.QuerySet.get` 会抛出一个 ``DoesNotExist`` 异常。
这个异常是正在查询的模型类的一个属性，所以在上面的代码中，如果没有主键为 1
的 `Entry`, Django 将抛出一个 ``Entry.DoesNotExist``。

同样，如果 :meth:`~django.db.models.query.QuerySet.get` 满足条件的结果超过1个,Django会抛出一个
:exc:`~django.core.exceptions.MultipleObjectsReturned` 异常。


其他 ``QuerySet`` 方法
------------------------

查询数据库时，大多数会使用方法 :meth:`~django.db.models.query.QuerySet.all`,
:meth:`~django.db.models.query.QuerySet.get`,
:meth:`~django.db.models.query.QuerySet.filter` 和
:meth:`~django.db.models.query.QuerySet.exclude` 。
但是这只是其中一小部分方法，有关 :class:`~django.db.models.query.QuerySet` 完整的方法列表，请参见
:ref:`QuerySet API Reference <queryset-api>` 。

.. _limiting-querysets:

``QuerySet`` 的Limit
-----------------------

可以使用Python 的切片语法来限制
:class:`~django.db.models.query.QuerySet` 记录的数目。它等同于SQL 的
``LIMIT`` 和 ``OFFSET`` 子句。

例如，下面的语句返回前面5个对象 (``LIMIT 5``)::

    >>> Entry.objects.all()[:5]

下面这条语句返回第6至第10个对像 (``OFFSET 5 LIMIT 5``)::

    >>> Entry.objects.all()[5:10]

不支持负数索引 (i.e. ``Entry.objects.all()[-1]``) 。

通常, :class:`~django.db.models.query.QuerySet`  的切片返回一个新的
:class:`~django.db.models.query.QuerySet` -- 它不会立即执行查询。但是，如果你使用了Python切片语法中的“步长”参数，
比如下面的语句将在前10个对象中每隔2个对象返回，这样会立即执行数据库查询::

    >>> Entry.objects.all()[:10:2]

若不想获取列表，要一个单一的对象(e.g. ``SELECT foo FROM bar LIMIT 1``)，可以直接使用位置索引而不是切片。
。例如，下面的语句返回数据库中根据标题排序后的第一条 `Entry` ::

    >>> Entry.objects.order_by('headline')[0]

它等同于::

    >>> Entry.objects.order_by('headline')[0:1].get()

不同的是，如果没有满足条件的结果，第一种方法将引发 ``IndexError`` 异常，第二种方法会引发 ``DoesNotExist`` 异常。
更多细节参见
:meth:`~django.db.models.query.QuerySet.get` 。

.. _field-lookups-intro:

字段查询
---------

字段查询是指如何指定SQL ``WHERE`` 子句的内容，
它们通过 :class:`~django.db.models.query.QuerySet`
的 :meth:`~django.db.models.query.QuerySet.filter`,
:meth:`~django.db.models.query.QuerySet.exclude` 和
:meth:`~django.db.models.query.QuerySet.get` 方法的关键字参数指定。

查询的关键字参数的基本形式是 ``field__lookuptype=value``.
(中间是两个下划线)。 例如::

    >>> Entry.objects.filter(pub_date__lte='2006-01-01')

翻译成SQL就是:

.. code-block:: sql

    SELECT * FROM blog_entry WHERE pub_date <= '2006-01-01';

.. admonition:: 如何实现

   Python 定义的函数可以接收任意的键/值对参数，这些名称和参数可以在运行时求值。更多信息，
   参见Python 官方文档中的 `关键字参数`_ 。

.. _关键字参数: https://docs.python.org/3/tutorial/controlflow.html#keyword-arguments

查询条件中指定的字段必须是模型字段的名称。但有一个例外，对于 :class:`~django.db.models.ForeignKey`
你可以使用字段名加上 ``_id`` 后缀。在这种情况下，该参数的值应该是外键的原始值。例如::

    >>> Entry.objects.filter(blog_id=4)

如果传入的是一个不合法的参数，查询函数将引发
``TypeError``。

这些数据库API 支持大约二十多种查询的类型; 完整的参考请参见 :ref:`字段查询 <field-lookups>` 。
下面是一些可能用到的常见查询：

:lookup:`exact`
    “精确”匹配。例如::

        >>> Entry.objects.get(headline__exact="Cat bites dog")

    将生成下面的SQL:

    .. code-block:: sql

        SELECT ... WHERE headline = 'Cat bites dog';

    如果没有提供查询类型 -- 即如果关键字参数不包含双下划线
    -- 默认查询类型就是
    ``exact`` 。

    因此，下面的两条语句相等::

        >>> Blog.objects.get(id__exact=14)  # Explicit form
        >>> Blog.objects.get(id=14)         # __exact is implied

    这是为了方便，因为 ``exact`` 查询是最常见的查询。

:lookup:`iexact`
    大小写不敏感的匹配。所以这个查询::

        >>> Blog.objects.get(name__iexact="beatles blog")

    将匹配到标题为 ``"Beatles Blog"`` 和 ``"beatles blog"`` 甚至 ``"BeAtlES blOG"`` 的 ``Blog`` 。

:lookup:`contains`
    大小写敏感的包含关系。 例如::

        Entry.objects.get(headline__contains='Lennon')

    可以翻译成下面的SQL:

    .. code-block:: sql

        SELECT ... WHERE headline LIKE '%Lennon%';

    注意，这种可以匹配到 ``'Today Lennon honored'`` 但匹配不到
    ``'today lennon honored'`` 。

    同样也有个大小写不敏感的版本 :lookup:`icontains` 。

:lookup:`startswith` 和 :lookup:`endswith`
    分别是查找以目标字符串开头和结尾的记录,同样的，它们都有一个不区分大小写的方法
    :lookup:`istartswith` 和
    :lookup:`iendswith` 。

上面罗列的仅仅是部分查询方法，完整的参考：
:ref:`字段查询参考 <field-lookups>`.

.. _lookups-that-span-relationships:

夸关联关系查询
----------------

Django 提供了强大而又直观的方式来“处理”查询中的关联关系，它在后台自动帮你处理 ``JOIN`` 。
若要使用关联关系的字段查询，只需使用关联的模型字段的名称，并使用双下划线分隔。

比如要获取所有 ``Entry`` 中所有 ``Blog`` 的 ``name`` 为 ``'Beatles Blog'`` 的对象::

    >>> Entry.objects.filter(blog__name='Beatles Blog')

而且这种查询可以是任意深度的。
反过来也是可行的。若要引用一个“反向”的关系，使用该模型的小写的名称即可。

比如，获取所有 ``Blog`` 中 ``Entry``
的 ``headline`` 包含 ``'Lennon'`` 的对象::

    >>> Blog.objects.filter(entry__headline__contains='Lennon')

如果多个关联关系直接过滤而且其中某个中间模型没有满足过滤条件的值，
Django 会把它当做一个空的（所有的值都为NULL）合法对象。这意味着不会引发任何错误。例如，在下面的过滤器中::

    Blog.objects.filter(entry__authors__name='Lennon')

(假设存在 ``Author`` 的关联模型), 如果没有找到符合条件的 ``author`` , 那么都会返回空,
而不是引发缺失 ``author`` 的异常，
这是一种比较好的处理方式，但是当使用
:lookup:`isnull` 就会有二义性。 例如::

    Blog.objects.filter(entry__authors__name__isnull=True)

这将会返回 ``author`` 中 ``name`` 为空的 ``Blog`` 对象，以及 ``entry`` 中  ``author`` 为空的``Blog`` 对象。
如果你不需要后者，你可以修改成::

    Blog.objects.filter(entry__authors__isnull=False, entry__authors__name__isnull=True)

跨关联关系多值查询
~~~~~~~~~~~~~~~~~~~~~~

当你使用
:class:`~django.db.models.ManyToManyField` 或者
:class:`~django.db.models.ForeignKey` 来过滤一个对象时,有两种不同的过滤方式。
对于 ``Blog``/``Entry`` 的关联关系
(``Blog`` 和 ``Entry`` 是一对多关系)。既可以查找headline为 *"Lennon"* 并且pub_date是2008的Entry,
也可以查找 headline为
*"Lennon"* 或者 pub_date为
2008的Entry。这两种查询都是有可能并且有意义的。

:class:`~django.db.models.ManyToManyField` 也有类似的情况。比如，如果 ``Entry`` 有一个
:class:`~django.db.models.ManyToManyField` ``tags``,这样可能想找到tag为 *"music"* and *"bands"* 的Entry,
或者我们想找一个tag名为 *"music"* 且状态为“public”的Entry。

这些情况都可以使用 :meth:`~django.db.models.query.QuerySet.filter` 来处理。
在单个 :meth:`~django.db.models.query.QuerySet.filter` 中的条件都会被同时应用到匹配。

这种描述可能不好理解，用一个例子来说明。比如要选择所有的entry包含 *"Lennon"* 标题并于2008年发表的博客(即这个Blog的entry要同时包含这两个条件)，
查询代码是这样的::

    Blog.objects.filter(entry__headline__contains='Lennon', entry__pub_date__year=2008)

要选择Blog的entry包含 *"Lennon"* 标题，或者是2008年出版的，查询代码是这样的::

    Blog.objects.filter(entry__headline__contains='Lennon').filter(entry__pub_date__year=2008)

假设有一个Blog拥有一个标题包含 *"Lennon"* 的entry和一个来自2008年的entry。
第一个查询将匹配不到Blog, 第二个查询才会匹配上这个Blog。

第二个例子中，第一个filter过滤出的查询集是所有关联有标题包含 *"Lennon"* 的entry的Blog,
第二个filter是在第一个的查询集中过滤出关联有发布时间是2008的entry的Blog。
第二个filter过滤出来的entry与第一个filter过滤出来的entry可能相同也可能不同。
每个filter语句过滤的是 ``Blog`` ，而不是 ``Entry`` 。

.. note::

    夸关联关系的多值 :meth:`~django.db.models.query.QuerySet.filter` 查询和
    :meth:`~django.db.models.query.QuerySet.exclude` 不同。
    单个 :meth:`~django.db.models.query.QuerySet.exclude` 方法的条件不必引用同一个记录。

    例如，要排除标题中包含 *"Lennon"* 的entry和 发布在2008的entry::

        Blog.objects.exclude(
            entry__headline__contains='Lennon',
            entry__pub_date__year=2008,
        )

    但是，这个和
    :meth:`~django.db.models.query.QuerySet.filter` 不一样,它并不是排除同时满足这两个条件的Blog。
    如果要排除Blog中entry的标题包含 *"Lennon"* 且发布时间为2008的，需要改成这样::

        Blog.objects.exclude(
            entry__in=Entry.objects.filter(
                headline__contains='Lennon',
                pub_date__year=2008,
            ),
        )

.. _using-f-expressions-in-filters:

Filters 引用模型字段
---------------------

在上面例子中，最多是将模型字段和常量进行比较。那么如何将模型的一个字段与模型的另外一个字段进行比较？

Django 提供了 :class:`F 表达式 <django.db.models.F>` 来完成这种操作。
``F()`` 的实例作为查询中模型字段的引用。可以在查询filter中使用这些引用来比较相同模型不同instance上两个不同字段的值。

比如, 如果要查找comments数目多于pingbacks的Entry，可以构造一个 ``F()`` 对象来引用pingback数目，
并在查询中使用该 ``F()`` 对象::

    >>> from django.db.models import F
    >>> Entry.objects.filter(n_comments__gt=F('n_pingbacks'))

Django 支持对 ``F()`` 对象使用加法、减法、乘法、除法、取模以及幂计算等算术操作，
操作符两边可以都是常数或 ``F()`` 对象。例如，查找comments 数目比pingbacks 两倍还要多的Entry，可以将查询修改为::

    >>> Entry.objects.filter(n_comments__gt=F('n_pingbacks') * 2)

查询rating 比pingback 和comment 数目总和要小的Entry，可以这样查询::

    >>> Entry.objects.filter(rating__lt=F('n_comments') + F('n_pingbacks'))

``F()`` 还支持在对象中使用双下划线标记来跨关联关系查询。带有双下划线的 ``F()``
对象将引入任何需要的join 操作以访问关联的对象。例如，如要获取author的名字与blog名字相同的Entry，可以这样查询::

    >>> Entry.objects.filter(authors__name=F('blog__name'))

对于date 和date/time 字段，支持给它们加上或减去一个
:class:`~datetime.timedelta` 对象。
下面的例子将返回修改时间位于发布3天后的Entry::

    >>> from datetime import timedelta
    >>> Entry.objects.filter(mod_date__gt=F('pub_date') + timedelta(days=3))

``F()`` 对象支持 ``.bitand()`` 和
``.bitor()`` 两种位操作，例如::

    >>> F('somefield').bitand(16)

``pk`` 快捷查询
-----------------

为了方便，Django 提供一个查询快捷方式 ``pk`` ，它表示“primary key” 的意思。

在 ``Blog`` 模型示例中，主键是 ``id`` 字段，所以下面三条语句是等同的::

    >>> Blog.objects.get(id__exact=14) # Explicit form
    >>> Blog.objects.get(id=14) # __exact is implied
    >>> Blog.objects.get(pk=14) # pk implies id__exact



``pk`` 的使用不仅限于 ``__exact`` 查询 —— 任何查询类型都可以与 ``pk`` 结合来完成一个模型上对主键的查询::

    # Get blogs entries with id 1, 4 and 7
    >>> Blog.objects.filter(pk__in=[1,4,7])

    # Get all blog entries with id > 14
    >>> Blog.objects.filter(pk__gt=14)


``pk`` 查询在 join 中适用。例如，下面三个语句是等同的::

    >>> Entry.objects.filter(blog__id__exact=3) # Explicit form
    >>> Entry.objects.filter(blog__id=3)        # __exact is implied
    >>> Entry.objects.filter(blog__pk=3)        # __pk implies __id__exact

``LIKE`` 中的%和_转义
-------------------------


与 ``LIKE`` SQL 语句等同的字段查询（ ``iexact`` 、 ``contains`` 、 ``icontains`` 、
``startswith`` 、 ``istartswith`` 、 ``endswith`` 和 ``iendswith`` ）将自动转义在 ``LIKE`` 语句中
使用的两个特殊的字符 —— 百分号和下划线。（在 ``LIKE`` 语句中，百分号通配符表示多个字符，下划线通配符表示单个字符）

这样语句将很直观，不会显得太抽象。例如，要获取包含一个百分号的所有的 ``Entry`` ，只需要像其它任何字符一样使用百分号::

    >>> Entry.objects.filter(headline__contains='%')

Django 会帮你转义；生成的SQL 看上去会是这样s:

.. code-block:: sql

    SELECT ... WHERE headline LIKE '%\%%';

对于下划线是同样的道理。百分号和下划线都会自动地帮你处理。

.. _caching-and-querysets:

``QuerySet`` 缓存
--------------------

每个 :class:`~django.db.models.query.QuerySet` 都会缓存一个最小化的数据库访问。编写高效的代码前你需要理解它是如何工作的。

在一个新创建的 :class:`~django.db.models.query.QuerySet` 中，缓存为空。
首次对查询集进行求值——即产生数据库查询，Django将保存查询的结果到
:class:`~django.db.models.query.QuerySet` 的缓存中，并明确返回请求的结果（
例如，如果正在迭代 :class:`~django.db.models.query.QuerySet` ，则返回下一个结果）
接下来对该 :class:`~django.db.models.query.QuerySet` 的求值将重用缓存的结果。

请牢记这个缓存行为，因为对 :class:`~django.db.models.query.QuerySet` 使用不当的话，它会坑你的。
例如，下面的语句创建两个 :class:`~django.db.models.query.QuerySet` ，对它们求值，然后扔掉它们::

    >>> print([e.headline for e in Entry.objects.all()])
    >>> print([e.pub_date for e in Entry.objects.all()])

这意味着相同的数据库查询将执行两次，显然增加了你的数据库负载。
同时，还有可能两个结果列表并不包含相同的数据库记录，因为在两次请求期间有可能有 ``Entry`` 被添加进来或删除掉。

为了避免这个问题，只需保存 :class:`~django.db.models.query.QuerySet` 并重新使用它::

    >>> queryset = Entry.objects.all()
    >>> print([p.headline for p in queryset]) # Evaluate the query set.
    >>> print([p.pub_date for p in queryset]) # Re-use the cache from the evaluation.

何时查询集不会被缓存
~~~~~~~~~~~~~~~~~~~~~~


查询集不会永远缓存它们的结果。当只对查询集的部分进行求值时会检查缓存，
但是如果这个部分不在缓存中，那么接下来查询返回的记录都将不会被缓存。
这意味着使用切片或 :ref:`限制查询集 <limiting-querysets>` 将不会填充缓存。

例如，重复获取查询集对象中一个特定的索引将每次都查询数据库::

    >>> queryset = Entry.objects.all()
    >>> print(queryset[5]) # Queries the database
    >>> print(queryset[5]) # Queries the database again

然而，如果已经对全部查询集求值过，则将检查缓存::

    >>> queryset = Entry.objects.all()
    >>> [entry for entry in queryset] # Queries the database
    >>> print(queryset[5]) # Uses cache
    >>> print(queryset[5]) # Uses cache

下面是一些其它例子，它们都会使得全部的查询集被求值并填充到缓存中::

    >>> [entry for entry in queryset]
    >>> bool(queryset)
    >>> entry in queryset
    >>> list(queryset)

.. note::

    简单地打印查询集不会填充缓存。因为 ``__repr__()`` 调用只返回全部查询集的一个切片。

.. _complex-lookups-with-q:

``Q`` 复杂查询
===============

:meth:`~django.db.models.query.QuerySet.filter` 等方法中的关键字参数查询都是一起进行 "AND" 操作,
如果你需要执行更复杂的查询(例如 ``OR`` 语句), 你可以使用 :class:`Q 查询对象 <django.db.models.Q>`.

:class:`Q 对象 <django.db.models.Q>` (``django.db.models.Q``) 对象用于封装一组关键字参数。
这些关键字参数就是上文“字段查询” 中所提及的那些。

例如，下面的 ``Q ``对象封装一个 ``LIKE`` 查询::

    from django.db.models import Q
    Q(question__startswith='What')

``Q`` 对象可以使用 ``&`` 和 ``|`` 操作符组合。 当使用操作符将两个对象组合是，将生成一个新的 ``Q`` 对象。

例如，下面的语句产生一个 ``Q`` 对象，表示两个 ``"question__startswith"``  查询的“OR”::

    Q(question__startswith='Who') | Q(question__startswith='What')

它等同于下面的SQL ``WHERE`` 句子::

    WHERE question LIKE 'Who%' OR question LIKE 'What%'

你可以组合 ``&`` 和 ``|``  操作符以及使用括号进行分组来编写任意复杂的 ``Q`` 对象。
同时，``Q`` 对象可以使用 ``~`` 操作符取反，这允许组合正常的查询和取反( ``NOT`` ) 查询::

    Q(question__startswith='Who') | ~Q(pub_date__year=2005)

每个接受关键字参数的查询函数
(e.g. :meth:`~django.db.models.query.QuerySet.filter`,
:meth:`~django.db.models.query.QuerySet.exclude`,
:meth:`~django.db.models.query.QuerySet.get`) 都可以传递一个或多个 ``Q`` 对象作为位置参数。
如果一个查询函数有多个 ``Q`` 对象参数，这些参数的逻辑关系为“AND"。例如::

    Poll.objects.get(
        Q(question__startswith='Who'),
        Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6))
    )

大体上可以翻译成这个SQL::

    SELECT * from polls WHERE question LIKE 'Who%'
        AND (pub_date = '2005-05-02' OR pub_date = '2005-05-06')

查询函数可以混合使用 ``Q`` 对象和关键字参数。
所有提供查询函数的参数（关键字参数或 ``Q`` 对象）都将"AND”在一起。
但是，如果出现 ``Q`` 对象，它必须位于所有关键字参数的前面。例如::

    Poll.objects.get(
        Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6)),
        question__startswith='Who',
    )

这是一个合法的查询，等同于前面的例子; 但是::

    # INVALID QUERY
    Poll.objects.get(
        question__startswith='Who',
        Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6))
    )

这个是不合法的。

.. seealso::

    Django 单元测试中的 `OR查询示例`_ 演示了几种 ``Q`` 的用法.

    .. _OR查询示例: https://github.com/django/django/blob/master/tests/or_lookups/tests.py

对象比较
=========

使用双等号 ``==`` 比较两个对象，其实是比较两个模型主键的值。

使用上面的 ``Entry`` 示范, 下面两个表达式是等同的::

    >>> some_entry == other_entry
    >>> some_entry.id == other_entry.id

即是模型的主键名称不是 ``id`` 也没关系，这种比较总是会使用主键不叫，不论叫什么名字。
例如，如果模型的主键字段叫 ``name`` ，下面的两条语句是等同的::

    >>> some_obj == other_obj
    >>> some_obj.name == other_obj.name

.. _topics-db-queries-delete:

删除对象
=========

删除方法叫做
:meth:`~django.db.models.Model.delete` 。这个方法将立即删除对象，
返回被删除的对象的总数和每个对象类型的删除数量的字典。举例::

    >>> e.delete()
    (1, {'weblog.Entry': 1})

.. versionchanged:: 1.9

    1.9开始删除方法才有返回值。

你还可以批量删除对象。每个
:class:`~django.db.models.query.QuerySet` 都有
:meth:`~django.db.models.query.QuerySet.delete` 方法，它作用是删除
:class:`~django.db.models.query.QuerySet` 中的所有成员。

例如, 下面语句删除所有 ``pub_date`` 为2005的 ``Entry`` ::

    >>> Entry.objects.filter(pub_date__year=2005).delete()
    (5, {'webapp.Entry': 5})

上面的过程是通过SQL实现的，并不是依次调用每个对象的 ``delete()`` 方法。
如果你给模型类提供了一个自定义的 ``delete()`` 方法，并且希望删除时方法被调用。
你需要"手动"调用实例的 ``delete()`` （例如:迭代 :class:`~django.db.models.query.QuerySet` 调用每个实例的
``delete()`` 方法，使用 :class:`~django.db.models.query.QuerySet` 的 ``delete()`` 方法）。


当Django 删除一个对象时，它默认使用SQL ``ON DELETE CASCADE`` 约束 —— 换句话讲，任何有外键指向要删除对象的对象都将一起删除。
例如::

    b = Blog.objects.get(pk=1)
    # 这会删除该 Blog 和它所有的Entry对象
    b.delete()

这种级联的行为可以通过 :class:`~django.db.models.ForeignKey` 的
:attr:`~django.db.models.ForeignKey.on_delete` 参数定义。

注意， :meth:`~django.db.models.query.QuerySet.delete` 是唯一没有在 :class:`~django.db.models.Manager`
上暴露出来的
:class:`~django.db.models.query.QuerySet` 方法。
这是一个安全机制来防止你意外地请求
``Entry.objects.delete()`` , 而删除所有的条目。
如果你确实想删除所有的对象，你必须明确地请求一个完整的查询集::

    Entry.objects.all().delete()

.. _topics-db-queries-copy:

复制对象
=========

Although there is no built-in method for copying model instances, it is
possible to easily create new instance with all fields' values copied. In the
simplest case, you can just set ``pk`` to ``None``. Using our blog example::

    blog = Blog(name='My blog', tagline='Blogging is easy')
    blog.save() # blog.pk == 1

    blog.pk = None
    blog.save() # blog.pk == 2

Things get more complicated if you use inheritance. Consider a subclass of
``Blog``::

    class ThemeBlog(Blog):
        theme = models.CharField(max_length=200)

    django_blog = ThemeBlog(name='Django', tagline='Django is easy', theme='python')
    django_blog.save() # django_blog.pk == 3

Due to how inheritance works, you have to set both ``pk`` and ``id`` to None::

    django_blog.pk = None
    django_blog.id = None
    django_blog.save() # django_blog.pk == 4

This process doesn't copy relations that aren't part of the model's database
table. For example, ``Entry`` has a ``ManyToManyField`` to ``Author``. After
duplicating an entry, you must set the many-to-many relations for the new
entry::

    entry = Entry.objects.all()[0] # some previous entry
    old_authors = entry.authors.all()
    entry.pk = None
    entry.save()
    entry.authors.set(old_authors)

For a ``OneToOneField``, you must duplicate the related object and assign it
to the new object's field to avoid violating the one-to-one unique constraint.
For example, assuming ``entry`` is already duplicated as above::

    detail = EntryDetail.objects.all()[0]
    detail.pk = None
    detail.entry = entry
    detail.save()

.. _topics-db-queries-update:

Updating multiple objects at once
=================================

Sometimes you want to set a field to a particular value for all the objects in
a :class:`~django.db.models.query.QuerySet`. You can do this with the
:meth:`~django.db.models.query.QuerySet.update` method. For example::

    # Update all the headlines with pub_date in 2007.
    Entry.objects.filter(pub_date__year=2007).update(headline='Everything is the same')

You can only set non-relation fields and :class:`~django.db.models.ForeignKey`
fields using this method. To update a non-relation field, provide the new value
as a constant. To update :class:`~django.db.models.ForeignKey` fields, set the
new value to be the new model instance you want to point to. For example::

    >>> b = Blog.objects.get(pk=1)

    # Change every Entry so that it belongs to this Blog.
    >>> Entry.objects.all().update(blog=b)

The ``update()`` method is applied instantly and returns the number of rows
matched by the query (which may not be equal to the number of rows updated if
some rows already have the new value). The only restriction on the
:class:`~django.db.models.query.QuerySet` being updated is that it can only
access one database table: the model's main table. You can filter based on
related fields, but you can only update columns in the model's main
table. Example::

    >>> b = Blog.objects.get(pk=1)

    # Update all the headlines belonging to this Blog.
    >>> Entry.objects.select_related().filter(blog=b).update(headline='Everything is the same')

Be aware that the ``update()`` method is converted directly to an SQL
statement. It is a bulk operation for direct updates. It doesn't run any
:meth:`~django.db.models.Model.save` methods on your models, or emit the
``pre_save`` or ``post_save`` signals (which are a consequence of calling
:meth:`~django.db.models.Model.save`), or honor the
:attr:`~django.db.models.DateField.auto_now` field option.
If you want to save every item in a :class:`~django.db.models.query.QuerySet`
and make sure that the :meth:`~django.db.models.Model.save` method is called on
each instance, you don't need any special function to handle that. Just loop
over them and call :meth:`~django.db.models.Model.save`::

    for item in my_queryset:
        item.save()

Calls to update can also use :class:`F expressions <django.db.models.F>` to
update one field based on the value of another field in the model. This is
especially useful for incrementing counters based upon their current value. For
example, to increment the pingback count for every entry in the blog::

    >>> Entry.objects.all().update(n_pingbacks=F('n_pingbacks') + 1)

However, unlike ``F()`` objects in filter and exclude clauses, you can't
introduce joins when you use ``F()`` objects in an update -- you can only
reference fields local to the model being updated. If you attempt to introduce
a join with an ``F()`` object, a ``FieldError`` will be raised::

    # This will raise a FieldError
    >>> Entry.objects.update(headline=F('blog__name'))

.. _topics-db-queries-related:

Related objects
===============

When you define a relationship in a model (i.e., a
:class:`~django.db.models.ForeignKey`,
:class:`~django.db.models.OneToOneField`, or
:class:`~django.db.models.ManyToManyField`), instances of that model will have
a convenient API to access the related object(s).

Using the models at the top of this page, for example, an ``Entry`` object ``e``
can get its associated ``Blog`` object by accessing the ``blog`` attribute:
``e.blog``.

(Behind the scenes, this functionality is implemented by Python descriptors_.
This shouldn't really matter to you, but we point it out here for the curious.)

Django also creates API accessors for the "other" side of the relationship --
the link from the related model to the model that defines the relationship.
For example, a ``Blog`` object ``b`` has access to a list of all related
``Entry`` objects via the ``entry_set`` attribute: ``b.entry_set.all()``.

All examples in this section use the sample ``Blog``, ``Author`` and ``Entry``
models defined at the top of this page.

.. _descriptors: http://users.rcn.com/python/download/Descriptor.htm

One-to-many relationships
-------------------------

Forward
~~~~~~~

If a model has a :class:`~django.db.models.ForeignKey`, instances of that model
will have access to the related (foreign) object via a simple attribute of the
model.

Example::

    >>> e = Entry.objects.get(id=2)
    >>> e.blog # Returns the related Blog object.

You can get and set via a foreign-key attribute. As you may expect, changes to
the foreign key aren't saved to the database until you call
:meth:`~django.db.models.Model.save`. Example::

    >>> e = Entry.objects.get(id=2)
    >>> e.blog = some_blog
    >>> e.save()

If a :class:`~django.db.models.ForeignKey` field has ``null=True`` set (i.e.,
it allows ``NULL`` values), you can assign ``None`` to remove the relation.
Example::

    >>> e = Entry.objects.get(id=2)
    >>> e.blog = None
    >>> e.save() # "UPDATE blog_entry SET blog_id = NULL ...;"

Forward access to one-to-many relationships is cached the first time the
related object is accessed. Subsequent accesses to the foreign key on the same
object instance are cached. Example::

    >>> e = Entry.objects.get(id=2)
    >>> print(e.blog)  # Hits the database to retrieve the associated Blog.
    >>> print(e.blog)  # Doesn't hit the database; uses cached version.

Note that the :meth:`~django.db.models.query.QuerySet.select_related`
:class:`~django.db.models.query.QuerySet` method recursively prepopulates the
cache of all one-to-many relationships ahead of time. Example::

    >>> e = Entry.objects.select_related().get(id=2)
    >>> print(e.blog)  # Doesn't hit the database; uses cached version.
    >>> print(e.blog)  # Doesn't hit the database; uses cached version.

.. _backwards-related-objects:

Following relationships "backward"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If a model has a :class:`~django.db.models.ForeignKey`, instances of the
foreign-key model will have access to a :class:`~django.db.models.Manager` that
returns all instances of the first model. By default, this
:class:`~django.db.models.Manager` is named ``FOO_set``, where ``FOO`` is the
source model name, lowercased. This :class:`~django.db.models.Manager` returns
``QuerySets``, which can be filtered and manipulated as described in the
"Retrieving objects" section above.

Example::

    >>> b = Blog.objects.get(id=1)
    >>> b.entry_set.all() # Returns all Entry objects related to Blog.

    # b.entry_set is a Manager that returns QuerySets.
    >>> b.entry_set.filter(headline__contains='Lennon')
    >>> b.entry_set.count()

You can override the ``FOO_set`` name by setting the
:attr:`~django.db.models.ForeignKey.related_name` parameter in the
:class:`~django.db.models.ForeignKey` definition. For example, if the ``Entry``
model was altered to ``blog = ForeignKey(Blog, on_delete=models.CASCADE,
related_name='entries')``, the above example code would look like this::

    >>> b = Blog.objects.get(id=1)
    >>> b.entries.all() # Returns all Entry objects related to Blog.

    # b.entries is a Manager that returns QuerySets.
    >>> b.entries.filter(headline__contains='Lennon')
    >>> b.entries.count()

.. _using-custom-reverse-manager:

Using a custom reverse manager
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default the :class:`~django.db.models.fields.related.RelatedManager` used
for reverse relations is a subclass of the :ref:`default manager <manager-names>`
for that model. If you would like to specify a different manager for a given
query you can use the following syntax::

    from django.db import models

    class Entry(models.Model):
        #...
        objects = models.Manager()  # Default Manager
        entries = EntryManager()    # Custom Manager

    b = Blog.objects.get(id=1)
    b.entry_set(manager='entries').all()

If ``EntryManager`` performed default filtering in its ``get_queryset()``
method, that filtering would apply to the ``all()`` call.

Of course, specifying a custom reverse manager also enables you to call its
custom methods::

    b.entry_set(manager='entries').is_published()

Additional methods to handle related objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In addition to the :class:`~django.db.models.query.QuerySet` methods defined in
"Retrieving objects" above, the :class:`~django.db.models.ForeignKey`
:class:`~django.db.models.Manager` has additional methods used to handle the
set of related objects. A synopsis of each is below, and complete details can
be found in the :doc:`related objects reference </ref/models/relations>`.

``add(obj1, obj2, ...)``
    Adds the specified model objects to the related object set.

``create(**kwargs)``
    Creates a new object, saves it and puts it in the related object set.
    Returns the newly created object.

``remove(obj1, obj2, ...)``
    Removes the specified model objects from the related object set.

``clear()``
    Removes all objects from the related object set.

``set(objs)``
    Replace the set of related objects.

To assign the members of a related set, use the ``set()`` method with an
iterable of object instances or a list of primary key values. For example::

    b = Blog.objects.get(id=1)
    b.entry_set.set([e1, e2])

In this example, ``e1`` and ``e2`` can be full Entry instances, or integer
primary key values.

If the ``clear()`` method is available, any pre-existing objects will be
removed from the ``entry_set`` before all objects in the iterable (in this
case, a list) are added to the set. If the ``clear()`` method is *not*
available, all objects in the iterable will be added without removing any
existing elements.

Each "reverse" operation described in this section has an immediate effect on
the database. Every addition, creation and deletion is immediately and
automatically saved to the database.

.. _m2m-reverse-relationships:

Many-to-many relationships
--------------------------

Both ends of a many-to-many relationship get automatic API access to the other
end. The API works just as a "backward" one-to-many relationship, above.

The only difference is in the attribute naming: The model that defines the
:class:`~django.db.models.ManyToManyField` uses the attribute name of that
field itself, whereas the "reverse" model uses the lowercased model name of the
original model, plus ``'_set'`` (just like reverse one-to-many relationships).

An example makes this easier to understand::

    e = Entry.objects.get(id=3)
    e.authors.all() # Returns all Author objects for this Entry.
    e.authors.count()
    e.authors.filter(name__contains='John')

    a = Author.objects.get(id=5)
    a.entry_set.all() # Returns all Entry objects for this Author.

Like :class:`~django.db.models.ForeignKey`,
:class:`~django.db.models.ManyToManyField` can specify
:attr:`~django.db.models.ManyToManyField.related_name`. In the above example,
if the :class:`~django.db.models.ManyToManyField` in ``Entry`` had specified
``related_name='entries'``, then each ``Author`` instance would have an
``entries`` attribute instead of ``entry_set``.

One-to-one relationships
------------------------

One-to-one relationships are very similar to many-to-one relationships. If you
define a :class:`~django.db.models.OneToOneField` on your model, instances of
that model will have access to the related object via a simple attribute of the
model.

For example::

    class EntryDetail(models.Model):
        entry = models.OneToOneField(Entry, on_delete=models.CASCADE)
        details = models.TextField()

    ed = EntryDetail.objects.get(id=2)
    ed.entry # Returns the related Entry object.

The difference comes in "reverse" queries. The related model in a one-to-one
relationship also has access to a :class:`~django.db.models.Manager` object, but
that :class:`~django.db.models.Manager` represents a single object, rather than
a collection of objects::

    e = Entry.objects.get(id=2)
    e.entrydetail # returns the related EntryDetail object

If no object has been assigned to this relationship, Django will raise
a ``DoesNotExist`` exception.

Instances can be assigned to the reverse relationship in the same way as
you would assign the forward relationship::

    e.entrydetail = ed

How are the backward relationships possible?
--------------------------------------------

Other object-relational mappers require you to define relationships on both
sides. The Django developers believe this is a violation of the DRY (Don't
Repeat Yourself) principle, so Django only requires you to define the
relationship on one end.

But how is this possible, given that a model class doesn't know which other
model classes are related to it until those other model classes are loaded?

The answer lies in the :data:`app registry <django.apps.apps>`. When Django
starts, it imports each application listed in :setting:`INSTALLED_APPS`, and
then the ``models`` module inside each application. Whenever a new model class
is created, Django adds backward-relationships to any related models. If the
related models haven't been imported yet, Django keeps tracks of the
relationships and adds them when the related models eventually are imported.

For this reason, it's particularly important that all the models you're using
be defined in applications listed in :setting:`INSTALLED_APPS`. Otherwise,
backwards relations may not work properly.

Queries over related objects
----------------------------

Queries involving related objects follow the same rules as queries involving
normal value fields. When specifying the value for a query to match, you may
use either an object instance itself, or the primary key value for the object.

For example, if you have a Blog object ``b`` with ``id=5``, the following
three queries would be identical::

    Entry.objects.filter(blog=b) # Query using object instance
    Entry.objects.filter(blog=b.id) # Query using id from instance
    Entry.objects.filter(blog=5) # Query using id directly

Falling back to raw SQL
=======================

If you find yourself needing to write an SQL query that is too complex for
Django's database-mapper to handle, you can fall back on writing SQL by hand.
Django has a couple of options for writing raw SQL queries; see
:doc:`/topics/db/sql`.

Finally, it's important to note that the Django database layer is merely an
interface to your database. You can access your database via other tools,
programming languages or database frameworks; there's nothing Django-specific
about your database.
