=============
模型实例
=============

.. currentmodule:: django.db.models

本文档将详细描述 ``Model`` . 在阅读本文档之前你需要先阅读和了解 :doc:`模型 </topics/db/models>` 和 :doc:`查询数据 </topics/db/queries>` .

本文档中将全部使用 :doc:`数据查询指南</topics/db/queries>` 中的 :ref:`Weblog Model  <queryset-model-example>` 作为演示模型.

创建对象
================

创建模型对象和实例化Python类一样:

.. class:: Model(**kwargs)

传入的关键字参数就是模型的字段名, 只有调用了 :meth:`~Model.save()` 才会执行数据库写入.

.. note::

    请不要通过重写 ``__init__`` 方法的方式来自定义创建模型. 因为这一行为可能会改变默认的调用路径.
    你可以通过下面两种方式来自定义创建模型:

    1. 使用类方法::

        from django.db import models

        class Book(models.Model):
            title = models.CharField(max_length=100)

            @classmethod
            def create(cls, title):
                book = cls(title=title)
                # do something with the book
                return book

        book = Book.create("Pride and Prejudice")

    2. 使用管理类(推荐)::

        class BookManager(models.Manager):
            def create_book(self, title):
                book = self.create(title=title)
                # do something with the book
                return book

        class Book(models.Model):
            title = models.CharField(max_length=100)

            objects = BookManager()

        book = Book.objects.create_book("Pride and Prejudice")

自定义模型加载
-------------------------

.. classmethod:: Model.from_db(db, field_names, values)

``from_db()`` 方法用于在数据库加载时自定义模型的创建.

``db`` 包含数据加载来源库的别名, ``field_names`` 包含所有已加载的字段名称,
``values`` 包含每个 ``field_names`` 字段的加载值. ``field_names`` 和 ``values`` 中顺序相同.
如果模型中所有字段都存在, 那么 ``values`` 中的顺序就必须要和 ``__init__()`` 所定义的顺序一致, 这时就可以通过 ``cls(*values)`` 创建实例.
如果有未加载字段则不会存在于 ``field_names`` 中, 这时需要给每个缺失字段分配一个 ``django.db.models.DEFERRED`` 值.

