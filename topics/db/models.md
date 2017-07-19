# 模型（model）

模型是你的数据的唯一的、权威的信息源。它包含你所储存数据的必要字段和操作行为。通常，每个模型都对应着数据库中的唯一一张表。

基础认识：

* 每个model都是一个继承`django.db.models.Model`的子类;

* model中的每个属性(attribute)都代表数据库中的一个字段;

* Django 提供一套自动生成的用于数据库访问的API；详见[执行查询](https://docs.djangoproject.com/en/1.11/topics/db/queries/)。

## 简短例子

例子中的模型定义了一个带有`first_name`和`last_name`的`Person`model:

```python
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)
```

其中`first_name`和`last_name`模型的都是[字段](#字段)，每个字段都是类的一个属性，每个属性映射到数据库中的列。

上面的`Person`model将会像下面sql语句类似的创建一张表:

```sql
CREATE TABLE myapp_person (
    "id" serial NOT NULL PRIMARY KEY,
    "first_name" varchar(30) NOT NULL,
    "last_name" varchar(30) NOT NULL
);
```

特别注意：

* 表的名字`myapp_person_name`是根据模型中的元数据自动生成的(app_name + class_name),也可以自定义表的名称,参考这里[表名 (0%)](https://docs.djangoproject.com/en/1.11/ref/models/options/#table-names);

* `id`字段是自动添加的，你可以重新改写这一行为，参见[自增主键](#Automatic_primary_key_fields);

* 本示例中的CREATE TABLE SQL使用的是PostgreSQL语法，但是在应用中，Django会根据[配置文件 (0%)](https://docs.djangoproject.com/en/1.10/topics/settings/)中指定的数据库类型来使用相应的SQL语句。

## 使用模型

定义好模型之后，接下来你需要告诉Django使用这些模型，你要做的就是修改配置文件中的`INSTALLED_APPS`设置，在其中添加`models.py`所在应用的名称。

例如，如果你的应用的模型位于`myapp.models`（文件结构是由`manage.py startapp`命令自动创建的），`INSTALLED_APPS`部分看上去应该是：

```python
INSTALLED_APPS = [
    #...
    'myapp',
    #...
]
```

当你在`INSTALLED_APPS`中添加新的应用名时，一定要运行命令`manage.py migrate`，你可以事先使用`manage.py makemigrations`给应用生成迁移脚本。


## 字段

对于一个模型来说，最重要的是列出该模型在数据库中定义的字段。字段由models类属性指定。要注意选择的字段名称不要和[模型API (0%)](https://docs.djangoproject.com/en/1.10/ref/models/instances/)冲突，比如`clean`、`save`` 或者`delete`等。

例子:
```python
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
```
### 字段类型

模型中的每个字段都是`Field`子类的某个实例,Django根据字段类的类型确定以下信息:

* 数据库中字段的类型(e.g. `INTEGER`, `VARCHAR`, `TEXT`);

* 渲染表单时使用的默认HTML[部件 (0%)](https://docs.djangoproject.com/en/1.10/ref/forms/widgets/)(e.g. `<input type="text">`, `<select>`);

* 在Django的admin和自动生成的表单中使用的最低验证要求;

Django有几十种内置字段类型；你可以在[这里 (0%)](https://docs.djangoproject.com/en/1.10/ref/models/fields/#model-field-types)找到所有的字段,如果Django内置字段类型无法满足要求，你也可以自定义字段,参见[自定义模型字段 (0%)](https://docs.djangoproject.com/en/1.10/howto/custom-model-fields/)。

### 字段选项

每个字段都有一些特有的参数，详见[模型字段参考 (0%)](https://docs.djangoproject.com/en/1.11/ref/models/fields/#model-field-types)。例如，`CharField`（和它的子类）需要`max_length`参数来指定`VARCHAR`数据库字段的大小。

下面是所有字段类型的通过选项，它们都是可选的，[这里](https://docs.djangoproject.com/en/1.11/ref/models/fields/#common-model-field-options)有它们详细的介绍。这里只对最常用的一种选项快速总结：

* `null`

如果为`True`，Django将会把数据库中的空值保存为`NULL`。默认值为`False`。

* `blank`

如果为`True`，该字段允许为空值，默认为`False`。

要注意，`blank`与`null`不同。`null`纯粹是数据库范畴,指数据库中字段内容是否允许为空，而`blank`是表单数据输入验证范畴的。如果一个字段的`blank=True`，表单的验证将允许输入一个空值。如果字段的`blank=False`，该字段就是必填的。

* `choices`

由二项元组构成的一个可迭代对象（列表或元组），用来给字段提供选择项。如果设置了`choices`，默认的表单将是一个选择框而不是文本框，而且这个选择框的选项就是`choices`中的选项。下面是一个关于`choices`列表的例子：

```python
YEAR_IN_SCHOOL_CHOICES = (
    ('FR', 'Freshman'),
    ('SO', 'Sophomore'),
    ('JR', 'Junior'),
    ('SR', 'Senior'),
    ('GR', 'Graduate'),
)
```

每个元组中的第一个元素是存储在数据库中的值。第二个元素是在管理界面或[ModelChoiceField](https://docs.djangoproject.com/en/1.11/ref/forms/fields/#django.forms.ModelChoiceField)中用作显示的内容。给定一个模型实例，可以使用`get_FOO_display()`方法获取一个选择字段的显示值((这里的FOO就是choices字段的名称 ))。例如:

```python
from django.db import models

class Person(models.Model):
    SHIRT_SIZES = (
        ('S', 'Small'),
        ('M', 'Medium'),
        ('L', 'Large'),
    )
    name = models.CharField(max_length=60)
    shirt_size = models.CharField(max_length=1, choices=SHIRT_SIZES)
```

```python
>>> p = Person(name="Fred Flintstone", shirt_size="L")
>>> p.save()
>>> p.shirt_size
'L'
>>> p.get_shirt_size_display()
'Large'
```

* `default`

字段的默认值。这可以是一个值，也可以是可调用对象。如果可调用，那么每次创建新对象时都会被调用。

* `help_text`

表单部件额外显示的帮助内容。即使字段不在表单中使用，它对生成文档也很有用。

* `primary_key`

如果为`True`，那么这个字段就是模型的主键。

如果你不在的模型中指定任何一个字段`primary_key=True`，那么Django将自动添加一个[IntegerField](https://docs.djangoproject.com/en/1.11/ref/models/fields/#django.db.models.IntegerField)来作为主键，所以你不需要在任何一个字段中设置`primary_key=True`，除非你想重写默认的主键行为。

主键字段是只读的。如果你在一个已存在的对象上面更改主键的值并且保存，那么其实是创建了一个新的对象。例如：

```python
from django.db import models

class Fruit(models.Model):
    name = models.CharField(max_length=100, primary_key=True)
```

```python
>>> fruit = Fruit.objects.create(name='Apple')
>>> fruit.name = 'Pear'
>>> fruit.save()
>>> Fruit.objects.values_list('name', flat=True)
<QuerySet ['Apple', 'Pear']>
```

* `unique`

如果该值设置为`True`, 这个数据字段在整张表中必须是唯一的。

最后重申，这些只是对最常见的字段选项的简短描述，更加详细内容参见[这里](https://docs.djangoproject.com/en/1.11/ref/models/fields/#common-model-field-options)。





