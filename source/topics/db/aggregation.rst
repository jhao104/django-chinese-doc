======
聚合
======

.. currentmodule:: django.db.models

:doc:`Django 数据库抽象API </topics/db/queries>` 描述了使用Django 查询来增删查改单个对象的方法。
然而，有时候你要获取的值需要根据一组对象聚合后才能得到。
这个主题指南描述了如何使用Django的查询来生成和返回聚合值的方法。

整篇指南我们都将引用以下模型。这些模型用来记录多个网上书店的库存。

.. _queryset-model-example:

.. code-block:: python

    from django.db import models

    class Author(models.Model):
        name = models.CharField(max_length=100)
        age = models.IntegerField()

    class Publisher(models.Model):
        name = models.CharField(max_length=300)
        num_awards = models.IntegerField()

    class Book(models.Model):
        name = models.CharField(max_length=300)
        pages = models.IntegerField()
        price = models.DecimalField(max_digits=10, decimal_places=2)
        rating = models.FloatField()
        authors = models.ManyToManyField(Author)
        publisher = models.ForeignKey(Publisher)
        pubdate = models.DateField()

    class Store(models.Model):
        name = models.CharField(max_length=300)
        books = models.ManyToManyField(Book)
        registered_users = models.PositiveIntegerField()

速查表
=========

下面是在上面的模型上如何执行常见的聚合查询:

.. code-block:: python

    # book 总数.
    >>> Book.objects.count()
    2452

    # publisher=BaloneyPress的book总数.
    >>> Book.objects.filter(publisher__name='BaloneyPress').count()
    73

    # 所有book的平均价格.
    >>> from django.db.models import Avg
    >>> Book.objects.all().aggregate(Avg('price'))
    {'price__avg': 34.35}

    # 所有book的最高价格
    >>> from django.db.models import Max
    >>> Book.objects.all().aggregate(Max('price'))
    {'price__max': Decimal('81.20')}

    # 每页均价
    >>> from django.db.models import F, FloatField, Sum
    >>> Book.objects.all().aggregate(
    ...    price_per_page=Sum(F('price')/F('pages'), output_field=FloatField()))
    {'price_per_page': 0.4470664529184653}

    # 下面的所有查询都涉及到遍历 Book<->Publisher
    # foreign key relationship backwards.

    # Each publisher, each with a count of books as a "num_books" attribute.
    >>> from django.db.models import Count
    >>> pubs = Publisher.objects.annotate(num_books=Count('book'))
    >>> pubs
    <QuerySet [<Publisher: BaloneyPress>, <Publisher: SalamiPress>, ...]>
    >>> pubs[0].num_books
    73

    # The top 5 publishers, in order by number of books.
    >>> pubs = Publisher.objects.annotate(num_books=Count('book')).order_by('-num_books')[:5]
    >>> pubs[0].num_books
    1323

``QuerySet`` 聚合
===================

Django提供了两种生成聚合的方法。第一种方法是从整个 ``QuerySet`` 生成统计值。
比如，你想要计算所有在售书的平均价钱。Django的查询语法提供了一种方式描述所有图书的集合。::

    >>> Book.objects.all()

我们需要在 ``QuerySet`` 对象上计算出汇总的值。这可以通过在  ``QuerySet`` 后面添加 ``aggregate()`` 子句来实现::

    >>> from django.db.models import Avg
    >>> Book.objects.all().aggregate(Avg('price'))
    {'price__avg': 34.35}

其实 ``all()`` 在这里可以省略，简化为::

    >>> Book.objects.aggregate(Avg('price'))
    {'price__avg': 34.35}

``aggregate()`` 子句的参数是想要计算的聚合值，在这个例子中，是 ``Book`` 模型中
``price`` 字段的平均值。 :ref:`查询集参考 <aggregation-functions>` 有所有的聚合函数。

