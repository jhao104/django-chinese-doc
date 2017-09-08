=========================
高级教程:开发可重用的应用
=========================

本教程接着 :doc:`Tutorial 7 </intro/tutorial07>` 部分。
我们将把网页投票转换成一个独立的Python包，这样你可以在其它项目中重用或者分享给其它人

可重用性很重要
==============

设计、开发、测试和维护一个web应用程序需要大量的工作。
许多Python和Django项目都有共同的问题。如果我们能把这些重复的工作储存起来，不是很好吗?

重用在Python 中是一种常见的方式。
这里有许多的包 `Python Package Index (PyPI) <https://pypi.python.org/pypi>`_ ，可以在自己的Python程序中使用。
检查 `Django Packages
<https://www.djangopackages.com>`_ 是否有适合你项目的可重用应用，你可以将他们结合到你的项目。
Django本身就是一个Python包。 这意味着你可以将已经存在的Python包和Django应用融合到你自己的网页项目。
这样你只需要开发你项目特有的部分。

假设你正启动一个新的项目，需要一个像之前教程里做的那样的投票应用程序。
如果让应用可重用呢? 幸运的是，你来的地儿了。在 :doc:`Tutorial 3 </intro/tutorial03>` 中,
我们可以使用 ``include`` 将投票应用从项目级别的URLconf中解耦。
在本教程中，我们将更进一步，让你的应用在新的项目中容易地使用并随时可以发布给其它人安装和使用。

.. admonition:: Package? App?

    Python package 按照简单重用的方式，将具有相关性的Python代码归为一组。
    一个包包含一个或多个Python文件（也叫做“模块”）。

    包可以通过 ``import foo.bar`` 或 ``from foo import bar`` 导入。
    如果一个目录（例如 ``polls`` ）想要形成一个包，它必须包含一个特殊的文件 ``__init__.py`` ，这个文件可以为空。

    一个Django *应用* 只是一个Python包，它专用于Django项目。应用可以使用常见的Django约定，例如具有
    ``models`` 、 ``tests`` 、 ``urls`` 和 ``views`` 子模块。

    后面我们使用 *打包* 这个词来描述将一个Python包变得让其他人易于安装的过程。这可能有点让人觉得困惑。

项目和可重用应用
=================

经过前面的教程之后，我们的项目应该看上去像这样::

    mysite/
        manage.py
        mysite/
            __init__.py
            settings.py
            urls.py
            wsgi.py
        polls/
            __init__.py
            admin.py
            migrations/
                __init__.py
                0001_initial.py
            models.py
            static/
                polls/
                    images/
                        background.gif
                    style.css
            templates/
                polls/
                    detail.html
                    index.html
                    results.html
            tests.py
            urls.py
            views.py
        templates/
            admin/
                base_site.html

你在 :doc:`教程 7 </intro/tutorial07>` 中创建了 ``mysite/templates`` ,在 :doc:`Tutorial 3 </intro/tutorial03>`
创建了 ``polls/templates`` 。
现在你可能会明白为什么我们为项目和应用选择单独的模板目录：属于投票应用的部分全部在 ``polls`` 中。
它使得该应用“自包含(self-contained)”且更加容易丢到一个新的项目中。

现在可以拷贝 ``polls`` 目录到一个新的Django项目， 尽管它还没有准备好到可以立即发布，但可以立即重用。
所以，我们需要打包这个应用来让它对其他人易于安装。

.. _installing-reusable-apps-prerequisites:

安装的前提条件
==============

Python打包的方式目前有点混乱，因为有多种工具都可以进行。 在本教程中将使用 setuptools_ 打包。
它是比较推荐的打包工具 (已经和 ``distribute`` 分支合并)。我们可以使用 `pip`_ 来安装或者卸载它。
所以你现在需要这两个包。 如果你有点不明白, 可以参考 :ref:`使用pip安装Django <installing-official-release>`。 安装 ``setuptools`` 也是同样的方式。

.. _setuptools: https://pypi.python.org/pypi/setuptools
.. _pip: https://pypi.python.org/pypi/pip

打包应用
=========

Python *打包* 会将你的应用预处理成一种特殊的格式，
这样安装和使用就会变得简单。Django自己就是是以非常相似的方式打包起来的。
对于一个像 ``polls`` 这样的小应用，这个过程不是太难。

1. 首先，在Django项目外为 ``polls`` 创建一个叫 ``django-polls`` 的父目录。

   .. admonition::  为应用取名

       为你的包选择一个名字时，检查一下PyPI中的资源以避免与已经存在的包有名字冲突。
       创建一个要发布的包，在你的模块名字前面加上 ``django-`` 很有用。
       这有助于其他正在查找Django应用的人区分你的应用是专门用于Django的。

       应用的标签（应用的包的点分路径的最后部分）在 :setting:`INSTALLED_APPS` 中必须唯一。
       避免使用与Django的 :doc:`contrib packages
       </ref/contrib/index>` 中任何一个使用相同的标签，例如 ``auth`` 、 ``admin`` 和 ``messages`` 。

2. 将 ``polls`` 目录移动到 ``django-polls`` 目录。

