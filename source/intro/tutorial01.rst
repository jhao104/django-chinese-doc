===========================
开发第一个Django应用,Part1
===========================

通过例子学习。在本教程中，将介绍创建一个简单的投票应用程序。

它包含下面两部分：

-  一个可以进行投票和查看结果的公开站点；

-  一个可以进行增删改查的后台admin管理界面；

如果你已经 :doc:`安装 </intro/install>` 了Django。您可以通过运行以下命令来查看Django版本以及验证是否安装：

.. code-block:: console

    $ python -m django --version

如果安装了Django，您应该将看到安装的版本。如果没有安装，你会得到一个错误，提示 ``No module named django`` 。

本教程是为Django |version| 和Python3.4或更高版本编写的。如果Django版本不匹配，
您可以去官网参考您的对应Django版本的教程，或者将Django更新到最新版本。

如果你仍然在使用Python 2.7，你需要稍微调整代码，注意代码中的注释。

创建project
===========

如果这是你第一次使用Django，你将需要处理一些初始设置。也就是说，这会自动生成一些建立Django项目的代码，
但是你需要设置一些配置，包括数据库配置，Django特定的选项和应用程序特定的设置等等。

从命令行，\ ``cd``\ 进入您将存放项目代码的目录，然后运行以下命令:

.. code-block:: console

   $ django-admin startproject mysite

如果运行出错，请参见 :ref:`troubleshooting-django-admin` 。这将在当前目录下生成一个 ``mysite`` 目录，
也就是你的这个Django项目的根目录。它包含了一系列自动生成的目录和文件，具备各自专有的用途。

.. note::

    在给项目命名的时候必须避开Django和Python的保留关键字。
    比如 ``django``（它会与Django本身冲突）或 ``test``（它与一个内置的Python包冲突）。

.. admonition:: 这些代码应该放在哪儿？

    如果你曾经学过普通的旧式的PHP（没有使用过现代的框架），
    你可能习惯于将代码放在Web服务器的文档根目录下（例如 ``/var/www``）。
    使用Django时，建议你不要这么做。将Python代码放在你的Web服务器的根目录不是个好主意，
    因为这可能会有让其他人看到你的代码的风险。
    你可以将你的代码放在根目录的 **外部**，比如
    :file:`/home/mycode`.

 :djadmin:`startproject` 会创建如下目录结构::

    mysite/
        manage.py
        mysite/
            __init__.py
            settings.py
            urls.py
            wsgi.py


这些文件分别是:

-  外层的 :file:`mysite/` 根目录仅仅是项目的一个容器。它的命名对Django无关紧要；你可以把它重新命名为任何你喜欢的名字;

-  :file:`manage.py` ：一个命令行工具，可以使你用多种方式对Django项目进行交互。
   你可以在 :doc:`/ref/django-admin` 中读到关于 :file:`manage.py` 的所有细节;

-  内层的mysite/目录是你的项目的真正的Python包。它的名字是你引用内部文件的包名（例如
   mysite.urls）;

-  :file:`mysite/__init__.py` ：一个空文件，它告诉Python这个目录应该被看做一个Python包;

-  :file:`mysite/settings.py` ：该Django项目的配置文件。具体内容可以参见 :doc:`/topics/settings` ;

-  :file:`mysite/urls.py` : 路由文件,相当于你的Django站点的“目录”。
   你可以在 :doc:`/topics/http/urls` 中阅读到关于URL的更多内容;

-  :file:`mysite/wsgi.py` ：用于你的项目的与WSGI兼容的Web服务器入口。用作服务部署，更多细节请参见 :doc:`/howto/deployment/wsgi/index` 。


The development server
======================

Let's verify your Django project works. Change into the outer :file:`mysite` directory, if
you haven't already, and run the following commands:

.. code-block:: console

   $ python manage.py runserver

You'll see the following output on the command line:

.. parsed-literal::

    Performing system checks...

    System check identified no issues (0 silenced).

    You have unapplied migrations; your app may not work properly until they are applied.
    Run 'python manage.py migrate' to apply them.

    |today| - 15:50:53
    Django version |version|, using settings 'mysite.settings'
    Starting development server at http://127.0.0.1:8000/
    Quit the server with CONTROL-C.

.. note::
    Ignore the warning about unapplied database migrations for now; we'll deal
    with the database shortly.

You've started the Django development server, a lightweight Web server written
purely in Python. We've included this with Django so you can develop things
rapidly, without having to deal with configuring a production server -- such as
Apache -- until you're ready for production.

Now's a good time to note: **don't** use this server in anything resembling a
production environment. It's intended only for use while developing. (We're in
the business of making Web frameworks, not Web servers.)

Now that the server's running, visit http://127.0.0.1:8000/ with your Web
browser. You'll see a "Welcome to Django" page, in pleasant, light-blue pastel.
It worked!

.. admonition:: Changing the port

    By default, the :djadmin:`runserver` command starts the development server
    on the internal IP at port 8000.

    If you want to change the server's port, pass
    it as a command-line argument. For instance, this command starts the server
    on port 8080:

    .. code-block:: console

        $ python manage.py runserver 8080

    If you want to change the server's IP, pass it along with the port. So to
    listen on all public IPs (useful if you want to show off your work on other
    computers on your network), use:

    .. code-block:: console

        $ python manage.py runserver 0.0.0.0:8000

    Full docs for the development server can be found in the
    :djadmin:`runserver` reference.

.. admonition:: Automatic reloading of :djadmin:`runserver`

    The development server automatically reloads Python code for each request
    as needed. You don't need to restart the server for code changes to take
    effect. However, some actions like adding files don't trigger a restart,
    so you'll have to restart the server in these cases.

Creating the Polls app
======================

Now that your environment -- a "project" -- is set up, you're set to start
doing work.

Each application you write in Django consists of a Python package that follows
a certain convention. Django comes with a utility that automatically generates
the basic directory structure of an app, so you can focus on writing code
rather than creating directories.

.. admonition:: Projects vs. apps

    What's the difference between a project and an app? An app is a Web
    application that does something -- e.g., a Weblog system, a database of
    public records or a simple poll app. A project is a collection of
    configuration and apps for a particular website. A project can contain
    multiple apps. An app can be in multiple projects.

Your apps can live anywhere on your :ref:`Python path <tut-searchpath>`. In
this tutorial, we'll create our poll app right next to your :file:`manage.py`
file so that it can be imported as its own top-level module, rather than a
submodule of ``mysite``.

To create your app, make sure you're in the same directory as :file:`manage.py`
and type this command:

.. code-block:: console

    $ python manage.py startapp polls

That'll create a directory :file:`polls`, which is laid out like this::

    polls/
        __init__.py
        admin.py
        apps.py
        migrations/
            __init__.py
        models.py
        tests.py
        views.py

This directory structure will house the poll application.

Write your first view
=====================

Let's write the first view. Open the file ``polls/views.py``
and put the following Python code in it:

.. snippet::
    :filename: polls/views.py

    from django.http import HttpResponse


    def index(request):
        return HttpResponse("Hello, world. You're at the polls index.")

This is the simplest view possible in Django. To call the view, we need to map
it to a URL - and for this we need a URLconf.

To create a URLconf in the polls directory, create a file called ``urls.py``.
Your app directory should now look like::

    polls/
        __init__.py
        admin.py
        apps.py
        migrations/
            __init__.py
        models.py
        tests.py
        urls.py
        views.py

In the ``polls/urls.py`` file include the following code:

.. snippet::
    :filename: polls/urls.py

    from django.conf.urls import url

    from . import views

    urlpatterns = [
        url(r'^$', views.index, name='index'),
    ]

The next step is to point the root URLconf at the ``polls.urls`` module. In
``mysite/urls.py``, add an import for ``django.conf.urls.include`` and insert
an :func:`~django.conf.urls.include` in the ``urlpatterns`` list, so you have:

.. snippet::
    :filename: mysite/urls.py

    from django.conf.urls import include, url
    from django.contrib import admin

    urlpatterns = [
        url(r'^polls/', include('polls.urls')),
        url(r'^admin/', admin.site.urls),
    ]

The :func:`~django.conf.urls.include` function allows referencing other
URLconfs. Note that the regular expressions for the
:func:`~django.conf.urls.include` function doesn't have a ``$`` (end-of-string
match character) but rather a trailing slash. Whenever Django encounters
:func:`~django.conf.urls.include`, it chops off whatever part of the URL
matched up to that point and sends the remaining string to the included URLconf
for further processing.

The idea behind :func:`~django.conf.urls.include` is to make it easy to
plug-and-play URLs. Since polls are in their own URLconf
(``polls/urls.py``), they can be placed under "/polls/", or under
"/fun_polls/", or under "/content/polls/", or any other path root, and the
app will still work.

.. admonition:: When to use :func:`~django.conf.urls.include()`

    You should always use ``include()`` when you include other URL patterns.
    ``admin.site.urls`` is the only exception to this.

.. admonition:: Doesn't match what you see?

    If you're seeing ``include(admin.site.urls)`` instead of just
    ``admin.site.urls``, you're probably using a version of Django that
    doesn't match this tutorial version.  You'll want to either switch to the
    older tutorial or the newer Django version.

You have now wired an ``index`` view into the URLconf. Lets verify it's
working, run the following command:

.. code-block:: console

   $ python manage.py runserver

Go to http://localhost:8000/polls/ in your browser, and you should see the
text "*Hello, world. You're at the polls index.*", which you defined in the
``index`` view.

The :func:`~django.conf.urls.url` function is passed four arguments, two
required: ``regex`` and ``view``, and two optional: ``kwargs``, and ``name``.
At this point, it's worth reviewing what these arguments are for.

:func:`~django.conf.urls.url` argument: regex
---------------------------------------------

The term "regex" is a commonly used short form meaning "regular expression",
which is a syntax for matching patterns in strings, or in this case, url
patterns. Django starts at the first regular expression and makes its way down
the list,  comparing the requested URL against each regular expression until it
finds one that matches.

Note that these regular expressions do not search GET and POST parameters, or
the domain name. For example, in a request to
``https://www.example.com/myapp/``, the URLconf will look for ``myapp/``. In a
request to ``https://www.example.com/myapp/?page=3``, the URLconf will also
look for ``myapp/``.

If you need help with regular expressions, see `Wikipedia's entry`_ and the
documentation of the :mod:`re` module. Also, the O'Reilly book "Mastering
Regular Expressions" by Jeffrey Friedl is fantastic. In practice, however,
you don't need to be an expert on regular expressions, as you really only need
to know how to capture simple patterns. In fact, complex regexes can have poor
lookup performance, so you probably shouldn't rely on the full power of regexes.

Finally, a performance note: these regular expressions are compiled the first
time the URLconf module is loaded. They're super fast (as long as the lookups
aren't too complex as noted above).

.. _Wikipedia's entry: https://en.wikipedia.org/wiki/Regular_expression

:func:`~django.conf.urls.url` argument: view
--------------------------------------------

When Django finds a regular expression match, Django calls the specified view
function, with an :class:`~django.http.HttpRequest` object as the first
argument and any “captured” values from the regular expression as other
arguments. If the regex uses simple captures, values are passed as positional
arguments; if it uses named captures, values are passed as keyword arguments.
We'll give an example of this in a bit.

:func:`~django.conf.urls.url` argument: kwargs
----------------------------------------------

Arbitrary keyword arguments can be passed in a dictionary to the target view. We
aren't going to use this feature of Django in the tutorial.

:func:`~django.conf.urls.url` argument: name
---------------------------------------------

Naming your URL lets you refer to it unambiguously from elsewhere in Django,
especially from within templates. This powerful feature allows you to make
global changes to the URL patterns of your project while only touching a single
file.

When you're comfortable with the basic request and response flow, read
:doc:`part 2 of this tutorial </intro/tutorial02>` to start working with the
database.
