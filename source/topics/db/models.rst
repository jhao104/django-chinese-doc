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

与普通的多对多字段不同，你不能使用 ``add()`` 、 ``create()`` 和 ``set()`` 语句来创建关系::

    >>> # The following statements will not work
    >>> beatles.members.add(john)
    >>> beatles.members.create(name="George Harrison")
    >>> beatles.members.set([john, paul, ringo, george])

为什么不能这样做？ 这是因为你不能只创建 ``Person`` 和 ``Group`` 之间的关联关系，
你还要指定 ``Membership`` 模型中所需要的所有信息；而简单的 ``add``、 ``create`` 和赋值语句是做不到这一点的。
所以它们不能在使用中介模型的多对多关系中使用。此时，唯一的办法就是创建中介模型的实例。

:meth:`~django.db.models.fields.related.RelatedManager.remove` 方法被禁用也是出于同样的原因。

如果中间模型定义的自定义表在 ``(model1,model2)`` 的执行不存在唯一性，则 ``remove()`` 调用将不能提供足够的信息来删除中间模型实例::

    >>> Membership.objects.create(person=ringo, group=beatles,
    ...     date_joined=date(1968, 9, 4),
    ...     invite_reason="You've been gone for a month and we miss you.")
    >>> beatles.members.all()
    <QuerySet [<Person: Ringo Starr>, <Person: Paul McCartney>, <Person: Ringo Starr>]>
    >>> # This will not work because it cannot tell which membership to remove
    >>> beatles.members.remove(ringo)

但是 :meth:`~django.db.models.fields.related.RelatedManager.clear` 方法却是可用的。它可以清空某个实例所有的多对多关系::

    >>> # Beatles have broken up
    >>> beatles.members.clear()
    >>> # Note that this deletes the intermediate model instances
    >>> Membership.objects.all()
    <QuerySet []>

只要通过创建中介模型的实例来建立对多对多关系后，你就可以执行查询了。
查询和普通的多对多字段一样，你可以直接使用被关联模型的属性进行查询：::

    # Find all the groups with a member whose name starts with 'Paul'
    >>> Group.objects.filter(members__name__startswith='Paul')
    <QuerySet [<Group: The Beatles>]>

如果你使用了中介模型，你也可以利用中介模型的属性进行查询::

    # Find all the members of the Beatles that joined after 1 Jan 1961
    >>> Person.objects.filter(
    ...     group__name='The Beatles',
    ...     membership__date_joined__gt=date(1961,1,1))
    <QuerySet [<Person: Ringo Starr]>

如果你需要访问一个成员的信息，你可以直接获取 ``Membership`` 模型::

    >>> ringos_membership = Membership.objects.get(group=beatles, person=ringo)
    >>> ringos_membership.date_joined
    datetime.date(1962, 8, 16)
    >>> ringos_membership.invite_reason
    'Needed a new drummer.'

另一种获取相同信息的方法是，在 ``Person`` 对象上查询 :ref:`多对多反转关系 <m2m-reverse-relationships>`::

    >>> ringos_membership = ringo.membership_set.get(group=beatles)
    >>> ringos_membership.date_joined
    datetime.date(1962, 8, 16)
    >>> ringos_membership.invite_reason
    'Needed a new drummer.'

一对一
~~~~~~~~~

使用 :class:`~django.db.models.OneToOneField` 来定义一对一关系。 用法和其他字段类型一样：在模型里面做为类属性包含进来。

当某个对象想扩展自另一个对象时，最常用的方式就是在这个对象的主键上添加一对一关系。

:class:`~django.db.models.OneToOneField` 接收一个位置参数：与之关联的模型。

例如，你想建一个“places” 数据库，里面有一些常用的字段，比如address、 phone number 等等。
接下来，如果你想在Place 数据库的基础上建立一个 ``Restaurant`` 数据库，而不想将已有的字段复制到Restaurant模型，
那你可以在 ``Restaurant`` 添加一个 :class:`~django.db.models.OneToOneField` 字段，
这个字段指向 ``Place`` （因为Restaurant 本身就是一个Place；事实上，在处理这个问题的时候，你应该使用 :ref:`继承 <model-inheritance>`，它隐含一个一对一关系)

