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

数据库路由器接收到的hints用来决定哪个数据库接收给的的请求.

目前, 唯一提供hint的是 ``instance``, 它是一个关联的正在进行读或写操作的对象实例.
它可以是是正在保存的实例, 或是正在添加多对多关系的实例.
在某些情况下不会提供实例hint. 路由其检查是否存在实例hint并确定其是否应该用来改变路由行为.

使用路由
-------------

使用 :setting:`DATABASE_ROUTERS` 配置启用数据库路由. 该配置定义了一组类名组成的列表,
其中每个类表示一个路由, 它们将被主路由(``django.db.router``)使用.

Django使用主路由来分配数据库操作使用的数据库. 当一个查询需要知道使用哪一个数据库时, 它将调用主路由并提供一个模型和一个Hint(可选).
随后Django依次尝试每个路由直至找到数据库. 如果没有找到, 它将尝试访问当前Hint实例的 ``_state.db``. 如果没有提供Hint实例, 或者该实例当前没有数据库state, 主路由将分配 ``default`` 数据库.

示例
----------

.. admonition:: 仅供参考!

    这个例子的目的是演示如何使用路由这个基本结构来决定使用的数据库. 它有意忽略一些复杂的问题, 目的是为了演示如何使用路由.

    如果 ``myapp`` 中的有任何模型包含与 ``其他`` 数据库之外的模型的关联关系, 这个例子将无法正常工作. :ref:`跨数据库关联关系 <no_cross_database_relations>` 介绍了Django目前无法解决的引用完整性问题.

    主/副(在某些数据库中叫做master/slave)配置也是有问题的 —— 它没有提供任何处理Replication滞后的解决办法(例如, 因为写入再同步到replica需要一定的时间, 这会导致查询的不一致). 它也没有考虑到事务与数据库利用率策略的交互.

所以 —— 在实际应用中这意味着什么? 我们考虑一个简单的配置例子. 该配置有几个数据库: 一个用于 ``auth`` 应用, 和其它应用使用一个带有两个只读副本的 主/副 配置. 以下是具体设置::

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

现在我们需要处理路由. 首先, 我们创建一个路由, 使它将 ``auth`` 应用的查询发送到 ``auth_db``::

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

然后再创建一个路由, 它将其他应用发送到 主/副 配置, 并且随机选择一个副本进行读操作::

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

最后, 在配置文件中添加下面的代码(用定义路由模块的实际Python路径替换 ``path.to.``)::

    DATABASE_ROUTERS = ['path.to.AuthRouter', 'path.to.PrimaryReplicaRouter']

路由配置的顺序非常重要. 路由将按照 :setting:`DATABASE_ROUTERS` 中设置的顺序进行查询.
在这个例子中, ``AuthRouter`` 将在 ``PrimaryReplicaRouter`` 之前处理, ``auth`` 处理在其它模型之前.
如果 :setting:`DATABASE_ROUTERS` 设置中按其他顺序列出这两个路由, ``PrimaryReplicaRouter.allow_migrate()`` 将优先处理.
PrimaryReplicaRouter中实现的捕获所有的查询, 这意味着所有的模型可以位于所有的数据库中.

设置好这个配置后, 让我们运行一些Django代码::

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

此示例定义一个路由来处理与 ``auth`` 应用的模型交互, 以及其他路由与所有其他应用程序的交互.
如果您将 ``default`` 数据库留空, 并且不想定义一个共用的数据库来处理所有未指定的应用, 那么则路由器必须在迁移之前处理 :setting:`INSTALLED_APPS` 的所有应用名.
有关contrib应用程序的行为, 请参考 :ref:`contrib_app_multiple_databases`.

手动指定数据库
=============================

Django还提供了了一个API用来在代码中控制数据库的使用. 手动指定的数据库的优先级高于路由分配的数据库.

指定 ``QuerySet`` 数据库
------------------------------------------------

可以在 ``QuerySet`` 查询链路上的任意点为 ``QuerySet`` 指定数据库.
调用 ``QuerySet`` 的 ``using()`` 方法指定数据库.

``using()`` 接受一个参数: 将要指定使用的数据库别名. 比如::

    >>> # This will run on the 'default' database.
    >>> Author.objects.all()

    >>> # So will this.
    >>> Author.objects.using('default').all()

    >>> # This will run on the 'other' database.
    >>> Author.objects.using('other').all()

指定 ``save()`` 数据库
-----------------------------------

``Model.save()`` 使用 ``using`` 参数指定将要保存的数据库.

例如将数据保存至 ``legacy_users`` 数据库::

    >>> my_object.save(using='legacy_users')

如果没有指定 ``using`` 参数, ``save()`` 方法将保存到路由分配的默认数据库中.

将对象从数据库移至另一个数据库
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果你已经将实例保存到数据库中, 有可能想使用 ``save(using=...)`` 来将该实例迁移到一个新的数据库中.
但是如果你没有使用正确的操作, 这可能会导致意想不到的结果.

考虑下面的例子::

    >>> p = Person(name='Fred')
    >>> p.save(using='first')  # (statement 1)
    >>> p.save(using='second') # (statement 2)

在statement 1中, 将一个新的 ``Person`` 对象保存到 ``first`` 数据库中. 此时 ``p`` 还没有主键, 所以Django发出一个SQL ``INSERT`` 语句. 这会创建一个主键且赋值给 ``p``.

当在statement 2中保存时, 此时 ``p`` 已经具有主键, Django将尝试在新的数据库上使用该主键. 如果该主键值在 ``second`` 数据库中不存在, 那么就不会有问题, 该对象将正常被复制到新的数据库中.

但是, 如果 ``p`` 的主键值已经存在于 ``second`` 数据库中, 已经存在的对象将在 ``p`` 保存时被覆盖.

可以用两种方法避免这种情况. 首先, 你可以清除实例的主键. 如果保存的对象没有主键, Django将把它当做一个新的对象, 这样可以避免 ``second`` 数据库上数据丢失的问题::

    >>> p = Person(name='Fred')
    >>> p.save(using='first')
    >>> p.pk = None # Clear the primary key.
    >>> p.save(using='second') # Write a completely new object.

第二个方式是调用 ``save()`` 方法时, 使用可选参数 ``force_insert`` 确保Django 执行 ``INSERT`` SQL::

    >>> p = Person(name='Fred')
    >>> p.save(using='first')
    >>> p.save(using='second', force_insert=True)

这可以确保 ``Fred`` 在两个数据库上拥有同样的主键. 当在 ``second`` 上保存时, 如果主键已经存在那将会引发一个异常.

指定删除的数据库
-----------------------------------

默认情况下, 删除一个已存在对象的调用将在获取对象时使用的数据库上执行::

    >>> u = User.objects.using('legacy_users').get(username='fred')
    >>> u.delete() # 将从 `legacy_users` 数据库删除

``Model.delete()`` 方法使用关键字参数 ``using`` 来指定从哪个数据库删除数据. 该参数的工作方式和 ``save()`` 方法的 ``using`` 参数一样.

例如, 将用户数据从 ``legacy_users`` 数据库迁移至 ``new_users`` 数据库::

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
