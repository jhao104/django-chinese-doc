==================================
内置标签和过滤器
==================================

本文档介绍Django的内置模板标签和过滤器. 建议您 :doc:`自动文档
</ref/contrib/admin/admindocs>` (如果可用), 因为这包括安装的所有自定义标签或过滤器的文档.

.. _ref-templates-builtins-tags:

内置标签参考
======================

.. highlight:: html+django

.. templatetag:: autoescape

``autoescape``
--------------

控制自动转义行为. 此标记接受 ``on`` 或 ``off`` 作为参数, 决定是否在block内自动转义. block需以 ``endautoescape`` 结束标签关闭.

当启用自动转义时, 在将结果输出之前(在所有任何过滤器之后), 所有变量内容都将进行HTML转义. 这相当于对每个变量手动应用 :tfilter:`escape` 过滤器.

唯一的例外是被标记为"safe"的变量, 也就是被应用了 :tfilter:`safe` 或 :tfilter:`escape` 过滤器的变量或者填充的代码.

用例::

    {% autoescape on %}
        {{ body }}
    {% endautoescape %}

.. templatetag:: block

``block``
---------

定义一个可以被子模板覆盖的块. 更多信息请参见 :ref:`模板继承 <template-inheritance>`.

.. templatetag:: comment

``comment``
-----------

注释 ``{% comment %}`` 和 ``{% endcomment %}`` 之间的内容. 可在第一个标记中插入可选的注释. 这在注释代码时记录代码被禁用的原因非常有用.

用例::

    <p>Rendered text with {{ pub_date|date:"c" }}</p>
    {% comment "可添加备注" %}
        <p>Commented out text with {{ create_date|date:"c" }}</p>
    {% endcomment %}

``comment`` 标签不能嵌套使用.

.. templatetag:: csrf_token

``csrf_token``
--------------

该标签用于CSRF保护, 具体可以参考
:doc:`跨站点请求伪造保护 </ref/csrf>`.

.. templatetag:: cycle

``cycle``
---------

每次访问这个标记时生成一个参数. 第一次访问时生成第一个参数, 第二次访问时生成第二个参数, 依此类推. 所有参数用完后, 将从第一个参数开始循环使用.

这个标签在循环中特别有用::

    {% for o in some_list %}
        <tr class="{% cycle 'row1' 'row2' %}">
            ...
        </tr>
    {% endfor %}

第一次迭代将生成class为 ``row1`` 的HTML, 第二次为 ``row2``, 第三次又变成 ``row1``, 每次迭代以此类推.

也可以使用变量. 例如, 假如有两个模板变量 ``rowvalue1`` 和 ``rowvalue2``, 可以像这样交替使用它们的值::

    {% for o in some_list %}
        <tr class="{% cycle rowvalue1 rowvalue2 %}">
            ...
        </tr>
    {% endfor %}

在循环中的变量将被转义. 可以通过以下方式禁用自动转义::

    {% for o in some_list %}
        <tr class="{% autoescape off %}{% cycle rowvalue1 rowvalue2 %}{% endautoescape %}">
            ...
        </tr>
    {% endfor %}

字符串和变量可以组合使用::

    {% for o in some_list %}
        <tr class="{% cycle 'row1' rowvalue2 'row3' %}">
            ...
        </tr>
    {% endfor %}

在某些情况下, 您可能希望引用循环的当前值, 而不是下一个循环值. 为此, 只需使用"as"给 ``{% cycle %}`` 标签一个别名, 如下所示::

    {% cycle 'row1' 'row2' as rowcolors %}

这样, 就可以通过引用循环名称作为上下文变量, 在模板中任意位置插入循环的当前值. 如果要独立于原始 ``cycle`` 标签将循环移动到下一个值, 可以使用另一个 ``cycle`` 标签并指定变量的名称. 因此::

    <tr>
        <td class="{% cycle 'row1' 'row2' as rowcolors %}">...</td>
        <td class="{{ rowcolors }}">...</td>
    </tr>
    <tr>
        <td class="{% cycle rowcolors %}">...</td>
        <td class="{{ rowcolors }}">...</td>
    </tr>

将会输出::

    <tr>
        <td class="row1">...</td>
        <td class="row1">...</td>
    </tr>
    <tr>
        <td class="row2">...</td>
        <td class="row2">...</td>
    </tr>

