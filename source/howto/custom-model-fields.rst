===========================
自定义模型字段
===========================

.. currentmodule:: django.db.models

介绍
============

在 :doc:`模型参考 </topics/db/models>` 中已经介绍了如何使用Django的标准字段类,比如
:class:`~django.db.models.CharField` 和 :class:`~django.db.models.DateField` 等等.
这些类可以满足你绝大部分的需求. 在某些情况可能Django的版本不能精准匹配到你的需求,
又或者你想使用的字段和Django内置的完全不同.

Django内置的字段类型并没有覆盖所有的数据库列类型 -- 它只有一些常见的字段,
比如 ``VARCHAR`` 和 ``INTEGER``. 对于其他的不常用类型, 比如地理数据(geographic polygons) 或者用户创建的类型比如
`PostgreSQL自定义类型`_, 这时可以自定义Django ``Field`` 子类.

.. _PostgreSQL自定义类型: https://www.postgresql.org/docs/current/static/sql-createtype.html

你可以编写一个复杂的Python对象, 使它以某种方式将数据序列化, 以适应数据库的列类型.
或是创建一个 ``Field`` 的子类, 从而让你可以使用model中的对象.

示例对象
------------------

创建自定义字段都很多需要注意的细节. 为了方便理解, 本文档中将全程使用一个案例:
封装一个代表 `桥牌`_ 的Python对象, 不用担心, 这并不要求你会玩桥牌.
你只需要知道52张牌被均分给4个玩家, 称他们为 *north*, *east*, *south* 和 *west*.
示例类::

    class Hand(object):
        """A hand of cards (bridge style)"""

        def __init__(self, north, east, south, west):
            # Input parameters are lists of cards ('Ah', '9s', etc.)
            self.north = north
            self.east = east
            self.south = south
            self.west = west

        # ... (other possibly useful methods omitted) ...

.. _桥牌: https://en.wikipedia.org/wiki/Contract_bridge

这是一个普通的Python类, 没有在Django中做特殊设定.
我们期望在模型中做如下操作(假设模型中的 ``hand`` 属性是 ``Hand`` 的一个实例)::

    example = MyModel.objects.get(pk=1)
    print(example.hand.north)

    new_hand = Hand(north, east, south, west)
    example.hand = new_hand
    example.save()

对模型中 ``hand`` 属性的取值和赋值与其他Python类一样. 关键是告诉Django如何保存和加载对象.

要在模型中使用 ``Hand`` 类, 我们 **不需要** 修改这个类.
这非常有用, 因为这可以让我们为已存在的类编写模型支持, 而不需要修改代码.

.. note::
    你可能只想利用自定义数据库列类型在模型中作为标准Python类型(字符串或浮点数)处理数据.
    这种情况与我们的 ``Hand`` 例子非常相似, 我们会随着文档的展开对两者的差异进行比较.

背景理论
=================

数据库存储
----------------

可以简单的认为模型字段提供了一种方法, 它接收普通的Python对象, 比如字符串, 布尔值, ``datetime`` 或者像是 ``Hand`` 这样的复杂对象.
然后在进行数据库操作时, 对对象进行格式转换来适应数据库格式.(序列化也和这个一样, 但是等掌握了数据库的转换, 序列化会更简单).

模型中的字段必须要以某种方式转换成数据库的现有列类型.
不同的数据库提供了各自的有效列类型集合, 任何想存储在数据库中的字段都必须转换成这些类型.

通常, 要么编写一个Django字段去匹配特定的数据库列类型, 要么需要一个方法将数据转换成字符串.

以 ``Hand`` 为例, 我们将卡牌以预定义好的顺序拼接成一个104个字符的字符串 --
第一个为 *north* 然后依次是 *east*, *south* 和 *west*.
这样 ``Hand`` 对象可以在数据库中储存在 text 或 character 类型列中.

field类做了什么?
---------------------------

