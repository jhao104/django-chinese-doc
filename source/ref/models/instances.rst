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

The ``pk`` property
~~~~~~~~~~~~~~~~~~~

.. attribute:: Model.pk

Regardless of whether you define a primary key field yourself, or let Django
supply one for you, each model will have a property called ``pk``. It behaves
like a normal attribute on the model, but is actually an alias for whichever
attribute is the primary key field for the model. You can read and set this
value, just as you would for any other attribute, and it will update the
correct field in the model.

Explicitly specifying auto-primary-key values
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If a model has an :class:`~django.db.models.AutoField` but you want to define a
new object's ID explicitly when saving, just define it explicitly before
saving, rather than relying on the auto-assignment of the ID::

    >>> b3 = Blog(id=3, name='Cheddar Talk', tagline='Thoughts on cheese.')
    >>> b3.id     # Returns 3.
    >>> b3.save()
    >>> b3.id     # Returns 3.

If you assign auto-primary-key values manually, make sure not to use an
already-existing primary-key value! If you create a new object with an explicit
primary-key value that already exists in the database, Django will assume you're
changing the existing record rather than creating a new one.

Given the above ``'Cheddar Talk'`` blog example, this example would override the
previous record in the database::

    b4 = Blog(id=3, name='Not Cheddar', tagline='Anything but cheese.')
    b4.save()  # Overrides the previous blog with ID=3!

See `How Django knows to UPDATE vs. INSERT`_, below, for the reason this
happens.

Explicitly specifying auto-primary-key values is mostly useful for bulk-saving
objects, when you're confident you won't have primary-key collision.

What happens when you save?
---------------------------

When you save an object, Django performs the following steps:

#. **Emit a pre-save signal.** The :data:`~django.db.models.signals.pre_save`
   signal is sent, allowing any functions listening for that signal to do
   something.

#. **Preprocess the data.** Each field's
   :meth:`~django.db.models.Field.pre_save` method is called to perform any
   automated data modification that's needed. For example, the date/time fields
   override ``pre_save()`` to implement
   :attr:`~django.db.models.DateField.auto_now_add` and
   :attr:`~django.db.models.DateField.auto_now`.

#. **Prepare the data for the database.** Each field's
   :meth:`~django.db.models.Field.get_db_prep_save` method is asked to provide
   its current value in a data type that can be written to the database.

   Most fields don't require data preparation. Simple data types, such as
   integers and strings, are 'ready to write' as a Python object. However, more
   complex data types often require some modification.

   For example, :class:`~django.db.models.DateField` fields use a Python
   ``datetime`` object to store data. Databases don't store ``datetime``
   objects, so the field value must be converted into an ISO-compliant date
   string for insertion into the database.

#. **Insert the data into the database.** The preprocessed, prepared data is
   composed into an SQL statement for insertion into the database.

#. **Emit a post-save signal.** The :data:`~django.db.models.signals.post_save`
   signal is sent, allowing any functions listening for that signal to do
   something.

How Django knows to UPDATE vs. INSERT
-------------------------------------

You may have noticed Django database objects use the same ``save()`` method
for creating and changing objects. Django abstracts the need to use ``INSERT``
or ``UPDATE`` SQL statements. Specifically, when you call ``save()``, Django
follows this algorithm:

* If the object's primary key attribute is set to a value that evaluates to
  ``True`` (i.e., a value other than ``None`` or the empty string), Django
  executes an ``UPDATE``.
* If the object's primary key attribute is *not* set or if the ``UPDATE``
  didn't update anything, Django executes an ``INSERT``.

The one gotcha here is that you should be careful not to specify a primary-key
value explicitly when saving new objects, if you cannot guarantee the
primary-key value is unused. For more on this nuance, see `Explicitly specifying
auto-primary-key values`_ above and `Forcing an INSERT or UPDATE`_ below.

In Django 1.5 and earlier, Django did a ``SELECT`` when the primary key
attribute was set. If the ``SELECT`` found a row, then Django did an ``UPDATE``,
otherwise it did an ``INSERT``. The old algorithm results in one more query in
the ``UPDATE`` case. There are some rare cases where the database doesn't
report that a row was updated even if the database contains a row for the
object's primary key value. An example is the PostgreSQL ``ON UPDATE`` trigger
which returns ``NULL``. In such cases it is possible to revert to the old
algorithm by setting the :attr:`~django.db.models.Options.select_on_save`
option to ``True``.

.. _ref-models-force-insert:

Forcing an INSERT or UPDATE
~~~~~~~~~~~~~~~~~~~~~~~~~~~

In some rare circumstances, it's necessary to be able to force the
:meth:`~Model.save()` method to perform an SQL ``INSERT`` and not fall back to
doing an ``UPDATE``. Or vice-versa: update, if possible, but not insert a new
row. In these cases you can pass the ``force_insert=True`` or
``force_update=True`` parameters to the :meth:`~Model.save()` method.
Obviously, passing both parameters is an error: you cannot both insert *and*
update at the same time!