和使用 :class:`~django.db.models.ForeignKey` 一样，你可以定义 :ref:`递归的关联关系 <recursive-relationships>` 和
:ref:`引用尚未定义的模型 <lazy-relationships>` 。

.. seealso::

    在 :doc:`一对一关系模型例子
    </topics/db/examples/one_to_one>` 中有完整例子。

:class:`~django.db.models.OneToOneField` 字段同时还接收一个可选的
:attr:`~django.db.models.OneToOneField.parent_link` 参数。

在以前的版本中，:class:`~django.db.models.OneToOneField` 字段会自动变成模型的 :attr:`~django.db.models.Field.primary_key`。
不过现在已经不这么做了(不过要是你愿意的话，你仍可以传递 :attr:`~django.db.models.Field.primary_key` 参数来创建主键字段)。
所以一个模型中可以有多个 :class:`~django.db.models.OneToOneField` 字段


跨文件的模型
-------------

也可以将一个模型与另一个应用程序中的一个模型关联。为此，在定义模型的文件顶部导入相关模型。然后，
只需在需要的地方引用其模型类即可。例如::

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

字段名称限制
-------------

Django对模型字段名有两个限制:

1. 字段名不能是Python保留字，因为这会导致Python语法错误。例如::

       class Example(models.Model):
           pass = models.IntegerField() # 'pass' is a reserved word!

2. 由于与Django的查询语法的工作方式冲突，字段名不能包含多个下划线。例如::

       class Example(models.Model):
           foo__bar = models.IntegerField() # 'foo__bar' has two underscores!

这些限制有变通的方法，因为没有要求字段名称必须与数据库的列名匹配。 参见
:attr:`~Field.db_column` 选项。

SQL 的保留字例如 ``join`` 、 ``where`` 和 ``select`` ，可以用作模型的字段名，
因为Django 会对底层的SQL 查询语句中的数据库表名和列名进行转义。 它根据你的数据库引擎使用不同的引用语法。

自定义字段类型
--------------

如果已有的模型字段都不合适，或者你想用到一些很少见的数据库列类型的优点，
你可以创建你自己的字段类型。创建你自己的字段在 :doc:`/howto/custom-model-fields` 中有完整讲述。


.. _meta-options:

``Meta`` 选项
================

使用内部的 ``class Meta`` 定义模型的元数据，例如::

    from django.db import models

    class Ox(models.Model):
        horn_length = models.IntegerField()

        class Meta:
            ordering = ["horn_length"]
            verbose_name_plural = "oxen"

模型元数据是“任何不是字段的数据”，比如排序选项
(:attr:`~Options.ordering`), 数据库表名 (:attr:`~Options.db_table`), 或者
人类可读的单复数名称 (:attr:`~Options.verbose_name` 和
:attr:`~Options.verbose_name_plural`)。在模型中添加 ``class Meta`` 是可选的，所有选项都不是必须的。

``Meta`` 选项的完整列表可以在 :doc:`模型选项参考 </ref/models/options>` 中找到。

.. _model-attributes:

模型属性
==========

``objects``
    模型最重要的属性是 :class:`~django.db.models.Manager` 。它是Django 模型进行数据库查询操作的接口，
    并用于从数据库 :ref:`获取实例 <retrieving-objects>` 。 如果没有定义自定义 ``Manager`` ，
    则默认名称为 :attr:`~django.db.models.Model.objects` 。Managers只能通过模型类访问，而不是模型实例。

.. _model-methods:

模型方法
=========

通过在模型中添加自定义方法来给对象添加“row级别”的功能。而 :class:`~django.db.models.Manager` 方法的目的是做“table范围”的事情，
模型方法应该着眼于一个特定的模型实例。

这是一个非常有价值的技术，让业务逻辑位于同一个地方 —— 模型中。

例如，下面的模型具有一些自定义的方法::

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