3. 创建名为 ``django-polls/README.rst`` 的文件，包含以下内容:

   .. snippet::
       :filename: django-polls/README.rst

       =====
       Polls
       =====

       Polls is a simple Django app to conduct Web-based polls. For each
       question, visitors can choose between a fixed number of answers.

       Detailed documentation is in the "docs" directory.

       Quick start
       -----------

       1. Add "polls" to your INSTALLED_APPS setting like this::

           INSTALLED_APPS = [
               ...
               'polls',
           ]

       2. Include the polls URLconf in your project urls.py like this::

           url(r'^polls/', include('polls.urls')),

       3. Run `python manage.py migrate` to create the polls models.

       4. Start the development server and visit http://127.0.0.1:8000/admin/
          to create a poll (you'll need the Admin app enabled).

       5. Visit http://127.0.0.1:8000/polls/ to participate in the poll.

4. 新建一个 ``django-polls/LICENSE`` 文件。 如何选择License(开源协议)不是本教程的范围，但值得一说的是,
   公开发布的代码如果没有License是毫无用处的。Django和许多与Django兼容的应用都是使用 BSD 协议发布;
   你可以自由选择License。请注意，您的License选择将影响谁能够使以及如何使用您的代码。

5. 接下来，新建一个 ``setup.py`` 文件，提供如何构建和安装该应用的详细信息。
   该文件完整的解释超出本教程的范围， `setuptools 文档
   <https://setuptools.readthedocs.io/en/latest/>`_ 有非常详细的解析。
   新建的 ``django-polls/setup.py`` 文件应包含如下内容:

   .. snippet::
       :filename: django-polls/setup.py

       import os
       from setuptools import find_packages, setup

       with open(os.path.join(os.path.dirname(__file__), 'README.rst')) as readme:
           README = readme.read()

       # allow setup.py to be run from any path
       os.chdir(os.path.normpath(os.path.join(os.path.abspath(__file__), os.pardir)))

       setup(
           name='django-polls',
           version='0.1',
           packages=find_packages(),
           include_package_data=True,
           license='BSD License',  # example license
           description='A simple Django app to conduct Web-based polls.',
           long_description=README,
           url='https://www.example.com/',
           author='Your Name',
           author_email='yourname@example.com',
           classifiers=[
               'Environment :: Web Environment',
               'Framework :: Django',
               'Framework :: Django :: X.Y',  # replace "X.Y" as appropriate
               'Intended Audience :: Developers',
               'License :: OSI Approved :: BSD License',  # example license
               'Operating System :: OS Independent',
               'Programming Language :: Python',
               # Replace these appropriately if you are stuck on Python 2.
               'Programming Language :: Python :: 3',
               'Programming Language :: Python :: 3.4',
               'Programming Language :: Python :: 3.5',
               'Topic :: Internet :: WWW/HTTP',
               'Topic :: Internet :: WWW/HTTP :: Dynamic Content',
           ],
       )

6. 默认只有Python模块和包才会包含进包中。如果需要包含额外的文件，需要创建一个 ``MANIFEST.in`` 文件。
   上一步提到的setuptools文档对这个文件有更详细的说明，如果要包含模板、 ``README.rst`` 和 ``LICENSE``
   文件, 就要创建一个 ``django-polls/MANIFEST.in`` 文件包含如下内容:

   .. snippet::
       :filename: django-polls/MANIFEST.in

       include LICENSE
       include README.rst
       recursive-include polls/static *
       recursive-include polls/templates *

7. 还有个推荐的可选项，将详细的文档包含进你的应用中，创建一个空的目录 ``django-polls/docs`` 用于将来存放文档。
   向 ``django-polls/MANIFEST.in`` 在添加一行::

    recursive-include docs *

   注意 ``docs`` 不会包含进你的包里面，除非你在目录中添加了文件。 需要Django应用还通过 `readthedocs.org <https://readthedocs.org>`_ 提供在线文件。

8. 尝试使用 ``python setup.py sdist`` 打包(在
   ``django-polls`` 里面运行)。 这会创建一个 ``dist`` 目录，并构建一个新的包 ``django-polls-0.1.tar.gz`` 。

更多关于打包的信息，参见 `Tutorial on Packaging and
Distributing Projects <https://packaging.python.org/en/latest/distributing.html>`_.

使用你自己的包
==============

因为，我们将 ``polls`` 目录移出项目目录了，所以项目也不能正常运行。现在通过安装我们的新的 ``django-polls`` 包来修复它。

.. admonition:: 安装成某个用户的库

   以下的步骤将安装 ``django-polls`` 成某个用户的库。用户级别的安装比系统级别的安装有许多优点，
   例如将包运行在普通用户级别上即不会影响系统服务也不会影响其他用户。

   注意根据用户的安装仍然可以影响以该用户身份运行的系统工具，所以 ``virtualenv`` 是更健壮的解决办法（见下文）。

1. 使用pip安装 (你已经 :ref:`安装
   <installing-reusable-apps-prerequisites>` 好了？)::

    pip install --user django-polls/dist/django-polls-0.1.tar.gz

2. 幸运的话，你的Django 项目现在应该又能正常工作了。

3. 使用pip卸载这个包::

    pip uninstall django-polls

.. _pip: https://pypi.python.org/pypi/pip

发布你的应用
=============

既然我们已经打包并测试过 ``django-polls`` ，是时候与其他人分享它了！如果它不仅仅是个例子，你现在可以：

* 用邮件分享给朋友。

* 上传到你的网站。

* 上传到到公共仓库, 比如 `Python Package Index (PyPI) <https://pypi.python.org/pypi>`_ 。 `packaging.python.org <https://packaging.python.org>`_ 上有 `很好的教程 <https://packaging.python.org/en/latest/distributing.html#uploading-your-project-to-pypi>`_
  讲述如何发布。

使用 virtualenv 安装
==========================================

前面，我们将 ``poll`` 安装成一个用户的库。它有一些缺点：:

* 修改这个用户的库可能影响你的系统上的其它Python 软件。

* 你将不可以运行这个包的多个版本（或者具有相同名字的其它包）

特别是一旦你维护几个Django项目，这些情况就会出现。如果真的出现，最好的解决办法是使用
`virtualenv
<https://virtualenv.pypa.io/>`_ 。这个工具允许你维护多个分离的Python环境，每个都具有它自己的库和包的命名空间。

