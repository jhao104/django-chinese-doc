　　Django的开发背景是快节奏的新闻编辑室环境，因此它被设计成一个大而全的web框架，能够快速简单的完成任务。本节将快速介绍如何利用Django搭建一个数据库驱动的WEB应用。
它不会有太多的技术细节，只是让你理解Django是如何工作的。当您准备开始一个项目时，您可以从[教程](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/intro/tutorial01/%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E4%B8%AADjango%E5%BA%94%E7%94%A8%2CPart1.md)开始，或者学习更详细的[文档](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/intro/%E4%BD%BF%E7%94%A8Django.md)

## 设计model

　　虽然您可以使用Django可以不需要数据库，但它附带一个[对象关系映射器](https://en.wikipedia.org/wiki/Object-relational_mapping)，你能直接使用Python代码来描述你的数据库设计。数据[模型](https://docs.djangoproject.com/en/1.10/topics/db/models/)语法提供了许多丰富的方法 - 到目前为止，它一直在解决了多年的数据库schema问题。以下是一个简单的例子：

```python
# mysite/news/models.py

from django.db import models

class Reporter(models.Model):
    full_name = models.CharField(max_length=70)

    def __str__(self):              # __unicode__ on Python 2
        return self.full_name

class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()
    reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)

    def __str__(self):              # __unicode__ on Python 2
        return self.headline
```

## 安装model

　　接下来，运行Django命令行自动创建数据库表：
```shell
$ python manage.py migrate
```