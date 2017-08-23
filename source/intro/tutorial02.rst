===========================
开发第一个Django应用,Part2
===========================

.. rubric:: 模型和管理站点

本教程继续 :doc:`Tutorial 1 </intro/tutorial01>。我们将设置数据库，创建您的第一个模型，并快速介绍Django的自动生成的管理网站。

数据库设置
==========

现在，编辑 :file:`mysite/settings.py` 。它是一个用模块级别变量表示Django配置的普通Python模块。

Django的默认数据库是SQLite。如果你是数据库初学者，或者你只是想要试用一下Django，
SQLite是最简单的选择。 SQLite包含在Python中，所以你不需要另外安装其他任何东西。
当然在你开始第一个真正的项目时，你可能想使用一个更健壮的数据库比如PostgreSQL来避免在未来遇到令人头疼的数据库切换问题。

如果你希望使用另外一种数据库，请配置合适的 :ref:`database bindings <database-installation>`，
并在 :file:`mysite/settings.py` 的 :setting:`DATABASES` ``'default'`` 条目中修改以下的配置以匹配你的数据库连接的设置：


* :setting:`ENGINE <DATABASE-ENGINE>` -- 支持
  ``'django.db.backends.sqlite3'``,
  ``'django.db.backends.postgresql'``,
  ``'django.db.backends.mysql'``, 和
  ``'django.db.backends.oracle'``. 更多参考 :ref:`also available <third-party-notes>`.

* :setting:`NAME` -- 数据库的名称。如果你使用SQLite，数据库将是你计算机上的一个文件； 如果是这样的话，NAME应该是这个文件的绝对路径，包括文件名。默认值是 ``os.path.join(BASE_DIR, 'db.sqlite3')``，它将文件保存在你项目的目录中；

如果不使用SQLite作为数据库，则必须添加其他设置，
例如 :setting:`USER`，:setting:`PASSWORD` 和 :setting:`HOST`。有关更多详细信息，请参阅 :setting:`DATABASES` 的参考文档。


.. admonition:: SQLite以外的数据库

    如果您使用SQLite之外的数据库，请确保您已经创建好了一个数据库。
    如果没有，请在数据库的交互中使用“ ``CREATE DATABASE database_name`` ”进行创建。

    并且要保证在 :file:`mysite/settings.py` 中配置的用户有创建表的权限。

    如果您使用的是SQLite，那么您不需要预先创建任何东西——数据库文件将在需要时自动创建。

当你编辑 :file:`mysite/settings.py` 时，请设置 :setting:`TIME_ZONE` 为你自己的时区。

:setting:`INSTALLED_APPS` 中是Django实例中所有Django应用的名称。
应用可以在多个项目中使用，而且你可以将这些应用打包和分发给其他人在他们的项目中使用。

:setting:`INSTALLED_APPS` 默认包含了以下应用:

* :mod:`django.contrib.admin` --  管理站点.

* :mod:`django.contrib.auth` -- 用户认证系统.

* :mod:`django.contrib.contenttypes` -- 用于内容类型的框架.

* :mod:`django.contrib.sessions` -- session框架.

* :mod:`django.contrib.messages` -- 消息框架.

* :mod:`django.contrib.staticfiles` -- 管理静态文件的框架.

这些应用，默认包含在Django中，以方便通用场合下使用。

其中一些应用程序需要数据库表才能使用，所以我们需要在数据库中创建表，然后才能使用它们。为此，请运行以下命令：

.. code-block:: console

    $ python manage.py migrate

:djadmin:`migrate` 查看 :setting:`INSTALLED_APPS` 设置并根据 :file:`mysite/settings.py` 文件中的数据库设置
创建任何必要的数据库表，数据库的迁移还会跟踪应用的变化。你会看到对每次迁移有一条信息。
如果你有兴趣，可以运行你的数据库的命令行客户端并输入 ``\dt`` (PostgreSQL), ``SHOW TABLES;`` (MySQL)
或 ``.schema`` (SQLite)或 ``SELECT TABLE_NAME FROM USER_TABLES;`` (Oracle) 来显示Django创建的表。

.. admonition:: 至极简主义者

    :setting:`INSTALLED_APPS` 包含的默认应用用于常见的场景，但并不是每个人都需要它们。
    如果你不需要它们中的任何一个或所有应用，
    可以在运行 :djadmin:`migrate` 之前从 :setting:`INSTALLED_APPS` 中自由地注释或删除相应的行。
    :djadmin:`migrate` 命令将只为 :setting:`INSTALLED_APPS` 中的应用运行数据库的迁移。

.. _creating-models:

创建模型
========

现在定义该应用的模型——本质上，就是定义该模型所对应的数据库设计及其附带的元数据。

.. admonition:: 定义

   模型是关于你的数据的唯一的、明确的来源。它包含您正在存储的数据的基本字段和行为。
   Django遵循 :ref:`DRY Principle <dry>` 。只在一个地方定义数据模型，并自动从中派生出其他东西。

   这包括迁移——与Ruby On Rails不同的是，例如迁移完全依照于你的模型文件且本质上只是一个历史记录，
   Django通过这个历史记录更新你的数据库模式使它与你现在的模型文件保持一致。


在这个简单的投票应用中，我们将创建两个模型：
``Question`` 和 ``Choice``。 ``Question`` 对象具有一个question_text（问题）属性和一个publish_date（发布时间）属性。
``Choice`` 有两个字段：选择的内容和选择的得票统计。 每个Choice与一个Question关联。

这些概念通过简单的Python类来表示。 编辑 :file:`polls/models.py` 文件，并让它看起来像这样：

.. snippet::
    :filename: polls/models.py

    from django.db import models


    class Question(models.Model):
        question_text = models.CharField(max_length=200)
        pub_date = models.DateTimeField('date published')


    class Choice(models.Model):
        question = models.ForeignKey(Question, on_delete=models.CASCADE)
        choice_text = models.CharField(max_length=200)
        votes = models.IntegerField(default=0)

代码很简单。每个模型由一个继承 :class:`django.db.models.Model` 的类表示。
每个模型都有一些类变量，每个变量表示模型中的数据库字段。


Each field is represented by an instance of a :class:`~django.db.models.Field`
class -- e.g., :class:`~django.db.models.CharField` for character fields and
:class:`~django.db.models.DateTimeField` for datetimes. This tells Django what
type of data each field holds.

The name of each :class:`~django.db.models.Field` instance (e.g.
``question_text`` or ``pub_date``) is the field's name, in machine-friendly
format. You'll use this value in your Python code, and your database will use
it as the column name.

You can use an optional first positional argument to a
:class:`~django.db.models.Field` to designate a human-readable name. That's used
in a couple of introspective parts of Django, and it doubles as documentation.
If this field isn't provided, Django will use the machine-readable name. In this
example, we've only defined a human-readable name for ``Question.pub_date``.
For all other fields in this model, the field's machine-readable name will
suffice as its human-readable name.

Some :class:`~django.db.models.Field` classes have required arguments.
:class:`~django.db.models.CharField`, for example, requires that you give it a
:attr:`~django.db.models.CharField.max_length`. That's used not only in the
database schema, but in validation, as we'll soon see.

A :class:`~django.db.models.Field` can also have various optional arguments; in
this case, we've set the :attr:`~django.db.models.Field.default` value of
``votes`` to 0.

Finally, note a relationship is defined, using
:class:`~django.db.models.ForeignKey`. That tells Django each ``Choice`` is
related to a single ``Question``. Django supports all the common database
relationships: many-to-one, many-to-many, and one-to-one.

Activating models
=================

That small bit of model code gives Django a lot of information. With it, Django
is able to:

* Create a database schema (``CREATE TABLE`` statements) for this app.
* Create a Python database-access API for accessing ``Question`` and ``Choice`` objects.

But first we need to tell our project that the ``polls`` app is installed.

.. admonition:: Philosophy

    Django apps are "pluggable": You can use an app in multiple projects, and
    you can distribute apps, because they don't have to be tied to a given
    Django installation.

To include the app in our project, we need to add a reference to its
configuration class in the :setting:`INSTALLED_APPS` setting. The
``PollsConfig`` class is in the :file:`polls/apps.py` file, so its dotted path
is ``'polls.apps.PollsConfig'``. Edit the :file:`mysite/settings.py` file and
add that dotted path to the :setting:`INSTALLED_APPS` setting. It'll look like
this:

.. snippet::
    :filename: mysite/settings.py

    INSTALLED_APPS = [
        'polls.apps.PollsConfig',
        'django.contrib.admin',
        'django.contrib.auth',
        'django.contrib.contenttypes',
        'django.contrib.sessions',
        'django.contrib.messages',
        'django.contrib.staticfiles',
    ]

Now Django knows to include the ``polls`` app. Let's run another command:

.. code-block:: console

    $ python manage.py makemigrations polls

You should see something similar to the following:

.. code-block:: text

    Migrations for 'polls':
      polls/migrations/0001_initial.py:
        - Create model Choice
        - Create model Question
        - Add field question to choice

By running ``makemigrations``, you're telling Django that you've made
some changes to your models (in this case, you've made new ones) and that
you'd like the changes to be stored as a *migration*.

Migrations are how Django stores changes to your models (and thus your
database schema) - they're just files on disk. You can read the migration
for your new model if you like; it's the file
``polls/migrations/0001_initial.py``. Don't worry, you're not expected to read
them every time Django makes one, but they're designed to be human-editable
in case you want to manually tweak how Django changes things.

There's a command that will run the migrations for you and manage your database
schema automatically - that's called :djadmin:`migrate`, and we'll come to it in a
moment - but first, let's see what SQL that migration would run. The
:djadmin:`sqlmigrate` command takes migration names and returns their SQL:

.. code-block:: console

    $ python manage.py sqlmigrate polls 0001

You should see something similar to the following (we've reformatted it for
readability):

.. code-block:: sql

    BEGIN;
    --
    -- Create model Choice
    --
    CREATE TABLE "polls_choice" (
        "id" serial NOT NULL PRIMARY KEY,
        "choice_text" varchar(200) NOT NULL,
        "votes" integer NOT NULL
    );
    --
    -- Create model Question
    --
    CREATE TABLE "polls_question" (
        "id" serial NOT NULL PRIMARY KEY,
        "question_text" varchar(200) NOT NULL,
        "pub_date" timestamp with time zone NOT NULL
    );
    --
    -- Add field question to choice
    --
    ALTER TABLE "polls_choice" ADD COLUMN "question_id" integer NOT NULL;
    ALTER TABLE "polls_choice" ALTER COLUMN "question_id" DROP DEFAULT;
    CREATE INDEX "polls_choice_7aa0f6ee" ON "polls_choice" ("question_id");
    ALTER TABLE "polls_choice"
      ADD CONSTRAINT "polls_choice_question_id_246c99a640fbbd72_fk_polls_question_id"
        FOREIGN KEY ("question_id")
        REFERENCES "polls_question" ("id")
        DEFERRABLE INITIALLY DEFERRED;

    COMMIT;

Note the following:

* The exact output will vary depending on the database you are using. The
  example above is generated for PostgreSQL.

* Table names are automatically generated by combining the name of the app
  (``polls``) and the lowercase name of the model -- ``question`` and
  ``choice``. (You can override this behavior.)

* Primary keys (IDs) are added automatically. (You can override this, too.)

* By convention, Django appends ``"_id"`` to the foreign key field name.
  (Yes, you can override this, as well.)

* The foreign key relationship is made explicit by a ``FOREIGN KEY``
  constraint. Don't worry about the ``DEFERRABLE`` parts; that's just telling
  PostgreSQL to not enforce the foreign key until the end of the transaction.

* It's tailored to the database you're using, so database-specific field types
  such as ``auto_increment`` (MySQL), ``serial`` (PostgreSQL), or ``integer
  primary key autoincrement`` (SQLite) are handled for you automatically. Same
  goes for the quoting of field names -- e.g., using double quotes or
  single quotes.

* The :djadmin:`sqlmigrate` command doesn't actually run the migration on your
  database - it just prints it to the screen so that you can see what SQL
  Django thinks is required. It's useful for checking what Django is going to
  do or if you have database administrators who require SQL scripts for
  changes.

If you're interested, you can also run
:djadmin:`python manage.py check <check>`; this checks for any problems in
your project without making migrations or touching the database.

Now, run :djadmin:`migrate` again to create those model tables in your database:

.. code-block:: console

    $ python manage.py migrate
    Operations to perform:
      Apply all migrations: admin, auth, contenttypes, polls, sessions
    Running migrations:
      Rendering model states... DONE
      Applying polls.0001_initial... OK

The :djadmin:`migrate` command takes all the migrations that haven't been
applied (Django tracks which ones are applied using a special table in your
database called ``django_migrations``) and runs them against your database -
essentially, synchronizing the changes you made to your models with the schema
in the database.

Migrations are very powerful and let you change your models over time, as you
develop your project, without the need to delete your database or tables and
make new ones - it specializes in upgrading your database live, without
losing data. We'll cover them in more depth in a later part of the tutorial,
but for now, remember the three-step guide to making model changes:

* Change your models (in ``models.py``).
* Run :djadmin:`python manage.py makemigrations <makemigrations>` to create
  migrations for those changes
* Run :djadmin:`python manage.py migrate <migrate>` to apply those changes to
  the database.

The reason that there are separate commands to make and apply migrations is
because you'll commit migrations to your version control system and ship them
with your app; they not only make your development easier, they're also
useable by other developers and in production.

Read the :doc:`django-admin documentation </ref/django-admin>` for full
information on what the ``manage.py`` utility can do.

Playing with the API
====================

Now, let's hop into the interactive Python shell and play around with the free
API Django gives you. To invoke the Python shell, use this command:

.. code-block:: console

    $ python manage.py shell

We're using this instead of simply typing "python", because :file:`manage.py`
sets the ``DJANGO_SETTINGS_MODULE`` environment variable, which gives Django
the Python import path to your :file:`mysite/settings.py` file.

.. admonition:: Bypassing manage.py

    If you'd rather not use :file:`manage.py`, no problem. Just set the
    :envvar:`DJANGO_SETTINGS_MODULE` environment variable to
    ``mysite.settings``, start a plain Python shell, and set up Django:

    .. code-block:: pycon

        >>> import django
        >>> django.setup()

    If this raises an :exc:`AttributeError`, you're probably using
    a version of Django that doesn't match this tutorial version. You'll want
    to either switch to the older tutorial or the newer Django version.

    You must run ``python`` from the same directory :file:`manage.py` is in,
    or ensure that directory is on the Python path, so that ``import mysite``
    works.

    For more information on all of this, see the :doc:`django-admin
    documentation </ref/django-admin>`.

Once you're in the shell, explore the :doc:`database API </topics/db/queries>`::

    >>> from polls.models import Question, Choice   # Import the model classes we just wrote.

    # No questions are in the system yet.
    >>> Question.objects.all()
    <QuerySet []>

    # Create a new Question.
    # Support for time zones is enabled in the default settings file, so
    # Django expects a datetime with tzinfo for pub_date. Use timezone.now()
    # instead of datetime.datetime.now() and it will do the right thing.
    >>> from django.utils import timezone
    >>> q = Question(question_text="What's new?", pub_date=timezone.now())

    # Save the object into the database. You have to call save() explicitly.
    >>> q.save()

    # Now it has an ID. Note that this might say "1L" instead of "1", depending
    # on which database you're using. That's no biggie; it just means your
    # database backend prefers to return integers as Python long integer
    # objects.
    >>> q.id
    1

    # Access model field values via Python attributes.
    >>> q.question_text
    "What's new?"
    >>> q.pub_date
    datetime.datetime(2012, 2, 26, 13, 0, 0, 775217, tzinfo=<UTC>)

    # Change values by changing the attributes, then calling save().
    >>> q.question_text = "What's up?"
    >>> q.save()

    # objects.all() displays all the questions in the database.
    >>> Question.objects.all()
    <QuerySet [<Question: Question object>]>

Wait a minute. ``<Question: Question object>`` is, utterly, an unhelpful representation
of this object. Let's fix that by editing the ``Question`` model (in the
``polls/models.py`` file) and adding a
:meth:`~django.db.models.Model.__str__` method to both ``Question`` and
``Choice``:

.. snippet::
    :filename: polls/models.py

    from django.db import models
    from django.utils.encoding import python_2_unicode_compatible

    @python_2_unicode_compatible  # only if you need to support Python 2
    class Question(models.Model):
        # ...
        def __str__(self):
            return self.question_text

    @python_2_unicode_compatible  # only if you need to support Python 2
    class Choice(models.Model):
        # ...
        def __str__(self):
            return self.choice_text

It's important to add :meth:`~django.db.models.Model.__str__` methods to your
models, not only for your own convenience when dealing with the interactive
prompt, but also because objects' representations are used throughout Django's
automatically-generated admin.

Note these are normal Python methods. Let's add a custom method, just for
demonstration:

.. snippet::
    :filename: polls/models.py

    import datetime

    from django.db import models
    from django.utils import timezone


    class Question(models.Model):
        # ...
        def was_published_recently(self):
            return self.pub_date >= timezone.now() - datetime.timedelta(days=1)

Note the addition of ``import datetime`` and ``from django.utils import
timezone``, to reference Python's standard :mod:`datetime` module and Django's
time-zone-related utilities in :mod:`django.utils.timezone`, respectively. If
you aren't familiar with time zone handling in Python, you can learn more in
the :doc:`time zone support docs </topics/i18n/timezones>`.

Save these changes and start a new Python interactive shell by running
``python manage.py shell`` again::

    >>> from polls.models import Question, Choice

    # Make sure our __str__() addition worked.
    >>> Question.objects.all()
    <QuerySet [<Question: What's up?>]>

    # Django provides a rich database lookup API that's entirely driven by
    # keyword arguments.
    >>> Question.objects.filter(id=1)
    <QuerySet [<Question: What's up?>]>
    >>> Question.objects.filter(question_text__startswith='What')
    <QuerySet [<Question: What's up?>]>

    # Get the question that was published this year.
    >>> from django.utils import timezone
    >>> current_year = timezone.now().year
    >>> Question.objects.get(pub_date__year=current_year)
    <Question: What's up?>

    # Request an ID that doesn't exist, this will raise an exception.
    >>> Question.objects.get(id=2)
    Traceback (most recent call last):
        ...
    DoesNotExist: Question matching query does not exist.

    # Lookup by a primary key is the most common case, so Django provides a
    # shortcut for primary-key exact lookups.
    # The following is identical to Question.objects.get(id=1).
    >>> Question.objects.get(pk=1)
    <Question: What's up?>

    # Make sure our custom method worked.
    >>> q = Question.objects.get(pk=1)
    >>> q.was_published_recently()
    True

    # Give the Question a couple of Choices. The create call constructs a new
    # Choice object, does the INSERT statement, adds the choice to the set
    # of available choices and returns the new Choice object. Django creates
    # a set to hold the "other side" of a ForeignKey relation
    # (e.g. a question's choice) which can be accessed via the API.
    >>> q = Question.objects.get(pk=1)

    # Display any choices from the related object set -- none so far.
    >>> q.choice_set.all()
    <QuerySet []>

    # Create three choices.
    >>> q.choice_set.create(choice_text='Not much', votes=0)
    <Choice: Not much>
    >>> q.choice_set.create(choice_text='The sky', votes=0)
    <Choice: The sky>
    >>> c = q.choice_set.create(choice_text='Just hacking again', votes=0)

    # Choice objects have API access to their related Question objects.
    >>> c.question
    <Question: What's up?>

    # And vice versa: Question objects get access to Choice objects.
    >>> q.choice_set.all()
    <QuerySet [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]>
    >>> q.choice_set.count()
    3

    # The API automatically follows relationships as far as you need.
    # Use double underscores to separate relationships.
    # This works as many levels deep as you want; there's no limit.
    # Find all Choices for any question whose pub_date is in this year
    # (reusing the 'current_year' variable we created above).
    >>> Choice.objects.filter(question__pub_date__year=current_year)
    <QuerySet [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]>

    # Let's delete one of the choices. Use delete() for that.
    >>> c = q.choice_set.filter(choice_text__startswith='Just hacking')
    >>> c.delete()

For more information on model relations, see :doc:`Accessing related objects
</ref/models/relations>`. For more on how to use double underscores to perform
field lookups via the API, see :ref:`Field lookups <field-lookups-intro>`. For
full details on the database API, see our :doc:`Database API reference
</topics/db/queries>`.

Introducing the Django Admin
============================

.. admonition:: Philosophy

    Generating admin sites for your staff or clients to add, change, and delete
    content is tedious work that doesn't require much creativity. For that
    reason, Django entirely automates creation of admin interfaces for models.

    Django was written in a newsroom environment, with a very clear separation
    between "content publishers" and the "public" site. Site managers use the
    system to add news stories, events, sports scores, etc., and that content is
    displayed on the public site. Django solves the problem of creating a
    unified interface for site administrators to edit content.

    The admin isn't intended to be used by site visitors. It's for site
    managers.

Creating an admin user
----------------------

First we'll need to create a user who can login to the admin site. Run the
following command:

.. code-block:: console

    $ python manage.py createsuperuser

Enter your desired username and press enter.

.. code-block:: text

    Username: admin

You will then be prompted for your desired email address:

.. code-block:: text

    Email address: admin@example.com

The final step is to enter your password. You will be asked to enter your
password twice, the second time as a confirmation of the first.

.. code-block:: text

    Password: **********
    Password (again): *********
    Superuser created successfully.

Start the development server
----------------------------

The Django admin site is activated by default. Let's start the development
server and explore it.

If the server is not running start it like so:

.. code-block:: console

    $ python manage.py runserver

Now, open a Web browser and go to "/admin/" on your local domain -- e.g.,
http://127.0.0.1:8000/admin/. You should see the admin's login screen:

.. image:: _images/admin01.png
   :alt: Django admin login screen

Since :doc:`translation </topics/i18n/translation>` is turned on by default,
the login screen may be displayed in your own language, depending on your
browser's settings and if Django has a translation for this language.

Enter the admin site
--------------------

Now, try logging in with the superuser account you created in the previous step.
You should see the Django admin index page:

.. image:: _images/admin02.png
   :alt: Django admin index page

You should see a few types of editable content: groups and users. They are
provided by :mod:`django.contrib.auth`, the authentication framework shipped
by Django.

Make the poll app modifiable in the admin
-----------------------------------------

But where's our poll app? It's not displayed on the admin index page.

Just one thing to do: we need to tell the admin that ``Question``
objects have an admin interface. To do this, open the :file:`polls/admin.py`
file, and edit it to look like this:

.. snippet::
    :filename: polls/admin.py

    from django.contrib import admin

    from .models import Question

    admin.site.register(Question)

Explore the free admin functionality
------------------------------------

Now that we've registered ``Question``, Django knows that it should be displayed on
the admin index page:

.. image:: _images/admin03t.png
   :alt: Django admin index page, now with polls displayed

Click "Questions". Now you're at the "change list" page for questions. This page
displays all the questions in the database and lets you choose one to change it.
There's the "What's up?" question we created earlier:

.. image:: _images/admin04t.png
   :alt: Polls change list page

Click the "What's up?" question to edit it:

.. image:: _images/admin05t.png
   :alt: Editing form for question object

Things to note here:

* The form is automatically generated from the ``Question`` model.

* The different model field types (:class:`~django.db.models.DateTimeField`,
  :class:`~django.db.models.CharField`) correspond to the appropriate HTML
  input widget. Each type of field knows how to display itself in the Django
  admin.

* Each :class:`~django.db.models.DateTimeField` gets free JavaScript
  shortcuts. Dates get a "Today" shortcut and calendar popup, and times get
  a "Now" shortcut and a convenient popup that lists commonly entered times.

The bottom part of the page gives you a couple of options:

* Save -- Saves changes and returns to the change-list page for this type of
  object.

* Save and continue editing -- Saves changes and reloads the admin page for
  this object.

* Save and add another -- Saves changes and loads a new, blank form for this
  type of object.

* Delete -- Displays a delete confirmation page.

If the value of "Date published" doesn't match the time when you created the
question in :doc:`Tutorial 1</intro/tutorial01>`, it probably
means you forgot to set the correct value for the :setting:`TIME_ZONE` setting.
Change it, reload the page and check that the correct value appears.

Change the "Date published" by clicking the "Today" and "Now" shortcuts. Then
click "Save and continue editing." Then click "History" in the upper right.
You'll see a page listing all changes made to this object via the Django admin,
with the timestamp and username of the person who made the change:

.. image:: _images/admin06t.png
   :alt: History page for question object

When you're comfortable with the models API and have familiarized yourself with
the admin site, read :doc:`part 3 of this tutorial</intro/tutorial03>` to learn
about how to add more views to our polls app.
