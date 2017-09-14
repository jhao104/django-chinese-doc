===============
开发Django补丁
===============

介绍
=====

有兴趣为社区做出点贡献吗？也许你会在Django中发现你想要修复的漏洞，或者你希望为它添加一个小功能。

为Django作贡献这件事本身就是使你的顾虑得到解决的最好方式。一开始这可能会使你怯步，
但事实上是很简单的。整个过程中我们会一步一步为你解说，所以你可以通过例子学习。

本教程的受众
-------------

.. seealso::

    如果您正在寻找关于如何提交补丁的参考，请参阅 :doc:`/internals/contributing/writing-code/submitting-patches` 。

使用教程前，我们希望你至少对于Django的运行方式有基础的了解。
这意味着你可以自如地在 :doc:`开发第一个Django应用</intro/tutorial01>` 时使用教程 。
除此之外，你应该对于Python本身有很好的了解。如果您并不太了解，
我们为您推荐 `Dive Into Python`__ ，对于初次使用Python的程序员来说这是一本很棒（而且免费）的在线电子书。

对于版本控制系统及Trac不熟悉的人来说，这份教程及其中的链接所包含的信息足以满足你们开始学习的需求。
然而，如果你希望定期为Django贡献代码，你可能会希望阅读更多关于这些不同工具的信息。

当然对于其中的大部分内容，Django会尽可能做出解释以帮助广大的读者。

.. admonition:: 如何获取帮助:

    如果你在使用本教程时遇到困难，你可以发送信息给 :ref:`django-developers-mailing-list`
    或者登陆 `#django-dev on irc.freenode.net`__ 向其他Django使用者需求帮助。

__ http://www.diveintopython3.net/
__ irc://irc.freenode.net/django-dev

教程包含的内容
---------------

一开始我们会帮助你为Django编写补丁，
在教程结束时，你将具备对工具和所包含过程的基本了解。准确来说，我们的教程将包含以下几点：

* 安装 Git。
* 如何下载Django的开发副本。
* 运行Django测试组件。
* 开发补丁测试。
* 开发补丁。
* 测试补丁。
* 提交pull request。
* 在哪里寻找更多的信息。

一旦你完成了这份教程，你可以浏览剩下的
:doc:`Django's documentation on contributing</internals/contributing/index>` 。
它包含了大量信息。任何想成为Django的正式贡献者必须去阅读它。如果你有问题，它也许会给你答案。

.. admonition:: Python 3 required!

    本教程的内容要求 Python 3版本。 你可以在
    `Python's download page <https://www.python.org/download/>`_ 或者系统包管理器中获取最新版本。

.. admonition:: For Windows 用户

    如果你在Windows上使用Python, 请 "将
    python.exe 添加到环境变量中", 这样才能在命令行中随意调用Python。

行为规范
=========

作为一个贡献者，您可以帮助我们保持Django社区的开放和包容。请阅读并遵守我们的 `Code of Conduct <https://www.djangoproject.com/conduct/>`_ 。

安装 Git
=========

使用教程前，你需要安装好Git，下载Django的最新开发版本并且对你的修改生成补丁文件。

为了确认你是否已经安装了Git, 输入 ``git`` 进入命令行。如果信息提示命令无法找到, 你就需要下载并安装Git,
详情阅读 `Git's download page`__ 。

如果你还不熟悉 Git, 你可以在命令行下输入 ``git help`` 了解更多关于它的命令（确认已安装）。

.. admonition:: For Windows users

    如果在Windows上安装Git，建议选择“Git Bash”选项，以便Git运行在自己的shell中。本教程假设您已经安装了它。


__ http://git-scm.com/download

下载Django 开发版的副本
========================

为Django贡献代码的第一步就是获取源代码复本。`fork Django on GitHub <https://github.com/django/django/fork>`__ 。
在命令行里， 使用 ``cd`` 命令进入你想要保存Django的目录

使用下面的命令来下载Django的源码库：

.. code-block:: console

    $ git clone git@github.com:YourGitHubName/django.git

现在您已经有了Django的本地副本，您可以安装它，就像您可以使用 ``pip`` 安装任何包一样。
最方便的方法是使用 *virtual environment* (或virtualenv)，
它是Python内置的特性，允许您为每个项目保留一个单独的安装包目录，这样它们就不会互相干扰。