这个例子中的最后一个方法是一个 :term:`property` 。

在 :doc:`模型实例参考 </ref/models/instances>` 中有完整的
:ref:`为模型自动生成的方法 <model-instance-methods>` 列表。
你可以重写其中的大部分 -- 参考 `重载预定义模型方法`_ ,
但是有些方法你会始终想要重新定义:

:meth:`~Model.__str__` (Python 3)

    Python的“魔法方法”，它返回任何对象的unicode“表示” 。
    这就是Python和Django在模型实例需要被显示和强制转为为普通字符串时将使用的内容。
    最明显是在交互式控制台或者管理站点显示一个对象的时候。

    你会经常需要重新定义这个方法；默认方法返回的内容几乎没有帮助。

``__unicode__()`` (Python 2)

    Python 2 中使用，相当于 ``__str__()`` 。

:meth:`~Model.get_absolute_url`

    它告诉Django 如何计算一个对象的URL。Django 在它的管理站点中使用到这个方法，
    在其它任何需要计算一个对象的URL 时也将用到。

    任何具有唯一标识自己的URL 的对象都应该定义这个方法。

.. _overriding-model-methods:

重载预定义模型方法
------------------

还有另外一部分封装数据库行为的 :ref:`模型方法 <model-instance-methods>` ，
你可能想要自定义它们。特别是，你将要经常改变 :meth:`~Model.save` 和 :meth:`~Model.delete` 的工作方式。


您可以自由地重写这些方法(以及任何其他模型方法)来改变行为。

重写内置方法的经典用例是，如果您想要在保存对象时做些其他事情。例如，(请参阅 :meth:`~Model.save` 文档以获得它所接受的参数)::

    from django.db import models

    class Blog(models.Model):
        name = models.CharField(max_length=100)
        tagline = models.TextField()

        def save(self, *args, **kwargs):
            do_something()
            super(Blog, self).save(*args, **kwargs) # Call the "real" save() method.
            do_something_else()

你还可以阻止保存::

    from django.db import models

    class Blog(models.Model):
        name = models.CharField(max_length=100)
        tagline = models.TextField()

        def save(self, *args, **kwargs):
            if self.name == "Yoko Ono's blog":
                return # Yoko shall never have her own blog!
            else:
                super(Blog, self).save(*args, **kwargs) # Call the "real" save() method.

必须要记住调用超类的方法——  ``super(Blog, self).save(*args, **kwargs)``  —— 来确保对象被保存到数据库中。
如果你忘记调用超类的这个方法，默认的行为将不会发生且数据库不会有任何改变。

还要记住传递参数给这个模型方法 —— 即 ``*args, **kwargs`` 。
Django 在未来一直会扩展内建模型方法的功能并添加新的参数。
如果在你的方法定义中使用 ``*args,
**kwargs`` 将保证你的代码自动支持这些新的参数。

.. admonition:: 批量操作中被重载的模型方法不会被调用

    注意，当 :ref:`使用查询集批量删除对象 <topics-db-queries-delete>` 或者使用 :attr:`cascading
    delete <django.db.models.ForeignKey.on_delete>` 时，将不会为每个对象调用
    :meth:`~Model.delete()` 方法。为确保自定义的删除逻辑得到执行，你可以使用
    :data:`~django.db.models.signals.pre_delete` and/or
    :data:`~django.db.models.signals.post_delete` 信号。

    不幸的是，当批量
    :meth:`creating<django.db.models.query.QuerySet.bulk_create>` or
    :meth:`updating<django.db.models.query.QuerySet.update>` 对象时没有变通方法，
    因为不会调用 :meth:`~Model.save()` 、
    :data:`~django.db.models.signals.pre_save`, 和
    :data:`~django.db.models.signals.post_save` 。

执行自定义SQL
--------------

另一种常见的需求是在模型方法和模块级方法中编写自定义SQL语句。有关使用原始SQL的更多细节参见
:doc:`使用原始 SQL</topics/db/sql>` 。

.. _model-inheritance:

模型继承
=========

