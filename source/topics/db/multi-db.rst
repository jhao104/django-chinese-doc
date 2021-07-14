==================
多数据库
==================

本文档主旨是描述Django对多数据库的支持. 大部分Django文档假设你只和一个数据库交互. 如果你想与多个数据库交互则需要执行一些额外的步骤.

定义数据库
=======================

要在Django使用多个数据库, 首先要在 :setting:`DATABASES` 设置中配置数据库连接信息.
此设置将数据库别名(在Django中引用指定数据库的一种方式)映射到该连接信息的字典.
字典内容中的设置项在 :setting:`DATABASES` 中有详细的描述.

可以为Databases设置除了 ``default`` 外的任何别名. ``default`` 具有特殊含义, 在没有选择指定数据库时Django会使用其作为默认数据库.

下面展示了一段 ``settings.py`` 代码片段, 它定义了两个数据库 -- 一个PostgreSQL的默认数据库和一个名为 ``users`` 的MySQL数据库::

    DATABASES = {
        'default': {
            'NAME': 'app_data',
            'ENGINE': 'django.db.backends.postgresql',
            'USER': 'postgres_user',
            'PASSWORD': 's3krit'
        },
        'users': {
            'NAME': 'user_data',
            'ENGINE': 'django.db.backends.mysql',
            'USER': 'mysql_user',
            'PASSWORD': 'priv4te'
        }
    }

如果在你项目中不会使用到 ``default`` 数据库, 特别注意一定要指定想要使用的数据库.
Django要求必须设置 ``default`` 数据库, 就算不使用 ``default`` 数据库也必须将其设置为空字典.
这样做的情况下就必须为所有应用的模型(包括contrib和第三方应用)设置 :setting:`DATABASE_ROUTERS`,
以保证不再引用到默认数据库. 下面是 ``settings.py`` 的一段示例代码, 它定义了一个空的 ``default`` 和另外两个数据库::

    DATABASES = {
        'default': {},
        'users': {
            'NAME': 'user_data',
            'ENGINE': 'django.db.backends.mysql',
            'USER': 'mysql_user',
            'PASSWORD': 'superS3cret'
        },
        'customers': {
            'NAME': 'customer_data',
            'ENGINE': 'django.db.backends.mysql',
            'USER': 'mysql_cust',
            'PASSWORD': 'veryPriv@ate'
        }
    }

在访问 :setting:`DATABASES` 中不存在的数据库时, Django会引发一个 ``django.db.utils.ConnectionDoesNotExist`` 异常.

同步数据库
============================

管理命令 :djadmin:`migrate` 一次只能操作一个数据库.
默认情况下它在 ``default`` 数据库上操作, 可以使用 :option:`--database <migrate --database>` 选项来同步其他指定的数据库,
因此如果在上面例子中要在所有数据库上同步模型, 需要执行两次::

    $ ./manage.py migrate
    $ ./manage.py migrate --database=users

如果你不希望每个应用程序都同步到特定的数据库上, 那么可以定义一个 :ref:`数据库路由<topics-db-multi-db-routing>`, 该路由实现一个约束特定模型可用性的策略.

如果在上面的第二个例子中, ``default`` 数据库被设置为空, 这时就必须在执行 :djadmin:`migrate` 命令时指定数据库别名.
否则会报错, 所以对于第二个例子::

    $ ./manage.py migrate --database=users
    $ ./manage.py migrate --database=customers

使用其他管理命令
-------------------------------

大部分的 ``django-admin`` 命令都和 :djadmin:`migrate` 一样 -- 一次只能操作一个数据库, 使用 ``--database`` 选项来指定要控制的数据库.

:djadmin:`makemigrations` 命令是个例外. 它会在创建新迁移之前验证数据库中的历史迁移记录,以便发现当前迁移文件之中的问题(可能是由于修改所产生的).
默认情况下它只会检查 ``default`` 数据库, 但是如果有的话它也会参考 :ref:`routers
<topics-db-multi-db-routing>` 中的 :meth:`allow_migrate` 方法.

.. versionchanged:: 1.10

    添加了迁移一致性检查. 1.10.1中添加了基于数据库routers的检查.

.. _topics-db-multi-db-routing:

