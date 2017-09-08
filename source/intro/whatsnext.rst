=============
接下来看什么
=============

看来你已经阅读了 :doc:`介绍文档 </intro/index>` ，而且决定继续使用Django。
前面我们只是概要性的介绍(实际上，即使你读了所有的介绍，也只看了整个文档的5%)。

所以下一步是什么呢？

我们是喜欢通过实践来学习。基于这一点，你应该开始动手你自己的项目。当你需要新的技能的时候，再回来查文档。

我们花了很多精力来让Django文档实用、易读、尽可能完备。
下面文档更多的是关于如何使用文档，以便于你可以最大化的利用它。

(是的，这篇文档是关于如何使用文档的。放心我们不会再为这篇文档的文档再写一篇文档了)

搜索文档
=========

Django有 *大量* 的文档--大概450,000个单词--所以学会如何查找文档是很重要的。 一些查看文档的好地方是 :ref:`search` 和
:ref:`genindex` 。


或者你可以到处逛逛!

文档是如何组织的
=================

Django 的主要文档可以分解为几个用于满足不同需求的部分：

* :doc:`入门教程 </intro/index>` 是为刚接触Django或Web开发的人所设计的。
  它并不包含深度的内容，它像是培养你如何使用Django的一种“感觉”。

* :doc:`主题指南 </topics/index>` ,通过另一种方式，在Django的每一块做了深入讲解。
  主题包括 :doc:`模型系统 </topics/db/index>`， :doc:`模板索引
  </topics/templates>` ， :doc:`表单框架 </topics/forms/index>` 等等。

  这里可能是你最需要花时间的地方；如果你动手完成了这些指导文档的内容，那么你应该对Django非常熟悉了。

* Web开发通常范围广，但是不深--问题会涉及很多领域。
  我们写了一系列 :doc:`使用指南 </howto/index>` 来回答常见的 “我该如何..?” 这类的问题。
  这里你会发现关于 :doc:`如何用Django生成PDF文档 </howto/outputting-pdf>` ，
  如何写 :doc:`通用模板标签 </howto/custom-template-tags>` 等等。

  对于细节性的问题可以在  :doc:`FAQ
  </faq/index>` 中找到。

* 主题指南和使用指南没有完全覆盖到Django中得每个类、函数、方法---如果那样的话会太多，不利于学习。
  实际上，每个类、函数、方法还有模块的细节在 :doc:`参考指南 </ref/index>` 中。
  那里才是当你需要查找函数细节或是其他什么细节的地方。

* 如果你对部署项目到公网感兴趣，我们有 :doc:`这些文档</howto/deployment/index>` 是关于部署的，
  比如部 :doc:`部署检查清单</howto/deployment/checklist>` 是你需要关心的。

* 最后，有一些"特殊"的文档通常与大多数开发者无关。包括 :doc:`版本记录 </releases/index>` 和
  :doc:`内部文档 </internals/index>` 是写给那些想贡献代码到Django的人，和一些 :doc:`不好分类杂散 </misc/index>` 的文档。

文档如何更新
=============

像Django代码一样通常每天都在开发和改进，我们的文档是持续改进的。我们改进文档的原因:

* 内容修改，比如语法、排版错误。

* 增加内容还有例子到已有需要扩充的章节。

* 没有列出来的Django特性文档。(这些特性列表是不完整的，但是功能确实存在。)

* 当新特性增加的时候，增加到文档，或者 Django API行为改变的时候。

Django 文档和代码一样是有版本控制的。它在我们 Git 仓库的 `docs`_ 目录下。每篇文章在仓库中是一个独立的文本文件。

.. _docs: https://github.com/django/django/tree/master/docs

如何获取它
===========

你可以通过几种不同方式阅读Django文档。以下用优先顺序排列:

在线阅读
----------

最新版本的Django文档来源于右边网址 https://docs.djangoproject.com/en/dev/ 。
这些HTML页面是由源控制的文本本件自动产生的。这意味着他们反映了Django“最新和最好”的方面——包括最新的更正和新添加的内容，
以及对于可能仅针对Django最新版本的用户开放的新特性的讨论。（见下文“版本之间的差异”）

我们鼓励您在 `ticket system`_  中提交更改、更正或者建议以促进文档的改善。
Django 的开发者会主动查看工单系统，并且使用你的反馈意见来改善文档。

注意，不管怎样，工单应该非常明确的是和文档相关的，而不是问一些技术支持的问题。如果你需要特别的 Django 帮助，
试试 Django 用户组邮件列表_ 或者 `#django
IRC channel`_ 频道。


.. _用户组邮件列表: https://groups.google.com/d/forum/django-users
.. _ticket system: https://code.djangoproject.com/
.. _#django IRC channel: irc://irc.freenode.net/django

纯文本
-------

离线阅读，或者移动阅读，你可以阅读 Django 纯文本文档。

如果你正在使用 Django 官方发行版，注意代码压缩包(tarball)包括一个 ``docs/`` 目录，包含了对应发行版的文档。

如果你在使用开发版的 Django(又称为"trunk")，注意 ``docs/`` 目录包含了所有的文档。
你可以通过 git checkout 来获取最新更新。

一个稍微有点技术含量的查看文档的方法是通过 Unix 系统的 ``grep`` 命令来查找关键字搜索文档。
例如，这将会展示 Django 文档中提到"max_length"的地方。

.. code-block:: console

    $ grep -r max_length /path/to/django/docs/

下载html到本地
----------------

你可以通过以下简单的方法获取 HTML 格式的文档：

* Django 的文档用了一个叫做 Sphinx__ 的文档系统来从纯文本转换到 HTML。你需要安装 Sphinx，
  通过 Sphinx 网站下载安装包，或者通过 ``pip`` 方式安装。

  .. code-block:: console

        $ pip install Sphinx

* 然后使用文档目录中的 ``Makefile`` 来转换纯文本到 HTML:

  .. code-block:: console

        $ cd path/to/django/docs
        $ make html

  进行此操作，你需要安装 `GNU Make`__ 。

  如果你在 Windows 系统，你可以选择使用目录中的批处理文件：

  .. code-block:: bat

        cd path\to\django\docs
        make.bat html

* 生成的 HTML 文件将会放在 ``docs/_build/html``。

__ http://sphinx-doc.org/
__ https://www.gnu.org/software/make/

.. _differences-between-doc-versions:

版本差异
=========

像之前提到的，我们 Git 仓库中的文本文档包含很多"最新"修改的文档 。
这些修改通常包含 Django 开发版增加的一些特性。因此，我们的策略是保留各种版本的开发文档。

我们遵从的策略:

* 在Git仓库中djangoproject.com的最新文档是HTML版本。
  这些文档对应Django官方的最新版本，加上我们在since新版本中添加或者更改的新特性。

* 当Django的开发版本中添加新特性的时候，我们尽力在相同的git提交动作中同步更新文档。

* 为了区分文档中要素的更改/添加，我们使用短语“新版本X.Y”，在下一个发行版本中为X.Y

* 文档修复和改进可以由提交者自行决定，返回到最后一个发布分支，
  但是，一旦Django的版本 :ref:`no longer supported<backwards-compatibility-policy>` ，
  该版本的文档将不再获得任何进一步的更新。

* 主文档_ 网页包含指向所有先前版本的文档的链接。确保您使用的是与您使用的Django版本相对应的文档版本！

.. _主文档: https://docs.djangoproject.com/en/dev/