Django 中的模型继承与 Python 中普通类继承方式几乎完全相同，
但是本页开头列出的模型基本的要求还是要遵守。这表示自定义的模型类应该继承 :class:`django.db.models.Model` 。

你唯一需要作出的决定就是你是想让父模型具有它们自己的数据库表，
还是让父模型只持有一些共同的信息而这些信息只有在子模型中才能看到。

在Django 中有3种风格的继承

1. 通常，你只想使用父类来持有一些信息，你不想在每个子模型中都敲一遍。
   这个类永远不会单独使用，所以你要使用 :ref:`abstract-base-classes` 。

2. 如果你继承一个已经存在的模型且想让每个模型具有它自己的数据库表，那么应该使用
   :ref:`multi-table-inheritance` 。

3. 最后，如果你只是想改变一个模块级别的行为，而不用修改模型的字段，你可以使用
   :ref:`proxy-models`.

.. _abstract-base-classes:

抽象基类
----------

当你想将一些共有信息放进其它一些model的时候，抽象化类是十分有用的。你编写完基类之后，在
:ref:`Meta <meta-options>` 中设置 ``abstract=True``
这个模型就不会被用来创建任何数据表。取而代之的是，当它被用来作为一个其他model的基类，它的字段将被加入那些子类中。
如果抽象基类和它的子类有相同的字段名，那么将会出现error（并且Django将抛出一个exception）。

例如::

    from django.db import models

    class CommonInfo(models.Model):
        name = models.CharField(max_length=100)
        age = models.PositiveIntegerField()

        class Meta:
            abstract = True

    class Student(CommonInfo):
        home_group = models.CharField(max_length=5)

``Student`` 将会有三个字段: ``name``, ``age`` 和
``home_group`` 。 ``CommonInfo`` 模型无法像一般的Django模型一样使用, 因为它是一个抽象基类。
它无法生成一张数据表或者拥有一个管理器，并且不能实例化或者直接储存。

许多应用场景下, 这种类型的模型继承恰好是你想要的。它提供一种在Python语言层级上提取公共信息的方式，
同时在数据库层级上，每个子类各自仍然只创建一个数据库表。

``Meta`` 继承
~~~~~~~~~~~~~~~

当一个抽象基类被创建的时候, Django把你在基类内部定义的 :ref:`Meta <meta-options>`
类作为一个属性使其可用。如果子类没有声明自己的 :ref:`Meta <meta-options>` 类, 他将会继承父类的
:ref:`Meta <meta-options>` 。如果子类想要扩展父类的 :ref:`Meta <meta-options>` 类,
它可以作为其子类。例如::

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

继承时，Django 会对抽象基类的 :ref:`Meta <meta-options>` 类做一个调整：在设置
:ref:`Meta <meta-options>` 属性之前, Django 会设置 ``abstract=False`` 。
这意味着抽象基类的子类本身不会自动变成抽象类。 当然，你可以让一个抽象基类继承自另一个抽象基类，
你只要记得每次都要显式地设置 ``abstract=True`` 。

一些属性被包含在抽象基类的 :ref:`Meta <meta-options>` 里面是没有意义的。例如，包含 ``db_table``
属性将意味着所有的子类(是指那些没有指定自己的 :ref:`Meta <meta-options>` 类的子类)都使用同一张数据表，
这总归不会是你想要的。

.. _abstract-related-name:

小心使用 ``related_name`` 和 ``related_query_name``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果你在 ``ForeignKey`` 或 ``ManyToManyField`` 字段是使用 :attr:`~django.db.models.ForeignKey.related_name` 或
:attr:`~django.db.models.ForeignKey.related_query_name` 属性，
您必须始终为该字段指定 *惟一的* 反向名称和查询名称。
但在抽象基类上这样做就会引发一个很严重的问题，
应为Django会将基类字段添加到每个子类中，因此每个子类都具有相同的属性值(包括related_name和related_query_name)。