自动数据库路由
==========================

使用多数据库最简单的方式就是设置数据库路由.
默认的路由设置会确保对象对原数据库保持粘性(比如, 从 ``foo`` 数据库检索到的对象将被保存到同一个数据库).
在默认路由模式下, 如果没有指定数据库所有查询都以 ``default`` 数据库为对象.

默认数据库路由是自动启用的, 如果想实现更多高级的数据库分配行为, 可以定义和应用自己的数据库路由.

数据库路由
----------------

数据库路由是一个类, 它提供了四个方法:

.. method:: db_for_read(model, **hints)

    指定 ``model`` 对象的读操作应该使用的数据库.

    如果数据库操作能够提供其它额外的信息可以帮助选择数据库, 它将在 ``hints`` 字典中列出.
    详细信息在 :ref:`下文 <topics-db-multi-db-hints>` 给出.

    如果不指定则返回 ``None``.

.. method:: db_for_write(model, **hints)

    指定 ``model`` 对象的写操作应该使用的数据库.

    如果数据库操作能够提供其它额外的信息可以帮助选择数据库, 它将在 ``hints`` 字典中列出.
    详细信息在 :ref:`下文 <topics-db-multi-db-hints>` 给出.

    如果不指定则返回 ``None``.

.. method:: allow_relation(obj1, obj2, **hints)

    如果允许 ``obj1`` 和 ``obj2`` 关联, 返回 ``True``. 如果要阻止关联则返回 ``False``.
    如果路由没法判断, 则返回 ``None``. 这是一个纯验证操作, 由外键和多对多操作使用它决定是否应该允许关联.

.. method:: allow_migrate(db, app_label, model_name=None, **hints)

    决定是否允许迁移操作在别名为 ``db`` 的数据库上运行. 如果允许运行则返回 ``True``,
    如果不应该运行则返回 ``False``, 如果路由无法判断则返回 ``None``.

    位置参数 ``app_label`` 是要迁移的应用的标签.

    大部分迁移操作的 ``model_name`` 为迁移模型的 ``model._meta.model_name`` 值(模型 ``__name__`` 的小写形式).
    :class:`~django.db.migrations.operations.RunPython` 和
    :class:`~django.db.migrations.operations.RunSQL` 操作的值为 ``None``, 除非使用了hints来提供.

    ``hints`` 用于某些操作来传递额外的信息给路由.

    当 ``model_name`` 有值时, ``hints`` 通常包含该模型 ``'model'`` 下的值.
    注意它可能是 :ref:`历史模型 <historical-models>`, 因此不会有自定的属性,方法和管理器.你应该只依赖 ``_meta``.

    这个方法还可以用来决定一个指定数据库上某个模型的可用性.

    :djadmin:`makemigrations` 会给修改的模型创建迁移, 但是如果 ``allow_migrate()`` 返回 ``False``, 那么
    ``model_name`` 在 ``db`` 上执行 :djadmin:`migrate` 时会被忽略.
    对于已经有迁移的模型, 修改 ``allow_migrate()`` 的行为可能为导致外键损坏, 表或者额外表,
    当 :djadmin:`makemigrations` 验证历史迁移记录时, 它会跳过不允许迁移的应用的数据库.

路由不用提供所有这些方法 —— 它可以省略一个或多个. 如果某个方法被省略, Django会在执行相关检查时候跳过这个路由.

.. _topics-db-multi-db-hints:

Hints
~~~~~

The hints received by the database router can be used to decide which
database should receive a given request.

At present, the only hint that will be provided is ``instance``, an
object instance that is related to the read or write operation that is
underway. This might be the instance that is being saved, or it might
be an instance that is being added in a many-to-many relation. In some
cases, no instance hint will be provided at all. The router checks for
the existence of an instance hint, and determine if that hint should be
used to alter routing behavior.

Using routers
-------------

Database routers are installed using the :setting:`DATABASE_ROUTERS`
setting. This setting defines a list of class names, each specifying a
router that should be used by the master router
(``django.db.router``).

