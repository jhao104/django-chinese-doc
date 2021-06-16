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

... ``Book.objects.all()`` 语句将返回数据库中所有书的记录.

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

默认管理器
----------------

.. attribute:: Model._default_manager

在自定义 ``Manager`` 对象时需要注意在Django中的第一个 ``Manager``
(按照定义的顺序)具有特殊地位. Django将第一个 ``Manager`` 视为"默认" ``Manager``, 在Django的好几处(包括
:djadmin:`dumpdata`)将会只使用该 ``Manager`` . 因此在自定义管理器时一定要注意默认管理器, 这样才能避免
因为重写 ``get_queryset()`` 而导致无法获取到预期数据的问题.

使用 :attr:`Meta.default_manager_name
<django.db.models.Options.default_manager_name>` 来指定默认管理器.

如果你代码中必须要处理未知的模型, 例如实现通用视图的第三方应用, 请使用该管理器(或者 :attr:`~Model._base_manager`)
而不是假定模型具有 ``objects`` 管理器.

基础管理器
-------------

.. attribute:: Model._base_manager

.. _managers-for-related-objects:

使用管理器访问关联对象
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

默认情况下, Django访问关联对象(i.e. ``choice.question``)使用 ``Model._base_manager`` 管理器类的实例,
而不是关联对象的 ``_default_manager``. 因为Django要检索可能被默认管理器过滤掉的关联对象.

如果默认的基础管理器类 (:class:`django.db.models.Manager`) 无法满足需求, 可以设置 :attr:`Meta.base_manager_name
<django.db.models.Options.base_manager_name>` 告诉Django使用哪个类.

通过关联模型查询时不会使用基础管理器. 例如, 在 :ref:`教程 <creating-models>` 中 ``Question`` 模型有一个 ``deleted`` 字段,
如果基础管理器过滤掉 ``deleted=True`` 的实例,  ``Choice.objects.filter(question__name__startswith='What')`` 查询还是会返回
和已删除questions关联的choices.

不要在这类管理器子类中过滤结果
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

该管理器会用于访问关联模型的关联对象. 因此在这种情况下Django需要能获取相关模型的所有对象, 这样才能根据关联关系得到所有的数据.

如果重写了 ``get_queryset()`` 方法并且过滤了某些行, 这会导致Django返回不正确的结果.
请不要在基础管理器的 ``get_queryset()`` 中过滤数据.

.. _calling-custom-queryset-methods-from-manager:

管理器调用自定义 ``QuerySet`` 方法
----------------------------------------------------

大部分的 ``QuerySet`` 标准方法都可以在 ``Manager`` 中直接访问, 下面是在 ``Manager`` 中调用自定义 ``QuerySet`` 方法的唯一途径::

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

使用上面代码就可以通过管理器 ``Person.people`` 直接调用 ``authors()`` 和 ``editors()``.

.. _create-manager-with-queryset-methods:

通过 ``QuerySet`` 方法创建管理器
--------------------------------------------

上例中的方案需要 ``QuerySet`` 和 ``Manager`` 都要创建同样的方法, 使用 :meth:`QuerySet.as_manager()
<django.db.models.query.QuerySet.as_manager>` 可以创建一个 ``Manager`` 实例和自定义 ``QuerySet`` 方法的副本::

    class Person(models.Model):
        ...
        people = PersonQuerySet.as_manager()

通过 :meth:`QuerySet.as_manager()
<django.db.models.query.QuerySet.as_manager>` 创建的 ``Manager`` 实例实际上等价于上例中的 ``PersonManager``.

实际使用时不是所有 ``QuerySet`` 方法都需要放到 ``Manager`` 中;
例如, 我们想阻止 :meth:`QuerySet.delete()
<django.db.models.query.QuerySet.delete>` 方法拷贝到 ``Manager`` 中.

方法拷贝遵循以下规则:

- Public方法默认会被拷贝.
- Private方法(以下划线开头)默认不会被拷贝.
- ``queryset_only`` 属性为 ``False`` 的方法总是会被拷贝.
- ``queryset_only`` 属性为 ``True`` 的方法不会被拷贝.

举个例子::

    class CustomQuerySet(models.QuerySet):
        # Manager and QuerySet都可以访问.
        def public_method(self):
            return

        # 仅QuerySet可以访问.
        def _private_method(self):
            return

        # 仅QuerySet可以访问.
        def opted_out_public_method(self):
            return
        opted_out_public_method.queryset_only = True

        # Manager and QuerySet都可以访问.
        def _opted_in_private_method(self):
            return
        _opted_in_private_method.queryset_only = False

``from_queryset()``
~~~~~~~~~~~~~~~~~~~

.. classmethod:: from_queryset(queryset_class)

