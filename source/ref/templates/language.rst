============================
Django模板语言
============================

本文介绍Django模板系统的语法. 如果你想从技术角度了解它的工作原理以及如何扩展它, 请参见 :doc:`/ref/templates/api`.

Django的模板语言旨在在功能和易用性之间取得平衡. 它的设计目的是让那些习惯于使用HTML的人能够自如应对.
如果您接触过其他基于文本的模板语言, 例如 Smarty_ 或 Jinja2_, 那么您将对Django的模板语言感到一见如故.

.. admonition:: Philosophy

    如果您有编程方面的背景知识, 或者您习惯于将程序代码直接混入HTML中的语言, 那么您需要记住,
    Django模板系统不仅仅是嵌入到HTML中的Python. 它是通过设计实现的: 模板系统旨在表示表现形式, 而不是程序逻辑.

    Django模板系统提供了类似于编程结构的标签 -- 用于布尔判断的 :ttag:`if` 标签, 用于循环的 :ttag:`for` 标签等. --
    这些标签并不是简单地作为相应的Python代码执行的, 模板系统也不会执行任何Python表达式.
    默认情况下, 仅支持下面列出的标签, 过滤器和语法(尽管您可以根据需要向模板系统中添加 :doc:`自定义扩展
    </howto/custom-template-tags>`).

.. _`The Django template language: For Python programmers`: ../templates_python/
.. _Smarty: http://www.smarty.net/
.. _Jinja2: http://jinja.pocoo.org/

模板
=========

.. highlightlang:: html+django

模板是一个文本文件. 它可以生成任何基于文本的格式(HTML, XML, CSV 等).

一个模板包含使用时会被值替换的 **变量** 和控制模版逻辑的 **标签**.

下面是一个基础的模板, 展示了一些基础的内容. 在后面的文档中会解释其中每个元素.

.. code-block:: html+django

    {% extends "base_generic.html" %}

    {% block title %}{{ section.title }}{% endblock %}

    {% block content %}
    <h1>{{ section.title }}</h1>

    {% for story in story_list %}
    <h2>
      <a href="{{ story.get_absolute_url }}">
        {{ story.headline|upper }}
      </a>
    </h2>
    <p>{{ story.tease|truncatewords:"100" }}</p>
    {% endfor %}
    {% endblock %}

.. admonition:: Philosophy

    为什么使用基于文本的模板而不是基于XML的模板(比如Zope的TAL)?
    我们希望Django的模板语言可以用在更多的地方, 而不仅仅是XML/HTML模版.
    你可以将模板语言用于任何基于文本的格式中, 如电子邮件,JavaScript和CSV.

    还有, 让人去编辑XML简直是一种虐待!

.. _template-variables:

变量
=========

变量看起来就像这样: ``{{ variable }}``. 当模板引擎遇到一个变量时, 它会计算这个变量并用结果替换它.
变量名可以由任意字母数字字符和下划线(``"_"``)组成. 点(``"."``)也会在变量部分出现,
它具有特殊含义, 我们将在后面说明. 重要的是, **变量名中不能有空格或标点符号**.

使用点号 (``.``)来访问变量的属性.

.. admonition:: Behind the scenes

    从技术上讲, 当模板系统遇到一个点时, 它会按照以下顺序进行查找:

    * 字典查找 (Dictionary lookup)
    * 属性或方法查找 (Attribute or method lookup)
    * 数字索引查找 (Numeric index lookup)

    如果结果值是可调用的, 则会无参调用. 调用的结果值成为模板值.

    这个查询顺序, 会对优先于字典查找的对象上造成意想不到的行为.
    例如, 思考下面的代码片段, 它原意是想遍历 ``collections.defaultdict``::

        {% for k, v in defaultdict.iteritems %}
            Do something with k and v here...
        {% endfor %}

    因为字典查找是先执行的, 这时返回了一个默认值, 而不是使用预期的 ``.iteritems()`` 方法.
    在这种情况下, 可以考虑先转换为字典.