``from_db()`` 除了需要创建模型外, 还需要为新的模型实例的 `_state`` 属性设置 ``adding`` 和 ``db`` 标识.

下面例子演示了如何将数据库中加载数据转换成模型实例::

    from django.db.models import DEFERRED

    @classmethod
    def from_db(cls, db, field_names, values):
        # Default implementation of from_db() (subject to change and could
        # be replaced with super()).
        if len(values) != len(cls._meta.concrete_fields):
            values = list(values)
            values.reverse()
            values = [
                values.pop() if f.attname in field_names else DEFERRED
                for f in cls._meta.concrete_fields
            ]
        new = cls(*values)
        instance._state.adding = False
        instance._state.db = db
        # customization to store the original field values on the instance
        instance._loaded_values = dict(zip(field_names, values))
        return instance

    def save(self, *args, **kwargs):
        # Check how the current values differ from ._loaded_values. For example,
        # prevent changing the creator_id of the model. (This example doesn't
        # support cases where 'creator_id' is deferred).
        if not self._state.adding and (
                self.creator_id != self._loaded_values['creator_id']):
            raise ValueError("Updating the value of creator isn't allowed")
        super(...).save(*args, **kwargs)

上面例子演示了 ``from_db()`` 的一个完整实现. 当然这里的 ``from_db()`` 中完全可以通过调用 ``super()`` 代替.

.. versionchanged:: 1.10

    在旧版本中, 可以通过调用
    ``cls._deferred`` 来检查是否所有字段都加载完. 该属性被移除, 新增
    ``django.db.models.DEFERRED``.

从数据库更新对象
================================

如果删除了一个模型实例的字段, 再次访问时会从数据库重新加载该值::

    >>> obj = MyModel.objects.first()
    >>> del obj.field
    >>> obj.field  # 从数据库加载字段

.. versionchanged:: 1.10

    在旧版本中, 访问删除的字段不会重新加载而是引发 ``AttributeError``.

.. method:: Model.refresh_from_db(using=None, fields=None)

``refresh_from_db()`` 方法用于从数据库中重新加载模型值. 当无参数调用时将做以下动作:

1. 所有非延迟加载字段都会被更新成数据库中的当前值.
2. 如果之前加载的关联实例不再合法将会从新实例中移除. 例如, 如果重新加载的模型有个关联的外键 ``Author``,
   如果此时 ``obj.author_id != obj.author.id``,  那么 ``obj.author`` 将会被遗弃, 在下次访问它时根据
   ``obj.author_id`` 重新加载.

只有模型字段才会被重新加载. 其他数据库值比如注解不会被重载. 所有
:func:`@cached_property <django.utils.functional.cached_property>` 也不会被清除.

重载是从原加载的库中加载, 如果模型实例不是从数据库中加载的则从默认数据库加载,
可以使用 ``using`` 参数来强制指定重载的数据库.

可以使用 ``fields`` 参数强制指定重新加载的字段.

例如, 下面代码用来测试 ``update()`` 是否按预期更新::

    def test_update_result(self):
        obj = MyModel.objects.create(val=1)
        MyModel.objects.filter(pk=obj.pk).update(val=F('val') + 1)
        # 这时obj.val仍然是1, 但是数据库中的值已经被更新成2, 最新值需要从数据库中重新加载
        obj.refresh_from_db()
        self.assertEqual(obj.val, 2)

注意延迟字段也是通过这个方法加载的, 所以也可以通过该方法自定义验证加载字段行为.
下面的例子展示了当一个延迟字段被重载时,如何重载实例的所有字段::

    class ExampleModel(models.Model):
        def refresh_from_db(self, using=None, fields=None, **kwargs):
            # fields contains the name of the deferred field to be
            # loaded.
            if fields is not None:
                fields = set(fields)
                deferred_fields = self.get_deferred_fields()
                # If any deferred field is going to be loaded
                if fields.intersection(deferred_fields):
                    # then load all of them
                    fields = fields.union(deferred_fields)
            super(ExampleModel, self).refresh_from_db(using, fields, **kwargs)

.. method:: Model.get_deferred_fields()

一个辅助方法,它返回包含模型当前所有延迟字段的属性名称.

.. _validating-objects:

验证对象
==================

验证模型涉及三个步骤:

1. 验证模型的字段 - :meth:`Model.clean_fields()`
2. 验证模型的完整性 - :meth:`Model.clean()`
3. 验证模型字段的唯一性 - :meth:`Model.validate_unique()`

当调用模型的 :meth:`~Model.full_clean()` 方法时,这三个方法都会被执行.

如果使用 :class:`~django.forms.ModelForm`, 调用
:meth:`~django.forms.Form.is_valid()` 方法将会对模型的所有方法执行这些验证. 详见 :doc:`ModelForm
documentation </topics/forms/modelforms>` . 如果你验证的字段不在 :class:`~django.forms.ModelForm` 中或者想要
自己验证错误, 可以调用模型的 :meth:`~Model.full_clean()` 方法.

.. method:: Model.full_clean(exclude=None, validate_unique=True)

该方法会依次调用 :meth:`Model.clean_fields()` 、 :meth:`Model.clean()` 和
:meth:`Model.validate_unique()` (如果 ``validate_unique`` 设置为 ``True``), 抛出
:exc:`~django.core.exceptions.ValidationError` 异常的
``message_dict`` 属性包含了这三个验证阶段错误信息.

可选参数 ``exclude`` 接收一组不参与验证和清除的字段列表.
:class:`~django.forms.ModelForm` 的场景下, 如果包含了不存在于表单的字段也会被排除, 因为用户也无法修改这些字段的错误.

注意 :meth:`~Model.save()` 方法 *不会* 自动调用 ``full_clean()`` .
如果你想执行一次模型验证需要手动运行, 例如::

    from django.core.exceptions import ValidationError
    try:
        article.full_clean()
    except ValidationError as e:
        # Do something based on the errors contained in e.message_dict.
        # Display them to a user, or handle them programmatically.
        pass

``full_clean()`` 第一步是执行模型字段验证.

.. method:: Model.clean_fields(exclude=None)

该方法会验证模型所有字段.可选参数 ``exclude`` 接收一个不参与验证字段组成的列表.
如果有字段验证失败将引发 :exc:`~django.core.exceptions.ValidationError` 异常.

``full_clean()`` 执行的第二步是 :meth:`Model.clean()`.
该方式是用来被重写实现自定义模型验证.

.. method:: Model.clean()

这个方法应该用来自定义模型验证和修改模型属性. 例如, 可以给模型字段赋值和组合验证多个字段::

    import datetime
    from django.core.exceptions import ValidationError
    from django.db import models
    from django.utils.translation import ugettext_lazy as _

    class Article(models.Model):
        ...
        def clean(self):
            # Don't allow draft entries to have a pub_date.
            if self.status == 'draft' and self.pub_date is not None:
                raise ValidationError(_('Draft entries may not have a publication date.'))
            # Set the pub_date for published items if it hasn't been set already.
            if self.status == 'published' and self.pub_date is None:
                self.pub_date = datetime.date.today()

注意, 和 :meth:`Model.full_clean()` 方法一样,  ``clean()`` 也不会在 :meth:`~Model.save()` 时调用.

在上面例子中, ``Model.clean()`` 抛出的 :exc:`~django.core.exceptions.ValidationError` 异常是由字符串实例化的,
因此在错误字典中以一个特殊的键 :data:`~django.core.exceptions.NON_FIELD_ERRORS` 储存.
这个键用于与整个模型相关的错误,而不是与某个特定字段相关的错误::

    from django.core.exceptions import ValidationError, NON_FIELD_ERRORS
    try:
        article.full_clean()
    except ValidationError as e:
        non_field_errors = e.message_dict[NON_FIELD_ERRORS]

如果要抛出指定字段的异常, 可以使用字典实例化
:exc:`~django.core.exceptions.ValidationError` , 其中字典的键为字段名.
我们可以修改上面的例子, 抛出一个 ``pub_date`` 字段的异常::

    class Article(models.Model):
        ...
        def clean(self):
            # Don't allow draft entries to have a pub_date.
            if self.status == 'draft' and self.pub_date is not None:
                raise ValidationError({'pub_date': _('Draft entries may not have a publication date.')})
            ...

如果 ``Model.clean()`` 检验出多个字段异常, 可以传入一个字段名和错误映射的字典::

    raise ValidationError({
        'title': ValidationError(_('Missing title.'), code='required'),
        'pub_date': ValidationError(_('Invalid date.'), code='invalid'),
    })

最后, ``full_clean()`` 将检验模型的唯一约束.

.. method:: Model.validate_unique(exclude=None)

该方法类似于 :meth:`~Model.clean_fields`, 只是验证的是模型所有字段的唯一约束而不是单个字段值.
``exclude`` 参数接收一个用于排除验证的字段名列表.
如果有字段验证失败将抛出 :exc:`~django.core.exceptions.ValidationError` 异常.

注意, 如果 ``validate_unique()`` 传入了 ``exclude`` , 任何涉及传入字段的
:attr:`~django.db.models.Options.unique_together` 约束都不会被检验.


保存对象
==============

调用 ``save()`` 方法将对象保存至数据库:

.. method:: Model.save(force_insert=False, force_update=False, using=DEFAULT_DB_ALIAS, update_fields=None)

如果要自定义保存行为, 可以重写 ``save()`` 方法. 详见 :ref:`overriding-model-methods`.

模型保存过程也有一些微妙的地方, 请看下面的章节.

自增主键
------------------------------

如果模型中有自增主键 :class:`~django.db.models.AutoField`,
它们会在第一次调用 ``save()`` 时被计算然后保存到对象的属性上::

    >>> b2 = Blog(name='Cheddar Talk', tagline='Thoughts on cheese.')
    >>> b2.id     # Returns None, because b doesn't have an ID yet.
    >>> b2.save()
    >>> b2.id     # Returns the ID of your new object.

没有办法在调用 ``save()`` 前知道ID的值, 因为这个值是由数据库计算的而不是Django.

为了方便, 每个模型都有一个名为 ``id`` 的默认 :class:`~django.db.models.AutoField` 字段,
除非为某个字段指定了 ``primary_key=True``, 详见: :class:`~django.db.models.AutoField`.

``pk`` 属性
~~~~~~~~~~~~~~~~~~~

.. attribute:: Model.pk

无论是你自己定义的主键字段还是Django提供的默认主键, 每个模型都有一个名为 ``pk`` 的属性.
它是模型主键字段的别名, 其行为和模型普通属性一样. 你可以读写它的值和普通属性没有区别, 它实际指向的是模型的主键字段.

设置自增主键值
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果要指定模型中 :class:`~django.db.models.AutoField` 字段的ID值, 只需要在定义时传入ID值而不是依赖ID自动生成::

    >>> b3 = Blog(id=3, name='Cheddar Talk', tagline='Thoughts on cheese.')
    >>> b3.id     # Returns 3.
    >>> b3.save()
    >>> b3.id     # Returns 3.

在手动设置自增主键值时请不要使用已经存在的值! 如果你创建新对象时使用了在数据库中已经存在的主键值,
Django会认为你是在更新数据而不是插入数据.

接着上面 ``'Cheddar Talk'`` 博客例子, 下面代码将会覆盖数据库中的记录::

    b4 = Blog(id=3, name='Not Cheddar', tagline='Anything but cheese.')
    b4.save()  # Overrides the previous blog with ID=3!

出现这种情况的原因请参见下面的: `Django如何知道该UPDATE还是INSERT`_.

在知道不会有主键冲突的情况下,指定自增主键值在批量保存数据时非常有用.

保存时会发生什么?
---------------------------

保存对象时Django会执行以下步骤:

#. **发送 pre-save 信号.** 使监听 :data:`~django.db.models.signals.pre_save` 信号的函数, 做一些保存前的事.

#. **预处理数据.** 调用每个字段的
   :meth:`~django.db.models.Field.pre_save` 方法来做一些数据自动处理.
   比如日期和时间字段重写了 ``pre_save()`` 方法来实现
   :attr:`~django.db.models.DateField.auto_now_add` 和
   :attr:`~django.db.models.DateField.auto_now`.

#. **准备数据库数据.** 要求每个字段的
   :meth:`~django.db.models.Field.get_db_prep_save` 方法返回可写入数据库类型的当前值.
   大部分的字段不需要数据准备. 简单的数据类型比如整数和字符串是可以直接写入数据库的Python对象.
   但是一些复杂的数据类型需要做一些转换.例如 :class:`~django.db.models.DateField` 字段使用的是
   Python的 ``datetime`` 对象来储存数据. 数据库并不支持写入 ``datetime`` 对象.
   所以需要将该字段的值转换成符合ISO标准的日期字符串才能写入数据库.

#. **向数据库中插入数据.** 将预处理后准备好的数据组成SQL语句用于数据插入.

#. **发送 post-save 信号.** 使监听 :data:`~django.db.models.signals.post_save`
   信号的函数, 做一些保存后的事.

Django如何知道该UPDATE还是INSERT
-------------------------------------

你可能已经注意到了Django使用一个方法 ``save()`` 来创建和保存数据.
Django对 ``INSERT`` 和 ``UPDATE`` SQL语句进行抽象. 当调用 ``save()`` 时, Django使用下面的算法进行判断:

* 如果对象的主键属性设置了值且是 ``真值`` (例如, 不是 ``None`` 也不是空串), Django将执行 ``UPDATE``.
* 如果对象的主键属性 *没有* 设置值或者 ``UPDATE`` 没有更新任何字段, Django将执行 ``INSERT``.

这里引出一点是, 创建新对象时如果不能确定主键值是否已存在的情况下,请不要手动设置主键.
关于这两种更细微的差别请参见: `设置自增主键值`_ 和下方的: `强制INSERT或UPDATE`_.

在Django1.5或更早版本, 如果设置了主键属性Django会做一个 ``SELECT`` 查询,
如果 ``SELECT`` 查询到有数据, Django会执行  ``UPDATE`` 否则执行 ``INSERT``.
这个旧算法会导致在 ``UPDATE`` 情况下多做一次查询. 在一些极少数的情况下,
即使数据库中包含了一条对象主键值的记录,数据库也不会报告某行被更新.
一个例子是PostgreSQL的 ``ON UPDATE`` 触发器, 它会返回 ``NULL`` .
在这种情况下, 可以通过将 :attr:`~django.db.models.Options.select_on_save` 选项设置为 ``True`` 来启用旧算法.

.. _ref-models-force-insert:

强制INSERT或UPDATE
~~~~~~~~~~~~~~~~~~~~~~~~~~~

在一些少数场景中, 需要 :meth:`~Model.save()` 方法强制执行 ``INSERT`` 而不是 ``UPDATE``. 或者相反的情况,更新一行而不是插入一行.
这些情况下可以传入 ``force_insert=True`` 或 ``force_update=True`` 参数给 :meth:`~Model.save()` 方法.
显然同时传入这两个参数是错误的, 因为不可能即插入又更新!

非极端情况下不要使用这两个参数. Django大概率会做出正确的选择, 强制将其覆盖会导致难以追踪到的错误. 这个功能只适合高级用法.

使用 ``update_fields`` 也会强制更新类似 ``force_update``.

.. _ref-models-field-updates-using-f-expressions:

更新现有字段属性
--------------------------------------------

有时需要对一个字段执行简单的算术计算, 比如加减某个值. 实现这一目的最简单方法::

    >>> product = Product.objects.get(name='Venezuelan Beaver Cheese')
    >>> product.number_sold += 1
    >>> product.save()

如果从数据库中读取的 ``number_sold`` 原始值为 10, 那么保存回数据库中的值为 11.

这个过程可以变得更健壮, 通过将更新基于原始字段的值而不是显式赋予一个新值, 这个过程可以 :ref:`避免竞争条件
<avoiding-race-conditions-using-f>` 而且更快. Django提供了 :class:`F表达式
<django.db.models.F>` 用于这种类型的相对更新. 使用
:class:`F表达式 <django.db.models.F>` 可以将上面的例子改成::

    >>> from django.db.models import F
    >>> product = Product.objects.get(name='Venezuelan Beaver Cheese')
    >>> product.number_sold = F('number_sold') + 1
    >>> product.save()

详见: :class:`F表达式 <django.db.models.F>` 和 :ref:`在更新查询中的使用
<topics-db-queries-update>`.

指定保存字段
-------------------------------

如果 ``save()`` 方法传入了一组字段名组成的列表的参数 ``update_fields``, 那么就只会有该列表中的字段会被更新.
如果只需要更新单个或几个字段可以使用该方法, 这比更新所有字段会带来少许的性能提升. 例如::

    product.name = 'Name changed again'
    product.save(update_fields=['name'])

``update_fields`` 参数可以是任何包含字符串的可迭代对象. 空的
``update_fields`` 可迭代对象将会跳过保存. 如果传入None会更新所有字段.

指定 ``update_fields`` 将强制执行更新.

当访问通过延迟加载模型加载
(:meth:`~django.db.models.query.QuerySet.only()` 和
:meth:`~django.db.models.query.QuerySet.defer()`) 的模型时, 只有通过数据库中加载的字段才会被更新.
实际上, 在这种情况下有一个自动的 ``update_fields``. 如果你赋值或修改延迟字段, 该字段将被添加到更新的字段中.

删除对象
================

.. method:: Model.delete(using=DEFAULT_DB_ALIAS, keep_parents=False)

执行SQL ``DELETE`` 操作. 它只会删除数据库中的数据, Python实例仍然存在并且有字段数据.
该方法返回被删除对象的数量和一个每个删除对象类型的数量的字典.

更多细节以及批量删除, 请参见: :ref:`topics-db-queries-delete`.

可以通过重写 ``delete()`` 来自定义删除行为. 详见: :ref:`overriding-model-methods`.

在 :ref:`多表继承 <multi-table-inheritance>` 的情况下, 可以通过设置 ``keep_parents=True`` 来只删除子模型保留父模型.

.. versionchanged:: 1.9

    新增 ``keep_parents`` 参数.

.. versionchanged:: 1.9

    新增返回值: 已删除对象数量.

Pickling 对象
================

当 :mod:`pickle` 模型时, 它会在当前状态被序列化. 在反序列化时会返回被序列化的模型实例而不是数据库中的数据.

.. admonition:: 不同版本的序列化结果不通用

    模型的pickle只对序列化它们的Django版本有效. 如果使用Django版本N序列化,
    不能保证Django版本N+1可以反序列化这个pickle. 不要将Pickles作为长期归档的策略.

    由于pickle导致的兼容性错误很难诊断,
    所以当你unpickle模型使用的Django版本与pickle时的不同会引发一个 ``RuntimeWarning`` 异常.

.. _model-instance-methods:

其他模型实例方法
============================

一些有特殊用图的对象方法.

``__str__()``
-------------

.. method:: Model.__str__()

``__str__()`` 方法会在对对象调用 ``str()`` 时调用.
Django会在许多地方使用到 ``str(obj)``. 最常用的是在Django Admin站点显示一个对象
和在模板中插入对象值的场景. 所以最好让 ``__str__()`` 返回一个友好的、可读的结果.

例如::

    from django.db import models
    from django.utils.encoding import python_2_unicode_compatible

    @python_2_unicode_compatible  # only if you need to support Python 2
    class Person(models.Model):
        first_name = models.CharField(max_length=50)
        last_name = models.CharField(max_length=50)

        def __str__(self):
            return '%s %s' % (self.first_name, self.last_name)

如果要兼容Python 2, 可以给模型加上 :func:`~django.utils.encoding.python_2_unicode_compatible` 装饰器.

``__eq__()``
------------

.. method:: Model.__eq__()

定义等值判断方法, 具有相同主键值和相同具体类的实例被认为是相等的. 主键值为None的实例除自身外对任何事物都不相等.
具体类被定义为模型的第一个非代理父类, 对于所有其他模型, 它就是模型的类.

例如::

    from django.db import models

    class MyModel(models.Model):
        id = models.AutoField(primary_key=True)

    class MyProxyModel(MyModel):
        class Meta:
            proxy = True

    class MultitableInherited(MyModel):
        pass

    # Primary keys compared
    MyModel(id=1) == MyModel(id=1)
    MyModel(id=1) != MyModel(id=2)
    # Primay keys are None
    MyModel(id=None) != MyModel(id=None)
    # Same instance
    instance = MyModel(id=None)
    instance == instance
    # Proxy model
    MyModel(id=1) == MyProxyModel(id=1)
    # Multi-table inheritance
    MyModel(id=1) != MultitableInherited(id=1)

``__hash__()``
--------------

.. method:: Model.__hash__()

``__hash__()`` 方法基于实例的主键值. 它实际上是 ``hash(obj.pk)``.
如果实例没有设置主键时将会引发 ``TypeError`` 异常(不然的话 ``__hash__()`` 方法会在实例保存前和保存后返回不同的值,
然而Python禁止改变实例的 :meth:`~object.__hash__` 值).

``get_absolute_url()``
----------------------

.. method:: Model.get_absolute_url()

定义一个 ``get_absolute_url()`` 方法告诉Django如何返回对象的URL. 调用方可以通过该方法返回的URL字符串访问到该对象.

例如::

    def get_absolute_url(self):
        return "/people/%i/" % self.id

虽然上面的代码是正确的且很简单, 但是使用 :func:`~django.urls.reverse` 函数才是最优的方法.

例如::

    def get_absolute_url(self):
        from django.urls import reverse
        return reverse('people.views.details', args=[str(self.id)])

在管理站点的对象编辑页面中, 如果定义了 ``get_absolute_url()`` 方法, 编辑页面会多出一个 "View on site" 链接, 点击该链接会进入 ``get_absolute_url()`` 返回的地址.

类似的, Django还有其他地方例如 :doc:`syndication feed
framework </ref/contrib/syndication>` 也使用了 ``get_absolute_url()`` . 如果模型的每个实例都有唯一的URL就可以定义 ``get_absolute_url()``.

.. warning::

    千万不要用没有验证的用户输入构建URL, 避免有害的链接和重定向::

        def get_absolute_url(self):
            return '/%s/' % self.name

    如果 ``self.name`` 为 ``'/example.com'`` 将返回 ``'//example.com/'``
    而这是另一个域下的URL, 而不是实际需要的 ``'/%2Fexample.com/'``.


最佳实践是在模板中使用 ``get_absolute_url()`` 而不是硬编码URL.
比如下面代码是个错误的示范:

.. code-block:: html+django

    <!-- BAD template code. Avoid! -->
    <a href="/people/{{ object.id }}/">{{ object.name }}</a>

下面是正确示范:

.. code-block:: html+django

    <a href="{{ object.get_absolute_url }}">{{ object.name }}</a>

这样做的好处是, 如果修改了对象的URL路径, 即便是一些简单的拼写错误,
这样不用修改每一个使用到该URL的地方. 只需要修改 ``get_absolute_url()`` 中的定义然后在其它代码中调用它.

.. note::
    ``get_absolute_url()`` 返回的字符 **必须** 只包含ASCII字符(URL规范 :rfc:`2396`) 而且在需要情况下必须要URL编码.

    在代码和模板中调用 ``get_absolute_url()`` 可以直接使用而不需要做额外处理.
    如果使用了ASCII范围之外的unicode字符串, 可以使用 ``django.utils.encoding.iri_to_uri()`` 函数来解决这个问题.

额外实例方法
======================

除了 :meth:`~Model.save()` 和 :meth:`~Model.delete()` 模型还可能具有如下方法:

.. method:: Model.get_FOO_display()

对于设置了 :attr:`~django.db.models.Field.choices` 的字段, 该对象将具有一个形似 ``get_FOO_display()`` 方法,
其中 ``FOO`` 是字段名, 该方法返回字段的"可读值".

例如::

    from django.db import models

    class Person(models.Model):
        SHIRT_SIZES = (
            ('S', 'Small'),
            ('M', 'Medium'),
            ('L', 'Large'),
        )
        name = models.CharField(max_length=60)
        shirt_size = models.CharField(max_length=2, choices=SHIRT_SIZES)

::

    >>> p = Person(name="Fred Flintstone", shirt_size="L")
    >>> p.save()
    >>> p.shirt_size
    'L'
    >>> p.get_shirt_size_display()
    'Large'

.. method:: Model.get_next_by_FOO(\**kwargs)
.. method:: Model.get_previous_by_FOO(\**kwargs)

对于没有设置 :attr:`null=True<django.db.models.Field.null>` 的 :class:`~django.db.models.DateField` 字段和
:class:`~django.db.models.DateTimeField` 字段, 该对象将具有一个形似 ``get_next_by_FOO()`` 和
``get_previous_by_FOO()`` 的方法, 其中 ``FOO`` 为字段名称. 它将根据日期字段返回下一个或下一个对象,
并适时抛出 :exc:`~django.db.models.Model.DoesNotExist` 异常.

这两个方法都将使用模型的默认管理器执行查询. 如果需要使用自定义管理器筛选或者执行单次自定义筛选, 这个两个方法还接受可选参数, 其格式如 :ref:`字段查找 <field-lookups>` 中提到的格式.

注意在日期值相同的情况下这些方法将使用主键作为比较, 以保证了没有记录被跳过或重复. 因此不能对未保存的对象使用这些方法.

其他属性
================

``DoesNotExist``
----------------

.. exception:: Model.DoesNotExist

    该异常会在ORM的许多地方使用到, 比如
    :meth:`QuerySet.get() <django.db.models.query.QuerySet.get>` 查询时给定的参数查询不到值.

    Django为每个类都提供 ``DoesNotExist`` 异常是为了区别找不到的对象的所属类.
    并且可以在代码中使用 ``try/except`` 捕获某个特定的模型类. 这个异常类是
    :exc:`django.core.exceptions.ObjectDoesNotExist` 的子类.