在更高阶的使用中, 你可能想要创建一个自定义的 ``Manager`` 和一个自定义的
``QuerySet``. 可以通过调用 ``Manager.from_queryset()`` 实现, 它返回一个 ``Manager`` *子类*
带有所有自定义 ``QuerySet`` 方法的副本::

    class BaseManager(models.Manager):
        def manager_only_method(self):
            return

    class CustomQuerySet(models.QuerySet):
        def manager_and_queryset_method(self):
            return

    class MyModel(models.Model):
        objects = BaseManager.from_queryset(CustomQuerySet)()

还可以将生成的类存储到变量中::

    CustomManager = BaseManager.from_queryset(CustomQuerySet)

    class MyModel(models.Model):
        objects = CustomManager()

.. _custom-managers-and-inheritance:

自定义管理器和模型继承
-------------------------------------

下面是Django处理自定义管理器和 :ref:`模型继承
<model-inheritance>` 原则:

#. 基类的管理器总是被子类以Python的正常名称解析顺序继承(子类的属性会覆盖父类上相同名称属性, 父类会覆盖更上一级的类, 依次类推).

#. 如果该模型或其父类都没有定义管理器, Django会自动创建名为 ``objects`` 的管理器.

#. 类的模型管理器来自于
   :attr:`Meta.default_manager_name
   <django.db.models.Options.default_manager_name>` 指定, 或者模型声明的第一个管理器或者其父类的默认管理器.

.. versionchanged:: 1.10

    上面的一些继承行为只有在模型的 ``Meta`` 类中设置了
    ``manager_inheritance_from_future = True`` 才会生效.
    在之前版本中, 如果没有设置该属性, 则管理器的继承行为根据模型继承的类型而异(:ref:`abstract-base-classes`,
    :ref:`multi-table-inheritance`, 或者 :ref:`proxy-models`), 特别是选择默认管理器部分.

如果你想在一组模型中实现一系列自定义管理器, 上面的规则就为你实现这一目的提供了必要的灵活性. 你可以继承一个抽象继承定义自定义的默认管理器,
假设你的基类是这样的::

    class AbstractBase(models.Model):
        # ...
        objects = CustomManager()

        class Meta:
            abstract = True

如果你在子类中没有定义管理器, 直接使用上面的代码, 默认管理器就是从基类中继承的  ``objects`` ::

    class ChildA(AbstractBase):
        # ...
        # 该类将 CustomManager 作为默认管理器.
        pass

如果你希望继承 ``AbstractBase``, 使用另一个默认管理器, 那么可以在子类中定义默认管理器::

    class ChildB(AbstractBase):
        # ...
        # 显式定义默认管理器.
        default_manager = OtherManager()

在这个例子中, ``default_manager`` 作为默认管理器. 从基类中继承的 ``objects`` 管理器也是可用的. 只不过它不再是默认管理器.

最后再举个例子, 假设你想在子类中添加额外的管理器, 但是仍想从 ``AbstractBase`` 继承的管理器作为默认管理器.
这个情况下, 不能在子类中添加新的管理器, 这样会覆盖掉继承的默认管理器, 并且还必须显式地指定来自抽象基类的所有管理器.
解决办法是定义另一个基类添加额外管理器, 然后继承时将其放在默认管理器所在基类 **之后** ::

    class ExtraManager(models.Model):
        extra_manager = OtherManager()

        class Meta:
            abstract = True

    class ChildC(AbstractBase, ExtraManager):
        # ...
        # 默认管理器为 CustomManager,
        # 额外管理器也可以通过 "extra_manager" 属性使用.
        pass

请注意, 虽然可以在抽象模型上 *定义* 自定义管理器, 但不是不能够
*调用* 抽象模型的方法::

    ClassA.objects.do_something()

是合法的, 但是::

    AbstractBase.objects.do_something()

会引发异常. 这是因为管理器意在封装管理映射对象集合的逻辑. 由于抽象的对象中并没有一个集合, 管理它们是毫无意义的.
如果你写了应用在抽象模型上的功能, 你应该把功能放到抽象模型 ``staticmethod`` 或者 ``classmethod`` 中.

实现时注意事项
-----------------------

无论在自定义的 ``Manager`` 中添加了什么特性, 都必须能够对 ``Manager`` 实例进行浅拷贝; 即以下代码必须有效::

    >>> import copy
    >>> manager = MyManager()
    >>> my_copy = copy.copy(manager)

Django在某些查询期间会生成管理器对象的浅拷贝; 如果管理器无法复制那么这些查询将失败.

这对于大多数自定义管理器不是什么问题. 如果仅仅是在 ``Manager`` 中添加一些简单的方法, 这不大可能会导致 ``Manager`` 实例变得不可复制,
但是, 如果重写了 ``Manager`` 对象用于控制对象状态的 ``__getattr__`` 或其它私有方法, 你需要确认你的修改不会影响 ``Manager`` 被复制.