当您在抽象基类中使用 :attr:`~django.db.models.ForeignKey.related_name`
或 :attr:`~django.db.models.ForeignKey.related_query_name`) 时，要解决这个问题，名称中就要包含
``“%(app_label)s”`` 和 ``“%(class)s”`` 。

- ``'%(class)s'`` 会替换为子类的小写加下划线格式的名称，字段在子类中使用。

- ``'%(app_label)s'`` 会替换为应用的小写加下划线格式的名称，应用包含子类。每个安装的应用名称都应该是唯一的，而且应用里每个模型类的名称也应该是唯一的，所以产生的名称应该彼此不同。

例如，假设有一个app叫做 ``common/models.py``::

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

以及另一个应用 ``rare/models.py``::

    from common.models import Base

    class ChildB(Base):
        pass

``common.ChildA.m2m`` 字段的反向名称是
``common_childa_related`` ，反向查询名称是 ``common_childas`` 。
``common.ChildB.m2m`` 字段的反向名称是
``common_childb_related`` ，反向查询名称是
``common_childbs`` 。最后，``rare.ChildB.m2m`` 字段的反向名称是 ``rare_childb_related``
，反向查询名称是
``rare_childbs`` 。这取决于你如何使用 ``'%(class)s'`` 和
``'%(app_label)s'`` 来构造。
如果你没有这样做，Django 就会在验证 model (或运行 :djadmin:`migrate`) 时抛出错误。


如果你没有在抽象基类中为某个关联字段定义 :attr:`~django.db.models.ForeignKey.related_name`
属性, 那么默认的反向名称就是子类名称加上 ``'_set'``, 它能否正常工作取决于你是否在子类中定义了同名字段。
例如，在上面的代码中，如果去掉 :attr:`~django.db.models.ForeignKey.related_name`
属性, ``ChildA`` 中， ``m2m`` 字段的反向名称是
``childa_set`` ；而 ``ChildB`` 中的 ``m2m`` 字段的反向名称就是 ``childb_set``


.. versionchanged:: 1.10

   ``related_query_name`` 可以添加  ``'%(app_label)s'`` 和 ``'%(class)s'`` 是在1.10中加入。

.. _multi-table-inheritance:

多表继承
---------

这是 Django 支持的第二种继承方式。使用这种继承方式时，
每一个层级下的每个 model 都是一个真正意义上完整的 model 。 每个 model 都有专属的数据表，
都可以查询和创建数据表。 继承关系在子 model 和它的每个父类之间都添加一个链接
(通过一个自动创建的 :class:`~django.db.models.OneToOneField` 来实现)。 例如::

    from django.db import models

    class Place(models.Model):
        name = models.CharField(max_length=50)
        address = models.CharField(max_length=80)

    class Restaurant(Place):
        serves_hot_dogs = models.BooleanField(default=False)
        serves_pizza = models.BooleanField(default=False)

``Place`` 里面的所有字段在 ``Restaurant`` 中也是有效的，
只不过没有保存在数据库中的 ``Restaurant`` 表中。
所以下面两个语句都是可以运行的::

    >>> Place.objects.filter(name="Bob's Cafe")
    >>> Restaurant.objects.filter(name="Bob's Cafe")

如果你有一个 ``Place`` 它同时也是一个 ``Restaurant``, 那么你可以使用 model 的小写形式从
``Place`` 对象中获得与其对应的 ``Restaurant`` 对象::

    >>> p = Place.objects.get(id=12)
    # If p is a Restaurant object, this will give the child class:
    >>> p.restaurant
    <Restaurant: ...>

但是，如果上例中的 ``p``  并不是 ``Restaurant`` (比如它仅仅只是 ``Place`` 对象，或是其他类的父类),
那么在引用 ``p.restaurant`` 就会抛出 ``Restaurant.DoesNotExist`` 异常。

.. _meta-and-multi-table-inheritance:

多表继承的 ``Meta``
~~~~~~~~~~~~~~~~~~~~~

在多表继承中，子类继承父类的 :ref:`Meta <meta-options>` 类是没什么意义的。
所有的 :ref:`Meta <meta-options>` 选项已经对父类起了作用，再次使用只会起反作用。
(这与使用抽象基类的情况正好相反，因为抽象基类并没有属于它自己的内容)