所有Django的fields (本文提到的 *字段* 都是指模型字段, 不是 :doc:`表单字段 </ref/forms/fields>`)
都是 :class:`django.db.models.Field` 的子类.
Django记录字段的大多数信息对于所有字段都是通用的 -- 名称, 帮助文本, 唯一性等等.
由 `Field`` 处理所有存储的信息. 稍后将深入介绍 ``Field`` 可以做的什么.
可以说万物源于 ``Field``, 并在其基础上自定义了类的关键行为.

要知道Django字段类并不以模型的属性存在这一点非常重要.
模型属性仅仅是普通的Python对象. 在模型中定义的字段类实际上存储在 ``Meta`` 类中(如何实现在这里不重要).
因为只是创建和修改属性时,字段类不是必需的.
它们只是提供了在属性值和存储在数据库的值之间进行转换的机制, 并决定了什么被存入数据库或发送给
:doc:`序列化器 </topics/serialization>`.

在创建自定义字段时请牢记. 你编写的Django ``Field`` 子类提供了以各种方式在Python实例和数据库/序列化值之间进行转换的机制
(比如,保存值和用来查询的值之间是不同的). 如果这有点难以理解, 不用担心, 通过下面的例子会让你有清晰的了解.
这里只需要记住, 当你要自定义一个字段时, 需要创建两个类:

* 第一个类是用于用户操作的Python对象. 它会被赋予模型属性, 用于读取和显示. 例如本例中的 ``Hand`` 类.

* 第二类是 ``Field`` 的子类. 这个类用于描述第一个类如何在存储格式和Python格式之间转换.

编写Field子类
========================

在计划创建 :class:`~django.db.models.Field` 子类前,
首先要考虑新字段是否与哪个现有字段类相似, 因为继承一个现有的Django字段再编写一些自己的内容可以节省大量工作.
如果没有则直接继承 :class:`~django.db.models.Field` 类, 所有字段类都是继承自该类.

初始化新字段比较麻烦的是从公共参数中分离你需要的参数, 然后传递给 :class:`~django.db.models.Field` 的 ``__init__()`` 方法(或其父类).

在示例中, 我们将新字段命名为 ``HandField``. (建议将创建的 :class:`~django.db.models.Field` 子类命名为 ``<Something>Field``,
这样可以很容易辨认出它是 :class:`~django.db.models.Field` 的子类.)
我们示例的类并不和任何已存在的类相同, 因此我们直接继承 :class:`~django.db.models.Field`::

    from django.db import models

    class HandField(models.Field):

        description = "A hand of cards (bridge style)"

        def __init__(self, *args, **kwargs):
            kwargs['max_length'] = 104
            super(HandField, self).__init__(*args, **kwargs)

``HandField`` 接收大多数标准字段选项(参考下面列表), 但是我们要确保参数是定长的, 因为它只保存52个卡片的值, 总计104个字符.

.. note::

    许多Django模型字段可以接受并没有什么用的可选参数, 比如同时传入
    :attr:`~django.db.models.Field.editable` 和
    :attr:`~django.db.models.DateField.auto_now` 给
    :class:`django.db.models.DateField` 字段, 它会无视
    :attr:`~django.db.models.Field.editable` 参数
    (设置了 :attr:`~django.db.models.DateField.auto_now` 就意味着
    ``editable=False``). 这种情况并不会引发异常.

    这种行为简化了字段类, 因为它不用检查那些不必要的可选参数.
    它将所有参数传递给父类, 然后就不再使用它们.
    你也可以更严格地设置字段可选参数, 或者对当前字段设置更放任的行为.

``Field.__init__()`` 接受如下参数:

* :attr:`~django.db.models.Field.verbose_name`
* ``name``
* :attr:`~django.db.models.Field.primary_key`
* :attr:`~django.db.models.CharField.max_length`
* :attr:`~django.db.models.Field.unique`
* :attr:`~django.db.models.Field.blank`
* :attr:`~django.db.models.Field.null`
* :attr:`~django.db.models.Field.db_index`
* ``rel``: 用于关联字段 (比如 :class:`ForeignKey`). 仅用于高阶用途.
* :attr:`~django.db.models.Field.default`
* :attr:`~django.db.models.Field.editable`
* ``serialize``: 若设置为 ``False``, 则字段被传递给Django的 :doc:`serializers </topics/serialization>` 时不会被序列化. 默认为 ``True``.
* :attr:`~django.db.models.Field.unique_for_date`
* :attr:`~django.db.models.Field.unique_for_month`
* :attr:`~django.db.models.Field.unique_for_year`
* :attr:`~django.db.models.Field.choices`
* :attr:`~django.db.models.Field.help_text`
* :attr:`~django.db.models.Field.db_column`
* :attr:`~django.db.models.Field.db_tablespace`: 仅用于创建索引, 这需要后端支持 :doc:`tablespaces </topics/db/tablespaces>`. 通常情况下可以忽略此选项.
* :attr:`~django.db.models.Field.auto_created`: 若 ``True`` 则自动创建字段, 用于 :class:`~django.db.models.OneToOneField` 的模型继承. 仅用于高阶用途.

上述列表中所有没有解释的参数与普通Django字段中的作用一样. 详见 :doc:`模型字段 </ref/models/fields>`.

.. _custom-field-deconstruct-method:

Field deconstruct
--------------------

和 ``__init__()`` 方法相对应的是 ``deconstruct()`` 方法.
此方法用来告诉Django如何获取新字段的实例, 并将其转化为序列化形式 - 特别是传递什么参数给 ``__init__()`` 以重新创建它.

如果你没有在继承的Field类上添加额外参数, 那就不需要重写 ``deconstruct()`` 方法.
但是, 如果你要修改在 ``__init__()`` 中传递的参数(例如我们在 ``HandField`` 中一样), 则需要在这里实现.

``deconstruct()`` 方法很简单; 它返回一个由四个元素组成的元组: 字段的属性名, 字段类的完整导入路径, 位置参数(列表形式)和关键字参数(字典形式).
注意这与 :ref:`自定义类<custom-deconstruct-method>` 中的 ``deconstruct()`` 方法不同, 后者返回一个包含三个元素的元组.

在编写自定义字段时, 你可以不用关心它返回的前两个元素. 因为 ``Field`` 类已经包含了处理字段属性名和导入路径的代码.
你需要处理的是位置参数和关键字参数, 因为你想修改的内容就在其中.

例如, 在 ``HandField`` 例子中, 我们在 ``__init__()`` 方法中显示设置了max_length.
基类 ``Field`` 的 ``deconstruct()`` 方法也会收到这个值并会将其返回到关键字参数中.
因此,为了可读性我们可以从关键字参数中删除它::

    from django.db import models

    class HandField(models.Field):

        def __init__(self, *args, **kwargs):
            kwargs['max_length'] = 104
            super(HandField, self).__init__(*args, **kwargs)

        def deconstruct(self):
            name, path, args, kwargs = super(HandField, self).deconstruct()
            del kwargs["max_length"]
            return name, path, args, kwargs

如果有新增关键字参数, 则需要手动将其添加到 ``kwargs`` 中::

    from django.db import models

    class CommaSepField(models.Field):
        "Implements comma-separated storage of lists"

        def __init__(self, separator=",", *args, **kwargs):
            self.separator = separator
            super(CommaSepField, self).__init__(*args, **kwargs)

        def deconstruct(self):
            name, path, args, kwargs = super(CommaSepField, self).deconstruct()
            # Only include kwarg if it's not the default
            if self.separator != ",":
                kwargs['separator'] = self.separator
            return name, path, args, kwargs

更复杂的例子超出了本文档的范围, 你只需要记住 - 对于字段实例的任何修改, ``deconstruct()``
必须返回能够传递给 ``__init__`` 的参数.

如果你在父类 ``Field`` 中设置参数的默认值, 请特别注意; 说明你希望总是包含它们, 而不是在它们采用默认值时消失.

另外, 应该尽量避免使用位置参数来返回值. 为了保证良好的兼容性最好使用关键字参数. 除非你在构造方法中修改参数的名称比修改参数的名称更加频繁.

你可以在含有字段的迁移文件中查看deconstruct结果,
除此之外也可以在单元测试中通过deconstruct和重构字段来测试::

    name, path, args, kwargs = my_field_instance.deconstruct()
    new_instance = MyField(*args, **kwargs)
    self.assertEqual(my_field_instance.some_attribute, new_instance.some_attribute)

修改自定义字段基类
------------------------------------

Django中无法在中途修改自定义字段的基类, 因为Django无法检测到修改并实施迁移.
例如, 你开始的代码是这样::

    class CustomCharField(models.CharField):
        ...

后面你决定继承另一个基类 ``TextField`` , 不能像这样直接修改代码::

    class CustomCharField(models.TextField):
        ...

正确方法是, 重新创建一个新的自定义类然后更新模型引用新类::

    class CustomCharField(models.CharField):
        ...

    class CustomTextField(models.TextField):
        ...

正如 :ref:`删除字段 <migrations-removing-model-fields>` 描述的那样, 只要有迁移指向原来的 ``CustomCharField`` 就必须保留它.

添加自定义类文档
-----------------------------

和往常一样, 你应该为自定义字段添加描述文档, 让使用者能够了解这一自定义字段.
除了为开发者提供文档字符串外, 还需要让后台管理员通过 :doc:`django.contrib.admindocs
</ref/contrib/admin/admindocs>` 在应用后台看到自定义字段的介绍.
这只需要在自定义类的 :attr:`~Field.description` 属性设置描述文本即可,
在上述例子中, 由 ``admindocs`` 应用为 ``HandField`` 字段提供的描述是 'A hand of cards (bridge style)'.

在 :mod:`django.contrib.admindocs` 展示的内容中, 允许在字段描述中使用 ``field.__dict__`` 插值.
例如, :class:`~django.db.models.CharField` 的描述::

    description = _("String (up to %(max_length)s)")

使用方法
--------------

一旦你创建了 :class:`~django.db.models.Field` 子类, 根据新字段的行为可能会重写一些标准方法.
下面按照其重要程度列出了一些方法.

.. _custom-database-types:

自定义数据库列类型
~~~~~~~~~~~~~~~~~~~~~

假如你创建了一个 PostgreSQL 的自定义字段 ``mytype``.
继承 ``Field`` 类并实现 :meth:`~Field.db_type` 方法, 例如::

    from django.db import models

    class MytypeField(models.Field):
        def db_type(self, connection):
            return 'mytype'

只要创建了 ``MytypeField``, 你就可以像其他
``Field`` 类型一样在模型中使用::

    class Person(models.Model):
        name = models.CharField(max_length=80)
        something_else = MytypeField()

如果你的应用需要兼容多种数据库, 这需要考虑各个数据库列类型的差异.
比如, date/time 列类型在 PostgreSQL 叫做 ``timestamp``, 而在 MySQL 中叫做 ``datetime``.
解决这一问题的最简单的办法是使用 :meth:`~Field.db_type` 方法检查 ``connection.settings_dict['ENGINE']`` 属性.

例如::

    class MyDateField(models.Field):
        def db_type(self, connection):
            if connection.settings_dict['ENGINE'] == 'django.db.backends.mysql':
                return 'datetime'
            else:
                return 'timestamp'

:meth:`~Field.db_type` 和 :meth:`~Field.rel_db_type` 方法会在Django框架构建 ``CREATE TABLE`` 语句时调用 -- 即首次创建数据表时.
这些方法还会在构建包含此字段的 ``WHERE`` 子句时调用 -- 即使用 ``get()``,
``filter()`` 和 ``exclude()`` 方法并且使用了该字段作为参数检索数据的时候.
除此之外它们不会被调用, 所以它可以执行一些复杂的代码, 例如上面例子中的 ``connection.settings_dict``.

有些数据库列类型可以接收参数, 例如 ``CHAR(25)``, 参数 ``25`` 表示数据列支持的最大长度.
这种情况下通过参数传入而不是硬编码在 ``db_type()`` 方法中会更加灵活.
例如,下面的 ``CharMaxlength25Field`` 其实没有多大意义, 如下所示::

    # 这是硬编码的错误例子.
    class CharMaxlength25Field(models.Field):
        def db_type(self, connection):
            return 'char(25)'

    # In the model:
    class MyModel(models.Model):
        # ...
        my_field = CharMaxlength25Field()

更好的方法是指定运行时参数 -- 即实例化的时候. 重写
``Field.__init__()`` 方法就可以实现, 例如::

    # 更加灵活的例子
    class BetterCharField(models.Field):
        def __init__(self, max_length, *args, **kwargs):
            self.max_length = max_length
            super(BetterCharField, self).__init__(*args, **kwargs)

        def db_type(self, connection):
            return 'char(%s)' % self.max_length

    # In the model:
    class MyModel(models.Model):
        # ...
        my_field = BetterCharField(25)

最后, 如果你的列确实需要配置非常复杂的SQL, 可以让 :meth:`.db_type` 返回 ``None``.
这样Django在创建SQL代码的时候将会跳过该字段. 随后你需要在其他地方以某种方式创建该列.

:meth:`~Field.rel_db_type` 方法由指向另一字段的 ``ForeignKey`` 或 ``OneToOneField`` 等调用.
例如, 如果你有个 ``UnsignedAutoField``, 那么指向该字段的外键也需要使用相同的数据类型::

    # MySQL unsigned integer (range 0 to 4294967295).
    class UnsignedAutoField(models.AutoField):
        def db_type(self, connection):
            return 'integer UNSIGNED AUTO_INCREMENT'

        def rel_db_type(self, connection):
            return 'integer UNSIGNED'

.. versionadded:: 1.10

    新增 :meth:`~Field.rel_db_type` 方法.

.. _converting-values-to-python-objects:

将值转换成Python对象
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果你自定义的 :class:`~Field` 类需要处理比字符串, 日期, 整数, 浮点数更复杂的数据, 那么你可能需要重写 :meth:`~Field.from_db_value` 和 :meth:`~Field.to_python` 方法.

如果字段子类存在, ``from_db_value()`` 将会在从数据库中载入值时调用, 包含聚合和 :meth:`~django.db.models.query.QuerySet.values` 调用.

``to_python()`` 会在反序列化和在表单中使用 :meth:`~django.db.models.Model.clean` 方法时调用.

作为一个通用规则, ``to_python()`` 需要一下参数情况:

* 正确的类型实例 (e.g., 本例中的 ``Hand``).

* 字符串

* ``None`` (如果字段设置了 ``null=True``)

在 ``HandField`` 列子中, 数据库中以 VARCHAR 类型字段储存数据, 因此我们需要在 ``from_db_value()`` 处理字符串和 ``None`` 的情况.
在 ``to_python()`` 中我们需要处理 ``Hand`` 实例::

    import re

    from django.core.exceptions import ValidationError
    from django.db import models
    from django.utils.translation import ugettext_lazy as _

    def parse_hand(hand_string):
        """Takes a string of cards and splits into a full hand."""
        p1 = re.compile('.{26}')
        p2 = re.compile('..')
        args = [p2.findall(x) for x in p1.findall(hand_string)]
        if len(args) != 4:
            raise ValidationError(_("Invalid input for a Hand instance"))
        return Hand(*args)

    class HandField(models.Field):
        # ...

        def from_db_value(self, value, expression, connection, context):
            if value is None:
                return value
            return parse_hand(value)

        def to_python(self, value):
            if isinstance(value, Hand):
                return value

            if value is None:
                return value

            return parse_hand(value)

注意, 这两个方法总会返回一个 ``Hand`` 实例. 这就是我们要保存在模型属性中的Python对象.

在 ``to_python()`` 中, 如果在转换过程中出现任何错误都应该抛出 :exc:`~django.core.exceptions.ValidationError` 异常.

.. _converting-python-objects-to-query-values:

转换Python对象至查询值
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

数据库值转换是双向的, 如果你重写了 :meth:`~Field.to_python` 方法, 对应的你也必须要重写 :meth:`~Field.get_prep_value` 方法来将Python对象转换成查询值.

例如::

    class HandField(models.Field):
        # ...

        def get_prep_value(self, value):
            return ''.join([''.join(l) for l in (value.north,
                    value.east, value.south, value.west)])

.. warning::

    如果你的自定义字段使用了MySQL的 ``CHAR``, ``VARCHAR`` 或 ``TEXT`` 类型,
    你需要确保 :meth:`.get_prep_value` 返回的是字符串类型.
    在MySQL中对这些类型操作非常灵活, 甚至有时超出预期, 在传入值为整数时,
    查询结果可能包含非期望的结果. 这个问题不会在 :meth:`.get_prep_value` 返回字符串类型的情况出现.

.. _converting-query-values-to-database-values:

转换查询值至数据库值
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

某些数据类型(比如日期)在数据库后端处理前需要转为某种特定格式.
:meth:`~Field.get_db_prep_value` 用于实现这种转换.
将用于查询的特定连接作为 `connection`` 参数传入. 这允许你使用后端特定的转换逻辑.

