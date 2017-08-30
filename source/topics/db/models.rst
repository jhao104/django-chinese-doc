=====
模型
=====

.. module:: django.db.models

模型是你的数据的唯一的、权威的信息源。它包含你所储存数据的必要字段和操作行为。通常，每个模型都对应着数据库中的唯一一张表。

基础认识:

* 每个model都是一个继承 :class:`django.db.models.Model` 的子类;

* model中的每个属性(attribute)都代表数据库中的一个字段;

* Django 提供一套自动生成的用于数据库访问的API；详见 :doc:`/topics/db/queries`。


简短例子
========

例子中的模型定义了一个带有 ``first_name`` 和 ``last_name`` 的 ``Person`` model::

    from django.db import models

    class Person(models.Model):
        first_name = models.CharField(max_length=30)
        last_name = models.CharField(max_length=30)

其中 ``first_name`` 和 ``last_name`` 都是模型的字段    ，每个字段(fields)都是类的一个属性，每个属性映射到数据库中的列。

上面的 ``Person`` model将会像下面sql语句类似的创建一张表:

.. code-block:: sql

    CREATE TABLE myapp_person (
        "id" serial NOT NULL PRIMARY KEY,
        "first_name" varchar(30) NOT NULL,
        "last_name" varchar(30) NOT NULL
    );

特别注意:

* 表的名字 ``myapp_person_name`` 是根据模型中的元数据自动生成的(app_name + class_name),也可以自定义表的名称,参考这里 :ref:`table-names`;

* ``id`` 字段是自动添加的，你可以重新改写这一行为，参见 :ref:`automatic-primary-key-fields` ;

* 本示例中的 ``CREATE TABLE SQL`` 使用的是PostgreSQL语法，但是在应用中，Django会根据 :doc:`settings 文件 </topics/settings>` 中指定的数据库类型来使用相应的SQL语句。

使用模型
========

定义好模型之后，接下来你需要告诉Django使用这些模型，你要做的就是修改配置文件中的 :setting:`INSTALLED_APPS` 设置，
在其中添加 ``models.py`` 所在应用的名称。

例如，如果你的应用的模型位于 ``myapp.models`` （文件结构是由 :djadmin:`manage.py startapp <startapp>` 命令自动创建的），
:setting:`INSTALLED_APPS` 部分看上去应该是::

    INSTALLED_APPS = [
        #...
        'myapp',
        #...
    ]

当你在 :setting:`INSTALLED_APPS` 中添加新的应用名时，一定要运行命令 :djadmin:`manage.py migrate <migrate>` ，
你可以事先使用 :djadmin:`manage.py makemigrations <makemigrations>` 给应用生成迁移脚本。


字段
=====

对于一个模型来说，最重要的是列出该模型在数据库中定义的字段。字段由models类属性指定。
要注意选择的字段名称不要和 :doc:`models API </ref/models/instances>` 冲突，比如 ``clean``、``save`` 或者 ``delete`` 等。

例子::

    from django.db import models

    class Musician(models.Model):
        first_name = models.CharField(max_length=50)
        last_name = models.CharField(max_length=50)
        instrument = models.CharField(max_length=100)

    class Album(models.Model):
        artist = models.ForeignKey(Musician, on_delete=models.CASCADE)
        name = models.CharField(max_length=100)
        release_date = models.DateField()
        num_stars = models.IntegerField()

字段类型
-----------

模型中的每个字段都是 :class:`~django.db.models.Field` 子类的某个实例,Django根据字段类的类型确定以下信息:

* 数据库中字段的类型(e.g. ``INTEGER`` , ``VARCHAR`` , ``TEXT`` );

* 渲染表单时使用的默认HTML :doc:`widget </ref/forms/widgets>` (e.g. ``<input type="text">``, ``<select>``);

* 在Django的admin和自动生成的表单中使用的最低验证要求;

Django有几十种内置字段类型；你可以在 :ref:`模型字段参考 <model-field-types>` 找到所有的字段,
如果Django内置字段类型无法满足要求，你也可以自定义字段,参见 :doc:`/howto/custom-model-fields`。

字段选项
---------

每个字段都有一些特有的参数，详见 :ref:`模型字段参考 <model-field-types>` 。
例如，:class:`~django.db.models.CharField`（和它的子类）需要 :attr:`~django.db.models.CharField.max_length`
参数来指定数据库字段 ``VARCHAR`` 的大小。

下面是所有字段类型的通用选项，它们都是可选的，:ref:`reference <common-model-field-options>` 有它们详细的介绍。下面只对最常用的一种选项快速总结：

:attr:`~Field.null`

    如果为 ``True`` ，Django将会把数据库中的空值保存为 ``NULL``。默认值为 ``False``.

:attr:`~Field.blank`

    如果为 ``True`` ，该字段允许为空值，默认为 ``False``.

    要注意，:attr:`~Field.blank` 与 :attr:`~Field.null` 不同。
    :attr:`~Field.null` 纯粹是数据库范畴,指数据库中字段内容是否允许为空，
    而 :attr:`~Field.blank` 是表单数据输入验证范畴的。如果一个字段的 :attr:`blank=True <Field.blank>` ，
    表单的验证将允许输入一个空值。如果字段的 :attr:`blank=False <Field.blank>`，该字段就是必填的。


:attr:`~Field.choices`

    由二项元组构成的一个可迭代对象（列表或元组），用来给字段提供选择项。
    如果设置了 ``choices``，默认的表单将是一个选择框而不是文本框，而且这个选择框的选项就是 ``choices`` 中的选项。
    下面是一个关于 ``choices`` 列表的例子::

        YEAR_IN_SCHOOL_CHOICES = (
            ('FR', 'Freshman'),
            ('SO', 'Sophomore'),
            ('JR', 'Junior'),
            ('SR', 'Senior'),
            ('GR', 'Graduate'),
        )

    每个元组中的第一个元素是存储在数据库中的值。第二个元素是在管理界面或
    :class:`~django.forms.ModelChoiceField` 中用作显示的内容。给定一个模型实例，
    可以使用 ``get_FOO_display()`` 方法获取一个选择字段的显示值(这里的 ``FOO`` 就是choices字段的名称 )。例如::

        from django.db import models

        class Person(models.Model):
            SHIRT_SIZES = (
                ('S', 'Small'),
                ('M', 'Medium'),
                ('L', 'Large'),
            )
            name = models.CharField(max_length=60)
            shirt_size = models.CharField(max_length=1, choices=SHIRT_SIZES)

    ::

        >>> p = Person(name="Fred Flintstone", shirt_size="L")
        >>> p.save()
        >>> p.shirt_size
        'L'
        >>> p.get_shirt_size_display()
        'Large'