所以子 model 并不能访问它父类的 :ref:`Meta
<meta-options>` 类。但是在某些受限的情况下，子类可以从父类继承某些行为：如果子类没有指定
:attr:`~django.db.models.Options.ordering` 属性或
:attr:`~django.db.models.Options.get_latest_by` 属性，它就会从父类中继承这些属性。

如果父类有了排序设置，而你并不想让子类有任何排序设置，你就可以显式地禁用排序::

    class ChildModel(ParentModel):
        # ...
        class Meta:
            # Remove parent's ordering effect
            ordering = []

继承与反向关联
~~~~~~~~~~~~~~~

因为多表继承使用了一个隐含的
:class:`~django.db.models.OneToOneField` 来链接子类与父类，
所以象上例那样，你可以用父类来指代子类。但是这个字段默认的
:attr:`~django.db.models.ForeignKey.related_name` 的值与
:class:`~django.db.models.ForeignKey` 和
:class:`~django.db.models.ManyToManyField` 默认的反向名称相同。
如果你与该父类的另一个子类做多对一或是多对多关系，你就必须在每个多对一和多对多字段上强制指定
:attr:`~django.db.models.ForeignKey.related_name` 。
如果你没这么做，Django 就会在你运行 验证(validation)  时抛出异常。

例如，仍以上面 ``Place`` 类为例，我们创建一个带有 :class:`~django.db.models.ManyToManyField` 字段的子类::

    class Supplier(Place):
        customers = models.ManyToManyField(Place)

这会产生一个错误::

    Reverse query name for 'Supplier.customers' clashes with reverse query
    name for 'Supplier.place_ptr'.

    HINT: Add or change a related_name argument to the definition for
    'Supplier.customers' or 'Supplier.place_ptr'.

给 ``customers`` 字段添加一个 ``related_name`` 就可以解决这个错误:
 ``models.ManyToManyField(Place, related_name='provider')`` 。

指定链接父类的字段
~~~~~~~~~~~~~~~~~~~~~

之前我们提到，Django 会自动创建一个 :class:`~django.db.models.OneToOneField` 字段将子类链接至非抽象的父 model。
如果你想指定链接父类的属性名称，你可以创建你自己的 :class:`~django.db.models.OneToOneField` 字段并设置
:attr:`parent_link=True <django.db.models.OneToOneField.parent_link>` ，从而使用该字段链接父类。


.. _proxy-models:

代理模型
-----------

使用 :ref:`多表继承 <multi-table-inheritance>` 时, model 的每个子类都会创建一张新数据表。
通常情况下，这正是我们想要的结果。这是因为子类需要一个空间来存储不包含在基类中的字段数据。
但有时，你可能只想更改 model 在 Python 层的行为实现。比如：更改默认的 manager ，或是添加一个新方法。

这就是代理模型继承的好处:为原始模型创建一个代理。您可以创建、删除和更新代理模型的实例，
并且所有的数据将被保存，就像您使用原始的(非代理的)模型一样。
不同之处在于，您可以在代理中更改默认的模型排序或manager，而不必更改原始数据。

声明代理 model 和声明普通 model 没有什么不同。 设置 ``Meta`` 类中 :attr:`~django.db.models.Options.proxy`
的值为 ``True`` ，就完成了对代理 model 的声明。

举个例子，假设你想给 ``Person`` 模型添加一个方法。你可以这样做::

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

``MyPerson`` 类和它的父类 ``Person`` 操作同一个数据表。
特别的是， ``Person`` 的任何实例也可以通过 ``MyPerson`` 访问，反之亦然::

    >>> p = Person.objects.create(first_name="foobar")
    >>> MyPerson.objects.get(first_name="foobar")
    <MyPerson: foobar>