The master router is used by Django's database operations to allocate
database usage. Whenever a query needs to know which database to use,
it calls the master router, providing a model and a hint (if
available). Django then tries each router in turn until a database
suggestion can be found. If no suggestion can be found, it tries the
current ``_state.db`` of the hint instance. If a hint instance wasn't
provided, or the instance doesn't currently have database state, the
master router will allocate the ``default`` database.

An example
----------

.. admonition:: Example purposes only!

    This example is intended as a demonstration of how the router
    infrastructure can be used to alter database usage. It
    intentionally ignores some complex issues in order to
    demonstrate how routers are used.

    This example won't work if any of the models in ``myapp`` contain
    relationships to models outside of the ``other`` database.
    :ref:`Cross-database relationships <no_cross_database_relations>`
    introduce referential integrity problems that Django can't
    currently handle.

    The primary/replica (referred to as master/slave by some databases)
    configuration described is also flawed -- it
    doesn't provide any solution for handling replication lag (i.e.,
    query inconsistencies introduced because of the time taken for a
    write to propagate to the replicas). It also doesn't consider the
    interaction of transactions with the database utilization strategy.

So - what does this mean in practice? Let's consider another sample
configuration. This one will have several databases: one for the
``auth`` application, and all other apps using a primary/replica setup
with two read replicas. Here are the settings specifying these
databases::

    DATABASES = {
        'default': {},
        'auth_db': {
            'NAME': 'auth_db',
            'ENGINE': 'django.db.backends.mysql',
            'USER': 'mysql_user',
            'PASSWORD': 'swordfish',
        },
        'primary': {
            'NAME': 'primary',
            'ENGINE': 'django.db.backends.mysql',
            'USER': 'mysql_user',
            'PASSWORD': 'spam',
        },
        'replica1': {
            'NAME': 'replica1',
            'ENGINE': 'django.db.backends.mysql',
            'USER': 'mysql_user',
            'PASSWORD': 'eggs',
        },
        'replica2': {
            'NAME': 'replica2',
            'ENGINE': 'django.db.backends.mysql',
            'USER': 'mysql_user',
            'PASSWORD': 'bacon',
        },
    }

Now we'll need to handle routing. First we want a router that knows to
send queries for the ``auth`` app to ``auth_db``::

    class AuthRouter(object):
        """
        A router to control all database operations on models in the
        auth application.
        """
        def db_for_read(self, model, **hints):
            """
            Attempts to read auth models go to auth_db.
            """
            if model._meta.app_label == 'auth':
                return 'auth_db'
            return None

        def db_for_write(self, model, **hints):
            """
            Attempts to write auth models go to auth_db.
            """
            if model._meta.app_label == 'auth':
                return 'auth_db'
            return None

        def allow_relation(self, obj1, obj2, **hints):
            """
            Allow relations if a model in the auth app is involved.
            """
            if obj1._meta.app_label == 'auth' or \
               obj2._meta.app_label == 'auth':
               return True
            return None

        def allow_migrate(self, db, app_label, model_name=None, **hints):
            """
            Make sure the auth app only appears in the 'auth_db'
            database.
            """
            if app_label == 'auth':
                return db == 'auth_db'
            return None

And we also want a router that sends all other apps to the
primary/replica configuration, and randomly chooses a replica to read
from::

    import random

    class PrimaryReplicaRouter(object):
        def db_for_read(self, model, **hints):
            """
            Reads go to a randomly-chosen replica.
            """
            return random.choice(['replica1', 'replica2'])

        def db_for_write(self, model, **hints):
            """
            Writes always go to primary.
            """
            return 'primary'

        def allow_relation(self, obj1, obj2, **hints):
            """
            Relations between objects are allowed if both objects are
            in the primary/replica pool.
            """
            db_list = ('primary', 'replica1', 'replica2')
            if obj1._state.db in db_list and obj2._state.db in db_list:
                return True
            return None

        def allow_migrate(self, db, app_label, model_name=None, **hints):
            """
            All non-auth models end up in this pool.
            """
            return True

Finally, in the settings file, we add the following (substituting
``path.to.`` with the actual Python path to the module(s) where the
routers are defined)::

    DATABASE_ROUTERS = ['path.to.AuthRouter', 'path.to.PrimaryReplicaRouter']

