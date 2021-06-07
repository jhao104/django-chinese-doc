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

从数据库刷新对象
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

Note that when deferred fields are accessed, the loading of the deferred
field's value happens through this method. Thus it is possible to customize
the way deferred loading happens. The example below shows how one can reload
all of the instance's fields when a deferred field is reloaded::

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

A helper method that returns a set containing the attribute names of all those
fields that are currently deferred for this model.

.. _validating-objects:

Validating objects
==================

There are three steps involved in validating a model:

1. Validate the model fields - :meth:`Model.clean_fields()`
2. Validate the model as a whole - :meth:`Model.clean()`
3. Validate the field uniqueness - :meth:`Model.validate_unique()`

All three steps are performed when you call a model's
:meth:`~Model.full_clean()` method.

When you use a :class:`~django.forms.ModelForm`, the call to
:meth:`~django.forms.Form.is_valid()` will perform these validation steps for
all the fields that are included on the form. See the :doc:`ModelForm
documentation </topics/forms/modelforms>` for more information. You should only
need to call a model's :meth:`~Model.full_clean()` method if you plan to handle
validation errors yourself, or if you have excluded fields from the
:class:`~django.forms.ModelForm` that require validation.

.. method:: Model.full_clean(exclude=None, validate_unique=True)

This method calls :meth:`Model.clean_fields()`, :meth:`Model.clean()`, and
:meth:`Model.validate_unique()` (if ``validate_unique`` is ``True``), in that
order and raises a :exc:`~django.core.exceptions.ValidationError` that has a
``message_dict`` attribute containing errors from all three stages.

The optional ``exclude`` argument can be used to provide a list of field names
that can be excluded from validation and cleaning.
:class:`~django.forms.ModelForm` uses this argument to exclude fields that
aren't present on your form from being validated since any errors raised could
not be corrected by the user.

Note that ``full_clean()`` will *not* be called automatically when you call
your model's :meth:`~Model.save()` method. You'll need to call it manually
when you want to run one-step model validation for your own manually created
models. For example::

    from django.core.exceptions import ValidationError
    try:
        article.full_clean()
    except ValidationError as e:
        # Do something based on the errors contained in e.message_dict.
        # Display them to a user, or handle them programmatically.
        pass

The first step ``full_clean()`` performs is to clean each individual field.

.. method:: Model.clean_fields(exclude=None)

This method will validate all fields on your model. The optional ``exclude``
argument lets you provide a list of field names to exclude from validation. It
will raise a :exc:`~django.core.exceptions.ValidationError` if any fields fail
validation.

The second step ``full_clean()`` performs is to call :meth:`Model.clean()`.
This method should be overridden to perform custom validation on your model.

.. method:: Model.clean()

This method should be used to provide custom model validation, and to modify
attributes on your model if desired. For instance, you could use it to
automatically provide a value for a field, or to do validation that requires
access to more than a single field::

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

Note, however, that like :meth:`Model.full_clean()`, a model's ``clean()``
method is not invoked when you call your model's :meth:`~Model.save()` method.

In the above example, the :exc:`~django.core.exceptions.ValidationError`
exception raised by ``Model.clean()`` was instantiated with a string, so it
will be stored in a special error dictionary key,
:data:`~django.core.exceptions.NON_FIELD_ERRORS`. This key is used for errors
that are tied to the entire model instead of to a specific field::

    from django.core.exceptions import ValidationError, NON_FIELD_ERRORS
    try:
        article.full_clean()
    except ValidationError as e:
        non_field_errors = e.message_dict[NON_FIELD_ERRORS]

To assign exceptions to a specific field, instantiate the
:exc:`~django.core.exceptions.ValidationError` with a dictionary, where the
keys are the field names. We could update the previous example to assign the
error to the ``pub_date`` field::

    class Article(models.Model):
        ...
        def clean(self):
            # Don't allow draft entries to have a pub_date.
            if self.status == 'draft' and self.pub_date is not None:
                raise ValidationError({'pub_date': _('Draft entries may not have a publication date.')})
            ...

If you detect errors in multiple fields during ``Model.clean()``, you can also
pass a dictionary mapping field names to errors::

    raise ValidationError({
        'title': ValidationError(_('Missing title.'), code='required'),
        'pub_date': ValidationError(_('Invalid date.'), code='invalid'),
    })

Finally, ``full_clean()`` will check any unique constraints on your model.

.. method:: Model.validate_unique(exclude=None)

This method is similar to :meth:`~Model.clean_fields`, but validates all
uniqueness constraints on your model instead of individual field values. The
optional ``exclude`` argument allows you to provide a list of field names to
exclude from validation. It will raise a
:exc:`~django.core.exceptions.ValidationError` if any fields fail validation.

Note that if you provide an ``exclude`` argument to ``validate_unique()``, any
:attr:`~django.db.models.Options.unique_together` constraint involving one of
the fields you provided will not be checked.


Saving objects
==============

To save an object back to the database, call ``save()``:

.. method:: Model.save(force_insert=False, force_update=False, using=DEFAULT_DB_ALIAS, update_fields=None)

If you want customized saving behavior, you can override this ``save()``
method. See :ref:`overriding-model-methods` for more details.

The model save process also has some subtleties; see the sections below.

Auto-incrementing primary keys
------------------------------

If a model has an :class:`~django.db.models.AutoField` — an auto-incrementing
primary key — then that auto-incremented value will be calculated and saved as
an attribute on your object the first time you call ``save()``::

    >>> b2 = Blog(name='Cheddar Talk', tagline='Thoughts on cheese.')
    >>> b2.id     # Returns None, because b doesn't have an ID yet.
    >>> b2.save()
    >>> b2.id     # Returns the ID of your new object.

There's no way to tell what the value of an ID will be before you call
``save()``, because that value is calculated by your database, not by Django.

For convenience, each model has an :class:`~django.db.models.AutoField` named
``id`` by default unless you explicitly specify ``primary_key=True`` on a field
in your model. See the documentation for :class:`~django.db.models.AutoField`
for more details.

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
