==============
如何安装Django
==============

这篇文档将会让你的Django运行起来。

安装 Python
==============

Django作为Python的一个Web框架，当然需要Python支持。这里有详细的版本说明
:ref:`faq-python-version-support`。

到这里下载最新的Python https://www.python.org/download/ 或者到你的操作系统的软件包管理器中获取。

.. admonition:: Django on Jython

    如果你使用的是 Jython_ (Java平台的Python实现), 您需要执行一些额外的步骤. 参考 :doc:`/howto/jython`。

.. _jython: http://jython.org/

.. admonition:: Python on Windows

    如果你是在Windows平台上使用Django, 这个
    :doc:`/howto/windows` 会对你很有帮助。

安装 Apache 和 ``mod_wsgi``
============================

如果你只是想尝试下Django，你可以直接跳到下一节；Django包括一个可以用于测试的轻量级web服务器。
因此，在您准备好部署Django之前，您不需要设置Apache。

如果您想在生产环境上使用Django, 请使用 `Apache`_ 和
`mod_wsgi`_。mod_wsgi提供两种模式：嵌入式模式和守护进程模式。

在嵌入模式下，mod_wsgi类似于mod_perl - 它在Apache中嵌入Python，并在服务器启动时将Python代码加载到内存中。
代码在Apache进程的整个生命周期内保留在内存中，这使其比其他服务器安排有显着的性能提升。

在守护进程模式下，mod_wsgi生成一个处理请求的独立守护进程。守护进程可以作为不同于Web服务器的用户运行，
从而可以提高安全性，并且可以重新启动守护进程，而无需重新启动整个Apache Web服务器，从而可以更加无缝地刷新代码库。

请查阅mod_wsgi文档以确定适合您的设置的模式。

有关如何在安装mod_wsgi之后配置mod_wsgi的信息，请参阅 :doc:`How to use Django with mod_wsgi </howto/deployment/wsgi/modwsgi>`

如果你出于某种原因不能使用mod_wsgi，不要害怕：Django支持许多其他部署选项。
其中一个是 :doc:`uWSGI </howto/deployment/wsgi/uwsgi>`；它同样契合 `nginx`_ 。
此外，Django遵循WSGI规范（ :pep:`3333` ），这允许它在各种服务器平台上运行。

.. _Apache: https://httpd.apache.org/
.. _nginx: http://nginx.org/
.. _mod_wsgi: http://www.modwsgi.org/

启动数据库
==========

如果您计划使用Django的数据库API功能，您需要确保数据库服务器已经运行。

Django支持许多不同的数据库服务器，官方正式支持 PostgreSQL_, MySQL_, Oracle_ 和 SQLite_.

如果您正在开发一个简单的项目或者您不打算在生产环境中部署，SQLite通常是最简单的选项，
因为它不需要运行单独的服务器。但是，SQLite与其他数据库有很多不同之处，因此如果您正在处理大量的工作，
建议您使用与生产中相同的数据库进行开发。

除了官方支持的数据库之外，还有由第三方提供的 :ref:`backends provided by 3rd parties <third-party-notes>`
支持您在Django中使用其他数据库。

除了数据库，您还需要确认您已经安装了数据库的操作库。

* 如果您使用 PostgreSQL, 您需要安装 `psycopg2`_ 模块。详细信息请参考
  :ref:`PostgreSQL notes <postgresql-notes>` 。

* 如果您使用的是 MySQL, 您需要安装 :ref:`DB API driver
  <mysql-db-api-drivers>` 比如 ``mysqlclient``。 详细信息参考 :ref:`notes for the MySQL
  backend <mysql-notes>` 。

* 如果您使用 SQLite 请阅读 :ref:`SQLite backend notes
  <sqlite-notes>`。

* 如果您使用 Oracle, 则需要一份 cx_Oracle_ 的副本 , 但请阅读 :ref:`notes for the Oracle backend <oracle-notes>`
  了解有关Oracle和 ``cx_Oracle`` 支持版本的详细信息。

* 如果您使用非官方的第三方数据库，请参阅其提供的文档。

如果您要使用Django的 ``manage.py migrate`` 命令为模型自动创建数据库表（在首次安装Django并创建项目之后） ，
您需要确保Django具有在您使用的数据库中创建和更改表的权限；如果您打算手动创建表格，
只需授予Django ``SELECT``，``INSERT``，``UPDATE`` 和 ``DELETE`` 权限即可。使用这些权限创建数据库用户后，
您将在项目的设置文件中指定详细信息，有关详细信息，请参阅 :setting:`DATABASES` 。