在上面例子中, ``{{ section.title }}`` 将被 ``section`` 对象的 ``title`` 属性替代.

如果你使用一个不存在的变量, 模板系统会插入 ``string_if_invalid`` 选项的值, 它被默认设置为 ``''`` (空字符串).

请注意, 像 ``{{ foo.bar }}`` 这样的模板表达式中的"bar", 如果在模板上下文中存在的话, 将被解释为一个字面意义字符串, 而不是使用变量"bar"的值.

过滤器
=======

**过滤器** 用来修改展示的变量.

过滤器看起来像这样: ``{{ name|lower }}``. 这将显示变量 ``{{ name }}`` 被过滤器 :tfilter:`lower` 修改后的值,
它将文本转换成小写形式. 使用管道符 (``|``) 来应用过滤器.

过滤器可以 "链式." 一个过滤器的输出将应用到下一个.
``{{ text|escape|linebreaks }}`` 就是一个常用的过滤器链, 它用于转义文本内容, 然后将换行符转换为 ``<p>`` 标签.

过滤器可以接收参数. 例如: ``{{ bio|truncatewords:30 }}``. 它显示变量 ``bio`` 的前30个字符.

包含空格的过滤器参数必须加引号; 例如, 连接一个包含逗号和空格的列表: ``{{ list|join:", " }}``.

Django提供了大约60个内置的模板过滤器. 你可以在
:ref:`内置过滤器参考 <ref-templates-builtins-filters>` 中查看.
为了让你了解模板过滤器, 这里列出了一些比较常用的模板过滤器:

:tfilter:`default`
    如果变量为false或者空, 则使用给定的默认值. 否则, 使用变量的值. 例如::

        {{ value|default:"nothing" }}

    如果 ``value`` 不存在或者为空, 那么将显示"``nothing``".

:tfilter:`length`
    返回值的长度. 这对字符串和列表都有效. 例如::

        {{ value|length }}

    如果 ``value`` 为 ``['a', 'b', 'c', 'd']`` 时, 则将显示 ``4``.

:tfilter:`filesizeformat`
    转换为"易读"的文件大小 (例如. ``'13 KB'``,
    ``'4.1 MB'``, ``'102 bytes'``, 等.). 例如::

        {{ value|filesizeformat }}

    如果 ``value`` 为 123456789, 则将显示为 ``117.7 MB``.

这些只是几个例子; 查看 :ref:`内置过滤器参考
<ref-templates-builtins-filters>` 了解所有的内置过滤器.

你也可以自定义过滤器; 详见
:doc:`/howto/custom-template-tags`.

.. seealso::

    Django的admin接口提供了一个模板标签和可用的过滤器的完整参考. 详见
    :doc:`/ref/contrib/admin/admindocs`.

标签
====

标签看起来像这样: ``{% tag %}``. 标签比变量更复杂: 有些用于在输出中创建文本, 有些用于控制循环或逻辑, 有些用于加载外部信息到模板中供以后的变量使用.

有些标签要求要有开始和结束标签 (例如. ``{% tag %} ... 标签正文
... {% endtag %}``).

Django附带了大约24个内置模板标签, 你可以在 :ref:`内置标签参考 <ref-templates-builtins-tags>` 中了解他们. 为了让你了解这些标签, 下面是一些常用的标签:

:ttag:`for`
    循环遍历数组中的每个元素. 例如, 显示 ``athlete_list`` 的athletes列表::

        <ul>
        {% for athlete in athlete_list %}
            <li>{{ athlete.name }}</li>
        {% endfor %}
        </ul>