``aggregate()`` 是 ``QuerySet``  的一个终止子句，意思是说，它返回一个包含键值对的字典。
键的名称是聚合值的标识符，值是计算出来的聚合值。键的名称是按照字段和聚合函数的名称自动生成出来的。
如果你想要为聚合值指定一个名称，可以在聚合子句中指定::

    >>> Book.objects.aggregate(average_price=Avg('price'))
    {'average_price': 34.35}

如果你想要计算多个聚合，你可以在 ``aggregate()`` 子句作为参数添加。比如，
如果你也想知道所有图书价格的最大值和最小值，可以这样查询::

    >>> from django.db.models import Avg, Max, Min
    >>> Book.objects.aggregate(Avg('price'), Max('price'), Min('price'))
    {'price__avg': 34.35, 'price__max': Decimal('81.20'), 'price__min': Decimal('12.99')}

``QuerySet`` 逐个对象的聚合
============================

生成汇总值的第二种方法，是为 ``QuerySet`` 中每一个对象都生成一个独立的汇总值。
比如，你可能想知道每一本书有多少作者参与。每本书和作者是多对多的关系。
我们需要汇总 ``QuerySet`` 中每本书的这种关系。

逐个对象的汇总结果可以由 ``annotate()`` 子句生成。
当 ``annotate()`` 子句被指定之后， ``QuerySet`` 中的每个对象都会被注上特定的值。

annotate(注解)的语法都和 ``aggregate()`` 子句相同。 ``annotate()`` 的每个参数都描述了将要被计算的聚合值。

比如，给图书添加作者数量的注解:

.. code-block:: python

    # Build an annotated queryset
    >>> from django.db.models import Count
    >>> q = Book.objects.annotate(Count('authors'))
    # 查询queryset中的第一个对象
    >>> q[0]
    <Book: The Definitive Guide to Django>
    >>> q[0].authors__count
    2
    # 查询queryset中的第二个对象
    >>> q[1]
    <Book: Practical Django Projects>
    >>> q[1].authors__count
    1

和 ``aggregate()`` 一样，注解的名称也根据聚合函数的名称和聚合字段的名称自动生成。
同样可以在指定注释时，通过提供别名来覆盖此默认名称::

    >>> q = Book.objects.annotate(num_authors=Count('authors'))
    >>> q[0].num_authors
    2
    >>> q[1].num_authors
    1

和 ``aggregate()`` 不同的是， ``annotate()``  *不是* 结束子句。它返回的结果是一个 ``QuerySet`` 。
这个 ``QuerySet`` 可以使用任何 ``QuerySet`` 方法进行再次操作。包括 ``filter()``,
``order_by()``, 甚至是再次使用 ``annotate()`` 。

.. _combining-multiple-aggregations:

组合多个聚合
-------------

组合多个 ``annotate()`` 会产生
`错误的结果 <https://code.djangoproject.com/ticket/10060>`_ ，因为使用的是join而不是子查询:

    >>> book = Book.objects.first()
    >>> book.authors.count()
    2
    >>> book.store_set.count()
    3
    >>> q = Book.objects.annotate(Count('authors'), Count('store'))
    >>> q[0].authors__count
    6
    >>> q[0].store__count
    6

对于大多数的聚合, 都有这个没法避免的问题, 但是
:class:`~django.db.models.Count` 聚合带有一个 ``distinct`` 参数可以避免:

    >>> q = Book.objects.annotate(Count('authors', distinct=True), Count('store', distinct=True))
    >>> q[0].authors__count
    2
    >>> q[0].store__count
    3

.. admonition:: 如果不明白，可以查看SQL query!

    如果想知道查询具体做了什么，请查看 ``QuerySet`` 的 ``query`` 属性。

Join和聚合
===========

到目前为止，都是被查询的模型相关字段的聚合。然而，有时可能聚合的值是与所查询模型相关的模型。

在聚合函数中指定聚合字段时，可以使用 :ref:`双下划线
<field-lookups-intro>` 指定关联关系，Django会自动读取关联表，计算关联对象的聚合。