It should be very rare that you'll need to use these parameters. Django will
almost always do the right thing and trying to override that will lead to
errors that are difficult to track down. This feature is for advanced use
only.

Using ``update_fields`` will force an update similarly to ``force_update``.

.. _ref-models-field-updates-using-f-expressions:

Updating attributes based on existing fields
--------------------------------------------

Sometimes you'll need to perform a simple arithmetic task on a field, such
as incrementing or decrementing the current value. The obvious way to
achieve this is to do something like::

    >>> product = Product.objects.get(name='Venezuelan Beaver Cheese')
    >>> product.number_sold += 1
    >>> product.save()

If the old ``number_sold`` value retrieved from the database was 10, then
the value of 11 will be written back to the database.

The process can be made robust, :ref:`avoiding a race condition
<avoiding-race-conditions-using-f>`, as well as slightly faster by expressing
the update relative to the original field value, rather than as an explicit
assignment of a new value. Django provides :class:`F expressions
<django.db.models.F>` for performing this kind of relative update. Using
:class:`F expressions <django.db.models.F>`, the previous example is expressed
as::

    >>> from django.db.models import F
    >>> product = Product.objects.get(name='Venezuelan Beaver Cheese')
    >>> product.number_sold = F('number_sold') + 1
    >>> product.save()

For more details, see the documentation on :class:`F expressions
<django.db.models.F>` and their :ref:`use in update queries
<topics-db-queries-update>`.

Specifying which fields to save
-------------------------------

If ``save()`` is passed a list of field names in keyword argument
``update_fields``, only the fields named in that list will be updated.
This may be desirable if you want to update just one or a few fields on
an object. There will be a slight performance benefit from preventing
all of the model fields from being updated in the database. For example::

    product.name = 'Name changed again'
    product.save(update_fields=['name'])

The ``update_fields`` argument can be any iterable containing strings. An
empty ``update_fields`` iterable will skip the save. A value of None will
perform an update on all fields.

Specifying ``update_fields`` will force an update.

When saving a model fetched through deferred model loading
(:meth:`~django.db.models.query.QuerySet.only()` or
:meth:`~django.db.models.query.QuerySet.defer()`) only the fields loaded
from the DB will get updated. In effect there is an automatic
``update_fields`` in this case. If you assign or change any deferred field
value, the field will be added to the updated fields.

Deleting objects
================

.. method:: Model.delete(using=DEFAULT_DB_ALIAS, keep_parents=False)

Issues an SQL ``DELETE`` for the object. This only deletes the object in the
database; the Python instance will still exist and will still have data in
its fields. This method returns the number of objects deleted and a dictionary
with the number of deletions per object type.

For more details, including how to delete objects in bulk, see
:ref:`topics-db-queries-delete`.

If you want customized deletion behavior, you can override the ``delete()``
method. See :ref:`overriding-model-methods` for more details.

Sometimes with :ref:`multi-table inheritance <multi-table-inheritance>` you may
want to delete only a child model's data. Specifying ``keep_parents=True`` will
keep the parent model's data.

.. versionchanged:: 1.9

    The ``keep_parents`` parameter was added.

.. versionchanged:: 1.9

    The return value describing the number of objects deleted was added.

Pickling objects
================

When you :mod:`pickle` a model, its current state is pickled. When you unpickle
it, it'll contain the model instance at the moment it was pickled, rather than
the data that's currently in the database.

.. admonition:: You can't share pickles between versions

    Pickles of models are only valid for the version of Django that
    was used to generate them. If you generate a pickle using Django
    version N, there is no guarantee that pickle will be readable with
    Django version N+1. Pickles should not be used as part of a long-term
    archival strategy.

    Since pickle compatibility errors can be difficult to diagnose, such as
    silently corrupted objects, a ``RuntimeWarning`` is raised when you try to
    unpickle a model in a Django version that is different than the one in
    which it was pickled.

.. _model-instance-methods:

Other model instance methods
============================

A few object methods have special purposes.

``__str__()``
-------------

.. method:: Model.__str__()

The ``__str__()`` method is called whenever you call ``str()`` on an object.
Django uses ``str(obj)`` in a number of places. Most notably, to display an
object in the Django admin site and as the value inserted into a template when
it displays an object. Thus, you should always return a nice, human-readable
representation of the model from the ``__str__()`` method.

For example::

    from django.db import models
    from django.utils.encoding import python_2_unicode_compatible

    @python_2_unicode_compatible  # only if you need to support Python 2
    class Person(models.Model):
        first_name = models.CharField(max_length=50)
        last_name = models.CharField(max_length=50)

        def __str__(self):
            return '%s %s' % (self.first_name, self.last_name)

If you'd like compatibility with Python 2, you can decorate your model class
with :func:`~django.utils.encoding.python_2_unicode_compatible` as shown above.

``__eq__()``
------------

.. method:: Model.__eq__()

