==========================
开发第一个Django应用,Part3
==========================

本教程上接 :doc:`Tutorial 2 </intro/tutorial02>`。我们将继续开发网页投票这个应用，
主要讲如何创建一个对用户开放的界面。

概览
=====

视图是Django应用中的一“类”网页，它通常使用一个特定的函数提供服务，并且具有一个特定的模板。
例如，在博客应用中，可能有以下几种视图：

* 博客首页 —— 显示最新发表的博客；

* 博客“详细”页面 —— 每博客的链接页面；

* 基于年份的归档页面 —— 显示特定年内所有月份发表过的博客；

* 基于月份的归档页面 —— 显示特定月份内每天发表过博客；

* 基于日期的归档页面 —— 显示特定日期内发表过的所有博客；

* 评论：处理针对某篇博客发布的评论。

在我们的投票应用中，我们将建立下面的四个视图：

* Question首页 —— 显示最新发布的几个Question；

* Question“详细”页面 —— 显示单个Question的具体内容，提供一个投票的表单，但不显示该议题的当前投票结果；

* Question“结果”页面 —— 显示特定的Question的投票结果；

* 投票功能 —— 处理对Question中Choice的投票。

在Django中，网页的页面和其他内容都是由视图(views.py)来传递的（视图对WEB请求进行回应）。
每个视图都是由一个Python函数(或者是基于类视图的方法)表示。Django通过对比请求的URL地址来选择对应的视图函数。

在你平时的网页上，你可能经常会碰到类似“ME2/Sites/dirmod.asp?sid=&type=gen&mod=Core+Pages&gid=A6CD4967199A42D9B65B1B”的url。
庆幸的是Django支持使用更加简介的URL模式(patterns)，而不需要编写上面那种复杂的url。

Django的URL模式就是一种URL的通用模式 —— 例如： ``/newsarchive/<year>/<month>/``。

Django使用‘URLconfs’的配置来为URL匹配视图函数。 URLconf使用正则表达式将URL匹配到视图上。

本教程提供URLconfs基本使用，更多信息请参考 :mod:`django.urls`


编辑视图
========

下面，让我们打开 ``polls/views.py`` 文件，添加下列代码：

.. snippet::
    :filename: polls/views.py

    def detail(request, question_id):
        return HttpResponse("You're looking at question %s." % question_id)

    def results(request, question_id):
        response = "You're looking at the results of question %s."
        return HttpResponse(response % question_id)

    def vote(request, question_id):
        return HttpResponse("You're voting on question %s." % question_id)

然后，在 ``polls/urls.py`` 文件中加入下面的url模式，将其映射到我们上面新增的视图。

.. snippet::
    :filename: polls/urls.py

    from django.conf.urls import url

    from . import views

    urlpatterns = [
        # ex: /polls/
        url(r'^$', views.index, name='index'),
        # ex: /polls/5/
        url(r'^(?P<question_id>[0-9]+)/$', views.detail, name='detail'),
        # ex: /polls/5/results/
        url(r'^(?P<question_id>[0-9]+)/results/$', views.results, name='results'),
        # ex: /polls/5/vote/
        url(r'^(?P<question_id>[0-9]+)/vote/$', views.vote, name='vote'),
    ]

现在去浏览器中访问“/polls/34/”它将运行 ``detail()`` 方法，然后在页面中会显示你在url里提供的ID。
访问“/polls/34/results/”和“/polls/34/vote/”，将分别显示预定义的伪结果和投票页面。

