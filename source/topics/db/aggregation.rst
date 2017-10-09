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

This doesn't apply just to foreign keys. It also works with many-to-many
relations. For example, we can ask for every author, annotated with the total
number of pages considering all the books the author has (co-)authored (note how we
use ``'book'`` to specify the ``Author`` -> ``Book`` reverse many-to-many hop)::

    >>> Author.objects.annotate(total_pages=Sum('book__pages'))

(Every ``Author`` in the resulting ``QuerySet`` will have an extra attribute
called ``total_pages``. If no such alias were specified, it would be the rather
long ``book__pages__sum``.)

Or ask for the average rating of all the books written by author(s) we have on
file::

    >>> Author.objects.aggregate(average_rating=Avg('book__rating'))

(The resulting dictionary will have a key called ``'average_rating'``. If no
such alias were specified, it would be the rather long ``'book__rating__avg'``.)

Aggregations and other ``QuerySet`` clauses
===========================================

``filter()`` and ``exclude()``
------------------------------

Aggregates can also participate in filters. Any ``filter()`` (or
``exclude()``) applied to normal model fields will have the effect of
constraining the objects that are considered for aggregation.

When used with an ``annotate()`` clause, a filter has the effect of
constraining the objects for which an annotation is calculated. For example,
you can generate an annotated list of all books that have a title starting
with "Django" using the query::

    >>> from django.db.models import Count, Avg
    >>> Book.objects.filter(name__startswith="Django").annotate(num_authors=Count('authors'))

When used with an ``aggregate()`` clause, a filter has the effect of
constraining the objects over which the aggregate is calculated.
For example, you can generate the average price of all books with a
title that starts with "Django" using the query::

    >>> Book.objects.filter(name__startswith="Django").aggregate(Avg('price'))

Filtering on annotations
~~~~~~~~~~~~~~~~~~~~~~~~

Annotated values can also be filtered. The alias for the annotation can be
used in ``filter()`` and ``exclude()`` clauses in the same way as any other
model field.

For example, to generate a list of books that have more than one author,
you can issue the query::

    >>> Book.objects.annotate(num_authors=Count('authors')).filter(num_authors__gt=1)

This query generates an annotated result set, and then generates a filter
based upon that annotation.

Order of ``annotate()`` and ``filter()`` clauses
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When developing a complex query that involves both ``annotate()`` and
``filter()`` clauses, pay particular attention to the order in which the
clauses are applied to the ``QuerySet``.

When an ``annotate()`` clause is applied to a query, the annotation is computed
over the state of the query up to the point where the annotation is requested.
The practical implication of this is that ``filter()`` and ``annotate()`` are
not commutative operations.

Given:

* Publisher A has two books with ratings 4 and 5.
* Publisher B has two books with ratings 1 and 4.
* Publisher C has one book with rating 1.

Here's an example with the ``Count`` aggregate::

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

Both queries return a list of publishers that have at least one book with a
rating exceeding 3.0, hence publisher C is excluded.

In the first query, the annotation precedes the filter, so the filter has no
effect on the annotation. ``distinct=True`` is required to avoid a :ref:`query
bug <combining-multiple-aggregations>`.

The second query counts the number of books that have a rating exceeding 3.0
for each publisher. The filter precedes the annotation, so the filter
constrains the objects considered when calculating the annotation.

Here's another example with the ``Avg`` aggregate::

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

The first query asks for the average rating of all a publisher's books for
publisher's that have at least one book with a rating exceeding 3.0. The second
query asks for the average of a publisher's book's ratings for only those
ratings exceeding 3.0.

It's difficult to intuit how the ORM will translate complex querysets into SQL
queries so when in doubt, inspect the SQL with ``str(queryset.query)`` and
write plenty of tests.

``order_by()``
--------------

Annotations can be used as a basis for ordering. When you
define an ``order_by()`` clause, the aggregates you provide can reference
any alias defined as part of an ``annotate()`` clause in the query.

For example, to order a ``QuerySet`` of books by the number of authors
that have contributed to the book, you could use the following query::

    >>> Book.objects.annotate(num_authors=Count('authors')).order_by('num_authors')

``values()``
------------

Ordinarily, annotations are generated on a per-object basis - an annotated
``QuerySet`` will return one result for each object in the original
``QuerySet``. However, when a ``values()`` clause is used to constrain the
columns that are returned in the result set, the method for evaluating
annotations is slightly different. Instead of returning an annotated result
for each result in the original ``QuerySet``, the original results are
grouped according to the unique combinations of the fields specified in the
``values()`` clause. An annotation is then provided for each unique group;
the annotation is computed over all members of the group.