你也可以使用代理 model 给 model 定义不同的默认排序设置。
如果你不想每次都给 ``Person`` 模型排序，但是使用代理的时候总是按照 ``last_name`` 属性排序。这非常容易::

    class OrderedPerson(Person):
        class Meta:
            ordering = ["last_name"]
            proxy = True

这样，普通的 ``Person`` 查询是无序的，而 ``OrderedPerson`` 查询会按照 ``last_name`` 排序。

代理模型的 ``Meta`` 属性继承和 :ref:`普通模型 <meta-and-multi-table-inheritance>` 相同。

``QuerySet`` 始终返回请求的模型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

也就是说，没有办法让django在查询 ``Person`` 对象时返回 ``MyPerson`` 对象。
``Person`` 对象的查询集会返回相同类型的对象。代理对象的特点是:它会使用依赖于原生
``Person`` 的代码，而你可以使用你添加进来的扩展对象（它不会依赖其它任何代码）。而并不是将
``Person`` 模型（或者其它）在所有地方替换为其它你自己创建的模型。

基类的限制
~~~~~~~~~~~~

代理模型必须继承一个非抽象模型类。由于代理模型不提供不同数据库表中行之间的任何连接，
所以也不能继承多个非抽象模型。代理模型可以继承任意数量的抽象模型类，
但是不能定义任何模型字段。代理模型也可以从共享非抽象父类的任意数量的代理模型继承。

.. versionchanged:: 1.10

    在早期版本中，代理模型不能继承同一个父类的多个代理模型。

代理模型的管理器
~~~~~~~~~~~~~~~~~

如果你没有在代理 模型中定义任何管理器 ，代理模型就会从父类中继承管理器 。
如果你在代理模型中定义了一个管理器 ，它就会变成默认的管理器 ，不过定义在父类中的管理器仍然有效。

继续上面的例子，当你查询 ``Person`` 模型的时候，你可以改变默认管理器，例如::

    from django.db import models

    class NewManager(models.Manager):
        # ...
        pass

    class MyPerson(Person):
        objects = NewManager()

        class Meta:
            proxy = True


如果你想要向代理中添加新的管理器，而不是替换现有的默认管理器，
你可以使用 :ref:`自定义管理器管理器 <custom-managers-and-inheritance>` 文档中描述的技巧：
创建一个含有新的管理器的基类，并继承时把他放在主基类的后面::

    # Create an abstract class for the new manager.
    class ExtraManagers(models.Model):
        secondary = NewManager()

        class Meta:
            abstract = True

    class MyPerson(Person, ExtraManagers):
        class Meta:
            proxy = True

你可能不需要经常这样做，但这样做是可行的。

.. _proxy-vs-unmanaged-models:

代理模型与非托管模型之间的差异
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

代理 model 继承看上去和使用 ``Meta`` 类中的 :attr:`~django.db.models.Options.managed` 属性的非托管 model
非常相似。但两者并不相同，使用时选用哪种方案是一个值得考虑的问题。

通过设置 :attr:`Meta.db_table
<django.db.models.Options.db_table>` 可以创建一个非托管的模型，它可以将现有模型隐藏，
并向它添加Python方法。但是，这非常繁冗和脆弱，因为不论您做任何更改，都需要同时保持两个副本同步。

另一方面，代理模型的行为与代理模型完全相同。它们总是与父模型保持同步，因为它们直接继承了它的字段和管理器。

所以，一般规则是:

1. 如果你要借鉴一个已有的 模型或数据表，并不想要所有原始的数据库表列，那就令
   ``Meta.managed=False`` 。通常情况下，对模型数据库创建视图和表格不需要由 Django 控制时，就使用这个选项。

2. 如果你想对 model 做 Python 层级的改动，又想保留字段不变，那就令 ``Meta.proxy=True`` 。
   因此在数据保存时，代理 model 相当于完全复制了原始 模型的存储结构。

.. _model-multiple-inheritance-topic:

多重继承
---------

和Python的普通类一样，django的模型可以继承自多个父类模型。
切记一般的Python名称解析规则也会适用。
一个特定名称(例如 :ref:`Meta
<meta-options>` )出现的第一个基类将将是真正使用基类。这意味着如果多个父类含有
:ref:`Meta <meta-options>` 类，只有第一个会被使用，剩下的会忽略掉。