例如, Django对 :class:`BinaryField` 使用以下方法::

    def get_db_prep_value(self, value, connection, prepared=False):
        value = super(BinaryField, self).get_db_prep_value(value, connection, prepared)
        if value is not None:
            return connection.Database.Binary(value)
        return value

如果你的自定义字段在保存时需要进行特殊转换, 且与常规参数查询的转换不同,
则可以重写 :meth:`~Field.get_db_prep_save`.

.. _preprocessing-values-before-saving:

保存前预处理数据
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以使用 :meth:`~Field.pre_save` 在保存前预处理数据. 例如, Django的
:class:`~django.db.models.DateTimeField` 在 :attr:`~django.db.models.DateField.auto_now` 和
:attr:`~django.db.models.DateField.auto_now_add` 中使用此方法正确设置属性.

如果你重写了此方法, 则必须在结尾返回属性的值. 如果修改了值, 那么也需要更新模型属性, 这样持有该引用的模型总会看到正确的值.

.. _specifying-form-field-for-model-field:

为模型字段指定表单字段
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以通过重写 :meth:`~Field.formfield` 方法来自定义 :class:`~django.forms.ModelForm` 使用的表单属性.

表单字段类通过 ``form_class`` 或 ``choices_form_class`` 参数指定; 如果字段设置了选项(choices)则使用后者, 否则使用前者.
如果没有设置这些参数将会使用 :class:`~django.forms.CharField` 或 :class:`~django.forms.TypedChoiceField`.