The order in which routers are processed is significant. Routers will
be queried in the order they are listed in the
:setting:`DATABASE_ROUTERS` setting. In this example, the
``AuthRouter`` is processed before the ``PrimaryReplicaRouter``, and as a
result, decisions concerning the models in ``auth`` are processed
before any other decision is made. If the :setting:`DATABASE_ROUTERS`
setting listed the two routers in the other order,
``PrimaryReplicaRouter.allow_migrate()`` would be processed first. The
catch-all nature of the PrimaryReplicaRouter implementation would mean
that all models would be available on all databases.

With this setup installed, lets run some Django code::

    >>> # This retrieval will be performed on the 'auth_db' database
    >>> fred = User.objects.get(username='fred')
    >>> fred.first_name = 'Frederick'

    >>> # This save will also be directed to 'auth_db'
    >>> fred.save()

    >>> # These retrieval will be randomly allocated to a replica database
    >>> dna = Person.objects.get(name='Douglas Adams')

    >>> # A new object has no database allocation when created
    >>> mh = Book(title='Mostly Harmless')

    >>> # This assignment will consult the router, and set mh onto
    >>> # the same database as the author object
    >>> mh.author = dna

    >>> # This save will force the 'mh' instance onto the primary database...
    >>> mh.save()

    >>> # ... but if we re-retrieve the object, it will come back on a replica
    >>> mh = Book.objects.get(title='Mostly Harmless')

This example defined a router to handle interaction with models from the
``auth`` app, and other routers to handle interaction with all other apps. If
you left your ``default`` database empty and don't want to define a catch-all
database router to handle all apps not otherwise specified, your routers must
handle the names of all apps in :setting:`INSTALLED_APPS` before you migrate.
See :ref:`contrib_app_multiple_databases` for information about contrib apps
that must be together in one database.

Manually selecting a database
=============================

Django also provides an API that allows you to maintain complete control
over database usage in your code. A manually specified database allocation
will take priority over a database allocated by a router.

Manually selecting a database for a ``QuerySet``
------------------------------------------------

You can select the database for a ``QuerySet`` at any point in the
``QuerySet`` "chain." Just call ``using()`` on the ``QuerySet`` to get
another ``QuerySet`` that uses the specified database.

``using()`` takes a single argument: the alias of the database on
which you want to run the query. For example::

    >>> # This will run on the 'default' database.
    >>> Author.objects.all()

    >>> # So will this.
    >>> Author.objects.using('default').all()

    >>> # This will run on the 'other' database.
    >>> Author.objects.using('other').all()

Selecting a database for ``save()``
-----------------------------------

Use the ``using`` keyword to ``Model.save()`` to specify to which
database the data should be saved.

For example, to save an object to the ``legacy_users`` database, you'd
use this::

    >>> my_object.save(using='legacy_users')

If you don't specify ``using``, the ``save()`` method will save into
the default database allocated by the routers.

Moving an object from one database to another
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you've saved an instance to one database, it might be tempting to
use ``save(using=...)`` as a way to migrate the instance to a new
database. However, if you don't take appropriate steps, this could
have some unexpected consequences.

Consider the following example::

    >>> p = Person(name='Fred')
    >>> p.save(using='first')  # (statement 1)
    >>> p.save(using='second') # (statement 2)

In statement 1, a new ``Person`` object is saved to the ``first``
database. At this time, ``p`` doesn't have a primary key, so Django
issues an SQL ``INSERT`` statement. This creates a primary key, and
Django assigns that primary key to ``p``.

When the save occurs in statement 2, ``p`` already has a primary key
value, and Django will attempt to use that primary key on the new
database. If the primary key value isn't in use in the ``second``
database, then you won't have any problems -- the object will be
copied to the new database.

However, if the primary key of ``p`` is already in use on the
``second`` database, the existing object in the ``second`` database
will be overridden when ``p`` is saved.

You can avoid this in two ways. First, you can clear the primary key
of the instance. If an object has no primary key, Django will treat it
as a new object, avoiding any loss of data on the ``second``
database::

    >>> p = Person(name='Fred')
    >>> p.save(using='first')
    >>> p.pk = None # Clear the primary key.
    >>> p.save(using='second') # Write a completely new object.

