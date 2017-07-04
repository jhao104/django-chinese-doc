# 开发第一个Django应用,Part7

　　本教程上接[Part6](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/intro/tutorial06/%E7%AC%AC%E4%B8%80%E4%B8%AADjango%E5%BA%94%E7%94%A8%2CPart6.md)。将继续完成这个投票应用,本节将着重讲解如果用Django自动生成后台管理网站。

## 自定义管理表单

　　通过`admin.site.register(Question)`注册了`Question`后，Django可以自动构建一个默认的表单。如果您需要自定义管理表单的外观和功能。你可以在注册时通过配置来实现。

　　现在先来试试重新排序表单上的字段。只需要将`admin.site.register(Question)`所在行替换为：
```python
# polls/admin.py

from django.contrib import admin

from .models import Question


class QuestionAdmin(admin.ModelAdmin):
    fields = ['pub_date', 'question_text']

admin.site.register(Question, QuestionAdmin)
```

　　你可以参照上面的形式，创建一个模型类，将之作为第二个参数传入`admin.site.register()`。而且这种操作在任何时候都可以进行。

　　经过上面修改"Publication date"字段会在"Question"字段前面：

![admin07](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/_images/admin07.png)

　　目前的表单只有两个字段可能看不出什么，但是对于一个字段很多的表单，设计一个直观合理的排序方式非常重要。并且在字段数据很多时，还可以将表单分割成多个字段的集合：
```python
# polls/admin.py
from django.contrib import admin

from .models import Question


class QuestionAdmin(admin.ModelAdmin):
    fieldsets = [
        (None,               {'fields': ['question_text']}),
        ('Date information', {'fields': ['pub_date']}),
    ]

admin.site.register(Question, QuestionAdmin)
```

　　字段集合中每一个元组的第一个元素是该字段集合的标题。它让页面看起来像下面的样子：

![admin08t](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/_images/admin08t.png)

## 添加关联对象

　　现在Question的管理页面有了，但是一个Question应该有多个Choices。而此时管理页面并没有显示。现在有两个方法可以解决这个问题。一是就像刚刚Question一样也将Choice注册到admin界面。代码像这样:
```python
# polls/admin.py

from django.contrib import admin

from .models import Choice, Question
# ...
admin.site.register(Choice)
```

　　现在Choice也可以在admin页面看见了，其中"Add choice"表单应该类似这样:

![admin09](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/_images/admin09.png)