所有 ``kwargs`` 字典都直接传递到表单的 ``__init__()`` 方法. 通常你需要做的是为 ``form_class`` (或者 ``choices_form_class``) 参数设置好默认值,
然后委托父类进一步处理. 这可能需要你编写自定义表单字段(甚至表单视图). 详情请查看 :doc:`表单文档
</topics/forms/index>`.

回到我们的例子, 编写 :meth:`~Field.formfield` 方法::

    class HandField(models.Field):
        # ...

        def formfield(self, **kwargs):
            # This is a fairly standard way to set up some defaults
            # while letting the caller override them.
            defaults = {'form_class': MyFormField}
            defaults.update(kwargs)
            return super(HandField, self).formfield(**defaults)

这假设我们已经导入了一个 ``MyFormField`` 字段类(它有默认视图). 本文档不包括编写自定义表单字段的所有详细信息.

.. _helper functions: ../forms/#generating-forms-for-models
.. _forms documentation: ../forms/

.. _emulating-built-in-field-types:

仿造内置字段类型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果已经创建了 :meth:`.db_type` 方法, 则不需要担心 :meth:`.get_internal_type` 方法 -- 它并不常用.
虽然很多时候, 数据库存储行为和其他字段类似，所以你能直接用其它字段的逻辑创建正确的列.

