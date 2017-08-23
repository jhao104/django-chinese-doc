===========
快速安装
===========

既然要使用Django，那肯定要先安装。这里是 :doc:`完整安装指南 </topics/install>`，
涵盖所有可能涉及的东西。本指南将引导您进行简单的，最简化的安装。

安装Python
==========

作为Python Web框架，Django需要Python支持。这里可以查看 :ref:`faq-python-version-support` 。
Python内建了一个 SQLite_ 的轻量级数据库，因此您不需要立即配置数据库。

.. _sqlite: https://sqlite.org/

可以到这里 https://www.python.org/download/ 下载最新版本的Python。

.. admonition:: Django on Jython

    如果您使用的是 `Jython`_ （基于Java的Python），则需要执行几个步骤。有关详细信息，
    请参阅 :doc:`在Jython上运行Django </howto/jython>`。

.. _jython: http://www.jython.org/

您可以通过从shell来验证python是否安装成功，在shell中键入python;你应该看到类似下面的内容::

    Python 3.4.x
    [GCC 4.x] on linux
    Type "help", "copyright", "credits" or "license" for more information.
    >>>


搭建数据库
==========

只有当你需要使用“大型”数据库例如PostgreSQL、MySQL或Oracle时，才需要这一步。
若要安装这样一个数据库，请参考 :ref:`数据库安装 <database-installation>`。

删除旧版本Django
=================

如果你是从旧版本的Django升级安装，你将需要在安装新版本之前先
:ref:`卸载旧版本的Django <removing-old-versions-of-django>`。

安装Django
===========

你可以使用下面三个简单的方式来安装 Django：

* :ref:`安装官方正式发行版本 <installing-official-release>` ，这是大多数用户最佳的选择。

* 安装 :ref:`操作系统提供的Django <installing-distribution-package>` 版本。

* :ref:`安装最新的开发版本 <installing-development-version>`。此选项适用于想要最新功能且不怕运行全新代码的爱好者。
  您可能在开发版本中遇到新的错误，报告它们有助于开发Django。
  此外，第三方软件包的版本与最新的稳定版本不太可能兼容开发版本。

.. admonition:: 请你一定要参考与你Django版本一致的文档！

    如果您使用前两个步骤中的任何一个，请密切关注在开发版本中标记为 *新版本的文档* 。
    该短语标记的特性只能在Django的开发版本中使用，
    而且它们很可能无法在正式版本中使用。进入翻译页面


验证
=====

若要验证Django是否安装成功，可以在shell中输入python。然后在Python提示符下，尝试导入Django：

.. parsed-literal::

    >>> import django
    >>> print(django.get_version())
    |version|

完成
==========

安装完成 ——现在你可以继续学习 :doc:`教程 </intro/tutorial01>`