:attr:`~Field.default`

    字段的默认值。这可以是一个值，也可以是可调用对象。如果可调用，那么每次创建新对象时都会被调用。

:attr:`~Field.help_text`

    表单部件额外显示的帮助内容。即使字段不在表单中使用，它对生成文档也很有用。

:attr:`~Field.primary_key`

    如果为 ``True``，那么这个字段就是模型的主键。

    如果你不在的模型中指定任何一个字段 :attr:`primary_key=True <Field.primary_key>`，
    那么Django将自动添加一个 :class:`IntegerField` 来作为主键，
    所以你不需要在任何一个字段中设置 :attr:`primary_key=True <Field.primary_key>` ，除非你想重写默认的主键行为。

    主键字段是只读的。如果你在一个已存在的对象上面更改主键的值并且保存，那么其实是创建了一个新的对象。例如::

        from django.db import models

        class Fruit(models.Model):
            name = models.CharField(max_length=100, primary_key=True)

    .. code-block:: pycon

        >>> fruit = Fruit.objects.create(name='Apple')
        >>> fruit.name = 'Pear'
        >>> fruit.save()
        >>> Fruit.objects.values_list('name', flat=True)
        ['Apple', 'Pear']

:attr:`~Field.unique`

    如果该值设置为 ``True``, 这个数据字段在整张表中必须是唯一的。

最后重申，这些只是对最常见的字段选项的简短描述，更加详细内容参见 :ref:`公共字段选项 <common-model-field-options>`。


.. _automatic-primary-key-fields:

自增主键
----------

默认情况下Django会给所有的model添加类似下面的字段::

    id = models.AutoField(primary_key=True)

这就是自增主键。

如果你想将某个字段设置为主键，在那个字段设置 :attr:`primary_key=True <Field.primary_key>` 即可，
Django发现已经有字段设置了 :attr:`primary_key=True <Field.primary_key>` 时，
将不会再自动添加 ``id`` 字段。

每个model都必须要有一个字段是 :attr:`primary_key=True <Field.primary_key>`。
如果你不添加，Django将自动添加。


.. _verbose-field-names:

字段描述名
------------

除 :class:`~django.db.models.ForeignKey` 、 :class:`~django.db.models.ManyToManyField`
和 :class:`~django.db.models.OneToOneField` 外，每个字段类型都接受一个可选的位置参数（第一个位置）——字段的描述名。
如果没有给定描述名，Django将根据字段的属性名称自动创建描述名——将属性名称的下划线替换成空格。


比如这个例子中描述名是 ``person's first name``::

    first_name = models.CharField("person's first name", max_length=30)

而没有主动设置时，则是 ``first name``::

    first_name = models.CharField(max_length=30)

:class:`~django.db.models.ForeignKey`,
:class:`~django.db.models.ManyToManyField` 和
:class:`~django.db.models.OneToOneField` 也是有描述名的，只是他们的第一个参数必须是模型类，
所以描述名需要通过关键字参数 :attr:`~Field.verbose_name` 传入::

    poll = models.ForeignKey(
        Poll,
        on_delete=models.CASCADE,
        verbose_name="the related poll",
    )
    sites = models.ManyToManyField(Site, verbose_name="list of sites")
    place = models.OneToOneField(
        Place,
        on_delete=models.CASCADE,
        verbose_name="related place",
    )

习惯上 :attr:`~Field.verbose_name` 不用大写首字母，在必要的时候Django会自动大写首字母。

关系
------

显然，关系数据库的特点在于相互关联的表。Django提供了定义三种最常见的数据库关系类型方法:多对一、多对多和一对一。

多对一
~~~~~~~~

Django使用 :class:`django.db.models.ForeignKey` 来定义一个多对一的关系,和 :class:`~django.db.models.Field` 类型一样，
在模型当中把它做为一个类属性包含进来。:class:`django.db.models.ForeignKey` 接收一个位置参数——与model关联的类。

例如:一辆汽车(Car)只有一家制造商(Manufacturer)，但是一家制造商可以生产很多辆汽车，所以可以按照这样方式来定义::

    from django.db import models

    class Manufacturer(models.Model):
        # ...
        pass

    class Car(models.Model):
        manufacturer = models.ForeignKey(Manufacturer, on_delete=models.CASCADE)
        # ...

你还可以创建 :ref:`递归关联关系 <recursive-relationships>` （对象和自己进行多对一关联）和
:ref:`与尚未定义的模型的关联关系 <lazy-relationships>`；详见。:ref:`模型字段参考 <ref-foreignkey>`

建议你用被关联的模型的小写名称做为 :class:`~django.db.models.ForeignKey` 字段的名字
（例如，上面 ``manufacturer``）。当然，你也可以起别的名字。例如::

    class Car(models.Model):
        company_that_makes_it = models.ForeignKey(
            Manufacturer,
            on_delete=models.CASCADE,
        )
        # ...

.. seealso::

    :ref:`模型字段参考 <foreign-key-arguments>` 中有详细的 :class:`~django.db.models.ForeignKey` 字段的其他参数。

    这些选项帮助定义关联关系应该如何工作；它们都是可选的参数。 访问反向关联对象的细节，请见 :ref:`反向关联示例 <backwards-related-objects>`.

    示例代码，请见 :doc:`多对一关系模型示例 </topics/db/examples/many_to_one>`.


多对多
~~~~~~~~

使用 :class:`~django.db.models.ManyToManyField` 定义多对多关系。
用法和其他 :class:`~django.db.models.Field` 字段类型一样：在模型中做为一个类属性包含进来。

:class:`~django.db.models.ManyToManyField` 需要传入一个位置参数: 与之关联的模型。