例如，要查找每个商店提供的图书的价格范围，可以使用注解::

    >>> from django.db.models import Max, Min
    >>> Store.objects.annotate(min_price=Min('books__price'), max_price=Max('books__price'))

这段代码告诉 Django 获取 ``Store`` 模型, join (通过多对多关系) 到 ``Book`` 模型，然后对每本书的价格进行聚合，得出最小值和最大值。

同样，这也适用于 ``aggregate()`` 子句。
如果你想知道所有书店中最便宜的书和最贵的书价格分别是多少::

    >>> Store.objects.aggregate(min_price=Min('books__price'), max_price=Max('books__price'))

Join链可以按需求一直延伸。 例如，想得到所有作者当中最小的年龄，可以这样写::

    >>> Store.objects.aggregate(youngest_age=Min('books__authors__age'))

遵循反向关系
------------

和 :ref:`lookups-that-span-relationships` 类似，作用在所查询模型的关联模型或者字段上的聚合和注解可以遍历"反向"关系。
这里也使用了相关模型的小写名称和双下划线。

例如，查询所有出版商，并注解它们一共出了多少本书（注意是使用 ``book`` 指定 ``Publisher`` -> ``Book`` 的反向外键关系）::

    >>> from django.db.models import Count, Min, Sum, Avg
    >>> Publisher.objects.annotate(Count('book'))

(``QuerySet``结果中，每个 ``Publisher`` 都会包含一个额外的属性 ``book__count`` 。)

也可以按照每个出版商，查询所有图书中最旧的那本::

    >>> Publisher.objects.aggregate(oldest_pubdate=Min('book__pubdate'))

(返回的字典会包含一个键叫做 ``'oldest_pubdate'`` 。如果没有指定这样的别名，它将是 ``'book__pubdate__min'``.)

这不仅仅是在外键关系上是这样。多对多关系也是如此。
例如，查询每个作者，注解上它写的所有书（以及合著的书）一共有多少页（
注意如何使用 ``'book'`` 来指定 ``Author`` -> ``Book`` 的多对多的反向关系）::

    >>> Author.objects.annotate(total_pages=Sum('book__pages'))

(返回的 ``QuerySet`` 中，每个 ``Author`` 都有一个额外属性
叫做 ``total_pages`` 。如果没有指定这样的别名，默认将是
``book__pages__sum`` 。)

或者查询所有图书的平均评分，这些图书由存档过的作者所写::

    >>> Author.objects.aggregate(average_rating=Avg('book__rating'))

(返回的字典会包含一个叫做  ``'average_rating'`` 的键。 如果没有指定这样的别名，它将是 ``'book__rating__avg'``.)

聚合 ``QuerySet`` 子句
===========================

``filter()`` and ``exclude()``
------------------------------

聚合也可以在过滤器中使用。
作用于普通模型字段的任何 ``filter()`` (或 ``exclude()`` ) 都会对聚合涉及的对象进行限制。

使用 ``annotate()`` 子句时, 筛选器具有约束注释被计算对象的作用。
例如，计算每本以 "Django" 为书名开头的图书的作者的总数::

    >>> from django.db.models import Count, Avg
    >>> Book.objects.filter(name__startswith="Django").annotate(num_authors=Count('authors'))

使用 ``aggregate()`` 子句, 筛选器具有约束聚合被计算对象的作用。
例如，计算所有以 "Django" 为书名开头的图书平均价格::

    >>> Book.objects.filter(name__startswith="Django").aggregate(Avg('price'))

对注解过滤
~~~~~~~~~~~

注解的值也可以被过滤。 而注解的别名也可以和模型字段一样，在
``filter()`` 和 ``exclude()`` 子句中使用。

例如，要得到不止一个作者的图书，可以用::

    >>> Book.objects.annotate(num_authors=Count('authors')).filter(num_authors__gt=1)

这个查询首先计算注解结果，然后再生成一个作用于注解上的过滤器。