For example, consider an author query that attempts to find out the average
rating of books written by each author:

    >>> Author.objects.annotate(average_rating=Avg('book__rating'))

This will return one result for each author in the database, annotated with
their average book rating.

However, the result will be slightly different if you use a ``values()`` clause::

    >>> Author.objects.values('name').annotate(average_rating=Avg('book__rating'))

In this example, the authors will be grouped by name, so you will only get
an annotated result for each *unique* author name. This means if you have
two authors with the same name, their results will be merged into a single
result in the output of the query; the average will be computed as the
average over the books written by both authors.

Order of ``annotate()`` and ``values()`` clauses
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As with the ``filter()`` clause, the order in which ``annotate()`` and
``values()`` clauses are applied to a query is significant. If the
``values()`` clause precedes the ``annotate()``, the annotation will be
computed using the grouping described by the ``values()`` clause.

However, if the ``annotate()`` clause precedes the ``values()`` clause,
the annotations will be generated over the entire query set. In this case,
the ``values()`` clause only constrains the fields that are generated on
output.

For example, if we reverse the order of the ``values()`` and ``annotate()``
clause from our previous example::

    >>> Author.objects.annotate(average_rating=Avg('book__rating')).values('name', 'average_rating')

This will now yield one unique result for each author; however, only
the author's name and the ``average_rating`` annotation will be returned
in the output data.

You should also note that ``average_rating`` has been explicitly included
in the list of values to be returned. This is required because of the
ordering of the ``values()`` and ``annotate()`` clause.

If the ``values()`` clause precedes the ``annotate()`` clause, any annotations
will be automatically added to the result set. However, if the ``values()``
clause is applied after the ``annotate()`` clause, you need to explicitly
include the aggregate column.

.. _aggregation-ordering-interaction:

Interaction with default ordering or ``order_by()``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Fields that are mentioned in the ``order_by()`` part of a queryset (or which
are used in the default ordering on a model) are used when selecting the
output data, even if they are not otherwise specified in the ``values()``
call. These extra fields are used to group "like" results together and they
can make otherwise identical result rows appear to be separate. This shows up,
particularly, when counting things.

By way of example, suppose you have a model like this::

    from django.db import models

    class Item(models.Model):
        name = models.CharField(max_length=10)
        data = models.IntegerField()

        class Meta:
            ordering = ["name"]

The important part here is the default ordering on the ``name`` field. If you
want to count how many times each distinct ``data`` value appears, you might
try this::

    # Warning: not quite correct!
    Item.objects.values("data").annotate(Count("id"))

...which will group the ``Item`` objects by their common ``data`` values and
then count the number of ``id`` values in each group. Except that it won't
quite work. The default ordering by ``name`` will also play a part in the
grouping, so this query will group by distinct ``(data, name)`` pairs, which
isn't what you want. Instead, you should construct this queryset::

    Item.objects.values("data").annotate(Count("id")).order_by()

...clearing any ordering in the query. You could also order by, say, ``data``
without any harmful effects, since that is already playing a role in the
query.

This behavior is the same as that noted in the queryset documentation for
:meth:`~django.db.models.query.QuerySet.distinct` and the general rule is the
same: normally you won't want extra columns playing a part in the result, so
clear out the ordering, or at least make sure it's restricted only to those
fields you also select in a ``values()`` call.

.. note::
    You might reasonably ask why Django doesn't remove the extraneous columns
    for you. The main reason is consistency with ``distinct()`` and other
    places: Django **never** removes ordering constraints that you have
    specified (and we can't change those other methods' behavior, as that
    would violate our :doc:`/misc/api-stability` policy).

Aggregating annotations
-----------------------

You can also generate an aggregate on the result of an annotation. When you
define an ``aggregate()`` clause, the aggregates you provide can reference
any alias defined as part of an ``annotate()`` clause in the query.

For example, if you wanted to calculate the average number of authors per
book you first annotate the set of books with the author count, then
aggregate that author count, referencing the annotation field::

    >>> from django.db.models import Count, Avg
    >>> Book.objects.annotate(num_authors=Count('authors')).aggregate(Avg('num_authors'))
    {'num_authors__avg': 1.66}
