.. _overview:

==========
Django概览
==========

　　Django的开发背景是快节奏的新闻编辑室环境，因此它被设计成一个大而全的web框架，能够快速简单的完成任务。本节将快速介绍如何利用Django搭建一个数据库驱动的WEB应用。
它不会有太多的技术细节，只是让你理解Django是如何工作的。当您准备开始一个项目时，您可以从 `教程`_ 开始，或者学习更详细的 `文档`_

设计model
=========

　　使用Django时可以不需要数据库，因为它附带一个 `对象关系映射器`_ ，能直接使用Python代码来描述数据库设计。数据 `模型`_ 语法提供了许多丰富的方法,到目前为止，它一直在解决遗留了多年的数据库schema问题。以下是一个简单的例子：

.. code:: python

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

安装model
=========

　　接下来，运行Django命令行自动创建数据库表：

.. code:: shell

    $ python manage.py migrate

　　migrate命令会查找所有可用的模型，如果它还没有在数据库中存在，将根据model创建相应的表。你可以查看 `更加丰富的模式控制`_ 。

使用API
=======

　　Django为你提供了大量的方便的数据库操作API，无需你编写额外的代码。下面是个例子：

.. code:: python


    # 从新建的应用中导入models
    >>> from news.models import Reporter, Article

    # reporter为空
    >>> Reporter.objects.all()
    <QuerySet []>

    # 新建一个 Reporter.
    >>> r = Reporter(full_name='John Smith')

    # 将对象保存到数据库中。你必须显式调用save()
    >>> r.save()

    # 现在它会有个id属性
    >>> r.id
    1

    # 现在这个新的reporter位于数据库中了
    >>> Reporter.objects.all()
    <QuerySet [<Reporter: John Smith>]>

    # 字段被作为对象的属性使用
    >>> r.full_name
    'John Smith'

    # Django 提供了非常丰富的查询api
    >>> Reporter.objects.get(id=1)
    <Reporter: John Smith>
    >>> Reporter.objects.get(full_name__startswith='John')
    <Reporter: John Smith>
    >>> Reporter.objects.get(full_name__contains='mith')
    <Reporter: John Smith>
    >>> Reporter.objects.get(id=2)
    Traceback (most recent call last):
        ...
    DoesNotExist: Reporter matching query does not exist.

    # 新建一个 article.
    >>> from datetime import date
    >>> a = Article(pub_date=date.today(), headline='Django is cool',
    ...     content='Yeah.', reporter=r)
    >>> a.save()

    # 现在article也存在数据库中了
    >>> Article.objects.all()
    <QuerySet [<Article: Django is cool>]>

    # Article 对象可以获取其关联的reporter对象属性
    >>> r = a.reporter
    >>> r.full_name
    'John Smith'

    # 反之: Reporter 对象也能过获取其关联的Article对象属性
    >>> r.article_set.all()
    <QuerySet [<Article: Django is cool>]>

    # API遵循您所需要的关系,并且高效的执行
    # 在背后执行join操作
    # 这是查询所有reporter的name以“John”开头的所有Article
    >>> Article.objects.filter(reporter__full_name__startswith='John')
    <QuerySet [<Article: Django is cool>]>

    # 修改字段后通过save()保存
    >>> r.full_name = 'Billy Goat'
    >>> r.save()

    # 使用delete()删除.
    >>> r.delete()

后台管理界面
===============================

　　只要创建好model，Django就可以自动生成一个专业的、可用于生产的管理界面。一个允许经过身份验证的用户添加，更改和删除对象的网站。将model注册到管理员网站上非常简单：

.. code:: python


    # mysite/news/models.py

    from django.db import models

    class Article(models.Model):
        pub_date = models.DateField()
        headline = models.CharField(max_length=200)
        content = models.TextField()
        reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)

.. code:: python


    # mysite/news/admin.py

    from django.contrib import admin

    from . import models

    admin.site.register(models.Article)

　　admin的设计理念是，您的网站可以由员工或客户编辑，也可能仅仅是您编辑，但是这都不需要创建新的后端口管理内容。创建Django应用程序的一个典型工作流程是创建模型并尽可能快地在管理站点的运行，因此您的员工（或客户）可以开始填充数据。然后，开发数据呈现给公众的方式。

设计路由系统(URLs)
==================

　　Django主张干净、优雅的路由设计，不建议在路由中出现类似.php或.asp之类的字眼。要设计应用程序的URL，需要创建名为`URLconf <https://docs.djangoproject.com/en/1.10/topics/http/urls/>`__\ 的Python模块。它包含URL模式和Python回调函数之间的简单映射。
URLconfs还用于将URL与Python代码分离。以下是上述Reporter/Article示例的URLconf可能是什么样的：

.. code:: python


    # mysite/news/urls.py

    from django.conf.urls import url

    from . import views

    urlpatterns = [
        url(r'^articles/([0-9]{4})/$', views.year_archive),
        url(r'^articles/([0-9]{4})/([0-9]{2})/$', views.month_archive),
        url(r'^articles/([0-9]{4})/([0-9]{2})/([0-9]+)/$', views.article_detail),
    ]

