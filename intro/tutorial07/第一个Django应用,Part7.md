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