最好把你所有的虚拟空间放在一个地方，例如在您的home目录下的 ``.virtualenvs/`` 目录 。如果它还不存在，就创建它

.. code-block:: console

    $ mkdir ~/.virtualenvs

现在通过运行下面代码创建一个新的virtualenv:

.. code-block:: console

    $ python3 -m venv ~/.virtualenvs/djangodev

这个路径是你的电脑上保存新环境的地方。

.. admonition:: For Windows users

    如果您还在Windows上使用Git Bash shell，那么使用内置的 ``venv`` 模块将不起作用，
    因为只为系统shell( ``.bat`` )和PowerShell( ``.ps1`` )创建了激活脚本。使用 ``virtualenv`` 包代替:

    .. code-block:: none

        $ pip install virtualenv
        $ virtualenv ~/.virtualenvs/djangodev

.. admonition:: For Ubuntu users

    在某些版本的Ubuntu上，上面的命令可能会失败。使用 ``virtualenv`` 包，首先确保有 ``pip3`` :

    .. code-block:: console

        $ sudo apt-get install python3-pip
        $ # Prefix the next command with sudo if it gives a permission denied error
        $ pip3 install virtualenv
        $ virtualenv --python=`which python3` ~/.virtualenvs/djangodev

设置virtualenv的最后一步是激活它:

.. code-block:: console

    $ source ~/.virtualenvs/djangodev/bin/activate

如果 ``source`` 命令不可用，您可以尝试使用一个点:

.. code-block:: console

    $ . ~/.virtualenvs/djangodev/bin/activate

.. admonition:: For Windows users

    在Windows上激活你的virtualenv，运行:

    .. code-block:: none

        $ source ~/virtualenvs/djangodev/Scripts/activate

每次打开一个新的终端窗口时，你必须激活 virtualenv。 virtualenvwrapper__ 是一个好用的工具，使用它更方便。

__ https://virtualenvwrapper.readthedocs.io/en/latest/

当你激活了virtualenv后，通过 ``pip`` 安装的任何东西都将安装在新的virtualenv中，和其他环境和系统包是完全分离的。
此外，当前激活的virtualenv的名称将显示在命令行上，以帮助您跟踪正在使用的对象。继续安装Django之前的克隆版本：


.. code-block:: console

    $ pip install -e /path/to/your/local/clone/django/

Django 的已安装的版本现在指向您的本地副本。您可以立即看到它所做的任何更改，这对开发补丁是非常有用的。

回滚到Django以前版本
=====================

这个教程中，我们使用 :ticket:`24788` 问题来作为学习用例，
所以我们要把git中Django的版本回滚到这个问题的补丁没有提交之前。
这样的话我们就可以参与到从草稿到补丁的所有过程，包括运行Django的测试套件

**请记住，我们使用Django的老版仅是为了学习，通常情况下你应当使用当前最新的开发版本来提交补丁**

.. note::

    这个补丁由 Paweł Marczewski 开发， Git 提交到 Django 源码
    `commit 4df7e8483b2679fc1cba3410f08960bac6f51115`__.
    因此，我们要回到补丁提交之前的版本号
    `commit 4ccfc4439a7add24f8db4ef3960d02ef8ae09887`__.

__ https://github.com/django/django/commit/4df7e8483b2679fc1cba3410f08960bac6f51115
__ https://github.com/django/django/commit/4ccfc4439a7add24f8db4ef3960d02ef8ae09887


首先打开Django源码的根目录（这个目录包含了  ``django`` , ``docs`` , ``tests`` , ``AUTHORS`` , 等）
然后你你可以根据下面的教程check out老版本的Django：

.. code-block:: console

    $ git checkout 4ccfc4439a7add24f8db4ef3960d02ef8ae09887

首先运行Django的测试套件
========================

当你贡献代码给Django的时候，非常重要的一点就是你修改的代码不要给其他部分引入新的bug。
有个办法可以在你更改代码之后检查Django是否能正常工作，就是运行Django的测试套件。如
果所有的测试用例都通过，你就有理由相信你的改动完全没有破坏Django。
如果你从来没有运行过Django的测试套件，那么比较好的做法是事先运行一遍，熟悉下正常情况下应该输出什么结果。