``annotate()`` 和 ``filter()`` 的顺序
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在写一个包含 ``annotate()`` 和
``filter()`` 的复杂查询时，要特别注意作用于 ``QuerySet`` 子句的顺序。

使用 ``annotate()`` 子句作用于某个查询时,要根据查询的状态才能得出注解值，而状态由
``annotate()``的位置决定，所以 ``filter()`` 和 ``annotate()`` 不能随意交换位置。

比如，有以下数据:

* Publisher A 有两本 book，ratings 分别为4和5.
* Publisher B 有两本 book，ratings 分别为1和4.
* Publisher C 有一本 book，rating 为1.

以 ``Count`` 聚合为例::

    >>> a, b = Publisher.objects.annotate(num_books=Count('book', distinct=True)).filter(book__rating__gt=3.0)
    >>> a, a.num_books
    (<Publisher: A>, 2)
    >>> b, b.num_books
    (<Publisher: B>, 2)

    >>> a, b = Publisher.objects.filter(book__rating__gt=3.0).annotate(num_books=Count('book'))
    >>> a, a.num_books
    (<Publisher: A>, 2)
    >>> b, b.num_books
    (<Publisher: B>, 1)

两个查询都返回至少有一本书的rating大于3的publisher列表，因此publisher C不在列表中。

在第一个查询中, 注解先于过滤, 因此过滤不会影响注解。设置 ``distinct=True`` 避免 :ref:`query bug <combining-multiple-aggregations>` 。

第二个查询先计算每个publisher中rating的值超过3.0的图书。筛选器先于注释，因此筛选器在计算注释时已经约束了对象。

下面是另一个是 ``Avg`` 聚合的例子::

    >>> a, b = Publisher.objects.annotate(avg_rating=Avg('book__rating')).filter(book__rating__gt=3.0)
    >>> a, a.avg_rating
    (<Publisher: A>, 4.5)  # (5+4)/2
    >>> b, b.avg_rating
    (<Publisher: B>, 2.5)  # (1+4)/2

    >>> a, b = Publisher.objects.filter(book__rating__gt=3.0).annotate(avg_rating=Avg('book__rating'))
    >>> a, a.avg_rating
    (<Publisher: A>, 4.5)  # (5+4)/2
    >>> b, b.avg_rating
    (<Publisher: B>, 4.0)  # 4/1 (book with rating 1 excluded)

第一个查询是计算至少有一本书的rating超过3.0的publisher的所有图书的平均rating，
第二个查询要计算publisher的rating超过3.0的书的平均rating。

这种情况，很难直观地了解ORM如何将复杂的queryset翻译成SQL查询，
因此在不清楚时，可以使用 ``str(queryset.query)`` 来检查SQL，并且多一点测试。

``order_by()``
--------------

注解可以用来做为排序项。当你定义 ``order_by()`` 子句时，
也可以引用定义在 ``annotate()`` 中的任何别名。

例如，根据图书作者数量的多少对 ``QuerySet`` 进行排序::

    >>> Book.objects.annotate(num_authors=Count('authors')).order_by('num_authors')

``values()``
------------

通常，注解值会添加到每个对象上- 一个被注解的
``QuerySet`` 会为初始 ``QuerySet`` 的每个对象返回一个结果集。
但是，如果使用了 ``values()`` 子句，它就会限制结果中列的范围，对注解赋值的方法就会完全不同。不是在原始的
``QuerySet`` 返回结果中对每个对象中添加注解, 而是根据定义在 ``values()``
子句中的字段组合先对结果进行分组，再根据每个分组算出注解值， 这个注解值是根据分组中所有的成员计算而得的。

例如，查询出每个作者所写的书的平均评分：:

    >>> Author.objects.annotate(average_rating=Avg('book__rating'))

这段代码返回的是数据库中所有的作者以及他们所著图书的平均评分。

但是如果你使用了 ``values()`` 子句，结果是完全不同的::

    >>> Author.objects.values('name').annotate(average_rating=Avg('book__rating'))

