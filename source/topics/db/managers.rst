========
管理器
========

.. currentmodule:: django.db.models

.. class:: Manager()

``Manager`` 是Django进行数据库操作的接口, Django应用的模型至少有一个 ``Manager``.

如何使用 ``Manager`` 请参考 :doc:`/topics/db/queries`, 本文着重介绍管理器属性以及自定义 ``Manager``.

.. _manager-names:

管理器名称
=============

Django会默认为每个模型赋予一个名为 ``objects`` 的 ``Manager``.
如果你想将 ``objects`` 用于字段名称, 或者单纯是换个名字不想用 ``objects`` 作为 ``Manager`` 名字,
可以在定义模型的基础类上, 定义一个类型为 ``models.Manager()`` 的属性. 例如::

    from django.db import models

    class Person(models.Model):
        #...
        people = models.Manager()

使用上面的模型时, 使用 ``Person.objects`` 会抛出 ``AttributeError`` 异常, 而应该使用 ``Person.people.all()`` 来查询所有 ``Person``.

.. _custom-managers:

自定义管理器
===============

可以通过继承基础类 ``Manager`` 定义自己 ``Manager`` 类来实现自定义模型 ``Manager``.

通常自定义 ``Manager`` 用于两个场景: 添加额外 ``Manager`` 方法, 或者是修改 ``Manager`` 返回的 ``QuerySet``.

添加额外管理器方法
----------------------------

添加额外 ``Manager`` 方法通常是用来添加 "表级别" 的功能. ("行级别"功能可以使用模型实例的 :ref:`模型方法 <model-methods>` 实现.)

自定义的 ``Manager`` 方法无需一定要返回 ``QuerySet``, 可以返回任何类型.

例如下面代码, 自定义 ``Manager`` 提供一个 ``with_counts()`` 方法, 用来实现一个聚合查询, 返回所有 ``OpinionPoll`` 对象, 每个对象带有 ``num_responses`` 属性::

    from django.db import models

    class PollManager(models.Manager):
        def with_counts(self):
            from django.db import connection
            with connection.cursor() as cursor:
                cursor.execute("""
                    SELECT p.id, p.question, p.poll_date, COUNT(*)
                    FROM polls_opinionpoll p, polls_response r
                    WHERE p.id = r.poll_id
                    GROUP BY p.id, p.question, p.poll_date
                    ORDER BY p.poll_date DESC""")
                result_list = []
                for row in cursor.fetchall():
                    p = self.model(id=row[0], question=row[1], poll_date=row[2])
                    p.num_responses = row[3]
                    result_list.append(p)
            return result_list

    class OpinionPoll(models.Model):
        question = models.CharField(max_length=200)
        poll_date = models.DateField()
        objects = PollManager()

    class Response(models.Model):
        poll = models.ForeignKey(OpinionPoll, on_delete=models.CASCADE)
        person_name = models.CharField(max_length=50)
        response = models.TextField()

在上面例子中, 调用 ``OpinionPoll.objects.with_counts()`` 返回带有 ``num_responses`` 属性的 ``OpinionPoll`` 列表.

``Manager`` 方法中可以通过 ``self.model`` 来获取当前模型类.

修改管理器的初始化 ``QuerySet``
------------------------------------------

``Manager`` 的默认 ``QuerySet`` 会返回系统中所有的对象. 以下面模型为例::

    from django.db import models

    class Book(models.Model):
        title = models.CharField(max_length=100)
        author = models.CharField(max_length=50)

...``Book.objects.all()`` 语句将返回数据库中所有书的记录.

可以通过重写 ``Manager.get_queryset()`` 方法来修改 ``Manager`` 的默认 ``QuerySet``.
``get_queryset()`` 必须返回 ``QuerySet``.

例如, 下面模型定义了 *两个* ``Manager`` -- 一个返回所有对象, 另一个只返回作者是Roald Dahl的书::

    # First, define the Manager subclass.
    class DahlBookManager(models.Manager):
        def get_queryset(self):
            return super(DahlBookManager, self).get_queryset().filter(author='Roald Dahl')

    # Then hook it into the Book model explicitly.
    class Book(models.Model):
        title = models.CharField(max_length=100)
        author = models.CharField(max_length=50)

        objects = models.Manager() # The default manager.
        dahl_objects = DahlBookManager() # The Dahl-specific manager.

在上面例子中, ``Book.objects.all()`` 会返回数据库中所有的书, ``Book.dahl_objects.all()`` 只返回作者为Roald Dahl的书.

当然, 因为 ``get_queryset()`` 返回的是 ``QuerySet`` 对象, 所以可以对它使用
``filter()``, ``exclude()`` 和其他 ``QuerySet`` 方法.
下面例子的语句都是可用的::

    Book.dahl_objects.all()
    Book.dahl_objects.filter(title='Matilda')
    Book.dahl_objects.count()