例如，一个 ``披萨`` 可以有多种 ``馅料``  ，一种 ``馅料`` 也可以位于多个 ``披萨`` 上。 如下展示::

    from django.db import models

    class Topping(models.Model):
        # ...
        pass

    class Pizza(models.Model):
        # ...
        toppings = models.ManyToManyField(Topping)

和 :class:`~django.db.models.ForeignKey` 一样, 你同样可以创建
:ref:`递归关联关系 <recursive-relationships>` (对象与自己的多对多关联) 和
:ref:`与尚未定义的模型的关联关系 <lazy-relationships>`.

建议你以被关联模型名称的复数形式做为 :class:`~django.db.models.ManyToManyField` 的名字(例如，上面的 ``toppings`` )。

:class:`~django.db.models.ManyToManyField` 设置到哪个模型中并不重要,但是你只能在两个模型中的一个设置，不能两个都设置。

通常，:class:`~django.db.models.ManyToManyField` 实例应该位于可以编辑的表单中。在上面的例子中，
``toppings`` 位于 ``Pizza`` 中（而不是在 ``Topping`` 里面设置 ``pizzas`` 的 :class:`~django.db.models.ManyToManyField` 字段），
因为设想一个 ``Pizza`` 有多种 ``Topping`` 比一个 ``Topping`` 位于多个 ``Pizza`` 上要更加自然。
按照上面的方式，在 ``Pizza`` 的表单中将允许用户选择不同的 ``Toppings``

.. seealso::

    完整的示例参见 :doc:`多对多关系模型示例 </topics/db/examples/many_to_many>`。

:class:`~django.db.models.ManyToManyField` 还接受其他参数，你可以在 :ref:`模型字段参考 <manytomany-arguments>` 中查看。
这些选项帮助定义关系应该如何工作；它们都是可选的

.. _intermediary-manytomany:

多对多关系中的其他字段
~~~~~~~~~~~~~~~~~~~~~~

处理类似搭配 ``pizza`` 和 ``topping`` 这样简单的多对多关系时，使用标准的 :class:`~django.db.models.ManyToManyField` 就可以了。
但是，有时你可能需要关联数据到两个模型之间的关系上。

例如，有这样一个应用，它记录音乐家所属的音乐小组。我们可以用一个 :class:`~django.db.models.ManyToManyField`
表示小组和成员之间的多对多关系。但是，有时你可能想知道更多成员关系的细节，比如成员是何时加入小组的。

对于这些情况，Django 允许你指定一个中介模型来定义多对多关系。 你可以将其他字段放在中介模型里面。
源模型的 :class:`~django.db.models.ManyToManyField` 字段将使用 :attr:`through <ManyToManyField.through>` 参数指向中介模型。
对于上面的音乐小组的例子，代码如下::

    from django.db import models

    class Person(models.Model):
        name = models.CharField(max_length=128)

        def __str__(self):              # __unicode__ on Python 2
            return self.name

    class Group(models.Model):
        name = models.CharField(max_length=128)
        members = models.ManyToManyField(Person, through='Membership')

        def __str__(self):              # __unicode__ on Python 2
            return self.name

    class Membership(models.Model):
        person = models.ForeignKey(Person, on_delete=models.CASCADE)
        group = models.ForeignKey(Group, on_delete=models.CASCADE)
        date_joined = models.DateField()
        invite_reason = models.CharField(max_length=64)

在设置中介模型时，要显式地指定外键并关联到多对多关系涉及的模型。这个显式声明定义两个模型之间是如何关联的。

但是中介模型还一些限制:

* 中介模型必须有且只有一个外键到源模型（上面例子中的 ``Group``），
  或者你必须使用 :attr:`ManyToManyField.through_fields <ManyToManyField.through_fields>` 显式指定Django
  应该在关系中使用的外键。

  如果你的模型中存在不止一个外键，并且 ``through_fields`` 没有指定，
  将会触发一个无效的错误。 对目标模型的外键有相同的限制（上面例子中的 Person）

* 对于通过中介模型与自己进行多对多关联的模型，允许存在到同一个模型的两个外键，
  但它们将被当做多对多关联中一个关系的两边。如果有超过两个外键，
  同样你必须像上面一样指定 ``through_fields``，否则将引发一个验证错误。

* 使用中介模型定义与自身的多对多关系时，你必须设置 :attr:`symmetrical=False <ManyToManyField.symmetrical>`
  详见 :ref:`模型字段参考 <manytomany-arguments>`。


既然你已经设置好 :class:`~django.db.models.ManyToManyField` 来使用中介模型（在这个例子中就是 ``Membership``），
接下来你要开始创建多对多关系。你要做的就是创建中介模型的实例::

    >>> ringo = Person.objects.create(name="Ringo Starr")
    >>> paul = Person.objects.create(name="Paul McCartney")
    >>> beatles = Group.objects.create(name="The Beatles")
    >>> m1 = Membership(person=ringo, group=beatles,
    ...     date_joined=date(1962, 8, 16),
    ...     invite_reason="Needed a new drummer.")
    >>> m1.save()
    >>> beatles.members.all()
    <QuerySet [<Person: Ringo Starr>]>
    >>> ringo.group_set.all()
    <QuerySet [<Group: The Beatles>]>
    >>> m2 = Membership.objects.create(person=paul, group=beatles,
    ...     date_joined=date(1960, 8, 1),
    ...     invite_reason="Wanted to form a band.")
    >>> beatles.members.all()
    <QuerySet [<Person: Ringo Starr>, <Person: Paul McCartney>]>

Unlike normal many-to-many fields, you *can't* use ``add()``, ``create()``,
or ``set()`` to create relationships::

    >>> # The following statements will not work
    >>> beatles.members.add(john)
    >>> beatles.members.create(name="George Harrison")
    >>> beatles.members.set([john, paul, ringo, george])

Why? You can't just create a relationship between a ``Person`` and a ``Group``
- you need to specify all the detail for the relationship required by the
``Membership`` model. The simple ``add``, ``create`` and assignment calls
don't provide a way to specify this extra detail. As a result, they are
disabled for many-to-many relationships that use an intermediate model.
The only way to create this type of relationship is to create instances of the
intermediate model.