　　在这个表单中，Question字段是一个select选择框，包含了当前数据库中所有的Question实例。Django在admin站点中，自动地将所有的[外键](https://docs.djangoproject.com/en/1.11/ref/models/fields/#django.db.models.ForeignKey)关系展示为一个select框。在我们的例子中，目前只有一个question对象存在。

　　请注意图中的绿色加号，它连接到Question模型。每一个包含外键关系的对象都会有这个绿色加号。点击它，会弹出一个新增Question的表单，类似Question自己的添加表单。填入相关信息点击保存后，Django自动将该Question保存在数据库，并作为当前Choice的关联外键对象。通俗讲就是，新建一个Question并作为当前Choice的外键。

　　但是，实话说，这种创建方式的效率不怎么样。如果在创建Question对象的时候就可以直接添加一些Choice，那样操作将会变得简单些。

　　删除Choice模型对register()方法的调用。然后，编辑Question的注册代码如下：
```python
# polls/admin.py

from django.contrib import admin

from .models import Choice, Question


class ChoiceInline(admin.StackedInline):
    model = Choice
    extra = 3


class QuestionAdmin(admin.ModelAdmin):
    fieldsets = [
        (None,               {'fields': ['question_text']}),
        ('Date information', {'fields': ['pub_date'], 'classes': ['collapse']}),
    ]
    inlines = [ChoiceInline]

admin.site.register(Question, QuestionAdmin)
```

　　上面的代码告诉Django：Choice对象将在Question管理页面进行编辑，默认情况，请提供3个Choice对象的编辑区域。

　　现在"增加question"页面变成了这样:

![admin10t](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/_images/admin10t.png)

　　它的工作机制是：这里有3个插槽用于关联Choices，而且每当你重新返回一个已经存在的对象的“Change”页面，你又将获得3个新的额外的插槽可用。

　　在3个插槽的最后，还有一个“Add another Choice”链接。点击它，又可以获得一个新的插槽。如果你想删除新增的插槽，点击它右上方的X图标即可。但是，默认的三个插槽不可删除。下面是新增插槽的样子：

![admin14t](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/_images/admin14t.png)

　　但是现在还有个小问题。上面页面中插槽纵队排列的方式需要占据大块的页面空间，看起来很不方便。为此，Django提供了一种扁平化的显示方式，你仅仅只需要将ChoiceInline继承的类改为admin.TabularInline：
```python
# polls/admin.py

class ChoiceInline(admin.TabularInline):
    #...
```
　　使用`TabularInline`代替`StackedInline``,相关的对象将以一种更紧凑的表格形式显示出来:
![admin11t](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/_images/admin11t.png)

　　注意，这样多了一个"删除"选项，它允许你删除已经存在的Choice.

## 自定义修改列表

　　现在Question的管理页面看起来已经差不多了，下面来看看修改列表页面，也就是显示了所有question的页面，即下图这个页面:
![admin04t](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/_images/admin04t.png)

　　Django默认只显示`str()`方法指定的内容。如果我们想要同时显示一些别的内容，可以使用list_display属性，它是一个由多个字段组成的元组，其中的每一个字段都会按顺序显示在页面上，代码如下：

```python
# polls/admin.py

class QuestionAdmin(admin.ModelAdmin):
    # ...
    list_display = ('question_text', 'pub_date')
```
　　同时，还可以把[Part2](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/intro/tutorial02/%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E4%B8%AADjango%E5%BA%94%E7%94%A8%2CPart2.md)中的`was_published_recently()`方法也加入进来：
```python
# polls/admin.py

class QuestionAdmin(admin.ModelAdmin):
    # ...
    list_display = ('question_text', 'pub_date', 'was_published_recently')
```

　　现在question的修改列表页面看起来像这样:
![admin12t](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/_images/admin12t.png)

　　你可以点击其中一列的表头来让列表按照这列的值来进行排序，但是`was_published_recently`这列的表头不行，因为Django不支持按照随便一个方法的输出进行排序。另请注意，默认情况下，`was_published_recently`的列标题是方法的名称（下划线替换为空格），内容则是输出的字符串表示形式。

　　可以通过给方法提供一些属性来改进输出的样式，就如下面所示:
```python
# polls/models.py

class Question(models.Model):
    # ...
    def was_published_recently(self):
        now = timezone.now()
        return now - datetime.timedelta(days=1) <= self.pub_date <= now
    was_published_recently.admin_order_field = 'pub_date'
    was_published_recently.boolean = True
    was_published_recently.short_description = 'Published recently?'
```
　　关于这些方法属性的更多信息，请参见[list_display](https://docs.djangoproject.com/en/1.10/ref/contrib/admin/#django.contrib.admin.ModelAdmin.list_display)。
　　
　　我们还可以对显示结果进行过滤，通过使用`list_filter`属性。在`QuestionAdmin`中添加下面的代码：
```python
list_filter = ['pub_date']
```

　　它添加了一个“过滤器”侧边栏，这样就可以通过pubdate字段来过滤显示question:
![admin13t](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/_images/admin13t.png)

　　过滤器显示的筛选类型取决与你过滤的字段，由于`pub_data`是` DateTimeField`，所以Django就自动给出了“今天”、“过去7天”、“本月”、“今年”这几个选项。

　　这一切进展顺利。再添加一些搜索功能：
```python
search_fields = ['question_text']
```

　　这行代码在修改列表的顶部添加了一个搜索框。 当进行搜索时，Django将在question_text字段中进行搜索。 你在search_fields中使用任意数量的字段，但由于它在后台使用LIKE进行查询，尽量不要添加太多的字段，不然会降低数据库查询能力。