如果您使用Django的 :doc:`testing framework</topics/testing/index>` 测试数据库查询，
Django将需要创建测试数据库的权限。


.. _PostgreSQL: https://www.postgresql.org/
.. _MySQL: https://www.mysql.com/
.. _psycopg2: http://initd.org/psycopg/
.. _SQLite: https://www.sqlite.org/
.. _cx_Oracle: http://cx-oracle.sourceforge.net/
.. _Oracle: http://www.oracle.com/

.. _removing-old-versions-of-django:

卸载旧版Django
===============

如果要从以前的版本升级Django的安装，则需要在安装新版本之前卸载旧的Django版本。

如果以前使用 pip_ 或 ``easy_install`` 安装Django，则再次使用 pip_ 或 ``easy_install`` 安装会自动处理旧版本，
所以你不需要自己做。


如果您以前使用 ``python setup.py`` 安装安装的Django，
卸载操作就是从您的 ``Python site-packages`` 删除django目录 。要找到需要删除的目录，可以在shell提示符下运行以下命令（不是交互式Python提示符）：

.. code-block:: console

    $ python -c "import django; print(django.__path__)"

.. _install-django-code:

安装Django
===========

安装说明稍有不同，具体取决于是安装那种发行版的软件包，下载最新的正式版本还是获取最新的开发版本。

无论你选择哪种方式，这都很容易。

.. _installing-official-release:

``pip`` 安装官方正式版
----------------------

这是安装Django的推荐方法。

1. 安装 pip_ 。最简单的是使用 `standalone pip installer`_ 。如果您的发布版本已安装 ``pip`` ，您可能需要更新它,
   （如果它已过时，您会知道，因为安装不会正常工作。）

2. 查看 virtualenv_ 和 virtualenvwrapper_ 。这些工具提供了独立的Python环境，这比在系统范围内安装包更实用。
   它们还允许安装无管理员权限的软件包。这取决于你决定是否要学习和使用它们。:doc:`contributing tutorial
   </intro/contributing>` 讲述了如何在Python3上安装它们。


3. 创建并激活虚拟环境后，在shell提示符处输入命令 ``pip install Django``。

.. _pip: https://pip.pypa.io/
.. _virtualenv: https://virtualenv.pypa.io/
.. _virtualenvwrapper: https://virtualenvwrapper.readthedocs.io/en/latest/
.. _standalone pip installer: https://pip.pypa.io/en/latest/installing/#installing-with-get-pip-py

.. _installing-distribution-package:

安装特定版本
-------------

在 :doc:`发布版本说明 </misc/distributions>` 中，看看你的平台/发行版是否提供官方的Django包/安装程序。
分配提供的包通常允许自动安装依赖项和升级;但是，这些包很少包含最新的Django版本。

.. _installing-development-version:

安装开发版
------------

.. admonition:: Tracking Django development

    如果您决定使用Django的最新开发版本，则需要密切注意 `the development timeline`_ ，
    并且您需要密切关注 :ref:`release notes for the
    upcoming release <development_release_notes>` 。这将帮助您保持任何您可能想要使用的新功能，
    以及更新您的Django的副本，您需要对您的代码进行任何更改。（对于稳定版本，任何必要的更改都记录在版本说明中。）

.. _the development timeline: https://code.djangoproject.com/timeline

如果您希望偶尔更新Django代码，并提供最新的错误修正和改进，请按照以下说明操作：

1. 请确保您已安装 Git_ ，并且可以从shell运行其命令。（在shell提示符下输入 ``git help`` 以测试此操作）。

2. 检查Django的开发主分支:

   .. code-block:: console

        $ git clone git://github.com/django/django.git

   这将在你当前目录下创建一个 ``django`` 目录。

3. 确保Python解释器可以加载Django的代码。最方便的方法是使用 virtualenv_ 、
   virtualenvwrapper_ 和 pip_ 。:doc:`contributing tutorial </intro/contributing>` 将介绍如何在
   python3上创建一个virtualenv。

4. 在设置和激活virtualenv之后，运行以下命令:

   .. code-block:: console

        $ pip install -e django/

   这将使Django的代码可以导入，并且还会使 ``django-admin`` 实用程序命令可用。换句话说，你已经准备好了！

当您想要更新Django源代码的副本时，只需在Django目录中运行命令 ``git pull``。当您这样做时，Git将自动下载所有更改。

.. _Git: http://git-scm.com/