例如::

    class HandField(models.Field):
        # ...

        def get_internal_type(self):
            return 'CharField'

无论使用了哪个数据库后端, :djadmin:`migrate` 或其它 SQL 命令总会在保存字符串时为其创建正确的列类型.

If :meth:`.get_internal_type` returns a string that is not known to Django for
the database backend you are using -- that is, it doesn't appear in
``django.db.backends.<db_name>.base.DatabaseWrapper.data_types`` -- the string
will still be used by the serializer, but the default :meth:`~Field.db_type`
method will return ``None``. See the documentation of :meth:`~Field.db_type`
for reasons why this might be useful. Putting a descriptive string in as the
type of the field for the serializer is a useful idea if you're ever going to
be using the serializer output in some other place, outside of Django.

.. _converting-model-field-to-serialization:

Converting field data for serialization
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To customize how the values are serialized by a serializer, you can override
:meth:`~Field.value_to_string`. Using ``value_from_object()`` is the best way
to get the field's value prior to serialization. For example, since our
``HandField`` uses strings for its data storage anyway, we can reuse some
existing conversion code::

    class HandField(models.Field):
        # ...

        def value_to_string(self, obj):
            value = self.value_from_object(obj)
            return self.get_prep_value(value)

Some general advice
--------------------

Writing a custom field can be a tricky process, particularly if you're doing
complex conversions between your Python types and your database and
serialization formats. Here are a couple of tips to make things go more
smoothly:

1. Look at the existing Django fields (in
   :file:`django/db/models/fields/__init__.py`) for inspiration. Try to find
   a field that's similar to what you want and extend it a little bit,
   instead of creating an entirely new field from scratch.

2. Put a ``__str__()`` (``__unicode__()`` on Python 2) method on the class you're
   wrapping up as a field. There are a lot of places where the default
   behavior of the field code is to call
   :func:`~django.utils.encoding.force_text` on the value. (In our
   examples in this document, ``value`` would be a ``Hand`` instance, not a
   ``HandField``). So if your ``__str__()`` method (``__unicode__()`` on
   Python 2) automatically converts to the string form of your Python object,
   you can save yourself a lot of work.

Writing a ``FileField`` subclass
================================

In addition to the above methods, fields that deal with files have a few other
special requirements which must be taken into account. The majority of the
mechanics provided by ``FileField``, such as controlling database storage and
retrieval, can remain unchanged, leaving subclasses to deal with the challenge
of supporting a particular type of file.

Django provides a ``File`` class, which is used as a proxy to the file's
contents and operations. This can be subclassed to customize how the file is
accessed, and what methods are available. It lives at
``django.db.models.fields.files``, and its default behavior is explained in the
:doc:`file documentation </ref/files/file>`.

Once a subclass of ``File`` is created, the new ``FileField`` subclass must be
told to use it. To do so, simply assign the new ``File`` subclass to the special
``attr_class`` attribute of the ``FileField`` subclass.

A few suggestions
------------------

In addition to the above details, there are a few guidelines which can greatly
improve the efficiency and readability of the field's code.

1. The source for Django's own ``ImageField`` (in
   ``django/db/models/fields/files.py``) is a great example of how to
   subclass ``FileField`` to support a particular type of file, as it
   incorporates all of the techniques described above.

2. Cache file attributes wherever possible. Since files may be stored in
   remote storage systems, retrieving them may cost extra time, or even
   money, that isn't always necessary. Once a file is retrieved to obtain
   some data about its content, cache as much of that data as possible to
   reduce the number of times the file must be retrieved on subsequent
   calls for that information.