在这个例子中，作者会按名称分组，所以你只能得到某个唯一的作者分组的注解值。
也就是说如果你有两个作者同名，那么他们原本各自的查询结果将被合并到同一个结果中；两个作者的所有评分将被计算为一个平均分。

``annotate()`` 和 ``values()`` 的顺序
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

和使用 ``filter()`` 一样, 作用于某个查询的 ``annotate()`` 和
``values()`` 子句的顺序非常重要。如果
``values()`` 子句在 ``annotate()`` 之前, 就会根据
``values()`` 子句产生的分组来计算注解。

但是，如果  ``annotate()`` 子句在 ``values()`` 之前,
就会根据整个查询集生成注解。这种情况下，
``values()`` 子句只能限制输出的字段。

举个例子，如果互换了上个例子中 ``values()`` 和 ``annotate()`` 子句的顺序::

    >>> Author.objects.annotate(average_rating=Avg('book__rating')).values('name', 'average_rating')

这段代码将给每个作者添加一个唯一的字段，但只有作者名称和 ``average_rating`` 注解会返回在输出结果中。

这里 ``average_rating`` 显式地包含在返回的列表当中。这也正是因为
 ``values()`` 和 ``annotate()`` 子句的顺序问题。

如果 ``values()`` 子句在 ``annotate()`` 之前, 注解会被自动添加到结果集中。 但是，如果 ``values()``
子句作用于 ``annotate()`` 之后, 您需要在 ``values`` 中显式地包含聚合列。

.. _aggregation-ordering-interaction:

默认排序和 ``order_by()``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

查询集中的 ``order_by()`` 部分(或是模型中默认定义的排序项) 会在选择输出数据时被用到。
即使这些字段没有在 ``values()`` 中指定也会被用到。
这些字段用来组合“相似”结果，它们可以使相似的结果行看起来是独立的。尤其是在计数的时候。

通过例子中的方法，假设有一个这样的模型::

    from django.db import models

    class Item(models.Model):
        name = models.CharField(max_length=10)
        data = models.IntegerField()

        class Meta:
            ordering = ["name"]

关键的部分就是在模型默认排序项中设置的 ``name`` 字段。
如果你想知道每个非重复的  ``data`` 值出现的次数。可以这样写::

    # Warning: not quite correct!
    Item.objects.values("data").annotate(Count("id"))

...这部分代码的用意是想通过它们相同的 ``data`` 值来分组 ``Item`` 对象。然后在每个分组中计算
``id`` 总数。但是上面那样做是行不通的，这是因为默认排序项中的 ``name`` 也是一个分组项。
所以这个查询会根据非重复的 ``(data, name)`` 进行分组。想要得到正确的结果应该这样写::

    Item.objects.values("data").annotate(Count("id")).order_by()

...这样就清空了查询中的所有排序项。你也可以在 ``order_by()`` 中使用“data”。这样结果还是一样的。

这个行为与查询集文档中提到的
:meth:`~django.db.models.query.QuerySet.distinct` 一样，
而且生成规则也一样：
通常情况下，如果希望在结果中有额外的列，就可以清除排序，或者确保它只有在 ``values()`` 中调用的字段。

.. note::
    您可能会问为什么Django没有删除多余的列。主要原因就是要保证使用
    ``distinct()`` 和其他方法的一致性。Django **从不** 从不删除您指定的排序约束
    (不会改动那些方法的行为，因为这会违背 :doc:`/misc/api-stability` 原则)。

聚合注解
---------

你也可以在注解的结果上生成聚合。当你定义一个 ``aggregate()`` 子句时，
可以使用 ``annotate()`` 子句中定义的任何别名。

例如，如果你想计算平均每本书有几个作者，可以注解每本图书的作者总数，然后再聚合作者总数::

    >>> from django.db.models import Count, Avg
    >>> Book.objects.annotate(num_authors=Count('authors')).aggregate(Avg('num_authors'))
    {'num_authors__avg': 1.66}