:ttag:`if`, ``elif``, 和 ``else``
    判断变量的布尔值, 如果为 "true" 则显示其中内容::

        {% if athlete_list %}
            Number of athletes: {{ athlete_list|length }}
        {% elif athlete_in_locker_room_list %}
            Athletes should be out of the locker room soon!
        {% else %}
            No athletes.
        {% endif %}

    在上面例子中, 如果 ``athlete_list`` 不为空, 则会根据 ``{{ athlete_list|length }}`` 显示athlete的数量.
    否则, 如果 ``athlete_in_locker_room_list`` 不为空, 将会显示 "Athletes
    should be out...". 如果两个列表都为空,
    将会显示"No athletes.".

    :ttag:`if` 标签中也可以使用过滤器或其他操作符::

        {% if athlete_list|length > 1 %}
           Team: {% for athlete in athlete_list %} ... {% endfor %}
        {% else %}
           Athlete: {{ athlete_list.0.name }}
        {% endif %}

    虽然上面的例子是可行的, 需要注意, 大多数模版过滤器返回字符串, 所以使用过滤器做数学的比较通常都不会像您期望的那样工作. :tfilter:`length` 是个例外.

:ttag:`block` 和 :ttag:`extends`
    参见 `模板继承`_ (见下文), 一种减少模板中重复代码的有效方法.

这些只是几个例子; 查看 :ref:`内置标签参考
<ref-templates-builtins-tags>` 了解所有的内置标签.

你也可以自定义过滤标签; 详见
:doc:`/howto/custom-template-tags`.

.. seealso::

    Django的admin接口提供了一个模板标签和可用的过滤器的完整参考. 详见
    :doc:`/ref/contrib/admin/admindocs`.

.. _模板继承:

注释
========