上面访问的路由过程如下：当有人访问“/polls/34/”地址时，Django将首先加载 ``mysite.urls`` 模块，
因为它是settings文件里设置的 :setting:`ROOT_URLCONF` 配置文件。在模块里找到 ``urlpatterns`` 变量，
按顺序对各项进行正则匹配。当它匹配到了 ``^polls/``，就剥离出url中匹配的文本 ``polls/``，然后将剩下的文本“34/”，
传递给“ ``polls.urls`` ”进行下一步的处理。在 ``polls.urls`` ，又匹配到了 ``r’^(?P<question_id>[0-9]+)/$’``，
最终结果就是调用该模式对应的 `detail()`` 视图，将34作为参数传入::

    detail(request=<HttpRequest object>, question_id='34')


``question_id='34'`` 的部分来自 ``(？P <question_id> [0-9])``。使用模式周围的括号“捕获”该模式匹配到的文本，
并将其作为参数传递给视图函数; ``?P<question_id>`` 定义一个名字用于标识匹配的模式；``[0-9]+`` 是匹配一串数字的正则表达。

因为URL模式是正则表达式，你如何使用它们没有什么限制。 不需要添加像.html这样繁琐的URL —— 除非你执意这么做，在这种情况下你可以这样做::

    url(r'^polls/latest\.html$', views.index),

But, don't do that. It's silly.

编写拥有实际功能的视图
======================

每个视图函数只负责处理下面两件事中的一件：返回一个包含所请求页面内容的 :class:`~django.http.HttpResponse` 对象，
或抛出一个诸如 :exc:`~django.http.Http404` 异常。该如何去做这两件事，就看你自己的想法了。

您的视图可以从数据库读取记录，也可以不读取。它可以使用模板系统：
如Django的或第三方Python模板系统 或不。可以生成PDF文件，输出XML，即时创建ZIP文件，
任何你想要的，使用任何你想要的Python库。Django只要求返回的是一个:class:`~django.http.HttpResponse`。 或者抛出一个异常。

为了方便，让我们使用 :doc:`Tutorial 2 </intro/tutorial02>` 中介绍的Django自己的数据库API。
下面是一个新的 ``index()`` 视图，它显示系统中最新发布的5条questions记录，并用逗号分隔：

.. snippet::
    :filename: polls/views.py

    from django.http import HttpResponse

    from .models import Question


    def index(request):
        latest_question_list = Question.objects.order_by('-pub_date')[:5]
        output = ', '.join([q.question_text for q in latest_question_list])
        return HttpResponse(output)

    # 保持其他的视图 (detail, results, vote) 不变

这里有一个问题：页面的设计被硬编码在视图中。 如果你想更改页面的外观，就得编辑这段Python代码。
因此，我们使用Django的模板系统，通过创建一个视图能够调用的模板，将页面的设计从Python中分离出来。

首先，在你的 ``polls`` 目录下创建一个叫做 ``templates`` 的目录。Django将在这里查找模板。

项目的settings.py中的 :setting:`TEMPLATES` 配置决定了Django如何加载渲染模板。
将 :setting:`APP_DIRS <TEMPLATES-APP_DIRS>` 设置为True。``DjangoTemplates`` 将在 :setting:`INSTALLED_APPS`
所包含的每个应用的目录下查找名为"templates"子目录。

在刚刚创建的 ``templates`` 目录中，创建另一个名为 ``polls`` 的目录，并在其中创建一个名为 ``index.html`` 的文件。
换句话说，你的模板应该是 ``polls/templates/polls/index.html``。由于 ``app_directories`` 模板加载器如上所述工作，
因此您可以在Django中简单地引用此模板为 ``polls/index.html`` (省掉前面的路径)。


.. admonition:: 模板命名空间

    如果我们把模板直接放在 ``polls/templates`` 中（而不是创建另一个 ``polls`` 子目录），但它实际上是一个坏主意。
    Django将选择它找到的名字匹配的第一个模板，如果你在不同的应用程序中有一个相同名称的模板，
    Django将无法区分它们。我们需要能够将Django指向正确的一个，确保这一点的最简单的方法是通过 **命名空间**。
    也就是说，将这些模板放在为应用程序本身命名的另一个目录中。

将以下的代码写到模板文件：

.. snippet:: html+django
    :filename: polls/templates/polls/index.html

    {% if latest_question_list %}
        <ul>
        {% for question in latest_question_list %}
            <li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>
        {% endfor %}
        </ul>
    {% else %}
        <p>No polls are available.</p>
    {% endif %}

现在更新 ``polls/views.py`` 中的 ``index`` 视图来使用模板：

.. snippet::
    :filename: polls/views.py

    from django.http import HttpResponse
    from django.template import loader

    from .models import Question


    def index(request):
        latest_question_list = Question.objects.order_by('-pub_date')[:5]
        template = loader.get_template('polls/index.html')
        context = {
            'latest_question_list': latest_question_list,
        }
        return HttpResponse(template.render(context, request))

该代码加载名为 ``polls/index.html`` 的模板，并传给它一个 ``context`` 。``Context`` 是一个字典，将模板变量的名字映射到Python对象。

然后你可以通过浏览器打开 http://127.0.0.1:8000/polls 查看效果。

快捷方法：render()
-------------------

常见的习惯是载入一个模板、填充一个 ``context`` 然后返回一个含有模板渲染结果的 :class:`~django.http.HttpResponse` 对象。
Django为此提供一个快捷方式。 下面是重写后的 ``index()`` 视图：

.. snippet::
    :filename: polls/views.py

    from django.shortcuts import render

    from .models import Question


    def index(request):
        latest_question_list = Question.objects.order_by('-pub_date')[:5]
        context = {'latest_question_list': latest_question_list}
        return render(request, 'polls/index.html', context)

注意，一旦我们在所有这些视图中完成这个操作，我们不再需要导入 :mod:`~django.template.loader` 和 :class:`~django.http.HttpResponse`
（如果您仍然有 ``detail``, ``results``, 和 ``vote`` 方法，您将需要保留HttpResponse）。

:func:`~django.shortcuts.render` 函数接受request对象作为其第一个参数，模板名称作为其第二个参数，
字典作为其可选的第三个参数。它返回一个 :class:`~django.http.HttpResponse` 对象，含有用给定的context 渲染后的模板。

404错误
========

现在，让我们处理Question 详细页面的视图 —— 显示Question内容的页面：

.. snippet::
    :filename: polls/views.py

    from django.http import Http404
    from django.shortcuts import render

    from .models import Question
    # ...
    def detail(request, question_id):
        try:
            question = Question.objects.get(pk=question_id)
        except Question.DoesNotExist:
            raise Http404("Question does not exist")
        return render(request, 'polls/detail.html', {'question': question})

这里的新概念：如果具有所请求的ID的问题不存在，则该视图引发 :exc:`~django.http.Http404` 异常。
我们将在以后讨论你可以在 ``polls/detail.html`` 模板文件里放些什么代码，但如果你想快点运行上面的例子，仅仅只需要：

.. snippet:: html+django
    :filename: polls/templates/polls/detail.html

    {{ question }}


快捷方法：get_object_or_404()
--------------------------------

一种常见的习惯是使用 :meth:`~django.db.models.query.QuerySet.get` 并在对象不存在时引发 :exc:`~django.http.Http404`。
Django为此提供一个快捷方式。 下面是重写后的 ``detail()`` 视图：

.. snippet::
    :filename: polls/views.py

    from django.shortcuts import get_object_or_404, render

    from .models import Question
    # ...
    def detail(request, question_id):
        question = get_object_or_404(Question, pk=question_id)
        return render(request, 'polls/detail.html', {'question': question})

:func:`~django.shortcuts.get_object_or_404` 函数将一个Django模型作为它的第一个参数，
任意数量的关键字参数作为它的第二个参数，它会将这些关键字参数传递给模型管理器中的 :meth:`~django.db.models.query.QuerySet.get` 函数。
如果对象不存在，它就引发一个 :exc:`~django.http.Http404` 异常。

.. admonition:: Philosophy

    为什么我们要使用一个辅助函数 :func:`~django.shortcuts.get_object_or_404` 而不是在更高层自动
    捕获 :exc:`~django.core.exceptions.ObjectDoesNotExist` 异常，或者让模型的API引发 :exc:`~django.http.Http404`
    而不是 :exc:`~django.core.exceptions.ObjectDoesNotExist` ？

    因为那样做将会使模型层与视图层耦合在一起。 Django最重要的一个设计目标就是保持松耦合。
    一些可控的耦合将会在 :mod:`django.shortcuts` 模块中介绍。

还有一个 :func:`~django.shortcuts.get_list_or_404` 函数，它的工作方式类似
:func:`~django.shortcuts.get_object_or_404`  —— 差别在于它使用 :meth:`~django.db.models.query.QuerySet.filter` 而
不是 :meth:`~django.db.models.query.QuerySet.get` 。如果列表为空则引发 :exc:`~django.http.Http404` 。


使用模板系统
=============

回到我们投票应用的 ``detail()`` 视图。 根据context 变量 ``question`` ，下面是 ``polls/detail.html`` 模板可能的样子：

.. snippet:: html+django
    :filename: polls/templates/polls/detail.html

    <h1>{{ question.question_text }}</h1>
    <ul>
    {% for choice in question.choice_set.all %}
        <li>{{ choice.choice_text }}</li>
    {% endfor %}
    </ul>

模板系统使用点查找语法访问变量属性。在 ``{{question.question_text}}`` 的示例中，
首先Django对对象问题进行字典查找。如果没有，它尝试一个属性查找 - 在这种情况下工作。
如果属性查找失败，它将尝试列表索引查找。

方法调用发生在 :ttag:`{% for %}<for>` 循环中：``question.choice_set.all`` 被解释为Python的代码
``question.choice_set.all()`` ，它返回一个由Choice对象组成的可迭代对象，
并将其用于 :ttag:`{% for %}<for>` 标签。访问 :doc:`模板指南</topics/templates>` 来了解更多关于模板的信息。

移除模板中硬编码的URLs
=======================

我们在 ``polls/index.html`` 模板中编写一个指向Question的链接时，链接中一部分是硬编码的：

.. code-block:: html+django

    <li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>

这种硬编码、紧耦合的方法有一个问题，就是如果我们想在拥有许多模板文件的项目中修改URLs，
那将会变得非常麻烦。 但是，因为你在 ``polls.urls`` 模块的 ``url()`` 函数中定义了name 参数，
所以你可以通过使用 ``{% url %}`` 模板标签来移除对你的URL配置中定义的特定的URL的依赖：

.. code-block:: html+django

    <li><a href="{% url 'detail' question.id %}">{{ question.question_text }}</a></li>

它的工作原理是在 ``polls.urls`` 模块里查找指定的URL的定义。你可以看到名为‘detail’的URL的准确定义::

    ...
    # the 'name' value as called by the {% url %} template tag
    url(r'^(?P<question_id>[0-9]+)/$', views.detail, name='detail'),
    ...

如果你想把polls应用中 ``detail`` 视图的URL改成其它样子比如 ``polls/specifics/12/`` ，就可以不必在该模板（或者多个模板）中修改它，
只需要修改 ``polls/urls.py`` ::

    ...
    # added the word 'specifics'
    url(r'^specifics/(?P<question_id>[0-9]+)/$', views.detail, name='detail'),
    ...

URL name的命名空间
===================

教程中的这个项目只有一个应用 ``polls`` 。在真实的Django项目中，可能会有五个、十个、二十个或者更多的应用。
Django如何区分它们URL的名字呢？ 例如，``polls`` 应用具有一个 ``detail`` 视图，
相同项目中的博客应用可能也有这样一个视图。当使用模板标签 ``{% url %}`` 时，
人们该如何做才能使得Django知道为一个URL创建哪个应用的视图？

答案是在你的主URLconf下添加命名空间。 在 ``mysite/urls.py`` 文件中，添加命名空间将它修改成：

.. snippet::
    :filename: polls/urls.py

    from django.conf.urls import url

    from . import views

    app_name = 'polls'
    urlpatterns = [
        url(r'^$', views.index, name='index'),
        url(r'^(?P<question_id>[0-9]+)/$', views.detail, name='detail'),
        url(r'^(?P<question_id>[0-9]+)/results/$', views.results, name='results'),
        url(r'^(?P<question_id>[0-9]+)/vote/$', views.vote, name='vote'),
    ]

现在将你的 ``polls/index.html`` 改为具有命名空间的详细视图：

.. snippet:: html+django
    :filename: polls/templates/polls/index.html

    <li><a href="{% url 'polls:detail' question.id %}">{{ question.question_text }}</a></li>

When you're comfortable with writing views, read  to learn about simple form processing and generic views.
当您熟悉编写视图时，请阅读本教程的 :doc:`第四部分 </intro/tutorial04>` ，了解简单的表单处理和通用视图