The :meth:`~django.db.models.fields.related.RelatedManager.remove` method is
disabled for similar reasons. For example, if the custom through table defined
by the intermediate model does not enforce uniqueness on the
``(model1, model2)`` pair, a ``remove()`` call would not provide enough
information as to which intermediate model instance should be deleted::

    >>> Membership.objects.create(person=ringo, group=beatles,
    ...     date_joined=date(1968, 9, 4),
    ...     invite_reason="You've been gone for a month and we miss you.")
    >>> beatles.members.all()
    <QuerySet [<Person: Ringo Starr>, <Person: Paul McCartney>, <Person: Ringo Starr>]>
    >>> # This will not work because it cannot tell which membership to remove
    >>> beatles.members.remove(ringo)

However, the :meth:`~django.db.models.fields.related.RelatedManager.clear`
method can be used to remove all many-to-many relationships for an instance::

    >>> # Beatles have broken up
    >>> beatles.members.clear()
    >>> # Note that this deletes the intermediate model instances
    >>> Membership.objects.all()
    <QuerySet []>

Once you have established the many-to-many relationships by creating instances
of your intermediate model, you can issue queries. Just as with normal
many-to-many relationships, you can query using the attributes of the
many-to-many-related model::

    # Find all the groups with a member whose name starts with 'Paul'
    >>> Group.objects.filter(members__name__startswith='Paul')
    <QuerySet [<Group: The Beatles>]>

As you are using an intermediate model, you can also query on its attributes::

    # Find all the members of the Beatles that joined after 1 Jan 1961
    >>> Person.objects.filter(
    ...     group__name='The Beatles',
    ...     membership__date_joined__gt=date(1961,1,1))
    <QuerySet [<Person: Ringo Starr]>

If you need to access a membership's information you may do so by directly
querying the ``Membership`` model::

    >>> ringos_membership = Membership.objects.get(group=beatles, person=ringo)
    >>> ringos_membership.date_joined
    datetime.date(1962, 8, 16)
    >>> ringos_membership.invite_reason
    'Needed a new drummer.'

Another way to access the same information is by querying the
:ref:`many-to-many reverse relationship<m2m-reverse-relationships>` from a
``Person`` object::

    >>> ringos_membership = ringo.membership_set.get(group=beatles)
    >>> ringos_membership.date_joined
    datetime.date(1962, 8, 16)
    >>> ringos_membership.invite_reason
    'Needed a new drummer.'

One-to-one relationships
~~~~~~~~~~~~~~~~~~~~~~~~

To define a one-to-one relationship, use
:class:`~django.db.models.OneToOneField`. You use it just like any other
``Field`` type: by including it as a class attribute of your model.

This is most useful on the primary key of an object when that object "extends"
another object in some way.

:class:`~django.db.models.OneToOneField` requires a positional argument: the
class to which the model is related.

For example, if you were building a database of "places", you would
build pretty standard stuff such as address, phone number, etc. in the
database. Then, if you wanted to build a database of restaurants on
top of the places, instead of repeating yourself and replicating those
fields in the ``Restaurant`` model, you could make ``Restaurant`` have
a :class:`~django.db.models.OneToOneField` to ``Place`` (because a
restaurant "is a" place; in fact, to handle this you'd typically use
:ref:`inheritance <model-inheritance>`, which involves an implicit
one-to-one relation).

As with :class:`~django.db.models.ForeignKey`, a :ref:`recursive relationship
<recursive-relationships>` can be defined and :ref:`references to as-yet
undefined models <lazy-relationships>` can be made.

.. seealso::

    See the :doc:`One-to-one relationship model example
    </topics/db/examples/one_to_one>` for a full example.

:class:`~django.db.models.OneToOneField` fields also accept an optional
:attr:`~django.db.models.OneToOneField.parent_link` argument.

:class:`~django.db.models.OneToOneField` classes used to automatically become
the primary key on a model. This is no longer true (although you can manually
pass in the :attr:`~django.db.models.Field.primary_key` argument if you like).
Thus, it's now possible to have multiple fields of type
:class:`~django.db.models.OneToOneField` on a single model.

Models across files
-------------------

It's perfectly OK to relate a model to one from another app. To do this, import
the related model at the top of the file where your model is defined. Then,
just refer to the other model class wherever needed. For example::

    from django.db import models
    from geography.models import ZipCode

    class Restaurant(models.Model):
        # ...
        zip_code = models.ForeignKey(
            ZipCode,
            on_delete=models.SET_NULL,
            blank=True,
            null=True,
        )

Field name restrictions
-----------------------

Django places only two restrictions on model field names:

1. A field name cannot be a Python reserved word, because that would result
   in a Python syntax error. For example::

       class Example(models.Model):
           pass = models.IntegerField() # 'pass' is a reserved word!

2. A field name cannot contain more than one underscore in a row, due to
   the way Django's query lookup syntax works. For example::

       class Example(models.Model):
           foo__bar = models.IntegerField() # 'foo__bar' has two underscores!

These limitations can be worked around, though, because your field name doesn't
necessarily have to match your database column name. See the
:attr:`~Field.db_column` option.

SQL reserved words, such as ``join``, ``where`` or ``select``, *are* allowed as
model field names, because Django escapes all database table names and column
names in every underlying SQL query. It uses the quoting syntax of your
particular database engine.

Custom field types
------------------

If one of the existing model fields cannot be used to fit your purposes, or if
you wish to take advantage of some less common database column types, you can
create your own field class. Full coverage of creating your own fields is
provided in :doc:`/howto/custom-model-fields`.

.. _meta-options:

``Meta`` options
================

Give your model metadata by using an inner ``class Meta``, like so::

    from django.db import models

    class Ox(models.Model):
        horn_length = models.IntegerField()

        class Meta:
            ordering = ["horn_length"]
            verbose_name_plural = "oxen"

Model metadata is "anything that's not a field", such as ordering options
(:attr:`~Options.ordering`), database table name (:attr:`~Options.db_table`), or
human-readable singular and plural names (:attr:`~Options.verbose_name` and
:attr:`~Options.verbose_name_plural`). None are required, and adding ``class
Meta`` to a model is completely optional.