　　上面的代码将URL作为简单的正则表达式映射到Python回调函数（“views”）的位置。正则表达式使用括号来从URL捕获值。当用户请求页面时，Django按顺序依次匹配每个模式，直到匹配到为止(如果都没有匹配上，Django将返回一个特殊的404页面)。这个过程其实是非常快的，因为正则表达式在加载时就编译了。

　　一旦正则表达式匹配，Django导入并调用给定的视图。每个视图都会传递一个请求对象（包含请求元数据）以及在正则表达式中捕获的值。例如，如果用户请求URL　``/articles/2005/05/39323/`` 时，Django会调用该函数 ``news.views.article\_detail(request,'2005', '05', '39323')`` 。

编写视图
========

　　每一个视图都必须做下面两件事情之一：返回一个包含请求页面数据的HttoResponse对象或者弹出一个类似404页面的异常。通常，视图通过参数获取数据，并利用它们渲染加载的模板。下面是一个例子：

.. code:: python


    # mysite/news/views.py

    from django.shortcuts import render

    from .models import Article

    def year_archive(request, year):
        a_list = Article.objects.filter(pub_date__year=year)
        context = {'year': year, 'article_list': a_list}
        return render(request, 'news/year_archive.html', context)

　　这个例子使用了`Django模板系统 `_ ，它不仅功能强大，而且还让非编程人员使用觉得足够简单。

设计模板
========

　　Django有一个模板查找路径，在settings文件中，你可以指定路径列表，Django自动按顺序在列表中查找你调用的模板。一个模板看起来是下面这样的：

.. code:: python


    # mysite/news/templates/news/year_archive.html

    {% extends "base.html" %}

    {% block title %}Articles for {{ year }}{% endblock %}

    {% block content %}
    <h1>Articles for {{ year }}</h1>

    {% for article in article_list %}
        <p>{{ article.headline }}</p>
        <p>By {{ article.reporter.full_name }}</p>
        <p>Published {{ article.pub_date|date:"F j, Y" }}</p>
    {% endfor %}
    {% endblock %}

　　用两个花括号定义变量。 ``{{ article.headline
}}`` 的含义就是显示article的headline属性值。而且这不仅仅用于属性引用，还可以用于字典键查找，索引查找和函数调用。

　　``{{article.pub\_date\|date：“F j,Y”}}`` 使用Unix风格的管道（“\|”）字符。这被称为模板过滤器，它是一种过滤变量值的方法。在这种情况下，日期过滤器会以给定的格式（在PHP的日期函数中找到）格式化Python
datetime对象。

　　您可以将过滤器串式调用。也可以编写`自定义模板过滤器`_ ，标签会自动运行自定义的Python代码。

　　最后，Django里面有个“模板继承”的概念。就是这里的 ``{% extends “base.html” %}`` 。这意味着首先加载名为base的模板，这样可以大大减少模板中的冗余：每个模板只能定义该模板的自己的功能。以下是“base.html”模板，使用静态文件，内容如下:

.. code:: python


    # mysite/templates/base.html

    {% load static %}
    <html>
    <head>
        <title>{% block title %}{% endblock %}</title>
    </head>
    <body>
        <img src="{% static "images/sitelogo.png" %}" alt="Logo" />
        {% block content %}{% endblock %}
    </body>
    </html>

　　简单来说，它定义了网站的外观，并为子模板填写提供了空间。这样站点在重新设计和更改单个文件（基本模板）就很简单。

　　而且还可以创建多个版本的站点，具有不同的基本模板，同时重用子模板。Django的创作者已经使用这种技术来创建非常不同的移动版本的网站。

　　请注意，如果您喜欢其他模板系统，可以不使用Django的模板系统。Django的模板系统与Django的模型层特别完美地结合在一起，但是并没有让你强制使用它。基于这一点，您也不是必须要使用Django的数据库API。您可以使用另一个数据库抽象层，您可以读取XML文件，您可以从磁盘读取文件，或任何您想要的内容。每个Django的模型、视图、模板都是解耦的。

Just a little
=============

　　这只是Django功能的一点点简要概述。还有更多更有用的功能：

-  可以集成memcached或其他的\ `缓存框架`_ 。

-  一个 `联合框架`_ ，使得创建RSS和Atom源就像编写一个小Python类一样简单。

-  更加舒适自动生成的管理功能。

.. _教程: https://github.com/jhao104/django-chinese-docs-1.10/blob/master/intro/tutorial01/%E5%BC%80%E5%8F%91%E7%AC%AC%E4%B8%80%E4%B8%AADjango%E5%BA%94%E7%94%A8%2CPart1.md
.. _文档: https://github.com/jhao104/django-chinese-docs-1.10/blob/master/intro/%E4%BD%BF%E7%94%A8Django.md
.. _对象关系映射器: https://en.wikipedia.org/wiki/Object-relational_mapping
.. _模型: http://docs.djangoproject.com/en/1.10/topics/db/models/
.. _更加丰富的模式控制: https://docs.djangoproject.com/en/1.10/topics/migrations/
.. _Django模板系统 : https://docs.djangoproject.com/en/1.10/topics/templates/
.. _自定义模板过滤器: https://docs.djangoproject.com/en/1.10/howto/custom-template-tags/
.. _缓存框架: https://docs.djangoproject.com/en/1.10/topics/cache/
.. _联合框架: https://docs.djangoproject.com/en/1.10/ref/contrib/syndication/