可以在 ``cycle`` 标签中使用任意数量的值, 用空格分隔. 用单引号(``'``)或双引号(``"``)括起来的值被视为字符串, 不带引号的值被视为模板变量.

默认情况下, 当 ``as`` 关键字与cycle标签一起使用时, 启动循环的 ``{% cycle %}`` 本身将生成循环中的第一个值.
如果要在嵌套循环或包含的模板中使用该值, 则可能会出现问题. 如果只想声明循环, 但不想生成第一个值, 那么可以添加一个 ``silent`` 关键字作为标签中的最后一个关键字. 例如::

    {% for obj in some_list %}
        {% cycle 'row1' 'row2' as rowcolors silent %}
        <tr class="{{ rowcolors }}">{% include "subtemplate.html" %}</tr>
    {% endfor %}

这将输出一个交替使用 ``row1`` 和 ``row2`` 作为class的 ``<tr>`` 列表.
子模板可以在上下文中访问 ``rowcolors``, 并且该值将匹配包含它的 ``<tr>`` 类.
如果省略 ``silent`` 关键字, ``row1`` 和 ``row2`` 将作为普通文本在 ``<tr>`` 元素外出现.

当在cycle中定义上silent关键字时, silent将自动适用于该循环标签的所有后续使用.
以下模板将 *不* 输出任何东西, 即使第二次调用 ``{% cycle %}`` 时并没有指定 ``silent``::

    {% cycle 'row1' 'row2' as rowcolors silent %}
    {% cycle rowcolors %}

.. templatetag:: debug

``debug``
---------

输出全部调试信息, 包括当前上下文和导入的模块

.. templatetag:: extends

``extends``
-----------

表示当前模板继承自一个父模板.

该标签有两种用法:

* ``{% extends "base.html" %}`` (带引号) 使用字符串
  ``"base.html"`` 作为继承的父模板名称.

* ``{% extends variable %}`` 使用 ``variable`` 的值. 如果该变量值是一个字符串, Django将使用这个字符串作为父模板的名称.
  如果该变量值是一个 ``Template`` 对象, Django将使用该对象作为父模板.

更多信息见 :ref:`template-inheritance`.

字符串参数可以是以 ``./`` 和 ``../`` 开头的相对路径. 例如, 假设目录结构如下::

    dir1/
        template.html
        base2.html
        my/
            base3.html
    base1.html

在 ``template.html``, 可以像这样使用::

    {% extends "./base2.html" %}
    {% extends "../base1.html" %}
    {% extends "./my/base3.html" %}

.. versionadded:: 1.10

   新增使用相对路径的功能.

.. templatetag:: filter

``filter``
----------

通过一个或多个过滤器过滤block中的内容. 可以使用管道符连接多个过滤器, 过滤器可以使用参数, 就像变量语法一样.

请注意, 该block包括 ``filter`` 和 ``endfilter`` 标签之间的 *所有* 文本.

示例用法::

    {% filter force_escape|lower %}
        该文本会被 HTML-escaped, 然后转换成小写形式.
    {% endfilter %}

.. note::

    :tfilter:`escape` 和 :tfilter:`safe` 过滤器是不可接受参数的. 取而代之的是, 使用 :ttag:`autoescape` 标签来管理模板代码块的自动转义.

.. templatetag:: firstof

``firstof``
-----------

输出第一个不为 ``False`` 的参数. 如果传入的所有参数都为 ``False``, 就什么也不输出.

示例::

    {% firstof var1 var2 var3 %}

它等价于::

    {% if var1 %}
        {{ var1 }}
    {% elif var2 %}
        {{ var2 }}
    {% elif var3 %}
        {{ var3 }}
    {% endif %}

可以用一个字符串作为默认输出以防传入的所有变量都是False::

    {% firstof var1 var2 var3 "fallback value" %}

该标签会自动转义变量值. 可以通过以下方式禁用自动转义::

    {% autoescape off %}
        {% firstof var1 var2 var3 "<strong>fallback value</strong>" %}
    {% endautoescape %}

如果只是对某些变量避免转义, 可以这样::

    {% firstof var1 var2|safe var3 "<strong>fallback value</strong>"|safe %}

还可以使用这样 ``{% firstof var1 var2 var3 as value %}`` 的语法将输出结果储存至变量中.

.. versionadded:: 1.9

    新增 "as" 语法.

.. templatetag:: for

``for``
-------

循环迭代数组中的每一项, 使这些项在上下文中可用. 例如, 展示 ``athlete_list`` 中的每一个athlete::

    <ul>
    {% for athlete in athlete_list %}
        <li>{{ athlete.name }}</li>
    {% endfor %}
    </ul>

可以使用
``{% for obj in list reversed %}`` 来反向遍历列表.

如果要循环一个包含列表的列表, 可以将每个子列表中的值解包为单个变量.
例如, 假设上下文中包含一个名为 ``points`` 的(x,y)坐标列表, 可以使用下面的方法来输出每点::

    {% for x, y in points %}
        There is a point at {{ x }},{{ y }}
    {% endfor %}

它也可以用来访问字典中的项. 例如, 假设上下文中有个名为 ``data`` 的字典, 可以使用下面的方法展示字典的建和值::

    {% for key, value in data.items %}
        {{ key }}: {{ value }}
    {% endfor %}

需要注意的是点运算符, 字典键值查找会优先于方法查找. 因此如果 ``data`` 字典中含有一个名为
``'items'`` 的键, ``data.items`` 将会返回 ``data['items']`` 而不是
``data.items()``. 因此如果要在模板中使用这些方法, 请避免添加和字典方法同名的键(``items``, ``values``, ``keys``,
等.). 更多关于点运算符的查找顺序请参见
:ref:`模板变量 <template-variables>`.

for循环体内可以直接使用的变量:

==========================  ===============================================
变量                         描述
==========================  ===============================================
``forloop.counter``         表示当前循环的索引 (从1开始)
``forloop.counter0``        表示当前循环的索引 (从0开始)
``forloop.revcounter``      到循环结束的迭代次数(最后一次为1)
``forloop.revcounter0``     到循环结束的迭代次数(最后一次为0)
``forloop.first``           当前循环为首个迭代时该变量为True
``forloop.last``            当前循环为最后个迭代时该变量为True
``forloop.parentloop``      在嵌套循环中, 指向当前循环的上级循环
==========================  ===============================================

``for`` ... ``empty``
---------------------

``for`` 标签还可以配合可选 ``{% empty %}`` 标签使用, 用于当列表为空或不存在时输出特定内容::

    <ul>
    {% for athlete in athlete_list %}
        <li>{{ athlete.name }}</li>
    {% empty %}
        <li>Sorry, no athletes in this list.</li>
    {% endfor %}
    </ul>

上面的代码与下面的代码是等同的, 但上面的代码更简短, 更清晰, 也可能更快::

    <ul>
      {% if athlete_list %}
        {% for athlete in athlete_list %}
          <li>{{ athlete.name }}</li>
        {% endfor %}
      {% else %}
        <li>Sorry, no athletes in this list.</li>
      {% endif %}
    </ul>

.. templatetag:: if

``if``
------

``{% if %}`` 标签会判断给定的变量, 当变量为 "true" (例如. 存在, 非空, 非布尔值False) 时就输出块内的内容::

    {% if athlete_list %}
        Number of athletes: {{ athlete_list|length }}
    {% elif athlete_in_locker_room_list %}
        Athletes should be out of the locker room soon!
    {% else %}
        No athletes.
    {% endif %}

在上例中, 如果 ``athlete_list`` 不为空, 则会通过
``{{ athlete_list|length }}`` 展示athlete的数量.

``if`` 标签可以配合一个或多个 ``{% elif %}`` 分支,
以及一个 ``{% else %}`` 分支, 当之前的所有分支条件都不满足时, 该分支会被显示.

布尔运算
~~~~~~~~~~~~~~~~~

:ttag:`if` 标签可以接受 ``and``, ``or`` 和 ``not`` 来判断多个变量或取反某个变量::

    {% if athlete_list and coach_list %}
        Both athletes and coaches are available.
    {% endif %}

    {% if not athlete_list %}
        There are no athletes.
    {% endif %}

    {% if athlete_list or coach_list %}
        There are some athletes or some coaches.
    {% endif %}

    {% if not athlete_list or coach_list %}
        There are no athletes or there are some coaches.
    {% endif %}

    {% if athlete_list and not coach_list %}
        There are some athletes and absolutely no coaches.
    {% endif %}

可以在同一个标签中同时使用 ``and`` 和 ``or``,
``and`` 优先级高于 ``or`` 例如.::

    {% if athlete_list and coach_list or cheerleader_list %}

将被解释为:

.. code-block:: python

    if (athlete_list and coach_list) or cheerleader_list

:ttag:`if` 标签中使用实际的括号是无效的语法, 如果需要括号来表示优先级, 应使用嵌套的 :ttag:`if` 标签.

:ttag:`if` 标签可以使用运算符 ``==``, ``!=``, ``<``, ``>``,
``<=``, ``>=``, ``in``, ``not in``, ``is``, 和 ``is not``, 其作用如下:

``==`` 运算
^^^^^^^^^^^^^^^

等于判断. 例如::

    {% if somevar == "x" %}
      This appears if variable somevar equals the string "x"
    {% endif %}

``!=`` 运算
^^^^^^^^^^^^^^^

不等于判断. 例如::

    {% if somevar != "x" %}
      This appears if variable somevar does not equal the string "x",
      or if somevar is not found in the context
    {% endif %}

``<`` 运算
^^^^^^^^^^^^^^

小于判断. 例如::

    {% if somevar < 100 %}
      This appears if variable somevar is less than 100.
    {% endif %}

``>`` 运算
^^^^^^^^^^^^^^

大于判断. 例如::

    {% if somevar > 0 %}
      This appears if variable somevar is greater than 0.
    {% endif %}

``<=`` 运算
^^^^^^^^^^^^^^^

小于等于判断. 例如::

    {% if somevar <= 100 %}
      This appears if variable somevar is less than 100 or equal to 100.
    {% endif %}

``>=`` 运算
^^^^^^^^^^^^^^^

大于等于判断. 例如::

    {% if somevar >= 1 %}
      This appears if variable somevar is greater than 1 or equal to 1.
    {% endif %}

``in`` 运算
^^^^^^^^^^^^^^^

包含判断. 该运算是许多Python容器都支持的运算符来判断给定值是否在容器中. 下面是一些解释 ``x in y`` 的示例::

    {% if "bc" in "abcdef" %}
      This appears since "bc" is a substring of "abcdef"
    {% endif %}

    {% if "hello" in greetings %}
      If greetings is a list or set, one element of which is the string
      "hello", this will appear.
    {% endif %}

    {% if user in users %}
      If users is a QuerySet, this will appear if user is an
      instance that belongs to the QuerySet.
    {% endif %}

``not in`` 运算
^^^^^^^^^^^^^^^^^^^

不包含判断. 这是 ``in`` 运算取反.

``is`` 运算
^^^^^^^^^^^^^^^

.. versionadded:: 1.10

对象判断. 判断两个变量是否是同一对象. 例如::

    {% if somevar is True %}
      This appears if and only if somevar is True.
    {% endif %}

    {% if somevar is None %}
      This appears if somevar is None, or if somevar is not found in the context.
    {% endif %}

``is not`` 运算
^^^^^^^^^^^^^^^^^^^

.. versionadded:: 1.10

对象判断取反. 判断两个变量是否不是同一对象. 它是 ``is`` 运算取反. 例如::

    {% if somevar is not True %}
      This appears if somevar is not True, or if somevar is not found in the
      context.
    {% endif %}

    {% if somevar is not None %}
      This appears if and only if somevar is not None.
    {% endif %}

过滤器
~~~~~~~

可以在 :ttag:`if` 表达式中使用过滤器. 例如::

    {% if messages|length >= 100 %}
       You have lots of messages today!
    {% endif %}

复合表达式
~~~~~~~~~~~~~~~~~~~

所有这些都可以组合起来变成复合表达式. 对于此类表达式, 了解表达式计算时运算符的分组方式 —— 也就是优先规则很重要. 运算符的优先级从低到高, 如下所示:

* ``or``
* ``and``
* ``not``
* ``in``
* ``==``, ``!=``, ``<``, ``>``, ``<=``, ``>=``

(这完全是按照Python来的). 所以, 下面这个 :ttag:`if` 标签::

    {% if a == b or c == d and e %}

...将被解释为:

.. code-block:: python

    (a == b) or ((c == d) and e)

如果需要不同的优先级, 则需使用嵌套的 :ttag:`if` 标签. 有时为了清楚起见, 为了那些不知道优先规则的人, 这样做更好.

比较运算符不能像在Python或数学符号中那样'串联'. 例如, 不能使用::

    {% if a > b > c %}  (WRONG)

正确方式::

    {% if a > b and b > c %}

``ifequal`` 和 ``ifnotequal``
------------------------------

``{% ifequal a b %} ... {% endifequal %}`` 是
``{% if a == b %} ... {% endif %}`` 的一种过时的方式. 同样, ``{% ifnotequal a b %} ...
{% endifnotequal %}`` 也被 ``{% if a != b %} ... {% endif %}`` 替代.
``ifequal`` 和 ``ifnotequal`` 标签将会在未来版本中弃用.

.. templatetag:: ifchanged

``ifchanged``
-------------

检查值是否在循环的上次迭代中发生变化.

``{% ifchanged %}`` 块标签被用在循环中. 它通常有两种用途.

1. 根据之前的状态检查自己渲染的内容, 只有在内容发生变化时才显示. 例如, 这将显示天数列表, 仅在月份发生变化时显示月份::

        <h1>Archive for {{ year }}</h1>

        {% for date in days %}
            {% ifchanged %}<h3>{{ date|date:"F" }}</h3>{% endifchanged %}
            <a href="{{ date|date:"M/d"|lower }}/">{{ date|date:"j" }}</a>
        {% endfor %}

2. 如果给定一个或多个变量, 检查是否有任何变量发生更改. 例如, 以下内容显示每次更改的日期, 同时显示小时(如果小时或日期已更改)::

        {% for date in days %}
            {% ifchanged date.date %} {{ date.date }} {% endifchanged %}
            {% ifchanged date.hour date.date %}
                {{ date.hour }}
            {% endifchanged %}
        {% endfor %}

``ifchanged`` 标签也可以使用一个可选的 ``{% else %}`` 子句, 如果值没有改变就会显示::

        {% for match in matches %}
            <div style="background-color:
                {% ifchanged match.ballot_id %}
                    {% cycle "red" "blue" %}
                {% else %}
                    gray
                {% endifchanged %}
            ">{{ match }}</div>
        {% endfor %}

.. templatetag:: include

``include``
-----------

加载一个模板并在当前上下文中进行渲染. 这是一种在模板中"引入"其他模板的方式.

模板名称可以是一个变量, 也可以是一个硬编码(引号)的字符串, 可以是单引号或双引号.

下面例子引入了模板 ``"foo/bar.html"`` 的内容::

    {% include "foo/bar.html" %}

字符串参数可以是相对路径, 如 :ttag:`extends` 标签中所述, 以 ``./`` 或 ``../`` 开头.

.. versionadded:: 1.10

   新增使用相对路径的功能.

这个例子引入模板内容, 其名称包含在变量 ``template_name`` 中::

    {% include template_name %}

变量也可以是任何实现了 ``render()`` 方法接口的对象, 这个对象要可以接收上下文context. 这允许你在你的上下文中引用一个编译的 ``Template``.

被引入的模板在引入它的模板的上下文中渲染. 下面这个示例生成输出 ``"Hello, John!"``:

* Context: 变量 ``person`` 设置为 ``"John"``, 变量 ``greeting`` 设置为 ``"Hello"``.

* Template::

    {% include "name_snippet.html" %}

* ``name_snippet.html``::

    {{ greeting }}, {{ person|default:"friend" }}!

使用关键字参数向模板传递额外的上下文::

    {% include "name_snippet.html" with person="Jane" greeting="Hello" %}

如果想只用提供的变量(甚至不使用变量)来渲染上下文, 请使用 ``only`` 选项. 没有其他变量可用于引入的模板::

    {% include "name_snippet.html" with greeting="Hi" only %}

如果引入的模板在渲染时发送异常(包括它不存在和有语法错误), 则会根据
:class:`模版引擎 <django.template.Engine>` 的 ``debug`` 选项会有不同的行为(如果未设置, 则此选项默认为 :setting:`DEBUG` 的值).
启用调试模式时, 将引发 :exc:`~django.template.TemplateDoesNotExist` 或 :exc:`~django.template.TemplateSyntaxError` 之类的异常.
关闭调试模式时, ``{% include %}`` 标签 会向 ``django.template`` 记录器记录一条警告, 除了在渲染所引入的模板并返回一个空字符串时发生的异常.

.. versionchanged:: 1.9

    模板日志新增上面提到的警告日志.

.. note::
    :ttag:`include` 标签被视为 "将子模版渲染并嵌入HTML中"的方法, 而不是"解析这个子模板并引入其内容, 就像它是父模板的一部分一样".
    这意味着, 引入的模板之间没有共享状态 -- 每个引入都是一个完全独立的渲染过程.

    块在被引入 *之前* 就已经被执行. 这意味着模版在被引入之前就已经从另一个block扩展并 *已经* 被执行完成渲染 - 而不是block被覆盖, 例如, 扩展模板.

.. templatetag:: load

``load``
--------

Loads a custom template tag set.

For example, the following template would load all the tags and filters
registered in ``somelibrary`` and ``otherlibrary`` located in package
``package``::

    {% load somelibrary package.otherlibrary %}

You can also selectively load individual filters or tags from a library, using
the ``from`` argument. In this example, the template tags/filters named ``foo``
and ``bar`` will be loaded from ``somelibrary``::

    {% load foo bar from somelibrary %}

See :doc:`Custom tag and filter libraries </howto/custom-template-tags>` for
more information.

.. templatetag:: lorem

``lorem``
---------

Displays random "lorem ipsum" Latin text. This is useful for providing sample
data in templates.

Usage::

    {% lorem [count] [method] [random] %}

The ``{% lorem %}`` tag can be used with zero, one, two or three arguments.
The arguments are:

===========  =============================================================
Argument     Description
===========  =============================================================
``count``    A number (or variable) containing the number of paragraphs or
             words to generate (default is 1).
``method``   Either ``w`` for words, ``p`` for HTML paragraphs or ``b``
             for plain-text paragraph blocks (default is ``b``).
``random``   The word ``random``, which if given, does not use the common
             paragraph ("Lorem ipsum dolor sit amet...") when generating
             text.
===========  =============================================================

Examples:

* ``{% lorem %}`` will output the common "lorem ipsum" paragraph.
* ``{% lorem 3 p %}`` will output the common "lorem ipsum" paragraph
  and two random paragraphs each wrapped in HTML ``<p>`` tags.
* ``{% lorem 2 w random %}`` will output two random Latin words.

.. templatetag:: now

``now``
-------

Displays the current date and/or time, using a format according to the given
string. Such string can contain format specifiers characters as described
in the :tfilter:`date` filter section.

Example::

    It is {% now "jS F Y H:i" %}

Note that you can backslash-escape a format string if you want to use the
"raw" value. In this example, both "o" and "f" are backslash-escaped, because
otherwise each is a format string that displays the year and the time,
respectively::

    It is the {% now "jS \o\f F" %}

This would display as "It is the 4th of September".

.. note::

    The format passed can also be one of the predefined ones
    :setting:`DATE_FORMAT`, :setting:`DATETIME_FORMAT`,
    :setting:`SHORT_DATE_FORMAT` or :setting:`SHORT_DATETIME_FORMAT`.
    The predefined formats may vary depending on the current locale and
    if :doc:`/topics/i18n/formatting` is enabled, e.g.::

        It is {% now "SHORT_DATETIME_FORMAT" %}

You can also use the syntax ``{% now "Y" as current_year %}`` to store the
output (as a string) inside a variable. This is useful if you want to use
``{% now %}`` inside a template tag like :ttag:`blocktrans` for example::

    {% now "Y" as current_year %}
    {% blocktrans %}Copyright {{ current_year }}{% endblocktrans %}

.. templatetag:: regroup

``regroup``
-----------

Regroups a list of alike objects by a common attribute.

This complex tag is best illustrated by way of an example: say that ``cities``
is a list of cities represented by dictionaries containing ``"name"``,
``"population"``, and ``"country"`` keys:

.. code-block:: python

    cities = [
        {'name': 'Mumbai', 'population': '19,000,000', 'country': 'India'},
        {'name': 'Calcutta', 'population': '15,000,000', 'country': 'India'},
        {'name': 'New York', 'population': '20,000,000', 'country': 'USA'},
        {'name': 'Chicago', 'population': '7,000,000', 'country': 'USA'},
        {'name': 'Tokyo', 'population': '33,000,000', 'country': 'Japan'},
    ]

...and you'd like to display a hierarchical list that is ordered by country,
like this:

* India

  * Mumbai: 19,000,000
  * Calcutta: 15,000,000

* USA

  * New York: 20,000,000
  * Chicago: 7,000,000

* Japan

  * Tokyo: 33,000,000

You can use the ``{% regroup %}`` tag to group the list of cities by country.
The following snippet of template code would accomplish this::

    {% regroup cities by country as country_list %}

    <ul>
    {% for country in country_list %}
        <li>{{ country.grouper }}
        <ul>
            {% for city in country.list %}
              <li>{{ city.name }}: {{ city.population }}</li>
            {% endfor %}
        </ul>
        </li>
    {% endfor %}
    </ul>

Let's walk through this example. ``{% regroup %}`` takes three arguments: the
list you want to regroup, the attribute to group by, and the name of the
resulting list. Here, we're regrouping the ``cities`` list by the ``country``
attribute and calling the result ``country_list``.

``{% regroup %}`` produces a list (in this case, ``country_list``) of
**group objects**. Each group object has two attributes:

* ``grouper`` -- the item that was grouped by (e.g., the string "India" or
  "Japan").
* ``list`` -- a list of all items in this group (e.g., a list of all cities
  with country='India').

Note that ``{% regroup %}`` does not order its input! Our example relies on
the fact that the ``cities`` list was ordered by ``country`` in the first place.
If the ``cities`` list did *not* order its members by ``country``, the
regrouping would naively display more than one group for a single country. For
example, say the ``cities`` list was set to this (note that the countries are not
grouped together):

.. code-block:: python

    cities = [
        {'name': 'Mumbai', 'population': '19,000,000', 'country': 'India'},
        {'name': 'New York', 'population': '20,000,000', 'country': 'USA'},
        {'name': 'Calcutta', 'population': '15,000,000', 'country': 'India'},
        {'name': 'Chicago', 'population': '7,000,000', 'country': 'USA'},
        {'name': 'Tokyo', 'population': '33,000,000', 'country': 'Japan'},
    ]

With this input for ``cities``, the example ``{% regroup %}`` template code
above would result in the following output:

* India

  * Mumbai: 19,000,000

* USA

  * New York: 20,000,000

* India

  * Calcutta: 15,000,000

* USA

  * Chicago: 7,000,000

* Japan

  * Tokyo: 33,000,000

The easiest solution to this gotcha is to make sure in your view code that the
data is ordered according to how you want to display it.

Another solution is to sort the data in the template using the
:tfilter:`dictsort` filter, if your data is in a list of dictionaries::

    {% regroup cities|dictsort:"country" by country as country_list %}

Grouping on other properties
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Any valid template lookup is a legal grouping attribute for the regroup
tag, including methods, attributes, dictionary keys and list items. For
example, if the "country" field is a foreign key to a class with
an attribute "description," you could use::

    {% regroup cities by country.description as country_list %}

Or, if ``country`` is a field with ``choices``, it will have a
:meth:`~django.db.models.Model.get_FOO_display` method available as an
attribute, allowing  you to group on the display string rather than the
``choices`` key::

    {% regroup cities by get_country_display as country_list %}

``{{ country.grouper }}`` will now display the value fields from the
``choices`` set rather than the keys.

.. templatetag:: spaceless

``spaceless``
-------------

Removes whitespace between HTML tags. This includes tab
characters and newlines.

Example usage::

    {% spaceless %}
        <p>
            <a href="foo/">Foo</a>
        </p>
    {% endspaceless %}

This example would return this HTML::

    <p><a href="foo/">Foo</a></p>

Only space between *tags* is removed -- not space between tags and text. In
this example, the space around ``Hello`` won't be stripped::

    {% spaceless %}
        <strong>
            Hello
        </strong>
    {% endspaceless %}

.. templatetag:: templatetag

``templatetag``
---------------

Outputs one of the syntax characters used to compose template tags.

Since the template system has no concept of "escaping", to display one of the
bits used in template tags, you must use the ``{% templatetag %}`` tag.

The argument tells which template bit to output:

==================  =======
Argument            Outputs
==================  =======
``openblock``       ``{%``
``closeblock``      ``%}``
``openvariable``    ``{{``
``closevariable``   ``}}``
``openbrace``       ``{``
``closebrace``      ``}``
``opencomment``     ``{#``
``closecomment``    ``#}``
==================  =======

Sample usage::

    {% templatetag openblock %} url 'entry_list' {% templatetag closeblock %}

.. templatetag:: url

``url``
-------

Returns an absolute path reference (a URL without the domain name) matching a
given view and optional parameters. Any special characters in the resulting
path will be encoded using :func:`~django.utils.encoding.iri_to_uri`.

This is a way to output links without violating the DRY principle by having to
hard-code URLs in your templates::

    {% url 'some-url-name' v1 v2 %}

The first argument is a :func:`~django.conf.urls.url` ``name``. It can be a
quoted literal or any other context variable. Additional arguments are optional
and should be space-separated values that will be used as arguments in the URL.
The example above shows passing positional arguments. Alternatively you may
use keyword syntax::

    {% url 'some-url-name' arg1=v1 arg2=v2 %}

Do not mix both positional and keyword syntax in a single call. All arguments
required by the URLconf should be present.

For example, suppose you have a view, ``app_views.client``, whose URLconf
takes a client ID (here, ``client()`` is a method inside the views file
``app_views.py``). The URLconf line might look like this:

.. code-block:: python

    ('^client/([0-9]+)/$', app_views.client, name='app-views-client')

If this app's URLconf is included into the project's URLconf under a path
such as this:

.. code-block:: python

    ('^clients/', include('project_name.app_name.urls'))

...then, in a template, you can create a link to this view like this::

    {% url 'app-views-client' client.id %}

The template tag will output the string ``/clients/client/123/``.

Note that if the URL you're reversing doesn't exist, you'll get an
:exc:`~django.urls.NoReverseMatch` exception raised, which will cause your
site to display an error page.

If you'd like to retrieve a URL without displaying it, you can use a slightly
different call::

    {% url 'some-url-name' arg arg2 as the_url %}

    <a href="{{ the_url }}">I'm linking to {{ the_url }}</a>

The scope of the variable created by the  ``as var`` syntax is the
``{% block %}`` in which the ``{% url %}`` tag appears.

This ``{% url ... as var %}`` syntax will *not* cause an error if the view is
missing. In practice you'll use this to link to views that are optional::

    {% url 'some-url-name' as the_url %}
    {% if the_url %}
      <a href="{{ the_url }}">Link to optional stuff</a>
    {% endif %}

If you'd like to retrieve a namespaced URL, specify the fully qualified name::

    {% url 'myapp:view-name' %}

This will follow the normal :ref:`namespaced URL resolution strategy
<topics-http-reversing-url-namespaces>`, including using any hints provided
by the context as to the current application.

.. warning::

    Don't forget to put quotes around the :func:`~django.conf.urls.url`
    ``name``, otherwise the value will be interpreted as a context variable!

.. templatetag:: verbatim

``verbatim``
------------

Stops the template engine from rendering the contents of this block tag.

A common use is to allow a JavaScript template layer that collides with
Django's syntax. For example::

    {% verbatim %}
        {{if dying}}Still alive.{{/if}}
    {% endverbatim %}

You can also designate a specific closing tag, allowing the use of
``{% endverbatim %}`` as part of the unrendered contents::

    {% verbatim myblock %}
        Avoid template rendering via the {% verbatim %}{% endverbatim %} block.
    {% endverbatim myblock %}

.. templatetag:: widthratio

``widthratio``
--------------

For creating bar charts and such, this tag calculates the ratio of a given
value to a maximum value, and then applies that ratio to a constant.

For example::

    <img src="bar.png" alt="Bar"
         height="10" width="{% widthratio this_value max_value max_width %}" />

If ``this_value`` is 175, ``max_value`` is 200, and ``max_width`` is 100, the
image in the above example will be 88 pixels wide
(because 175/200 = .875; .875 * 100 = 87.5 which is rounded up to 88).

In some cases you might want to capture the result of ``widthratio`` in a
variable. It can be useful, for instance, in a :ttag:`blocktrans` like this::

    {% widthratio this_value max_value max_width as width %}
    {% blocktrans %}The width is: {{ width }}{% endblocktrans %}

.. templatetag:: with

``with``
--------

Caches a complex variable under a simpler name. This is useful when accessing
an "expensive" method (e.g., one that hits the database) multiple times.

For example::

    {% with total=business.employees.count %}
        {{ total }} employee{{ total|pluralize }}
    {% endwith %}

The populated variable (in the example above, ``total``) is only available
between the ``{% with %}`` and ``{% endwith %}`` tags.

You can assign more than one context variable::

    {% with alpha=1 beta=2 %}
        ...
    {% endwith %}

.. note:: The previous more verbose format is still supported:
   ``{% with business.employees.count as total %}``

.. _ref-templates-builtins-filters:

Built-in filter reference
=========================

.. templatefilter:: add

``add``
-------

Adds the argument to the value.

For example::

    {{ value|add:"2" }}

If ``value`` is ``4``, then the output will be ``6``.

This filter will first try to coerce both values to integers. If this fails,
it'll attempt to add the values together anyway. This will work on some data
types (strings, list, etc.) and fail on others. If it fails, the result will
be an empty string.

For example, if we have::

    {{ first|add:second }}

and ``first`` is ``[1, 2, 3]`` and ``second`` is ``[4, 5, 6]``, then the
output will be ``[1, 2, 3, 4, 5, 6]``.

.. warning::

    Strings that can be coerced to integers will be **summed**, not
    concatenated, as in the first example above.

.. templatefilter:: addslashes

``addslashes``
--------------

Adds slashes before quotes. Useful for escaping strings in CSV, for example.

For example::

    {{ value|addslashes }}

If ``value`` is ``"I'm using Django"``, the output will be
``"I\'m using Django"``.

.. templatefilter:: capfirst

``capfirst``
------------

Capitalizes the first character of the value. If the first character is not
a letter, this filter has no effect.

For example::

    {{ value|capfirst }}

If ``value`` is ``"django"``, the output will be ``"Django"``.

.. templatefilter:: center

``center``
----------

Centers the value in a field of a given width.

For example::

    "{{ value|center:"15" }}"

If ``value`` is ``"Django"``, the output will be ``"     Django    "``.

.. templatefilter:: cut

``cut``
-------

Removes all values of arg from the given string.

For example::

    {{ value|cut:" " }}

If ``value`` is ``"String with spaces"``, the output will be
``"Stringwithspaces"``.

.. templatefilter:: date

``date``
--------

Formats a date according to the given format.

Uses a similar format as PHP's ``date()`` function (https://php.net/date)
with some differences.

.. note::
    These format characters are not used in Django outside of templates. They
    were designed to be compatible with PHP to ease transitioning for designers.

.. _date-and-time-formatting-specifiers:

Available format strings:

================  ========================================  =====================
Format character  Description                               Example output
================  ========================================  =====================
a                 ``'a.m.'`` or ``'p.m.'`` (Note that       ``'a.m.'``
                  this is slightly different than PHP's
                  output, because this includes periods
                  to match Associated Press style.)
A                 ``'AM'`` or ``'PM'``.                     ``'AM'``
b                 Month, textual, 3 letters, lowercase.     ``'jan'``
B                 Not implemented.
c                 ISO 8601 format. (Note: unlike others     ``2008-01-02T10:30:00.000123+02:00``,
                  formatters, such as "Z", "O" or "r",      or ``2008-01-02T10:30:00.000123`` if the datetime is naive
                  the "c" formatter will not add timezone
                  offset if value is a naive datetime
                  (see :class:`datetime.tzinfo`).
d                 Day of the month, 2 digits with           ``'01'`` to ``'31'``
                  leading zeros.
D                 Day of the week, textual, 3 letters.      ``'Fri'``
e                 Timezone name. Could be in any format,
                  or might return an empty string,          ``''``, ``'GMT'``, ``'-500'``, ``'US/Eastern'``, etc.
                  depending on the datetime.
E                 Month, locale specific alternative
                  representation usually used for long
                  date representation.                      ``'listopada'`` (for Polish locale, as opposed to ``'Listopad'``)
f                 Time, in 12-hour hours and minutes,       ``'1'``, ``'1:30'``
                  with minutes left off if they're zero.
                  Proprietary extension.
F                 Month, textual, long.                     ``'January'``
g                 Hour, 12-hour format without leading      ``'1'`` to ``'12'``
                  zeros.
G                 Hour, 24-hour format without leading      ``'0'`` to ``'23'``
                  zeros.
h                 Hour, 12-hour format.                     ``'01'`` to ``'12'``
H                 Hour, 24-hour format.                     ``'00'`` to ``'23'``
i                 Minutes.                                  ``'00'`` to ``'59'``
I                 Daylight Savings Time, whether it's       ``'1'`` or ``'0'``
                  in effect or not.
j                 Day of the month without leading          ``'1'`` to ``'31'``
                  zeros.
l                 Day of the week, textual, long.           ``'Friday'``
L                 Boolean for whether it's a leap year.     ``True`` or ``False``
m                 Month, 2 digits with leading zeros.       ``'01'`` to ``'12'``
M                 Month, textual, 3 letters.                ``'Jan'``
n                 Month without leading zeros.              ``'1'`` to ``'12'``
N                 Month abbreviation in Associated Press    ``'Jan.'``, ``'Feb.'``, ``'March'``, ``'May'``
                  style. Proprietary extension.
o                 ISO-8601 week-numbering year,             ``'1999'``
                  corresponding to the ISO-8601 week
                  number (W) which uses leap weeks. See Y
                  for the more common year format.
O                 Difference to Greenwich time in hours.    ``'+0200'``
P                 Time, in 12-hour hours, minutes and       ``'1 a.m.'``, ``'1:30 p.m.'``, ``'midnight'``, ``'noon'``, ``'12:30 p.m.'``
                  'a.m.'/'p.m.', with minutes left off
                  if they're zero and the special-case
                  strings 'midnight' and 'noon' if
                  appropriate. Proprietary extension.
r                 :rfc:`5322` formatted date.               ``'Thu, 21 Dec 2000 16:01:07 +0200'``
s                 Seconds, 2 digits with leading zeros.     ``'00'`` to ``'59'``
S                 English ordinal suffix for day of the     ``'st'``, ``'nd'``, ``'rd'`` or ``'th'``
                  month, 2 characters.
t                 Number of days in the given month.        ``28`` to ``31``
T                 Time zone of this machine.                ``'EST'``, ``'MDT'``
u                 Microseconds.                             ``000000`` to ``999999``
U                 Seconds since the Unix Epoch
                  (January 1 1970 00:00:00 UTC).
w                 Day of the week, digits without           ``'0'`` (Sunday) to ``'6'`` (Saturday)
                  leading zeros.
W                 ISO-8601 week number of year, with        ``1``, ``53``
                  weeks starting on Monday.
y                 Year, 2 digits.                           ``'99'``
Y                 Year, 4 digits.                           ``'1999'``
z                 Day of the year.                          ``0`` to ``365``
Z                 Time zone offset in seconds. The          ``-43200`` to ``43200``
                  offset for timezones west of UTC is
                  always negative, and for those east of
                  UTC is always positive.
================  ========================================  =====================

For example::

    {{ value|date:"D d M Y" }}

If ``value`` is a :py:class:`~datetime.datetime` object (e.g., the result of
``datetime.datetime.now()``), the output will be the string
``'Wed 09 Jan 2008'``.

The format passed can be one of the predefined ones :setting:`DATE_FORMAT`,
:setting:`DATETIME_FORMAT`, :setting:`SHORT_DATE_FORMAT` or
:setting:`SHORT_DATETIME_FORMAT`, or a custom format that uses the format
specifiers shown in the table above. Note that predefined formats may vary
depending on the current locale.

Assuming that :setting:`USE_L10N` is ``True`` and :setting:`LANGUAGE_CODE` is,
for example, ``"es"``, then for::

    {{ value|date:"SHORT_DATE_FORMAT" }}

the output would be the string ``"09/01/2008"`` (the ``"SHORT_DATE_FORMAT"``
format specifier for the ``es`` locale as shipped with Django is ``"d/m/Y"``).

When used without a format string, the ``DATE_FORMAT`` format specifier is
used. Assuming the same settings as the previous example::

    {{ value|date }}

outputs ``9 de Enero de 2008`` (the ``DATE_FORMAT`` format specifier for the
``es`` locale is ``r'j \d\e F \d\e Y'``.

.. versionchanged:: 1.10

    In older versions, the :setting:`DATE_FORMAT` setting (without
    localization) is always used when a format string isn't given.

You can combine ``date`` with the :tfilter:`time` filter to render a full
representation of a ``datetime`` value. E.g.::

    {{ value|date:"D d M Y" }} {{ value|time:"H:i" }}

.. templatefilter:: default

``default``
-----------

If value evaluates to ``False``, uses the given default. Otherwise, uses the
value.

For example::

    {{ value|default:"nothing" }}

If ``value`` is ``""`` (the empty string), the output will be ``nothing``.

.. templatefilter:: default_if_none

``default_if_none``
-------------------

If (and only if) value is ``None``, uses the given default. Otherwise, uses the
value.

Note that if an empty string is given, the default value will *not* be used.
Use the :tfilter:`default` filter if you want to fallback for empty strings.

For example::

    {{ value|default_if_none:"nothing" }}

If ``value`` is ``None``, the output will be the string ``"nothing"``.

.. templatefilter:: dictsort

``dictsort``
------------

Takes a list of dictionaries and returns that list sorted by the key given in
the argument.

For example::

    {{ value|dictsort:"name" }}

If ``value`` is:

.. code-block:: python

    [
        {'name': 'zed', 'age': 19},
        {'name': 'amy', 'age': 22},
        {'name': 'joe', 'age': 31},
    ]

then the output would be:

.. code-block:: python

    [
        {'name': 'amy', 'age': 22},
        {'name': 'joe', 'age': 31},
        {'name': 'zed', 'age': 19},
    ]

You can also do more complicated things like::

    {% for book in books|dictsort:"author.age" %}
        * {{ book.title }} ({{ book.author.name }})
    {% endfor %}

If ``books`` is:

.. code-block:: python

    [
        {'title': '1984', 'author': {'name': 'George', 'age': 45}},
        {'title': 'Timequake', 'author': {'name': 'Kurt', 'age': 75}},
        {'title': 'Alice', 'author': {'name': 'Lewis', 'age': 33}},
    ]

then the output would be::

    * Alice (Lewis)
    * 1984 (George)
    * Timequake (Kurt)

``dictsort`` can also order a list of lists (or any other object implementing
``__getitem__()``) by elements at specified index. For example::

    {{ value|dictsort:0 }}

If ``value`` is:

.. code-block:: python

    [
        ('a', '42'),
        ('c', 'string'),
        ('b', 'foo'),
    ]

then the output would be:

.. code-block:: python

    [
        ('a', '42'),
        ('b', 'foo'),
        ('c', 'string'),
    ]

You must pass the index as an integer rather than a string. The following
produce empty output::

    {{ values|dictsort:"0" }}

.. versionchanged:: 1.10

    The ability to order a list of lists was added.

.. templatefilter:: dictsortreversed

``dictsortreversed``
--------------------

Takes a list of dictionaries and returns that list sorted in reverse order by
the key given in the argument. This works exactly the same as the above filter,
but the returned value will be in reverse order.

.. templatefilter:: divisibleby

``divisibleby``
---------------

Returns ``True`` if the value is divisible by the argument.

For example::

    {{ value|divisibleby:"3" }}

If ``value`` is ``21``, the output would be ``True``.

.. templatefilter:: escape

``escape``
----------

Escapes a string's HTML. Specifically, it makes these replacements:

* ``<`` is converted to ``&lt;``
* ``>`` is converted to ``&gt;``
* ``'`` (single quote) is converted to ``&#39;``
* ``"`` (double quote) is converted to ``&quot;``
* ``&`` is converted to ``&amp;``

The escaping is only applied when the string is output, so it does not matter
where in a chained sequence of filters you put ``escape``: it will always be
applied as though it were the last filter. If you want escaping to be applied
immediately, use the :tfilter:`force_escape` filter.

Applying ``escape`` to a variable that would normally have auto-escaping
applied to the result will only result in one round of escaping being done. So
it is safe to use this function even in auto-escaping environments. If you want
multiple escaping passes to be applied, use the :tfilter:`force_escape` filter.

For example, you can apply ``escape`` to fields when :ttag:`autoescape` is off::

    {% autoescape off %}
        {{ title|escape }}
    {% endautoescape %}

.. deprecated:: 1.10

    The "lazy" behavior of the ``escape`` filter is deprecated. It will change
    to immediately apply :func:`~django.utils.html.conditional_escape` in
    Django 2.0.

.. templatefilter:: escapejs

``escapejs``
------------

Escapes characters for use in JavaScript strings. This does *not* make the
string safe for use in HTML, but does protect you from syntax errors when using
templates to generate JavaScript/JSON.

For example::

    {{ value|escapejs }}

If ``value`` is ``"testing\r\njavascript \'string" <b>escaping</b>"``,
the output will be ``"testing\\u000D\\u000Ajavascript \\u0027string\\u0022 \\u003Cb\\u003Eescaping\\u003C/b\\u003E"``.

.. templatefilter:: filesizeformat

``filesizeformat``
------------------

Formats the value like a 'human-readable' file size (i.e. ``'13 KB'``,
``'4.1 MB'``, ``'102 bytes'``, etc.).

For example::

    {{ value|filesizeformat }}

If ``value`` is 123456789, the output would be ``117.7 MB``.

.. admonition:: File sizes and SI units

    Strictly speaking, ``filesizeformat`` does not conform to the International
    System of Units which recommends using KiB, MiB, GiB, etc. when byte sizes
    are calculated in powers of 1024 (which is the case here). Instead, Django
    uses traditional unit names (KB, MB, GB, etc.) corresponding to names that
    are more commonly used.

.. templatefilter:: first

``first``
---------

Returns the first item in a list.

For example::

    {{ value|first }}

If ``value`` is the list ``['a', 'b', 'c']``, the output will be ``'a'``.

.. templatefilter:: floatformat

``floatformat``
---------------

When used without an argument, rounds a floating-point number to one decimal
place -- but only if there's a decimal part to be displayed. For example:

============  ===========================  ========
``value``     Template                     Output
============  ===========================  ========
``34.23234``  ``{{ value|floatformat }}``  ``34.2``
``34.00000``  ``{{ value|floatformat }}``  ``34``
``34.26000``  ``{{ value|floatformat }}``  ``34.3``
============  ===========================  ========

If used with a numeric integer argument, ``floatformat`` rounds a number to
that many decimal places. For example:

============  =============================  ==========
``value``     Template                       Output
============  =============================  ==========
``34.23234``  ``{{ value|floatformat:3 }}``  ``34.232``
``34.00000``  ``{{ value|floatformat:3 }}``  ``34.000``
``34.26000``  ``{{ value|floatformat:3 }}``  ``34.260``
============  =============================  ==========

Particularly useful is passing 0 (zero) as the argument which will round the
float to the nearest integer.

============  ================================  ==========
``value``     Template                          Output
============  ================================  ==========
``34.23234``  ``{{ value|floatformat:"0" }}``   ``34``
``34.00000``  ``{{ value|floatformat:"0" }}``   ``34``
``39.56000``  ``{{ value|floatformat:"0" }}``   ``40``
============  ================================  ==========

If the argument passed to ``floatformat`` is negative, it will round a number
to that many decimal places -- but only if there's a decimal part to be
displayed. For example:

============  ================================  ==========
``value``     Template                          Output
============  ================================  ==========
``34.23234``  ``{{ value|floatformat:"-3" }}``  ``34.232``
``34.00000``  ``{{ value|floatformat:"-3" }}``  ``34``
``34.26000``  ``{{ value|floatformat:"-3" }}``  ``34.260``
============  ================================  ==========

Using ``floatformat`` with no argument is equivalent to using ``floatformat``
with an argument of ``-1``.

.. templatefilter:: force_escape

``force_escape``
----------------

Applies HTML escaping to a string (see the :tfilter:`escape` filter for
details). This filter is applied *immediately* and returns a new, escaped
string. This is useful in the rare cases where you need multiple escaping or
want to apply other filters to the escaped results. Normally, you want to use
the :tfilter:`escape` filter.

For example, if you want to catch the ``<p>`` HTML elements created by
the :tfilter:`linebreaks` filter::

    {% autoescape off %}
        {{ body|linebreaks|force_escape }}
    {% endautoescape %}

.. templatefilter:: get_digit

``get_digit``
-------------

Given a whole number, returns the requested digit, where 1 is the right-most
digit, 2 is the second-right-most digit, etc. Returns the original value for
invalid input (if input or argument is not an integer, or if argument is less
than 1). Otherwise, output is always an integer.

For example::

    {{ value|get_digit:"2" }}

If ``value`` is ``123456789``, the output will be ``8``.

.. templatefilter:: iriencode

``iriencode``
-------------

Converts an IRI (Internationalized Resource Identifier) to a string that is
suitable for including in a URL. This is necessary if you're trying to use
strings containing non-ASCII characters in a URL.

It's safe to use this filter on a string that has already gone through the
:tfilter:`urlencode` filter.

For example::

    {{ value|iriencode }}

If ``value`` is ``"?test=1&me=2"``, the output will be ``"?test=1&amp;me=2"``.

.. templatefilter:: join

``join``
--------

Joins a list with a string, like Python's ``str.join(list)``

For example::

    {{ value|join:" // " }}

If ``value`` is the list ``['a', 'b', 'c']``, the output will be the string
``"a // b // c"``.

.. templatefilter:: last

``last``
--------

Returns the last item in a list.

For example::

    {{ value|last }}

If ``value`` is the list ``['a', 'b', 'c', 'd']``, the output will be the
string ``"d"``.

.. templatefilter:: length

``length``
----------

Returns the length of the value. This works for both strings and lists.

For example::

    {{ value|length }}

If ``value`` is ``['a', 'b', 'c', 'd']`` or ``"abcd"``, the output will be
``4``.

The filter returns ``0`` for an undefined variable.

.. templatefilter:: length_is

``length_is``
-------------

Returns ``True`` if the value's length is the argument, or ``False`` otherwise.

For example::

    {{ value|length_is:"4" }}

If ``value`` is ``['a', 'b', 'c', 'd']`` or ``"abcd"``, the output will be
``True``.

.. templatefilter:: linebreaks

``linebreaks``
--------------

Replaces line breaks in plain text with appropriate HTML; a single
newline becomes an HTML line break (``<br />``) and a new line
followed by a blank line becomes a paragraph break (``</p>``).

For example::

    {{ value|linebreaks }}

If ``value`` is ``Joel\nis a slug``, the output will be ``<p>Joel<br />is a
slug</p>``.

.. templatefilter:: linebreaksbr

``linebreaksbr``
----------------

Converts all newlines in a piece of plain text to HTML line breaks
(``<br />``).

For example::

    {{ value|linebreaksbr }}

If ``value`` is ``Joel\nis a slug``, the output will be ``Joel<br />is a
slug``.

.. templatefilter:: linenumbers

``linenumbers``
---------------

Displays text with line numbers.

For example::

    {{ value|linenumbers }}

If ``value`` is::

    one
    two
    three

the output will be::

    1. one
    2. two
    3. three

.. templatefilter:: ljust

``ljust``
---------

Left-aligns the value in a field of a given width.

**Argument:** field size

For example::

    "{{ value|ljust:"10" }}"

If ``value`` is ``Django``, the output will be ``"Django    "``.

.. templatefilter:: lower

``lower``
---------

Converts a string into all lowercase.

For example::

    {{ value|lower }}

If ``value`` is ``Totally LOVING this Album!``, the output will be
``totally loving this album!``.

.. templatefilter:: make_list

``make_list``
-------------

Returns the value turned into a list. For a string, it's a list of characters.
For an integer, the argument is cast into an unicode string before creating a
list.

For example::

    {{ value|make_list }}

If ``value`` is the string ``"Joel"``, the output would be the list
``['J', 'o', 'e', 'l']``. If ``value`` is ``123``, the output will be the
list ``['1', '2', '3']``.

.. templatefilter:: phone2numeric

``phone2numeric``
-----------------

Converts a phone number (possibly containing letters) to its numerical
equivalent.

The input doesn't have to be a valid phone number. This will happily convert
any string.

For example::

    {{ value|phone2numeric }}

If ``value`` is ``800-COLLECT``, the output will be ``800-2655328``.

.. templatefilter:: pluralize

``pluralize``
-------------

Returns a plural suffix if the value is not 1. By default, this suffix is
``'s'``.

Example::

    You have {{ num_messages }} message{{ num_messages|pluralize }}.

If ``num_messages`` is ``1``, the output will be ``You have 1 message.``
If ``num_messages`` is ``2``  the output will be ``You have 2 messages.``

For words that require a suffix other than ``'s'``, you can provide an alternate
suffix as a parameter to the filter.

Example::

    You have {{ num_walruses }} walrus{{ num_walruses|pluralize:"es" }}.

For words that don't pluralize by simple suffix, you can specify both a
singular and plural suffix, separated by a comma.

Example::

    You have {{ num_cherries }} cherr{{ num_cherries|pluralize:"y,ies" }}.

.. note:: Use :ttag:`blocktrans` to pluralize translated strings.

.. templatefilter:: pprint

``pprint``
----------

A wrapper around :func:`pprint.pprint` -- for debugging, really.

.. templatefilter:: random

``random``
----------

Returns a random item from the given list.

For example::

    {{ value|random }}

If ``value`` is the list ``['a', 'b', 'c', 'd']``, the output could be ``"b"``.

.. templatefilter:: rjust

``rjust``
---------

Right-aligns the value in a field of a given width.

**Argument:** field size

For example::

    "{{ value|rjust:"10" }}"

If ``value`` is ``Django``, the output will be ``"    Django"``.

.. templatefilter:: safe

``safe``
--------

Marks a string as not requiring further HTML escaping prior to output. When
autoescaping is off, this filter has no effect.

.. note::

    If you are chaining filters, a filter applied after ``safe`` can
    make the contents unsafe again. For example, the following code
    prints the variable as is, unescaped::

        {{ var|safe|escape }}

.. templatefilter:: safeseq

``safeseq``
-----------

Applies the :tfilter:`safe` filter to each element of a sequence. Useful in
conjunction with other filters that operate on sequences, such as
:tfilter:`join`. For example::

    {{ some_list|safeseq|join:", " }}

You couldn't use the :tfilter:`safe` filter directly in this case, as it would
first convert the variable into a string, rather than working with the
individual elements of the sequence.

.. templatefilter:: slice

``slice``
---------

Returns a slice of the list.

Uses the same syntax as Python's list slicing. See
http://www.diveintopython3.net/native-datatypes.html#slicinglists
for an introduction.

Example::

    {{ some_list|slice:":2" }}

If ``some_list`` is ``['a', 'b', 'c']``, the output will be ``['a', 'b']``.

.. templatefilter:: slugify

``slugify``
-----------

Converts to ASCII. Converts spaces to hyphens. Removes characters that aren't
alphanumerics, underscores, or hyphens. Converts to lowercase. Also strips
leading and trailing whitespace.

For example::

    {{ value|slugify }}

If ``value`` is ``"Joel is a slug"``, the output will be ``"joel-is-a-slug"``.

.. templatefilter:: stringformat

``stringformat``
----------------

Formats the variable according to the argument, a string formatting specifier.
This specifier uses the :ref:`old-string-formatting` syntax, with the exception
that the leading "%" is dropped.

For example::

    {{ value|stringformat:"E" }}

If ``value`` is ``10``, the output will be ``1.000000E+01``.

.. templatefilter:: striptags

``striptags``
-------------

Makes all possible efforts to strip all [X]HTML tags.

For example::

    {{ value|striptags }}

If ``value`` is ``"<b>Joel</b> <button>is</button> a <span>slug</span>"``, the
output will be ``"Joel is a slug"``.

.. admonition:: No safety guarantee

    Note that ``striptags`` doesn't give any guarantee about its output being
    HTML safe, particularly with non valid HTML input. So **NEVER** apply the
    ``safe`` filter to a ``striptags`` output. If you are looking for something
    more robust, you can use the ``bleach`` Python library, notably its
    `clean`_ method.

.. _clean: https://bleach.readthedocs.io/en/latest/clean.html

.. templatefilter:: time

``time``
--------

Formats a time according to the given format.

Given format can be the predefined one :setting:`TIME_FORMAT`, or a custom
format, same as the :tfilter:`date` filter. Note that the predefined format
is locale-dependent.

For example::

    {{ value|time:"H:i" }}

If ``value`` is equivalent to ``datetime.datetime.now()``, the output will be
the string ``"01:23"``.

Another example:

Assuming that :setting:`USE_L10N` is ``True`` and :setting:`LANGUAGE_CODE` is,
for example, ``"de"``, then for::

    {{ value|time:"TIME_FORMAT" }}

the output will be the string ``"01:23:00"`` (The ``"TIME_FORMAT"`` format
specifier for the ``de`` locale as shipped with Django is ``"H:i:s"``).

The ``time`` filter will only accept parameters in the format string that
relate to the time of day, not the date (for obvious reasons). If you need to
format a ``date`` value, use the :tfilter:`date` filter instead (or along
``time`` if you need to render a full :py:class:`~datetime.datetime` value).

There is one exception the above rule: When passed a ``datetime`` value with
attached timezone information (a :ref:`time-zone-aware
<naive_vs_aware_datetimes>` ``datetime`` instance) the ``time`` filter will
accept the timezone-related :ref:`format specifiers
<date-and-time-formatting-specifiers>` ``'e'``, ``'O'`` , ``'T'`` and ``'Z'``.

When used without a format string, the ``TIME_FORMAT`` format specifier is
used::

    {{ value|time }}

is the same as::

    {{ value|time:"TIME_FORMAT" }}

.. versionchanged:: 1.10

    In older versions, the :setting:`TIME_FORMAT` setting (without
    localization) is always used when a format string isn't given.

.. templatefilter:: timesince

``timesince``
-------------

Formats a date as the time since that date (e.g., "4 days, 6 hours").

Takes an optional argument that is a variable containing the date to use as
the comparison point (without the argument, the comparison point is *now*).
For example, if ``blog_date`` is a date instance representing midnight on 1
June 2006, and ``comment_date`` is a date instance for 08:00 on 1 June 2006,
then the following would return "8 hours"::

    {{ blog_date|timesince:comment_date }}

Comparing offset-naive and offset-aware datetimes will return an empty string.

Minutes is the smallest unit used, and "0 minutes" will be returned for any
date that is in the future relative to the comparison point.

.. templatefilter:: timeuntil

``timeuntil``
-------------

Similar to ``timesince``, except that it measures the time from now until the
given date or datetime. For example, if today is 1 June 2006 and
``conference_date`` is a date instance holding 29 June 2006, then
``{{ conference_date|timeuntil }}`` will return "4 weeks".

Takes an optional argument that is a variable containing the date to use as
the comparison point (instead of *now*). If ``from_date`` contains 22 June
2006, then the following will return "1 week"::

    {{ conference_date|timeuntil:from_date }}

Comparing offset-naive and offset-aware datetimes will return an empty string.

Minutes is the smallest unit used, and "0 minutes" will be returned for any
date that is in the past relative to the comparison point.

.. templatefilter:: title

``title``
---------

Converts a string into titlecase by making words start with an uppercase
character and the remaining characters lowercase. This tag makes no effort to
keep "trivial words" in lowercase.

For example::

    {{ value|title }}

If ``value`` is ``"my FIRST post"``, the output will be ``"My First Post"``.

.. templatefilter:: truncatechars

``truncatechars``
-----------------

Truncates a string if it is longer than the specified number of characters.
Truncated strings will end with a translatable ellipsis sequence ("...").

**Argument:** Number of characters to truncate to

For example::

    {{ value|truncatechars:9 }}

If ``value`` is ``"Joel is a slug"``, the output will be ``"Joel i..."``.

.. templatefilter:: truncatechars_html

``truncatechars_html``
----------------------

Similar to :tfilter:`truncatechars`, except that it is aware of HTML tags. Any
tags that are opened in the string and not closed before the truncation point
are closed immediately after the truncation.

For example::

    {{ value|truncatechars_html:9 }}

If ``value`` is ``"<p>Joel is a slug</p>"``, the output will be
``"<p>Joel i...</p>"``.

Newlines in the HTML content will be preserved.

.. templatefilter:: truncatewords

``truncatewords``
-----------------

Truncates a string after a certain number of words.

**Argument:** Number of words to truncate after

For example::

    {{ value|truncatewords:2 }}

If ``value`` is ``"Joel is a slug"``, the output will be ``"Joel is ..."``.

Newlines within the string will be removed.

.. templatefilter:: truncatewords_html

``truncatewords_html``
----------------------

Similar to :tfilter:`truncatewords`, except that it is aware of HTML tags. Any
tags that are opened in the string and not closed before the truncation point,
are closed immediately after the truncation.

This is less efficient than :tfilter:`truncatewords`, so should only be used
when it is being passed HTML text.

For example::

    {{ value|truncatewords_html:2 }}

If ``value`` is ``"<p>Joel is a slug</p>"``, the output will be
``"<p>Joel is ...</p>"``.

Newlines in the HTML content will be preserved.

.. templatefilter:: unordered_list

``unordered_list``
------------------

Recursively takes a self-nested list and returns an HTML unordered list --
WITHOUT opening and closing <ul> tags.

The list is assumed to be in the proper format. For example, if ``var``
contains ``['States', ['Kansas', ['Lawrence', 'Topeka'], 'Illinois']]``, then
``{{ var|unordered_list }}`` would return::

    <li>States
    <ul>
            <li>Kansas
            <ul>
                    <li>Lawrence</li>
                    <li>Topeka</li>
            </ul>
            </li>
            <li>Illinois</li>
    </ul>
    </li>

.. templatefilter:: upper

``upper``
---------

Converts a string into all uppercase.

For example::

    {{ value|upper }}

If ``value`` is ``"Joel is a slug"``, the output will be ``"JOEL IS A SLUG"``.

.. templatefilter:: urlencode

``urlencode``
-------------

Escapes a value for use in a URL.

For example::

    {{ value|urlencode }}

If ``value`` is ``"https://www.example.org/foo?a=b&c=d"``, the output will be
``"https%3A//www.example.org/foo%3Fa%3Db%26c%3Dd"``.

An optional argument containing the characters which should not be escaped can
be provided.

If not provided, the '/' character is assumed safe. An empty string can be
provided when *all* characters should be escaped. For example::

    {{ value|urlencode:"" }}

If ``value`` is ``"https://www.example.org/"``, the output will be
``"https%3A%2F%2Fwww.example.org%2F"``.

.. templatefilter:: urlize

``urlize``
----------

Converts URLs and email addresses in text into clickable links.

This template tag works on links prefixed with ``http://``, ``https://``, or
``www.``. For example, ``https://goo.gl/aia1t`` will get converted but
``goo.gl/aia1t`` won't.

It also supports domain-only links ending in one of the original top level
domains (``.com``, ``.edu``, ``.gov``, ``.int``, ``.mil``, ``.net``, and
``.org``). For example, ``djangoproject.com`` gets converted.

Links can have trailing punctuation (periods, commas, close-parens) and leading
punctuation (opening parens), and ``urlize`` will still do the right thing.

Links generated by ``urlize`` have a ``rel="nofollow"`` attribute added
to them.

For example::

    {{ value|urlize }}

If ``value`` is ``"Check out www.djangoproject.com"``, the output will be
``"Check out <a href="http://www.djangoproject.com"
rel="nofollow">www.djangoproject.com</a>"``.

In addition to web links, ``urlize`` also converts email addresses into
``mailto:`` links. If ``value`` is
``"Send questions to foo@example.com"``, the output will be
``"Send questions to <a href="mailto:foo@example.com">foo@example.com</a>"``.

The ``urlize`` filter also takes an optional parameter ``autoescape``. If
``autoescape`` is ``True``, the link text and URLs will be escaped using
Django's built-in :tfilter:`escape` filter. The default value for
``autoescape`` is ``True``.

.. note::

    If ``urlize`` is applied to text that already contains HTML markup,
    things won't work as expected. Apply this filter only to plain text.

.. templatefilter:: urlizetrunc

``urlizetrunc``
---------------

Converts URLs and email addresses into clickable links just like urlize_, but
truncates URLs longer than the given character limit.

**Argument:** Number of characters that link text should be truncated to,
including the ellipsis that's added if truncation is necessary.

For example::

    {{ value|urlizetrunc:15 }}

If ``value`` is ``"Check out www.djangoproject.com"``, the output would be
``'Check out <a href="http://www.djangoproject.com"
rel="nofollow">www.djangopr...</a>'``.

As with urlize_, this filter should only be applied to plain text.

.. templatefilter:: wordcount

``wordcount``
-------------

Returns the number of words.

For example::

    {{ value|wordcount }}

If ``value`` is ``"Joel is a slug"``, the output will be ``4``.

.. templatefilter:: wordwrap

``wordwrap``
------------

Wraps words at specified line length.

**Argument:** number of characters at which to wrap the text

For example::

    {{ value|wordwrap:5 }}

If ``value`` is ``Joel is a slug``, the output would be::

    Joel
    is a
    slug

.. templatefilter:: yesno

``yesno``
---------

Maps values for ``True``, ``False``, and (optionally) ``None``, to the strings
"yes", "no", "maybe", or a custom mapping passed as a comma-separated list, and
returns one of those strings according to the value:

For example::

    {{ value|yesno:"yeah,no,maybe" }}

==========  ======================  ===========================================
Value       Argument                Outputs
==========  ======================  ===========================================
``True``                            ``yes``
``True``    ``"yeah,no,maybe"``     ``yeah``
``False``   ``"yeah,no,maybe"``     ``no``
``None``    ``"yeah,no,maybe"``     ``maybe``
``None``    ``"yeah,no"``           ``no`` (converts ``None`` to ``False``
                                    if no mapping for ``None`` is given)
==========  ======================  ===========================================

Internationalization tags and filters
=====================================

Django provides template tags and filters to control each aspect of
:doc:`internationalization </topics/i18n/index>` in templates. They allow for
granular control of translations, formatting, and time zone conversions.

``i18n``
--------

This library allows specifying translatable text in templates.
To enable it, set :setting:`USE_I18N` to ``True``, then load it with
``{% load i18n %}``.

See :ref:`specifying-translation-strings-in-template-code`.

``l10n``
--------

This library provides control over the localization of values in templates.
You only need to load the library using ``{% load l10n %}``, but you'll often
set :setting:`USE_L10N` to ``True`` so that localization is active by default.

See :ref:`topic-l10n-templates`.

``tz``
------

This library provides control over time zone conversions in templates.
Like ``l10n``, you only need to load the library using ``{% load tz %}``,
but you'll usually also set :setting:`USE_TZ` to ``True`` so that conversion
to local time happens by default.

See :ref:`time-zones-in-templates`.

Other tags and filters libraries
================================

Django comes with a couple of other template-tag libraries that you have to
enable explicitly in your :setting:`INSTALLED_APPS` setting and enable in your
template with the :ttag:`{% load %}<load>` tag.

``django.contrib.humanize``
---------------------------

A set of Django template filters useful for adding a "human touch" to data. See
:doc:`/ref/contrib/humanize`.

``static``
----------

.. templatetag:: static

``static``
~~~~~~~~~~

To link to static files that are saved in :setting:`STATIC_ROOT` Django ships
with a :ttag:`static` template tag. If the :mod:`django.contrib.staticfiles`
app is installed, the tag will serve files using ``url()`` method of the
storage specified by :setting:`STATICFILES_STORAGE`. For example::

    {% load static %}
    <img src="{% static "images/hi.jpg" %}" alt="Hi!" />

It is also able to consume standard context variables, e.g. assuming a
``user_stylesheet`` variable is passed to the template::

    {% load static %}
    <link rel="stylesheet" href="{% static user_stylesheet %}" type="text/css" media="screen" />

If you'd like to retrieve a static URL without displaying it, you can use a
slightly different call::

    {% load static %}
    {% static "images/hi.jpg" as myphoto %}
    <img src="{{ myphoto }}"></img>

.. admonition:: Using Jinja2 templates?

    See :class:`~django.template.backends.jinja2.Jinja2` for information on
    using the ``static`` tag with Jinja2.

.. versionchanged:: 1.10

    In older versions, you had to use ``{% load static from staticfiles %}`` in
    your template to serve files from the storage defined in
    :setting:`STATICFILES_STORAGE`. This is no longer required.

.. templatetag:: get_static_prefix

``get_static_prefix``
~~~~~~~~~~~~~~~~~~~~~

You should prefer the :ttag:`static` template tag, but if you need more control
over exactly where and how :setting:`STATIC_URL` is injected into the template,
you can use the :ttag:`get_static_prefix` template tag::

    {% load static %}
    <img src="{% get_static_prefix %}images/hi.jpg" alt="Hi!" />

There's also a second form you can use to avoid extra processing if you need
the value multiple times::

    {% load static %}
    {% get_static_prefix as STATIC_PREFIX %}

    <img src="{{ STATIC_PREFIX }}images/hi.jpg" alt="Hi!" />
    <img src="{{ STATIC_PREFIX }}images/hi2.jpg" alt="Hello!" />

.. templatetag:: get_media_prefix

``get_media_prefix``
~~~~~~~~~~~~~~~~~~~~

Similar to the :ttag:`get_static_prefix`, ``get_media_prefix`` populates a
template variable with the media prefix :setting:`MEDIA_URL`, e.g.::

    {% load static %}
    <body data-media-url="{% get_media_prefix %}">

By storing the value in a data attribute, we ensure it's escaped appropriately
if we want to use it in a JavaScript context.