The second option is to use the ``force_insert`` option to ``save()``
to ensure that Django does an SQL ``INSERT``::

    >>> p = Person(name='Fred')
    >>> p.save(using='first')
    >>> p.save(using='second', force_insert=True)

This will ensure that the person named ``Fred`` will have the same
primary key on both databases. If that primary key is already in use
when you try to save onto the ``second`` database, an error will be
raised.

Selecting a database to delete from
-----------------------------------

By default, a call to delete an existing object will be executed on
the same database that was used to retrieve the object in the first
place::

    >>> u = User.objects.using('legacy_users').get(username='fred')
    >>> u.delete() # will delete from the `legacy_users` database

To specify the database from which a model will be deleted, pass a
``using`` keyword argument to the ``Model.delete()`` method. This
argument works just like the ``using`` keyword argument to ``save()``.

For example, if you're migrating a user from the ``legacy_users``
database to the ``new_users`` database, you might use these commands::

    >>> user_obj.save(using='new_users')
    >>> user_obj.delete(using='legacy_users')

Using managers with multiple databases
--------------------------------------

Use the ``db_manager()`` method on managers to give managers access to
a non-default database.

For example, say you have a custom manager method that touches the
database -- ``User.objects.create_user()``. Because ``create_user()``
is a manager method, not a ``QuerySet`` method, you can't do
``User.objects.using('new_users').create_user()``. (The
``create_user()`` method is only available on ``User.objects``, the
manager, not on ``QuerySet`` objects derived from the manager.) The
solution is to use ``db_manager()``, like this::

    User.objects.db_manager('new_users').create_user(...)

``db_manager()`` returns a copy of the manager bound to the database you specify.

Using ``get_queryset()`` with multiple databases
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you're overriding ``get_queryset()`` on your manager, be sure to
either call the method on the parent (using ``super()``) or do the
appropriate handling of the ``_db`` attribute on the manager (a string
containing the name of the database to use).

For example, if you want to return a custom ``QuerySet`` class from
the ``get_queryset`` method, you could do this::

    class MyManager(models.Manager):
        def get_queryset(self):
            qs = CustomQuerySet(self.model)
            if self._db is not None:
                qs = qs.using(self._db)
            return qs

Exposing multiple databases in Django's admin interface
=======================================================

Django's admin doesn't have any explicit support for multiple
databases. If you want to provide an admin interface for a model on a
database other than that specified by your router chain, you'll
need to write custom :class:`~django.contrib.admin.ModelAdmin` classes
that will direct the admin to use a specific database for content.

``ModelAdmin`` objects have five methods that require customization for
multiple-database support::

    class MultiDBModelAdmin(admin.ModelAdmin):
        # A handy constant for the name of the alternate database.
        using = 'other'

        def save_model(self, request, obj, form, change):
            # Tell Django to save objects to the 'other' database.
            obj.save(using=self.using)

        def delete_model(self, request, obj):
            # Tell Django to delete objects from the 'other' database
            obj.delete(using=self.using)

        def get_queryset(self, request):
            # Tell Django to look for objects on the 'other' database.
            return super(MultiDBModelAdmin, self).get_queryset(request).using(self.using)

        def formfield_for_foreignkey(self, db_field, request, **kwargs):
            # Tell Django to populate ForeignKey widgets using a query
            # on the 'other' database.
            return super(MultiDBModelAdmin, self).formfield_for_foreignkey(db_field, request, using=self.using, **kwargs)

        def formfield_for_manytomany(self, db_field, request, **kwargs):
            # Tell Django to populate ManyToMany widgets using a query
            # on the 'other' database.
            return super(MultiDBModelAdmin, self).formfield_for_manytomany(db_field, request, using=self.using, **kwargs)

The implementation provided here implements a multi-database strategy
where all objects of a given type are stored on a specific database
(e.g., all ``User`` objects are in the ``other`` database). If your
usage of multiple databases is more complex, your ``ModelAdmin`` will
need to reflect that strategy.

