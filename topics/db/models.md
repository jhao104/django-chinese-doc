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
