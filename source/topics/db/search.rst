======
搜索
======

根据用户输入检索数据库中数据是Web应用的常见任务.
简单的情况下是按类别过滤对象列表,
复杂的情况有加权, 分类, 高亮显示, 多语言搜索等.
本文档介绍了一些可能的用例和可以使用的工具.

本文将引用 :doc:`/topics/db/queries` 的模型用例.

用例
=========

标准文本查询
------------------------

文本字段可以通过匹配计算进行筛选. 例如, 像这样查找author::

    >>> Author.objects.filter(name__contains='Terry')
    [<Author: Terry Gilliam>, <Author: Terry Jones>]

这个一个非常脆弱的解决方案, 因为它要求用户必须知道author name字段的子串.
有一个稍微好一点的方法是不区分大小写的匹配(:lookup:`icontains`).

更高级的数据库比较函数
-----------------------------------------------

如果你使用的是 PostgreSQL, Django内置了 :doc:`数据库特殊筛选工具 </ref/contrib/postgres/search>` 帮助你使用更复杂的查询条件.
其他数据库可以通过插件或者用户自定义函数来实现, Django此时还没有内置对它们的支持.
下面将使用PostgreSQL中的一些示例来演示其具有的高阶功能.

.. admonition:: 其他数据库

    所有由 :mod:`django.contrib.postgres` 提供的筛选工具都是基于
    :doc:`定义查询
    </ref/models/lookups>` 和 :doc:`数据库函数
    </ref/models/database-functions>` 实现.
    你应该根据你的数据库构建类似的API. 若有某个东西不能以这种方式实现, 请提交工单.

在上面例子中, 使用大小写不敏感的查询更实用. 在处理非英文name时, 可以使用 :lookup:`无重音比较 <unaccent>` 来优化::

    >>> Author.objects.filter(name__unaccent__icontains='Helen')
    [<Author: Helen Mirren>, <Author: Helena Bonham Carter>, <Actor: Hélène Joy>]

这又引出了另一个问题, 通过名字的不同拼写进行比较.
但是这种比较是不对称的, 比如搜索 ``Helen`` 可以得到 ``Helena`` 和 ``Hélène``, 但是反过来却不行.
另一个选择是使用 :lookup:`trigram_similar` 进行比较.

例如::

    >>> Author.objects.filter(name__unaccent__lower__trigram_similar='Hélène')
    [<Author: Helen Mirren>, <Actor: Hélène Joy>]

现在又出现了另一个问题 - “Helena Bonham Carter”太长了所以没有搜索出来.
Trigram搜索考虑三种字母的所有组合, 并比较搜索和源字符串中出现的数量.
对于较长的name, 就有更多的组合出现在源字符串中, 所以其不再被认为是一种近似匹配.

比较函数需要根据你的数据集来选择, 例如根据使用的语言和搜索文本的类型.
上面所有例子都是关于短字符串的, 这使用户输入的内容与源数据可以有较大的关联.

文档搜索
---------------------

对于大文本检索来说普通的数据库操作已经捉襟见肘.
虽然上面的方法也可以对字符串使用, 但是全文检索作用对象是单个词, 根据所使用的系统还可以采用下面的方法:

- 忽略"停用词" 比如 "a", "the", "and".
- 词干化, 这样 "pony" 和 "ponies" 会被认为是一样的.
- 设置单词权重, 例如根据在文本中出现的频率, 或者其所属字段(比如标题或关键字)的重要性.

可以使用的搜索工具有很多, 常见的有 Elastic_ 和 Solr_.
它们都是基于全文搜索的解决方案.
如果要将它们与Django模型的数据一起使用, 需要在抽象层将数据转换为文本文档, 包括对数据库id的反向引用,
当使用该引擎搜索返回某个文档时, 可以在数据库中查找到它.
已经有很多第三方库实现了这一功能.

.. _Elastic: https://www.elastic.co/
.. _Solr: http://lucene.apache.org/solr/

PostgreSQL支持
~~~~~~~~~~~~~~~~~~

PostgreSQL内置了其自己的全文搜索实现.
虽然不如其他专业的搜索引擎强大, 但它的优点是内置在数据库中, 可以很方便的与其它关联查询条件进行联合查询, 如按分类查询.

The :mod:`django.contrib.postgres` 模块提供了一些辅助函数来执行这些查询.
例如, 查询包含了 "cheese" 的entry::

    >>> Entry.objects.filter(body_text__search='cheese')
    [<Entry: Cheese on Toast recipes>, <Entry: Pizza recipes>]

还可以对字段和关联模型组合筛选::

    >>> Entry.objects.annotate(
    ...     search=SearchVector('blog__tagline', 'body_text'),
    ... ).filter(search='cheese')
    [
        <Entry: Cheese on Toast recipes>,
        <Entry: Pizza Recipes>,
        <Entry: Dairy farming in Argentina>,
    ]

完整描述请参见 ``contrib.postgres`` :doc:`/ref/contrib/postgres/search`.