:class:`~django.contrib.admin.InlineModelAdmin` objects can be handled in a
similar fashion. They require three customized methods::

    class MultiDBTabularInline(admin.TabularInline):
        using = 'other'

        def get_queryset(self, request):
            # Tell Django to look for inline objects on the 'other' database.
            return super(MultiDBTabularInline, self).get_queryset(request).using(self.using)

        def formfield_for_foreignkey(self, db_field, request, **kwargs):
            # Tell Django to populate ForeignKey widgets using a query
            # on the 'other' database.
            return super(MultiDBTabularInline, self).formfield_for_foreignkey(db_field, request, using=self.using, **kwargs)

        def formfield_for_manytomany(self, db_field, request, **kwargs):
            # Tell Django to populate ManyToMany widgets using a query
            # on the 'other' database.
            return super(MultiDBTabularInline, self).formfield_for_manytomany(db_field, request, using=self.using, **kwargs)

Once you've written your model admin definitions, they can be
registered with any ``Admin`` instance::

    from django.contrib import admin

    # Specialize the multi-db admin objects for use with specific models.
    class BookInline(MultiDBTabularInline):
        model = Book

    class PublisherAdmin(MultiDBModelAdmin):
        inlines = [BookInline]

    admin.site.register(Author, MultiDBModelAdmin)
    admin.site.register(Publisher, PublisherAdmin)

    othersite = admin.AdminSite('othersite')
    othersite.register(Publisher, MultiDBModelAdmin)

This example sets up two admin sites. On the first site, the
``Author`` and ``Publisher`` objects are exposed; ``Publisher``
objects have an tabular inline showing books published by that
publisher. The second site exposes just publishers, without the
inlines.

Using raw cursors with multiple databases
=========================================

If you are using more than one database you can use
``django.db.connections`` to obtain the connection (and cursor) for a
specific database. ``django.db.connections`` is a dictionary-like
object that allows you to retrieve a specific connection using its
alias::

    from django.db import connections
    cursor = connections['my_db_alias'].cursor()

Limitations of multiple databases
=================================

.. _no_cross_database_relations:

Cross-database relations
------------------------

Django doesn't currently provide any support for foreign key or
many-to-many relationships spanning multiple databases. If you
have used a router to partition models to different databases,
any foreign key and many-to-many relationships defined by those
models must be internal to a single database.

This is because of referential integrity. In order to maintain a
relationship between two objects, Django needs to know that the
primary key of the related object is valid. If the primary key is
stored on a separate database, it's not possible to easily evaluate
the validity of a primary key.

If you're using Postgres, Oracle, or MySQL with InnoDB, this is
enforced at the database integrity level -- database level key
constraints prevent the creation of relations that can't be validated.

However, if you're using SQLite or MySQL with MyISAM tables, there is
no enforced referential integrity; as a result, you may be able to
'fake' cross database foreign keys. However, this configuration is not
officially supported by Django.

.. _contrib_app_multiple_databases:

Behavior of contrib apps
------------------------

Several contrib apps include models, and some apps depend on others. Since
cross-database relationships are impossible, this creates some restrictions on
how you can split these models across databases:

- each one of ``contenttypes.ContentType``, ``sessions.Session`` and
  ``sites.Site`` can be stored in any database, given a suitable router.
- ``auth`` models — ``User``, ``Group`` and ``Permission`` — are linked
  together and linked to ``ContentType``, so they must be stored in the same
  database as ``ContentType``.
- ``admin`` depends on ``auth``, so its models must be in the same database
  as ``auth``.
- ``flatpages`` and ``redirects`` depend on ``sites``, so their models must be
  in the same database as ``sites``.

In addition, some objects are automatically created just after
:djadmin:`migrate` creates a table to hold them in a database:

- a default ``Site``,
- a ``ContentType`` for each model (including those not stored in that
  database),
- three ``Permission`` for each model (including those not stored in that
  database).

For common setups with multiple databases, it isn't useful to have these
objects in more than one database. Common setups include primary/replica and
connecting to external databases. Therefore, it's recommended to write a
:ref:`database router<topics-db-multi-db-routing>` that allows synchronizing
these three models to only one database. Use the same approach for contrib
and third-party apps that don't need their tables in multiple databases.

.. warning::

    If you're synchronizing content types to more than one database, be aware
    that their primary keys may not match across databases. This may result in
    data corruption or data loss.