该例还展示了另外一个很有意思的技巧: 同一模型使用多个管理器.
你可以依据自己偏好在模型里面添加多个 ``Manager()`` 实例.
下例是给模型添加通用过滤器的一个简单方::

    class AuthorManager(models.Manager):
        def get_queryset(self):
            return super(AuthorManager, self).get_queryset().filter(role='A')

    class EditorManager(models.Manager):
        def get_queryset(self):
            return super(EditorManager, self).get_queryset().filter(role='E')

    class Person(models.Model):
        first_name = models.CharField(max_length=50)
        last_name = models.CharField(max_length=50)
        role = models.CharField(max_length=1, choices=(('A', _('Author')), ('E', _('Editor'))))
        people = models.Manager()
        authors = AuthorManager()
        editors = EditorManager()

在上面例子中可以使用 ``Person.authors.all()``, ``Person.editors.all()``,
和 ``Person.people.all()`` 来过滤.

.. _default-managers:

Default managers
----------------

.. attribute:: Model._default_manager

If you use custom ``Manager`` objects, take note that the first ``Manager``
Django encounters (in the order in which they're defined in the model) has a
special status. Django interprets the first ``Manager`` defined in a class as
the "default" ``Manager``, and several parts of Django (including
:djadmin:`dumpdata`) will use that ``Manager`` exclusively for that model. As a
result, it's a good idea to be careful in your choice of default manager in
order to avoid a situation where overriding ``get_queryset()`` results in an
inability to retrieve objects you'd like to work with.

You can specify a custom default manager using :attr:`Meta.default_manager_name
<django.db.models.Options.default_manager_name>`.

If you're writing some code that must handle an unknown model, for example, in
a third-party app that implements a generic view, use this manager (or
:attr:`~Model._base_manager`) rather than assuming the model has an ``objects``
manager.

Base managers
-------------

.. attribute:: Model._base_manager

.. _managers-for-related-objects:

Using managers for related object access
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default, Django uses an instance of the ``Model._base_manager`` manager
class when accessing related objects (i.e. ``choice.question``), not the
``_default_manager`` on the related object. This is because Django needs to be
able to retrieve the related object, even if it would otherwise be filtered out
(and hence be inaccessible) by the default manager.

If the normal base manager class (:class:`django.db.models.Manager`) isn't
appropriate for your circumstances, you can tell Django which class to use by
setting :attr:`Meta.base_manager_name
<django.db.models.Options.base_manager_name>`.

Manager's aren't used when querying on related models. For example, if the
``Question`` model :ref:`from the tutorial <creating-models>` had a ``deleted``
field and a base manager that filters out instances with ``deleted=True``, a
queryset like ``Choice.objects.filter(question__name__startswith='What')``
would include choices related to deleted questions.

Don't filter away any results in this type of manager subclass
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This manager is used to access objects that are related to from some other
model. In those situations, Django has to be able to see all the objects for
the model it is fetching, so that *anything* which is referred to can be
retrieved.

If you override the ``get_queryset()`` method and filter out any rows, Django
will return incorrect results. Don't do that. A manager that filters results
in ``get_queryset()`` is not appropriate for use as a base manager.

.. _calling-custom-queryset-methods-from-manager:

Calling custom ``QuerySet`` methods from the manager
----------------------------------------------------

While most methods from the standard ``QuerySet`` are accessible directly from
the ``Manager``, this is only the case for the extra methods defined on a
custom ``QuerySet`` if you also implement them on the ``Manager``::

    class PersonQuerySet(models.QuerySet):
        def authors(self):
            return self.filter(role='A')

        def editors(self):
            return self.filter(role='E')

    class PersonManager(models.Manager):
        def get_queryset(self):
            return PersonQuerySet(self.model, using=self._db)

        def authors(self):
            return self.get_queryset().authors()

        def editors(self):
            return self.get_queryset().editors()

    class Person(models.Model):
        first_name = models.CharField(max_length=50)
        last_name = models.CharField(max_length=50)
        role = models.CharField(max_length=1, choices=(('A', _('Author')), ('E', _('Editor'))))
        people = PersonManager()

This example allows you to call both ``authors()`` and ``editors()`` directly from
the manager ``Person.people``.

.. _create-manager-with-queryset-methods:

Creating a manager with ``QuerySet`` methods
--------------------------------------------

In lieu of the above approach which requires duplicating methods on both the
``QuerySet`` and the ``Manager``, :meth:`QuerySet.as_manager()
<django.db.models.query.QuerySet.as_manager>` can be used to create an instance
of ``Manager`` with a copy of a custom ``QuerySet``’s methods::

    class Person(models.Model):
        ...
        people = PersonQuerySet.as_manager()

The ``Manager`` instance created by :meth:`QuerySet.as_manager()
<django.db.models.query.QuerySet.as_manager>` will be virtually
identical to the ``PersonManager`` from the previous example.

Not every ``QuerySet`` method makes sense at the ``Manager`` level; for
instance we intentionally prevent the :meth:`QuerySet.delete()
<django.db.models.query.QuerySet.delete>` method from being copied onto
the ``Manager`` class.

Methods are copied according to the following rules:

- Public methods are copied by default.
- Private methods (starting with an underscore) are not copied by default.
- Methods with a ``queryset_only`` attribute set to ``False`` are always copied.
- Methods with a ``queryset_only`` attribute set to ``True`` are never copied.

For example::

    class CustomQuerySet(models.QuerySet):
        # Available on both Manager and QuerySet.
        def public_method(self):
            return

        # Available only on QuerySet.
        def _private_method(self):
            return

        # Available only on QuerySet.
        def opted_out_public_method(self):
            return
        opted_out_public_method.queryset_only = True

        # Available on both Manager and QuerySet.
        def _opted_in_private_method(self):
            return
        _opted_in_private_method.queryset_only = False

``from_queryset()``
~~~~~~~~~~~~~~~~~~~

.. classmethod:: from_queryset(queryset_class)

For advanced usage you might want both a custom ``Manager`` and a custom
``QuerySet``. You can do that by calling ``Manager.from_queryset()`` which
returns a *subclass* of your base ``Manager`` with a copy of the custom
``QuerySet`` methods::

    class BaseManager(models.Manager):
        def manager_only_method(self):
            return

    class CustomQuerySet(models.QuerySet):
        def manager_and_queryset_method(self):
            return

    class MyModel(models.Model):
        objects = BaseManager.from_queryset(CustomQuerySet)()

You may also store the generated class into a variable::

    CustomManager = BaseManager.from_queryset(CustomQuerySet)

    class MyModel(models.Model):
        objects = CustomManager()

.. _custom-managers-and-inheritance:

Custom managers and model inheritance
-------------------------------------

Here's how Django handles custom managers and :ref:`model inheritance
<model-inheritance>`:

#. Managers from base classes are always inherited by the child class,
   using Python's normal name resolution order (names on the child
   class override all others; then come names on the first parent class,
   and so on).

#. If no managers are declared on a model and/or its parents, Django
   automatically creates the ``objects`` manager.

#. The default manager on a class is either the one chosen with
   :attr:`Meta.default_manager_name
   <django.db.models.Options.default_manager_name>`, or the first manager
   declared on the model, or the default manager of the first parent model.

.. versionchanged:: 1.10

    Some inheritance behaviors described above don't apply unless you set
    ``manager_inheritance_from_future = True`` on the model's ``Meta`` class.
    In older versions and if you don't set that attribute, manager inheritance
    varies depending on the type of model inheritance (:ref:`abstract-base-classes`,
    :ref:`multi-table-inheritance`, or :ref:`proxy-models`), especially
    with regards to electing the default manager.

These rules provide the necessary flexibility if you want to install a
collection of custom managers on a group of models, via an abstract base
class, but still customize the default manager. For example, suppose you have
this base class::

    class AbstractBase(models.Model):
        # ...
        objects = CustomManager()

        class Meta:
            abstract = True

If you use this directly in a subclass, ``objects`` will be the default
manager if you declare no managers in the base class::

    class ChildA(AbstractBase):
        # ...
        # This class has CustomManager as the default manager.
        pass

If you want to inherit from ``AbstractBase``, but provide a different default
manager, you can provide the default manager on the child class::

    class ChildB(AbstractBase):
        # ...
        # An explicit default manager.
        default_manager = OtherManager()

Here, ``default_manager`` is the default. The ``objects`` manager is
still available, since it's inherited. It just isn't used as the default.

Finally for this example, suppose you want to add extra managers to the child
class, but still use the default from ``AbstractBase``. You can't add the new
manager directly in the child class, as that would override the default and you would
have to also explicitly include all the managers from the abstract base class.
The solution is to put the extra managers in another base class and introduce
it into the inheritance hierarchy *after* the defaults::

    class ExtraManager(models.Model):
        extra_manager = OtherManager()

        class Meta:
            abstract = True

    class ChildC(AbstractBase, ExtraManager):
        # ...
        # Default manager is CustomManager, but OtherManager is
        # also available via the "extra_manager" attribute.
        pass

Note that while you can *define* a custom manager on the abstract model, you
can't *invoke* any methods using the abstract model. That is::

    ClassA.objects.do_something()

is legal, but::

    AbstractBase.objects.do_something()

will raise an exception. This is because managers are intended to encapsulate
logic for managing collections of objects. Since you can't have a collection of
abstract objects, it doesn't make sense to be managing them. If you have
functionality that applies to the abstract model, you should put that functionality
in a ``staticmethod`` or ``classmethod`` on the abstract model.

Implementation concerns
-----------------------

Whatever features you add to your custom ``Manager``, it must be
possible to make a shallow copy of a ``Manager`` instance; i.e., the
following code must work::

    >>> import copy
    >>> manager = MyManager()
    >>> my_copy = copy.copy(manager)

Django makes shallow copies of manager objects during certain queries;
if your Manager cannot be copied, those queries will fail.

This won't be an issue for most custom managers. If you are just
adding simple methods to your ``Manager``, it is unlikely that you
will inadvertently make instances of your ``Manager`` uncopyable.
However, if you're overriding ``__getattr__`` or some other private
method of your ``Manager`` object that controls object state, you
should ensure that you don't affect the ability of your ``Manager`` to
be copied.