在运行测试套件之前，先将它的依赖项安装到Django ``tests/`` 目录中，运行:

.. code-block:: console

    $ pip install -r requirements/py3.txt

如果在安装过程中遇到错误，您的系统可能缺少对一个或多个Python包的依赖。
查阅失败的软件包的文档，或者在Web上搜索您遇到的错误消息。

现在我们已经准备好运行测试套件了。如果您使用的是 GNU/Linux、Mac OS X或其他Unix系统，请运行:

.. code-block:: console

    $ ./runtests.py

现在坐下来放松一下。Django的整个测试套件有超过9600个不同的测试用例，
所以它可能需要5到15分钟时间运行，这也取决于您的计算机的速度。

当Django的测试套件正在运行时，您将看到字符流，表示每次测试的状态。
``E`` 表示测试期间出现错误， ``F`` 表示测试断言失败。这两种方法都被认为是测试失败。
与此同时， ``x`` 和 ``s`` 分别表示预期的故障和跳过测试。Dots 表示通过测试。

Skipped tests are typically due to missing external libraries required to run
the test; see :ref:`running-unit-tests-dependencies` for a list of dependencies
and be sure to install any for tests related to the changes you are making (we
won't need any for this tutorial). Some tests are specific to a particular
database backend and will be skipped if not testing with that backend. SQLite
is the database backend for the default settings. To run the tests using a
different backend, see :ref:`running-unit-tests-settings`.

Once the tests complete, you should be greeted with a message informing you
whether the test suite passed or failed. Since you haven't yet made any changes
to Django's code, the entire test suite **should** pass. If you get failures or
errors make sure you've followed all of the previous steps properly. See
:ref:`running-unit-tests` for more information. If you're using Python 3.5+,
there will be a couple failures related to deprecation warnings that you can
ignore. These failures have since been fixed in Django.

Note that the latest Django trunk may not always be stable. When developing
against trunk, you can check `Django's continuous integration builds`__ to
determine if the failures are specific to your machine or if they are also
present in Django's official builds. If you click to view a particular build,
you can view the "Configuration Matrix" which shows failures broken down by
Python version and database backend.

__ http://djangoci.com

.. note::

    For this tutorial and the ticket we're working on, testing against SQLite
    is sufficient, however, it's possible (and sometimes necessary) to
    :ref:`run the tests using a different database
    <running-unit-tests-settings>`.

给补丁创建分支
================

Before making any changes, create a new branch for the ticket:

.. code-block:: console

    $ git checkout -b ticket_24788

You can choose any name that you want for the branch, "ticket_24788" is an
example. All changes made in this branch will be specific to the ticket and
won't affect the main copy of the code that we cloned earlier.

给ticket写一些测试用例
========================

In most cases, for a patch to be accepted into Django it has to include tests.
For bug fix patches, this means writing a regression test to ensure that the
bug is never reintroduced into Django later on. A regression test should be
written in such a way that it will fail while the bug still exists and pass
once the bug has been fixed. For patches containing new features, you'll need
to include tests which ensure that the new features are working correctly.
They too should fail when the new feature is not present, and then pass once it
has been implemented.

A good way to do this is to write your new tests first, before making any
changes to the code. This style of development is called
`test-driven development`__ and can be applied to both entire projects and
single patches. After writing your tests, you then run them to make sure that
they do indeed fail (since you haven't fixed that bug or added that feature
yet). If your new tests don't fail, you'll need to fix them so that they do.
After all, a regression test that passes regardless of whether a bug is present
is not very helpful at preventing that bug from reoccurring down the road.

Now for our hands-on example.

__ https://en.wikipedia.org/wiki/Test-driven_development

给分支#24788写测试
--------------------

Ticket :ticket:`24788` proposes a small feature addition: the ability to
specify the class level attribute ``prefix`` on Form classes, so that::

    […] forms which ship with apps could effectively namespace themselves such
    that N overlapping form fields could be POSTed at once and resolved to the
    correct form.

In order to resolve this ticket, we'll add a ``prefix`` attribute to the
``BaseForm`` class. When creating instances of this class, passing a prefix to
the ``__init__()`` method will still set that prefix on the created instance.
But not passing a prefix (or passing ``None``) will use the class-level prefix.
Before we make those changes though, we're going to write a couple tests to
verify that our modification functions correctly and continues to function
correctly in the future.

Navigate to Django's ``tests/forms_tests/tests/`` folder and open the
``test_forms.py`` file. Add the following code on line 1674 right before the
``test_forms_with_null_boolean`` function::

    def test_class_prefix(self):
        # Prefix can be also specified at the class level.
        class Person(Form):
            first_name = CharField()
            prefix = 'foo'

        p = Person()
        self.assertEqual(p.prefix, 'foo')

        p = Person(prefix='bar')
        self.assertEqual(p.prefix, 'bar')

This new test checks that setting a class level prefix works as expected, and
that passing a ``prefix`` parameter when creating an instance still works too.

.. admonition:: But this testing thing looks kinda hard...

    If you've never had to deal with tests before, they can look a little hard
    to write at first glance. Fortunately, testing is a *very* big subject in
    computer programming, so there's lots of information out there:

    * A good first look at writing tests for Django can be found in the
      documentation on :doc:`/topics/testing/overview`.
    * Dive Into Python (a free online book for beginning Python developers)
      includes a great `introduction to Unit Testing`__.
    * After reading those, if you want something a little meatier to sink
      your teeth into, there's always the Python :mod:`unittest` documentation.

__ http://www.diveintopython.net/unit_testing/index.html

运行测试
---------

Remember that we haven't actually made any modifications to ``BaseForm`` yet,
so our tests are going to fail. Let's run all the tests in the ``forms_tests``
folder to make sure that's really what happens. From the command line, ``cd``
into the Django ``tests/`` directory and run:

.. code-block:: console

    $ ./runtests.py forms_tests

If the tests ran correctly, you should see one failure corresponding to the test
method we added. If all of the tests passed, then you'll want to make sure that
you added the new test shown above to the appropriate folder and class.

开发ticket代码
================

Next we'll be adding the functionality described in ticket :ticket:`24788` to
Django.

开发 ticket #24788 代码
----------------------------------

Navigate to the ``django/django/forms/`` folder and open the ``forms.py`` file.
Find the ``BaseForm`` class on line 72 and add the ``prefix`` class attribute
right after the ``field_order`` attribute::

    class BaseForm(object):
        # This is the main implementation of all the Form logic. Note that this
        # class is different than Form. See the comments by the Form class for
        # more information. Any improvements to the form API should be made to
        # *this* class, not to the Form class.
        field_order = None
        prefix = None

确保测试通过
-------------

Once you're done modifying Django, we need to make sure that the tests we wrote
earlier pass, so we can see whether the code we wrote above is working
correctly. To run the tests in the ``forms_tests`` folder, ``cd`` into the
Django ``tests/`` directory and run:

.. code-block:: console

    $ ./runtests.py forms_tests

Oops, good thing we wrote those tests! You should still see one failure with
the following exception::

    AssertionError: None != 'foo'

We forgot to add the conditional statement in the ``__init__`` method. Go ahead
and change ``self.prefix = prefix`` that is now on line 87 of
``django/forms/forms.py``, adding a conditional statement::

    if prefix is not None:
        self.prefix = prefix

Re-run the tests and everything should pass. If it doesn't, make sure you
correctly modified the ``BaseForm`` class as shown above and copied the new test
correctly.

再次运行Django测试套件
======================

Once you've verified that your patch and your test are working correctly, it's
a good idea to run the entire Django test suite just to verify that your change
hasn't introduced any bugs into other areas of Django. While successfully
passing the entire test suite doesn't guarantee your code is bug free, it does
help identify many bugs and regressions that might otherwise go unnoticed.

To run the entire Django test suite, ``cd`` into the Django ``tests/``
directory and run:

.. code-block:: console

    $ ./runtests.py

As long as you don't see any failures, you're good to go.

编写文档
==========

This is a new feature, so it should be documented. Add the following section on
line 1068 (at the end of the file) of ``django/docs/ref/forms/api.txt``::

    The prefix can also be specified on the form class::

        >>> class PersonForm(forms.Form):
        ...     ...
        ...     prefix = 'person'

    .. versionadded:: 1.9

        The ability to specify ``prefix`` on the form class was added.

Since this new feature will be in an upcoming release it is also added to the
release notes for Django 1.9, on line 164 under the "Forms" section in the file
``docs/releases/1.9.txt``::

    * A form prefix can be specified inside a form class, not only when
      instantiating a form. See :ref:`form-prefix` for details.

For more information on writing documentation, including an explanation of what
the ``versionadded`` bit is all about, see
:doc:`/internals/contributing/writing-documentation`. That page also includes
an explanation of how to build a copy of the documentation locally, so you can
preview the HTML that will be generated.

预览修改
=========

Now it's time to go through all the changes made in our patch. To display the
differences between your current copy of Django (with your changes) and the
revision that you initially checked out earlier in the tutorial:

.. code-block:: console

    $ git diff

Use the arrow keys to move up and down.

.. code-block:: diff

    diff --git a/django/forms/forms.py b/django/forms/forms.py
    index 509709f..d1370de 100644
    --- a/django/forms/forms.py
    +++ b/django/forms/forms.py
    @@ -75,6 +75,7 @@ class BaseForm(object):
         # information. Any improvements to the form API should be made to *this*
         # class, not to the Form class.
         field_order = None
    +    prefix = None

         def __init__(self, data=None, files=None, auto_id='id_%s', prefix=None,
                      initial=None, error_class=ErrorList, label_suffix=None,
    @@ -83,7 +84,8 @@ class BaseForm(object):
             self.data = data or {}
             self.files = files or {}
             self.auto_id = auto_id
    -        self.prefix = prefix
    +        if prefix is not None:
    +            self.prefix = prefix
             self.initial = initial or {}
             self.error_class = error_class
             # Translators: This is the default suffix added to form field labels
    diff --git a/docs/ref/forms/api.txt b/docs/ref/forms/api.txt
    index 3bc39cd..008170d 100644
    --- a/docs/ref/forms/api.txt
    +++ b/docs/ref/forms/api.txt
    @@ -1065,3 +1065,13 @@ You can put several Django forms inside one ``<form>`` tag. To give each
         >>> print(father.as_ul())
         <li><label for="id_father-first_name">First name:</label> <input type="text" name="father-first_name" id="id_father-first_name" /></li>
         <li><label for="id_father-last_name">Last name:</label> <input type="text" name="father-last_name" id="id_father-last_name" /></li>
    +
    +The prefix can also be specified on the form class::
    +
    +    >>> class PersonForm(forms.Form):
    +    ...     ...
    +    ...     prefix = 'person'
    +
    +.. versionadded:: 1.9
    +
    +    The ability to specify ``prefix`` on the form class was added.
    diff --git a/docs/releases/1.9.txt b/docs/releases/1.9.txt
    index 5b58f79..f9bb9de 100644
    --- a/docs/releases/1.9.txt
    +++ b/docs/releases/1.9.txt
    @@ -161,6 +161,9 @@ Forms
       :attr:`~django.forms.Form.field_order` attribute, the ``field_order``
       constructor argument , or the :meth:`~django.forms.Form.order_fields` method.

    +* A form prefix can be specified inside a form class, not only when
    +  instantiating a form. See :ref:`form-prefix` for details.
    +
     Generic Views
     ^^^^^^^^^^^^^

    diff --git a/tests/forms_tests/tests/test_forms.py b/tests/forms_tests/tests/test_forms.py
    index 690f205..e07fae2 100644
    --- a/tests/forms_tests/tests/test_forms.py
    +++ b/tests/forms_tests/tests/test_forms.py
    @@ -1671,6 +1671,18 @@ class FormsTestCase(SimpleTestCase):
             self.assertEqual(p.cleaned_data['last_name'], 'Lennon')
             self.assertEqual(p.cleaned_data['birthday'], datetime.date(1940, 10, 9))

    +    def test_class_prefix(self):
    +        # Prefix can be also specified at the class level.
    +        class Person(Form):
    +            first_name = CharField()
    +            prefix = 'foo'
    +
    +        p = Person()
    +        self.assertEqual(p.prefix, 'foo')
    +
    +        p = Person(prefix='bar')
    +        self.assertEqual(p.prefix, 'bar')
    +
         def test_forms_with_null_boolean(self):
             # NullBooleanField is a bit of a special case because its presentation (widget)
             # is different than its data. This is handled transparently, though.

When you're done previewing the patch, hit the ``q`` key to return to the
command line. If the patch's content looked okay, it's time to commit the
changes.

提交更改到分支
==============

To commit the changes:

.. code-block:: console

    $ git commit -a

This opens up a text editor to type the commit message. Follow the :ref:`commit
message guidelines <committing-guidelines>` and write a message like:

.. code-block:: text

    Fixed #24788 -- Allowed Forms to specify a prefix at the class level.

推送到远程并请求合并
====================

After committing the patch, send it to your fork on GitHub (substitute
"ticket_24788" with the name of your branch if it's different):

.. code-block:: console

    $ git push origin ticket_24788

You can create a pull request by visiting the `Django GitHub page
<https://github.com/django/django/>`_. You'll see your branch under "Your
recently pushed branches". Click "Compare & pull request" next to it.

Please don't do it for this tutorial, but on the next page that displays a
preview of the patch, you would click "Create pull request".

下一步
=========

Congratulations, you've learned how to make a pull request to Django! Details
of more advanced techniques you may need are in
:doc:`/internals/contributing/writing-code/working-with-git`.

Now you can put those skills to good use by helping to improve Django's
codebase.

contributors的更多信息
----------------------

Before you get too into writing patches for Django, there's a little more
information on contributing that you should probably take a look at:

* You should make sure to read Django's documentation on
  :doc:`claiming tickets and submitting patches
  </internals/contributing/writing-code/submitting-patches>`.
  It covers Trac etiquette, how to claim tickets for yourself, expected
  coding style for patches, and many other important details.
* First time contributors should also read Django's :doc:`documentation
  for first time contributors</internals/contributing/new-contributors/>`.
  It has lots of good advice for those of us who are new to helping out
  with Django.
* After those, if you're still hungry for more information about
  contributing, you can always browse through the rest of
  :doc:`Django's documentation on contributing</internals/contributing/index>`.
  It contains a ton of useful information and should be your first source
  for answering any questions you might have.

Finding your first real ticket
------------------------------

Once you've looked through some of that information, you'll be ready to go out
and find a ticket of your own to write a patch for. Pay special attention to
tickets with the "easy pickings" criterion. These tickets are often much
simpler in nature and are great for first time contributors. Once you're
familiar with contributing to Django, you can move on to writing patches for
more difficult and complicated tickets.

If you just want to get started already (and nobody would blame you!), try
taking a look at the list of `easy tickets that need patches`__ and the
`easy tickets that have patches which need improvement`__. If you're familiar
with writing tests, you can also look at the list of
`easy tickets that need tests`__. Just remember to follow the guidelines about
claiming tickets that were mentioned in the link to Django's documentation on
:doc:`claiming tickets and submitting patches
</internals/contributing/writing-code/submitting-patches>`.

__ https://code.djangoproject.com/query?status=new&status=reopened&has_patch=0&easy=1&col=id&col=summary&col=status&col=owner&col=type&col=milestone&order=priority
__ https://code.djangoproject.com/query?status=new&status=reopened&needs_better_patch=1&easy=1&col=id&col=summary&col=status&col=owner&col=type&col=milestone&order=priority
__ https://code.djangoproject.com/query?status=new&status=reopened&needs_tests=1&easy=1&col=id&col=summary&col=status&col=owner&col=type&col=milestone&order=priority

What's next after creating a pull request?
------------------------------------------

After a ticket has a patch, it needs to be reviewed by a second set of eyes.
After submitting a pull request, update the ticket metadata by setting the
flags on the ticket to say "has patch", "doesn't need tests", etc, so others
can find it for review. Contributing doesn't necessarily always mean writing a
patch from scratch. Reviewing existing patches is also a very helpful
contribution. See :doc:`/internals/contributing/triaging-tickets` for details.