一般来说，你并不需要继承多个父类。多重继承主要对“mix-in”类有用：
向每个继承mix-in的类添加一个特定的、额外的字段或者方法。
你应该尝试将你的继承关系保持得尽可能简洁和直接，这样你就不必费很大力气来弄清楚某段特定的信息来自哪里。

注意，从具有普通 ``id`` 主键字段的多个模型中继承会引起错误。要正确使用多重继承，可以在基类模型中使用显式使用 :class:`~django.db.models.AutoField` ::

    class Article(models.Model):
        article_id = models.AutoField(primary_key=True)
        ...

    class Book(models.Model):
        book_id = models.AutoField(primary_key=True)
        ...

    class BookReview(Book, Article):
        pass

或者是使用一个父类来持有 :class:`~django.db.models.AutoField`::

    class Piece(models.Model):
        pass

    class Article(Piece):
        ...

    class Book(Piece):
        ...

    class BookReview(Book, Article):
        pass

不“重写”字段
------------------

在普通的Python类继承中，子类可以覆盖父类的任何属性。在Django中，模型字段通常不允许这样做。
如果一个非抽象模型基类有一个名为 ``author`` 的字段，那么您就不能创建另一个模型字段，
也不能在任何继承自基类的类中定义一个名为 ``author`` 的属性。

这个限制并不适用于从抽象模型继承的模型字段。这种情况字段可以被另一个字段或值覆盖，
或者被设置成 ``field_name = None`` 删除掉。

.. versionchanged:: 1.10

    在1.10版本中新增了重写抽象字段的功能。

.. warning::

    模型管理器是从抽象基类继承而来的。重写继承 :class:`~django.db.models.Manager` 引用的继承字段
    可能会导致一些的错误。参见 :ref:`自定义管理器和模型继承 <custom-managers-and-inheritance>` 。

.. note::
    一些字段在模型上定义了额外的属性，例如，一个 :class:`~django.db.models.ForeignKey` 定义了一个额外的属性，
    并将 ``_id`` 附加到字段名，以及在外键模型上的 ``related_name`` 和 ``related_query_name`` 。

    除非更改或删除了定义它的字段，否则不能重写这些额外的属性。

重写父类的字段会导致很多麻烦，比如：初始化实例(指定在 Model.__init__ 中被实例化的字段) 和序列化。
而普通的 Python 类继承机制并不能处理好这些特性。
所以 Django 的继承机制被设计成与 Python 有所不同，这样做并不是随意而为的。

这些限制仅仅针对做为属性使用的 :class:`~django.db.models.Field` 实例，
并不是针对 Python 属性，Python 属性仍是可以被重写的。 在 Python 看来，上面的限制仅仅针对字段实例的名称：
如果你手动指定了数据库的列名称，那么在多重继承中，你就可以在子类和某个祖先类当中使用同一个列名称。
(因为它们使用的是两个不同数据表的字段)。

如果你在祖先类中重写了某个 model 字段，Django 都会抛出 :exc:`~django.core.exceptions.FieldError` 异常

将模型放到包中
================

:djadmin:`manage.py startapp <startapp>` 命令创建一个包含 ``models.py`` 的应用文件结构。
如果您有许多模型，把他们放在单独的文件夹中非常有用。

为此，创建一个 ``models`` 包。删除 ``models.py`` 。
新建一个包含 ``__init__.py`` 文件的路径 ``myapp/models/`` 来放置模型。
必须在 ``__init__.py`` 中导入模型。

.. snippet::
    :filename: myapp/models/__init__.py

    from .organic import Person
    from .synthetic import Robot

显式导入每个模型而不是使用  ``from .models import *`` 。这样具有命名空间明确，代码更具可读性等优点。


.. seealso::

    :doc:`模型参考 </ref/models/index>`
        涵盖模型相关的API，包括模型字段、关联对象和 ``查询集`` 。