The equality method is defined such that instances with the same primary
key value and the same concrete class are considered equal, except that
instances with a primary key value of ``None`` aren't equal to anything except
themselves. For proxy models, concrete class is defined as the model's first
non-proxy parent; for all other models it's simply the model's class.

For example::

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

The ``__hash__()`` method is based on the instance's primary key value. It
is effectively ``hash(obj.pk)``. If the instance doesn't have a primary key
value then a ``TypeError`` will be raised (otherwise the ``__hash__()``
method would return different values before and after the instance is
saved, but changing the :meth:`~object.__hash__` value of an instance is
forbidden in Python.

``get_absolute_url()``
----------------------

.. method:: Model.get_absolute_url()

Define a ``get_absolute_url()`` method to tell Django how to calculate the
canonical URL for an object. To callers, this method should appear to return a
string that can be used to refer to the object over HTTP.

For example::

    def get_absolute_url(self):
        return "/people/%i/" % self.id

While this code is correct and simple, it may not be the most portable way to
to write this kind of method. The :func:`~django.urls.reverse` function is
usually the best approach.

For example::

    def get_absolute_url(self):
        from django.urls import reverse
        return reverse('people.views.details', args=[str(self.id)])

One place Django uses ``get_absolute_url()`` is in the admin app. If an object
defines this method, the object-editing page will have a "View on site" link
that will jump you directly to the object's public view, as given by
``get_absolute_url()``.

Similarly, a couple of other bits of Django, such as the :doc:`syndication feed
framework </ref/contrib/syndication>`, use ``get_absolute_url()`` when it is
defined. If it makes sense for your model's instances to each have a unique
URL, you should define ``get_absolute_url()``.

.. warning::

    You should avoid building the URL from unvalidated user input, in order to
    reduce possibilities of link or redirect poisoning::

        def get_absolute_url(self):
            return '/%s/' % self.name

    If ``self.name`` is ``'/example.com'`` this returns ``'//example.com/'``
    which, in turn, is a valid schema relative URL but not the expected
    ``'/%2Fexample.com/'``.


It's good practice to use ``get_absolute_url()`` in templates, instead of
hard-coding your objects' URLs. For example, this template code is bad:

.. code-block:: html+django

    <!-- BAD template code. Avoid! -->
    <a href="/people/{{ object.id }}/">{{ object.name }}</a>

This template code is much better:

.. code-block:: html+django

    <a href="{{ object.get_absolute_url }}">{{ object.name }}</a>

The logic here is that if you change the URL structure of your objects, even
for something simple such as correcting a spelling error, you don't want to
have to track down every place that the URL might be created. Specify it once,
in ``get_absolute_url()`` and have all your other code call that one place.

.. note::
    The string you return from ``get_absolute_url()`` **must** contain only
    ASCII characters (required by the URI specification, :rfc:`2396`) and be
    URL-encoded, if necessary.

    Code and templates calling ``get_absolute_url()`` should be able to use the
    result directly without any further processing. You may wish to use the
    ``django.utils.encoding.iri_to_uri()`` function to help with this if you
    are using unicode strings containing characters outside the ASCII range at
    all.

Extra instance methods
======================

In addition to :meth:`~Model.save()`, :meth:`~Model.delete()`, a model object
might have some of the following methods:

.. method:: Model.get_FOO_display()

For every field that has :attr:`~django.db.models.Field.choices` set, the
object will have a ``get_FOO_display()`` method, where ``FOO`` is the name of
the field. This method returns the "human-readable" value of the field.

For example::

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

For every :class:`~django.db.models.DateField` and
:class:`~django.db.models.DateTimeField` that does not have :attr:`null=True
<django.db.models.Field.null>`, the object will have ``get_next_by_FOO()`` and
``get_previous_by_FOO()`` methods, where ``FOO`` is the name of the field. This
returns the next and previous object with respect to the date field, raising
a :exc:`~django.db.models.Model.DoesNotExist` exception when appropriate.

Both of these methods will perform their queries using the default
manager for the model. If you need to emulate filtering used by a
custom manager, or want to perform one-off custom filtering, both
methods also accept optional keyword arguments, which should be in the
format described in :ref:`Field lookups <field-lookups>`.

Note that in the case of identical date values, these methods will use the
primary key as a tie-breaker. This guarantees that no records are skipped or
duplicated. That also means you cannot use those methods on unsaved objects.

Other attributes
================

``DoesNotExist``
----------------

.. exception:: Model.DoesNotExist

    This exception is raised by the ORM in a couple places, for example by
    :meth:`QuerySet.get() <django.db.models.query.QuerySet.get>` when an object
    is not found for the given query parameters.

    Django provides a ``DoesNotExist`` exception as an attribute of each model
    class to identify the class of object that could not be found and to allow
    you to catch a particular model class with ``try/except``. The exception is
    a subclass of :exc:`django.core.exceptions.ObjectDoesNotExist`.
