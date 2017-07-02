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