A complete list of all possible ``Meta`` options can be found in the :doc:`model
option reference </ref/models/options>`.

.. _model-attributes:

Model attributes
================

``objects``
    The most important attribute of a model is the
    :class:`~django.db.models.Manager`. It's the interface through which
    database query operations are provided to Django models and is used to
    :ref:`retrieve the instances <retrieving-objects>` from the database. If no
    custom ``Manager`` is defined, the default name is
    :attr:`~django.db.models.Model.objects`. Managers are only accessible via
    model classes, not the model instances.

.. _model-methods:

Model methods
=============

Define custom methods on a model to add custom "row-level" functionality to your
objects. Whereas :class:`~django.db.models.Manager` methods are intended to do
"table-wide" things, model methods should act on a particular model instance.

This is a valuable technique for keeping business logic in one place -- the
model.

For example, this model has a few custom methods::

    from django.db import models

    class Person(models.Model):
        first_name = models.CharField(max_length=50)
        last_name = models.CharField(max_length=50)
        birth_date = models.DateField()

        def baby_boomer_status(self):
            "Returns the person's baby-boomer status."
            import datetime
            if self.birth_date < datetime.date(1945, 8, 1):
                return "Pre-boomer"
            elif self.birth_date < datetime.date(1965, 1, 1):
                return "Baby boomer"
            else:
                return "Post-boomer"

        def _get_full_name(self):
            "Returns the person's full name."
            return '%s %s' % (self.first_name, self.last_name)
        full_name = property(_get_full_name)

The last method in this example is a :term:`property`.

The :doc:`model instance reference </ref/models/instances>` has a complete list
of :ref:`methods automatically given to each model <model-instance-methods>`.
You can override most of these -- see `overriding predefined model methods`_,
below -- but there are a couple that you'll almost always want to define:

:meth:`~Model.__str__` (Python 3)
    A Python "magic method" that returns a unicode "representation" of any
    object. This is what Python and Django will use whenever a model
    instance needs to be coerced and displayed as a plain string. Most
    notably, this happens when you display an object in an interactive
    console or in the admin.

    You'll always want to define this method; the default isn't very helpful
    at all.

``__unicode__()`` (Python 2)
    Python 2 equivalent of ``__str__()``.

:meth:`~Model.get_absolute_url`
    This tells Django how to calculate the URL for an object. Django uses
    this in its admin interface, and any time it needs to figure out a URL
    for an object.

    Any object that has a URL that uniquely identifies it should define this
    method.

.. _overriding-model-methods:

Overriding predefined model methods
-----------------------------------

There's another set of :ref:`model methods <model-instance-methods>` that
encapsulate a bunch of database behavior that you'll want to customize. In
particular you'll often want to change the way :meth:`~Model.save` and
:meth:`~Model.delete` work.

You're free to override these methods (and any other model method) to alter
behavior.

A classic use-case for overriding the built-in methods is if you want something
to happen whenever you save an object. For example (see
:meth:`~Model.save` for documentation of the parameters it accepts)::

    from django.db import models

    class Blog(models.Model):
        name = models.CharField(max_length=100)
        tagline = models.TextField()

        def save(self, *args, **kwargs):
            do_something()
            super(Blog, self).save(*args, **kwargs) # Call the "real" save() method.
            do_something_else()

You can also prevent saving::

    from django.db import models

    class Blog(models.Model):
        name = models.CharField(max_length=100)
        tagline = models.TextField()

        def save(self, *args, **kwargs):
            if self.name == "Yoko Ono's blog":
                return # Yoko shall never have her own blog!
            else:
                super(Blog, self).save(*args, **kwargs) # Call the "real" save() method.

It's important to remember to call the superclass method -- that's
that ``super(Blog, self).save(*args, **kwargs)`` business -- to ensure
that the object still gets saved into the database. If you forget to
call the superclass method, the default behavior won't happen and the
database won't get touched.

It's also important that you pass through the arguments that can be
passed to the model method -- that's what the ``*args, **kwargs`` bit
does. Django will, from time to time, extend the capabilities of
built-in model methods, adding new arguments. If you use ``*args,
**kwargs`` in your method definitions, you are guaranteed that your
code will automatically support those arguments when they are added.

.. admonition:: Overridden model methods are not called on bulk operations

    Note that the :meth:`~Model.delete()` method for an object is not
    necessarily called when :ref:`deleting objects in bulk using a
    QuerySet <topics-db-queries-delete>` or as a result of a :attr:`cascading
    delete <django.db.models.ForeignKey.on_delete>`. To ensure customized
    delete logic gets executed, you can use
    :data:`~django.db.models.signals.pre_delete` and/or
    :data:`~django.db.models.signals.post_delete` signals.

    Unfortunately, there isn't a workaround when
    :meth:`creating<django.db.models.query.QuerySet.bulk_create>` or
    :meth:`updating<django.db.models.query.QuerySet.update>` objects in bulk,
    since none of :meth:`~Model.save()`,
    :data:`~django.db.models.signals.pre_save`, and
    :data:`~django.db.models.signals.post_save` are called.

Executing custom SQL
--------------------

Another common pattern is writing custom SQL statements in model methods and
module-level methods. For more details on using raw SQL, see the documentation
on :doc:`using raw SQL</topics/db/sql>`.

.. _model-inheritance:

Model inheritance
=================

Model inheritance in Django works almost identically to the way normal
class inheritance works in Python, but the basics at the beginning of the page
should still be followed. That means the base class should subclass
:class:`django.db.models.Model`.

The only decision you have to make is whether you want the parent models to be
models in their own right (with their own database tables), or if the parents
are just holders of common information that will only be visible through the
child models.

There are three styles of inheritance that are possible in Django.

1. Often, you will just want to use the parent class to hold information that
   you don't want to have to type out for each child model. This class isn't
   going to ever be used in isolation, so :ref:`abstract-base-classes` are
   what you're after.
2. If you're subclassing an existing model (perhaps something from another
   application entirely) and want each model to have its own database table,
   :ref:`multi-table-inheritance` is the way to go.
3. Finally, if you only want to modify the Python-level behavior of a model,
   without changing the models fields in any way, you can use
   :ref:`proxy-models`.