要注释模板中的内容, 请使用注释语法: ``{# #}``.

例如, 下面内容会显示为 ``'hello'``::

    {# greeting #}hello

注释可以包含任何有效或无效的代码. 例如::

    {# {% if foo %}bar{% else %} #}

这种语法只能用于单行注释 (在 ``{#`` 和 ``#}`` 中不允许有换行). 如果你需要注释掉模版中的多行内容, 请参见 :ttag:`comment` 标签.

.. _template-inheritance:

模板继承
====================

Django的模板引擎中 -- 最强大 -- 也是最复杂的部分是模板继承. 模板继承允许你创建一个基础的“骨架”模板,
它包含了网页的所有常用元素, 并定义子模板可以覆盖的 **blocks**.

通过下面这个例子, 可以很容易的理解模版继承::

    <!DOCTYPE html>
    <html lang="en">
    <head>
        <link rel="stylesheet" href="style.css" />
        <title>{% block title %}My amazing site{% endblock %}</title>
    </head>

    <body>
        <div id="sidebar">
            {% block sidebar %}
            <ul>
                <li><a href="/">Home</a></li>
                <li><a href="/blog/">Blog</a></li>
            </ul>
            {% endblock %}
        </div>

        <div id="content">
            {% block content %}{% endblock %}
        </div>
    </body>
    </html>

我们将上面模板称为 ``base.html``, 它定义了一个基础的HTML骨架模板, 用来制作一个简单的两栏式页面. "子" 模板的任务是用内容填充空块.

在上面例子中, 使用 :ttag:`block` 标签定义三个可以被子模板填充的块. :ttag:`block` 的作用就是告诉模版引擎: 子模版可能会覆盖掉模版中的这些位置.

子模板可能是这样的::

    {% extends "base.html" %}

    {% block title %}My amazing blog{% endblock %}

    {% block content %}
    {% for entry in blog_entries %}
        <h2>{{ entry.title }}</h2>
        <p>{{ entry.body }}</p>
    {% endfor %}
    {% endblock %}

:ttag:`extends` 标签是其中的关键. 它告诉模版引擎, 这个模版"继承"了另一个模版. 当模版引擎处理这个模版时, 首先它将定位父模版 -- 在此例中, 就是"base.html".

此时, 模板引擎将注意到 ``base.html`` 中的三个 :ttag:`block` 标签, 并用子模板的内容替换这些块. 依据 ``blog_entries`` 的值可能显示如下::

    <!DOCTYPE html>
    <html lang="en">
    <head>
        <link rel="stylesheet" href="style.css" />
        <title>My amazing blog</title>
    </head>

    <body>
        <div id="sidebar">
            <ul>
                <li><a href="/">Home</a></li>
                <li><a href="/blog/">Blog</a></li>
            </ul>
        </div>

        <div id="content">
            <h2>Entry one</h2>
            <p>This is my first entry.</p>

            <h2>Entry two</h2>
            <p>This is my second entry.</p>
        </div>
    </body>
    </html>

注意, 由于子模板中没有定义 ``sidebar`` 块, 所以直接使用父模板的值. 父模板  ``{% block %}`` 标签中的内容总是作为备选.

你可以根据需要使用任意层次的继承. 一种常见的使用继承的方式是下面的三层继承结构:

* 创建一个 ``base.html`` 模板来控制网站的主要外观及风格.
* 为站点的每个"部分"创建一个 ``base_SECTIONNAME.html`` 模板. 比如, ``base_news.html``, ``base_sports.html``.
  这些模板都继承于 ``base.html``, 用来控制这些特定部分的样式和内容.
* 为每一种页面类型创建独立的模版, 例如新闻内容或者博客文章. 这些模版继承于对应部分的模版.

这种方式使代码得到最大程度的复用, 并且使得添加内容到共享的内容区域更加简单, 例如部分的导航.

下面是一些使用继承技巧:

* 使用模板继承时, :ttag:`{% extends %}<extends>` 必须是模板文件中的第一个标签, 否则模板继承不会生效.

* 在基础模板中使用的 :ttag:`{% block %}<block>` 标签越多越好. 记住, 在子模板中不必实现父模板中所有的blocks,
  所以你可以在某些的blocks中填入合理的默认值, 然后之后只需要定义需要的block, 钩子多总比钩子少好.

* 如果你发现自己的内容在多个模板中重复, 那么可能你需要考虑将这些内容移到父模板的 ``{% block %}`` 中.

* 如果你需要获取父模板block中的内容, 可以使用 ``{{ block.super }}`` 变量.
  如果你想在父模板中添加的内容而不是覆盖它, 这很有用.
  使用 ``{{ block.super }}`` 插入的数据不会被自动转义(参见 `下一节`_), 因为如果需要的话, 它已经在父模板中被转义了.

* 为增加可读性, 你可以给
  ``{% endblock %}`` 标签起一个 *名字*. 例如::

      {% block content %}
      ...
      {% endblock content %}

  在大型模版中, 这个方法可以帮你清楚的看到哪一个 ``{% block %}`` 标签被关闭了.

最后, 请注意不要在同一模板中定义多个具有相同名称标签. 此限制的存在是因为 :ttag:`block` 标签是"双向"工作的.
这个意思是, :ttag:`block` 标签不仅是提供了一个填充的位置, 它定义了向父模版的位置中所填的内容.
如果在一个模版中有两个名字一样的 :ttag:`block` 标签, 模版的父模版将不知道使用哪个block的内容.

.. _下一节: #automatic-html-escaping
.. _automatic-html-escaping:

自动转义HTML
=======================

从模板生成HTML总是存在这样一个风险: 即变量值可能会包含影响HTML最终呈现的字符. 例如, 考虑这个模板片段::

    Hello, {{ name }}

乍一看, 这似乎是一种无伤大雅的显示用户姓名的方式, 但考虑一下, 如果用户将自己的姓名设置为::

    <script>alert('hello')</script>

使用这个值, 该模板会被渲染为::

    Hello, <script>alert('hello')</script>

...这意味着浏览器会弹出一个JavaScript警报框!

同样, 如果名称中包含一个 ``'<'`` 符号，像这样呢?

.. code-block:: html

    <b>username

这样一来, 呈现出来的模板就会是这样的::

    Hello, <b>username

...这又会导致网页的其余部分被加粗!

显然, 不应该盲目信任用户提交的数据, 并且直接插入到你的网页中, 因为恶意用户可能会利用这种漏洞做潜在的坏事.
这种类型的安全漏洞被称为 `跨站点脚本`_ (XSS) 攻击.

为避免这个问题, 你有两个选择:

* 第一, 对每个不被信任的变量使用 :tfilter:`escape` 过滤器(见下文), 它可以将潜在的有害HTML字符转换为无害的字符.
  在Django的开始几年里这是默认的解决方案, 但是这样就把责任给了 *你*, 也就是开发者/模板作者, 需要你来确保都被转义了,
  但是很容易忘记对数据进行转义.

* 第二, 你可以利用Django的HTML自动转义功能. 本节剩余部分将介绍自动转义的工作原理.

默认情况下, 在Django中模板都会为每个变量输出自动转义. 明确地说, 这五个字符会被转义:

* ``<`` 会被转成 ``&lt;``
* ``>`` 会被转成 ``&gt;``
* ``'`` (单引号) 会被转成 ``&#39;``
* ``"`` (双引号) 会被转成 ``&quot;``
* ``&`` 会被转成 ``&amp;``

再次强调这个行为是默认启用的. 如果你使用Django的模板系统, 你就会受到此保护.

.. _跨站点脚本: https://en.wikipedia.org/wiki/Cross-site_scripting

如何关闭它
------------------

如果你不希望数据自动转义, 无论是在站点, 模板还是变量级别, 可以使用下面几种方法来关闭它.

为什么想要关闭它呢? 因为有时, 模板变量中包含一些希望以原始HTML形式呈现的数据,
并不想转义这些内容. 例如, 在数据库中储存一些HTML代码, 并希望将其直接嵌入到模板中.
或者, 使用Django的模板系统来生成非HTML的文本 -- 比如邮件内容.

对于单个变量
~~~~~~~~~~~~~~~~~~~~~~~~

关闭单个变量的自动转义可以使用 :tfilter:`safe` 过滤器::

    This will be escaped: {{ data }}
    This will not be escaped: {{ data|safe }}

可以把 *safe* 看作是 *safe from further escaping* 或者 *can be
safely interpreted as HTML* 的意思. 在这个例子中, 如果 ``data`` 带有 ``'<b>'``,
输出将是::

    This will be escaped: &lt;b&gt;
    This will not be escaped: <b>

对于模板块
~~~~~~~~~~~~~~~~~~~

要控制模板的自动转义, 可以将模板(或模板的某一部分)放在 :ttag:`autoescape` 标签中, 像这样::

    {% autoescape off %}
        Hello {{ name }}
    {% endautoescape %}

:ttag:`autoescape` 标签接收 ``on`` 或者 ``off`` 参数. 有时, 可能会想在没有启用自动转义的情况下强制转义. 以下是一个模板示例::

    Auto-escaping is on by default. Hello {{ name }}

    {% autoescape off %}
        This will not be auto-escaped: {{ data }}.

        Nor this: {{ other_data }}
        {% autoescape on %}
            Auto-escaping applies again: {{ name }}
        {% endautoescape %}
    {% endautoescape %}

自动转义标签会作用于扩展当前模板的模板以及通过 :ttag:`include` 标签包含的模板, 就像所有block标签那样. 例如:

.. snippet::
    :filename: base.html

    {% autoescape off %}
    <h1>{% block title %}{% endblock %}</h1>
    {% block content %}
    {% endblock %}
    {% endautoescape %}

.. snippet::
    :filename: child.html

    {% extends "base.html" %}
    {% block title %}This &amp; that{% endblock %}
    {% block content %}{{ greeting }}{% endblock %}

由于在基础模板中关闭了自动转义, 所以在子模板中也会关闭,
当 ``greeting`` 变量带有 ``<b>Hello!</b>`` 字符时, 会呈现以下的HTML渲染结果::

    <h1>This &amp; that</h1>
    <b>Hello!</b>

注意
-----

一般来说, 模板作者不需要太担心自动转义的问题. Python端的开发人员(编写视图和自定义过滤器的人)会考虑在哪些情况下数据不应该被转义, 并进行适当地标记数据让其在模板中正常工作.

如果你在不确定是否启用自动转义的情况下创建的模板, 那么可以在所有需要转义的变量中添加一个 :tfilter:`escape` 过滤器.
在自动转义开启时, 也不会有 :tfilter:`escape` 过滤器 *双重转义* 数据的问题  --  :tfilter:`escape` 过滤器不会影响已自动转义的变量.

.. _string-literals-and-automatic-escaping:

字符串和自动转义
--------------------------------------

正如前面提到的, 过滤器的参数可以是字符串::

    {{ data|default:"This is a string literal." }}

所有的字符串文本都是在 **没有** 自动转义的情况下插入到模板中的 --  它们的行为类似于通过 :tfilter:`safe` 过滤器传入一样.
这背后的原因是, 模板作者可以控制字符串文本的内容, 因此他们可以确保在编写模板时正确地转义文本.

这意味着你应当这么写::

    {{ data|default:"3 &lt; 2" }}

...而不是::

    {{ data|default:"3 < 2" }}  {# Bad! Don't do this. #}

这并不影响来源于变量自身的数据. 变量的内容在必要时仍然会自动转义, 因为它们不受模板作者的控制.

.. _template-accessing-methods:

方法调用
======================

对象的大多数方法同样可以在模板中调用. 这意味着模板能够访问到的不仅仅是类属性(比如字段名称)和视图中传入的变量.
例如, Django ORM提供了 :ref:`"entry_set"<topics-db-queries-related>` 语法用于查找关联到外键的对象集合.
因此, 如果模型"comment"有一个外键关联到模型"task", 你可以通过 "task" 遍历其所有的comments, 像这样::

    {% for comment in task.comment_set.all %}
        {{ comment }}
    {% endfor %}

同样, :doc:`QuerySets</ref/models/querysets>` 提供了一个 ``count()`` 方法来计算它们所包含的对象数量.
因此, 可以通过以下方法获得与当前task相关的所有comment的数量::

    {{ task.comment_set.all.count }}

你也可以访问在模型上定义的方法:

.. snippet::
    :filename: models.py

    class Task(models.Model):
        def foo(self):
            return "bar"

.. snippet::
    :filename: template.html

    {{ task.foo }}

由于Django有意限制了模板语言中的逻辑处理, 不能够在模板中传递参数来调用方法. 所以数据应该在视图中处理然后传递给模板展示.

.. _loading-custom-template-libraries:

自定义标签和自定义库
===============================

某些应用提供了自定义标签和过滤器库. 如果要在模板中访问它们, 请确保应用在 :setting:`INSTALLED_APPS` 中(本例中我们会添加 'django.contrib.humanize'),
然后在模板中使用 :ttag:`load` 标签::

    {% load humanize %}

    {{ 45000|intcomma }}

在上面例子中, :ttag:`load` 标签加载了 ``humanize`` 标签库, 然后使 ``intcomma`` 过滤器可以使.
如果你启用了 :mod:`django.contrib.admindocs`, 你可以在admin站点的文档区查找安装的自定义库列表.

:ttag:`load` 标签可以接收多个库名, 使用空格分开.
例如::

    {% load humanize i18n %}

如何自定义模板库请参见 :doc:`/howto/custom-template-tags`.

自定义库和模板继承
-----------------------------------------

当你加载自定义标签或过滤器库时, 标签/过滤器仅对当前模板可用, 而不是沿模板继承路径的所有父模板或子模板都可用.

例如, 假设模板 ``foo.html`` 带有  ``{% load humanize %}``, 子模版(例如, 带有 ``{% extends "foo.html" %}`` 的模板)中
*不能* 访问humanize模板标签和过滤器. 子模版需要自己添加 ``{% load humanize %}``.

这是因为这能使模板更健全且更好维护.

.. seealso::

    :doc:`模板参考 </ref/templates/index>`
        涵盖内置标签, 内置过滤器, 使用替代模板, 语言等.