.. _abstract-base-classes:

Abstract base classes
---------------------

Abstract base classes are useful when you want to put some common
information into a number of other models. You write your base class
and put ``abstract=True`` in the :ref:`Meta <meta-options>`
class. This model will then not be used to create any database
table. Instead, when it is used as a base class for other models, its
fields will be added to those of the child class. It is an error to
have fields in the abstract base class with the same name as those in
the child (and Django will raise an exception).

An example::

    from django.db import models

    class CommonInfo(models.Model):
        name = models.CharField(max_length=100)
        age = models.PositiveIntegerField()

        class Meta:
            abstract = True

    class Student(CommonInfo):
        home_group = models.CharField(max_length=5)

The ``Student`` model will have three fields: ``name``, ``age`` and
``home_group``. The ``CommonInfo`` model cannot be used as a normal Django
model, since it is an abstract base class. It does not generate a database
table or have a manager, and cannot be instantiated or saved directly.

For many uses, this type of model inheritance will be exactly what you want.
It provides a way to factor out common information at the Python level, while
still only creating one database table per child model at the database level.

``Meta`` inheritance
~~~~~~~~~~~~~~~~~~~~

When an abstract base class is created, Django makes any :ref:`Meta <meta-options>`
inner class you declared in the base class available as an
attribute. If a child class does not declare its own :ref:`Meta <meta-options>`
class, it will inherit the parent's :ref:`Meta <meta-options>`. If the child wants to
extend the parent's :ref:`Meta <meta-options>` class, it can subclass it. For example::

    from django.db import models

    class CommonInfo(models.Model):
        # ...
        class Meta:
            abstract = True
            ordering = ['name']

    class Student(CommonInfo):
        # ...
        class Meta(CommonInfo.Meta):
            db_table = 'student_info'

Django does make one adjustment to the :ref:`Meta <meta-options>` class of an abstract base
class: before installing the :ref:`Meta <meta-options>` attribute, it sets ``abstract=False``.
This means that children of abstract base classes don't automatically become
abstract classes themselves. Of course, you can make an abstract base class
that inherits from another abstract base class. You just need to remember to
explicitly set ``abstract=True`` each time.

Some attributes won't make sense to include in the :ref:`Meta <meta-options>` class of an
abstract base class. For example, including ``db_table`` would mean that all
the child classes (the ones that don't specify their own :ref:`Meta <meta-options>`) would use
the same database table, which is almost certainly not what you want.

.. _abstract-related-name:

Be careful with ``related_name`` and ``related_query_name``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you are using :attr:`~django.db.models.ForeignKey.related_name` or
:attr:`~django.db.models.ForeignKey.related_query_name` on a ``ForeignKey`` or
``ManyToManyField``, you must always specify a *unique* reverse name and query
name for the field. This would normally cause a problem in abstract base
classes, since the fields on this class are included into each of the child
classes, with exactly the same values for the attributes (including
:attr:`~django.db.models.ForeignKey.related_name` and
:attr:`~django.db.models.ForeignKey.related_query_name`) each time.

To work around this problem, when you are using
:attr:`~django.db.models.ForeignKey.related_name` or
:attr:`~django.db.models.ForeignKey.related_query_name` in an abstract base
class (only), part of the value should contain ``'%(app_label)s'`` and
``'%(class)s'``.

- ``'%(class)s'`` is replaced by the lower-cased name of the child class
  that the field is used in.
- ``'%(app_label)s'`` is replaced by the lower-cased name of the app the child
  class is contained within. Each installed application name must be unique
  and the model class names within each app must also be unique, therefore the
  resulting name will end up being different.

For example, given an app ``common/models.py``::

    from django.db import models

    class Base(models.Model):
        m2m = models.ManyToManyField(
            OtherModel,
            related_name="%(app_label)s_%(class)s_related",
            related_query_name="%(app_label)s_%(class)ss",
        )

        class Meta:
            abstract = True

    class ChildA(Base):
        pass

    class ChildB(Base):
        pass

Along with another app ``rare/models.py``::

    from common.models import Base

    class ChildB(Base):
        pass

The reverse name of the ``common.ChildA.m2m`` field will be
``common_childa_related`` and the reverse query name will be ``common_childas``.
The reverse name of the ``common.ChildB.m2m`` field will be
``common_childb_related`` and the reverse query name will be
``common_childbs``. Finally, the reverse name of the ``rare.ChildB.m2m`` field
will be ``rare_childb_related`` and the reverse query name will be
``rare_childbs``. It's up to you how you use the ``'%(class)s'`` and
``'%(app_label)s'`` portion to construct your related name or related query name
but if you forget to use it, Django will raise errors when you perform system
checks (or run :djadmin:`migrate`).

If you don't specify a :attr:`~django.db.models.ForeignKey.related_name`
attribute for a field in an abstract base class, the default reverse name will
be the name of the child class followed by ``'_set'``, just as it normally
would be if you'd declared the field directly on the child class. For example,
in the above code, if the :attr:`~django.db.models.ForeignKey.related_name`
attribute was omitted, the reverse name for the ``m2m`` field would be
``childa_set`` in the ``ChildA`` case and ``childb_set`` for the ``ChildB``
field.

.. versionchanged:: 1.10

   Interpolation of  ``'%(app_label)s'`` and ``'%(class)s'`` for
   ``related_query_name`` was added.

.. _multi-table-inheritance:

Multi-table inheritance
-----------------------

The second type of model inheritance supported by Django is when each model in
the hierarchy is a model all by itself. Each model corresponds to its own
database table and can be queried and created individually. The inheritance
relationship introduces links between the child model and each of its parents
(via an automatically-created :class:`~django.db.models.OneToOneField`).
For example::

    from django.db import models

    class Place(models.Model):
        name = models.CharField(max_length=50)
        address = models.CharField(max_length=80)

    class Restaurant(Place):
        serves_hot_dogs = models.BooleanField(default=False)
        serves_pizza = models.BooleanField(default=False)

All of the fields of ``Place`` will also be available in ``Restaurant``,
although the data will reside in a different database table. So these are both
possible::

    >>> Place.objects.filter(name="Bob's Cafe")
    >>> Restaurant.objects.filter(name="Bob's Cafe")

If you have a ``Place`` that is also a ``Restaurant``, you can get from the
``Place`` object to the ``Restaurant`` object by using the lower-case version
of the model name::

    >>> p = Place.objects.get(id=12)
    # If p is a Restaurant object, this will give the child class:
    >>> p.restaurant
    <Restaurant: ...>

However, if ``p`` in the above example was *not* a ``Restaurant`` (it had been
created directly as a ``Place`` object or was the parent of some other class),
referring to ``p.restaurant`` would raise a ``Restaurant.DoesNotExist``
exception.

.. _meta-and-multi-table-inheritance:

``Meta`` and multi-table inheritance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the multi-table inheritance situation, it doesn't make sense for a child
class to inherit from its parent's :ref:`Meta <meta-options>` class. All the :ref:`Meta <meta-options>` options
have already been applied to the parent class and applying them again would
normally only lead to contradictory behavior (this is in contrast with the
abstract base class case, where the base class doesn't exist in its own
right).

So a child model does not have access to its parent's :ref:`Meta
<meta-options>` class. However, there are a few limited cases where the child
inherits behavior from the parent: if the child does not specify an
:attr:`~django.db.models.Options.ordering` attribute or a
:attr:`~django.db.models.Options.get_latest_by` attribute, it will inherit
these from its parent.

If the parent has an ordering and you don't want the child to have any natural
ordering, you can explicitly disable it::

    class ChildModel(ParentModel):
        # ...
        class Meta:
            # Remove parent's ordering effect
            ordering = []

Inheritance and reverse relations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Because multi-table inheritance uses an implicit
:class:`~django.db.models.OneToOneField` to link the child and
the parent, it's possible to move from the parent down to the child,
as in the above example. However, this uses up the name that is the
default :attr:`~django.db.models.ForeignKey.related_name` value for
:class:`~django.db.models.ForeignKey` and
:class:`~django.db.models.ManyToManyField` relations.  If you
are putting those types of relations on a subclass of the parent model, you
**must** specify the :attr:`~django.db.models.ForeignKey.related_name`
attribute on each such field. If you forget, Django will raise a validation
error.

For example, using the above ``Place`` class again, let's create another
subclass with a :class:`~django.db.models.ManyToManyField`::

    class Supplier(Place):
        customers = models.ManyToManyField(Place)

This results in the error::

    Reverse query name for 'Supplier.customers' clashes with reverse query
    name for 'Supplier.place_ptr'.

    HINT: Add or change a related_name argument to the definition for
    'Supplier.customers' or 'Supplier.place_ptr'.

Adding ``related_name`` to the ``customers`` field as follows would resolve the
error: ``models.ManyToManyField(Place, related_name='provider')``.

Specifying the parent link field
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As mentioned, Django will automatically create a
:class:`~django.db.models.OneToOneField` linking your child
class back to any non-abstract parent models. If you want to control the
name of the attribute linking back to the parent, you can create your
own :class:`~django.db.models.OneToOneField` and set
:attr:`parent_link=True <django.db.models.OneToOneField.parent_link>`
to indicate that your field is the link back to the parent class.

.. _proxy-models:

Proxy models
------------

When using :ref:`multi-table inheritance <multi-table-inheritance>`, a new
database table is created for each subclass of a model. This is usually the
desired behavior, since the subclass needs a place to store any additional
data fields that are not present on the base class. Sometimes, however, you
only want to change the Python behavior of a model -- perhaps to change the
default manager, or add a new method.

This is what proxy model inheritance is for: creating a *proxy* for the
original model. You can create, delete and update instances of the proxy model
and all the data will be saved as if you were using the original (non-proxied)
model. The difference is that you can change things like the default model
ordering or the default manager in the proxy, without having to alter the
original.

Proxy models are declared like normal models. You tell Django that it's a
proxy model by setting the :attr:`~django.db.models.Options.proxy` attribute of
the ``Meta`` class to ``True``.

For example, suppose you want to add a method to the ``Person`` model. You can do it like this::

    from django.db import models

    class Person(models.Model):
        first_name = models.CharField(max_length=30)
        last_name = models.CharField(max_length=30)

    class MyPerson(Person):
        class Meta:
            proxy = True

        def do_something(self):
            # ...
            pass

The ``MyPerson`` class operates on the same database table as its parent
``Person`` class. In particular, any new instances of ``Person`` will also be
accessible through ``MyPerson``, and vice-versa::

    >>> p = Person.objects.create(first_name="foobar")
    >>> MyPerson.objects.get(first_name="foobar")
    <MyPerson: foobar>

You could also use a proxy model to define a different default ordering on
a model. You might not always want to order the ``Person`` model, but regularly
order by the ``last_name`` attribute when you use the proxy. This is easy::

    class OrderedPerson(Person):
        class Meta:
            ordering = ["last_name"]
            proxy = True

Now normal ``Person`` queries will be unordered
and ``OrderedPerson`` queries will be ordered by ``last_name``.

Proxy models inherit ``Meta`` attributes :ref:`in the same way as regular
models <meta-and-multi-table-inheritance>`.

``QuerySet``\s still return the model that was requested
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There is no way to have Django return, say, a ``MyPerson`` object whenever you
query for ``Person`` objects. A queryset for ``Person`` objects will return
those types of objects. The whole point of proxy objects is that code relying
on the original ``Person`` will use those and your own code can use the
extensions you included (that no other code is relying on anyway). It is not
a way to replace the ``Person`` (or any other) model everywhere with something
of your own creation.

Base class restrictions
~~~~~~~~~~~~~~~~~~~~~~~

A proxy model must inherit from exactly one non-abstract model class. You
can't inherit from multiple non-abstract models as the proxy model doesn't
provide any connection between the rows in the different database tables. A
proxy model can inherit from any number of abstract model classes, providing
they do *not* define any model fields. A proxy model may also inherit from any
number of proxy models that share a common non-abstract parent class.

.. versionchanged:: 1.10

    In earlier versions, a proxy model couldn't inherit more than one proxy
    model that shared the same parent class.

Proxy model managers
~~~~~~~~~~~~~~~~~~~~

If you don't specify any model managers on a proxy model, it inherits the
managers from its model parents. If you define a manager on the proxy model,
it will become the default, although any managers defined on the parent
classes will still be available.

Continuing our example from above, you could change the default manager used
when you query the ``Person`` model like this::

    from django.db import models

    class NewManager(models.Manager):
        # ...
        pass

    class MyPerson(Person):
        objects = NewManager()

        class Meta:
            proxy = True

If you wanted to add a new manager to the Proxy, without replacing the
existing default, you can use the techniques described in the :ref:`custom
manager <custom-managers-and-inheritance>` documentation: create a base class
containing the new managers and inherit that after the primary base class::

    # Create an abstract class for the new manager.
    class ExtraManagers(models.Model):
        secondary = NewManager()

        class Meta:
            abstract = True

    class MyPerson(Person, ExtraManagers):
        class Meta:
            proxy = True

You probably won't need to do this very often, but, when you do, it's
possible.

.. _proxy-vs-unmanaged-models:

Differences between proxy inheritance and unmanaged models
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Proxy model inheritance might look fairly similar to creating an unmanaged
model, using the :attr:`~django.db.models.Options.managed` attribute on a
model's ``Meta`` class.

With careful setting of :attr:`Meta.db_table
<django.db.models.Options.db_table>` you could create an unmanaged model that
shadows an existing model and adds Python methods to it. However, that would be
very repetitive and fragile as you need to keep both copies synchronized if you
make any changes.

On the other hand, proxy models are intended to behave exactly like the model
they are proxying for. They are always in sync with the parent model since they
directly inherit its fields and managers.

The general rules are:

1. If you are mirroring an existing model or database table and don't want
   all the original database table columns, use ``Meta.managed=False``.
   That option is normally useful for modeling database views and tables
   not under the control of Django.
2. If you are wanting to change the Python-only behavior of a model, but
   keep all the same fields as in the original, use ``Meta.proxy=True``.
   This sets things up so that the proxy model is an exact copy of the
   storage structure of the original model when data is saved.

.. _model-multiple-inheritance-topic:

Multiple inheritance
--------------------

Just as with Python's subclassing, it's possible for a Django model to inherit
from multiple parent models. Keep in mind that normal Python name resolution
rules apply. The first base class that a particular name (e.g. :ref:`Meta
<meta-options>`) appears in will be the one that is used; for example, this
means that if multiple parents contain a :ref:`Meta <meta-options>` class,
only the first one is going to be used, and all others will be ignored.

Generally, you won't need to inherit from multiple parents. The main use-case
where this is useful is for "mix-in" classes: adding a particular extra
field or method to every class that inherits the mix-in. Try to keep your
inheritance hierarchies as simple and straightforward as possible so that you
won't have to struggle to work out where a particular piece of information is
coming from.

Note that inheriting from multiple models that have a common ``id`` primary
key field will raise an error. To properly use multiple inheritance, you can
use an explicit :class:`~django.db.models.AutoField` in the base models::

    class Article(models.Model):
        article_id = models.AutoField(primary_key=True)
        ...

    class Book(models.Model):
        book_id = models.AutoField(primary_key=True)
        ...

    class BookReview(Book, Article):
        pass

Or use a common ancestor to hold the :class:`~django.db.models.AutoField`::

    class Piece(models.Model):
        pass

    class Article(Piece):
        ...

    class Book(Piece):
        ...

    class BookReview(Book, Article):
        pass

Field name "hiding" is not permitted
-------------------------------------

In normal Python class inheritance, it is permissible for a child class to
override any attribute from the parent class. In Django, this isn't usually
permitted for model fields. If a non-abstract model base class has a field
called ``author``, you can't create another model field or define
an attribute called ``author`` in any class that inherits from that base class.

This restriction doesn't apply to model fields inherited from an abstract
model. Such fields may be overridden with another field or value, or be removed
by setting ``field_name = None``.

.. versionchanged:: 1.10

    The ability to override abstract fields was added.

.. warning::

    Model managers are inherited from abstract base classes. Overriding an
    inherited field which is referenced by an inherited
    :class:`~django.db.models.Manager` may cause subtle bugs. See :ref:`custom
    managers and model inheritance <custom-managers-and-inheritance>`.

.. note::

    Some fields define extra attributes on the model, e.g. a
    :class:`~django.db.models.ForeignKey` defines an extra attribute with
    ``_id`` appended to the field name, as well as ``related_name`` and
    ``related_query_name`` on the foreign model.

    These extra attributes cannot be overridden unless the field that defines
    it is changed or removed so that it no longer defines the extra attribute.

Overriding fields in a parent model leads to difficulties in areas such as
initializing new instances (specifying which field is being initialized in
``Model.__init__``) and serialization. These are features which normal Python
class inheritance doesn't have to deal with in quite the same way, so the
difference between Django model inheritance and Python class inheritance isn't
arbitrary.

This restriction only applies to attributes which are
:class:`~django.db.models.Field` instances. Normal Python attributes
can be overridden if you wish. It also only applies to the name of the
attribute as Python sees it: if you are manually specifying the database
column name, you can have the same column name appearing in both a child and
an ancestor model for multi-table inheritance (they are columns in two
different database tables).

Django will raise a :exc:`~django.core.exceptions.FieldError` if you override
any model field in any ancestor model.

Organizing models in a package
==============================

The :djadmin:`manage.py startapp <startapp>` command creates an application
structure that includes a ``models.py`` file. If you have many models,
organizing them in separate files may be useful.

To do so, create a ``models`` package. Remove ``models.py`` and create a
``myapp/models/`` directory with an ``__init__.py`` file and the files to
store your models. You must import the models in the ``__init__.py`` file.

For example, if you had ``organic.py`` and ``synthetic.py`` in the ``models``
directory:

.. snippet::
    :filename: myapp/models/__init__.py

    from .organic import Person
    from .synthetic import Robot

Explicitly importing each model rather than using ``from .models import *``
has the advantages of not cluttering the namespace, making code more readable,
and keeping code analysis tools useful.

.. seealso::

    :doc:`The Models Reference </ref/models/index>`
        Covers all the model related APIs including model fields, related
        objects, and ``QuerySet``.
