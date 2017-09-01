===================
应用(Applications)
===================

.. module:: django.apps

Django 包含一个由已安装应用组成的注册表，它保存应用的配置并提供自省。它还维护一个可用 :doc:`models </topics/db/models>` 的列表。

这个注册表叫做 :mod:`django.apps` ，位于 :attr:`~django.apps.apps` 中::

    >>> from django.apps import apps
    >>> apps.get_app_config('admin').verbose_name
    'Admin'

项目和应用
===========
历史上，Django 使用 **项目** 这个词来描述Django的一个web应用。现在项目主要通过一个设置模块定义。

例如，当你运行 ``django-admin startproject mysite`` 时，将会创建一个 ``mysite`` 的项目目录，目录包含一个名为 ``mysite`` 的Python
包，包含文件 ``settings.py`` 、 ``urls.py`` 和 ``wsgi.py``。 项目包通常被扩展成一个包括fixture、CSS和模板的特定应用程序模板。

项目的 **根目录** (包含 ``manage.py`` 的目录)通常是项目里所有应用程序的容器，这些应用程序并不是单独存在的。

**应用** 这个词表示一个Python包，它提供某些功能的集合。应用可以在各种项目中 :doc:`重用 </intro/reusable-apps/>`。

应用包含模型、视图、模板、模板标签、静态文件、URL、中间件等等。它们一般通过 :setting:`INSTALLED_APPS` 设置和
其它例如URLconf、:setting:`MIDDLEWARE` 设置以及模板继承等机制接入到项目中。

Django 的应用只是一个代码集，它与框架的其它部分进行交互，理解这点很重要。并没有应用程序对象之类的东西。
然而，有一些地方Django 需要与安装的应用交互，主要用于配置和自省。这是为什么应用注册表要在 :class:`~django.apps.AppConfig`
实例中维持每个安装的应用的元数据。

也没有规定项目不能被认为是包含模型的应用。（这就需要将其添加到 :setting:`INSTALLED_APPS` 中）

.. _configuring-applications-ref:

配置应用
=========

配置应用首先要创建 :class:`~django.apps.AppConfig` 的子类，并且将子类路径加入 :setting:`INSTALLED_APPS`。

当 :setting:`INSTALLED_APPS` 包含某个应用模块的路径后，Django 将在这个模块中检查一个 ``default_app_config`` 变量。

如果这个变量有定义，它应该是这个应用的 :class:`~django.apps.AppConfig` 子类的路径。

如果没有 ``default_app_config``, Django将使用 :class:`~django.apps.AppConfig` 类。

新的应用最好不要使用 ``default_app_config``. 应该在 :setting:`INSTALLED_APPS` 中显式地配置适当的 :class:`~django.apps.AppConfig` 子类的路径。

对于应用开发者
-----------------------

如果你正在创建一个名叫“Rock ’n’ roll”的可插式应用，下面展示了如何给admin提供一个合适的名字::

    # rock_n_roll/apps.py

    from django.apps import AppConfig

    class RockNRollConfig(AppConfig):
        name = 'rock_n_roll'
        verbose_name = "Rock ’n’ roll"


可以让你的应用像下面一样以默认方式加载这个 :class:`~django.apps.AppConfig` 的子类::

    # rock_n_roll/__init__.py

    default_app_config = 'rock_n_roll.apps.RockNRollConfig'

这将使得 :setting:`INSTALLED_APPS` 只包含 ``'rock_n_roll'`` 时将使用 ``RockNRollConfig``
这允许你使用 :class:`~django.apps.AppConfig` 功能而不用要求用户更新他们的 :setting:`INSTALLED_APPS` 设置

当然，你也可以将 ``'rock_n_roll.apps.RockNRollConfig'`` 放在 :setting:`INSTALLED_APPS` 设置中。
你甚至可以提供几个具有不同行为的 :class:`~django.apps.AppConfig` 子类，并让使用者通过他们的
:setting:`INSTALLED_APPS` 设置选择。

建议的做法是将配置类放在应用的 ``apps`` 子模块中。但是，Django 不强制这一点。

你必须包含 :attr:`~django.apps.AppConfig.name` 属性来让Django 决定该配置适用的应用。
你可以定义 :class:`~django.apps.AppConfig` API 参考中记录的任何属性。

.. note::

    如果你的代码在应用的 ``__init__.py`` 中导入应用程序注册表，
    名称 ``apps`` 将与 ``pps`` 子模块发生冲突。最佳做法是将这些代码移到子模块，并将其导入。
    一种解决方法是以一个不同的名称导入注册表::

        from django.apps import apps as django_apps

For application users
---------------------

If you're using "Rock ’n’ roll" in a project called ``anthology``, but you
want it to show up as "Jazz Manouche" instead, you can provide your own
configuration::

    # anthology/apps.py

    from rock_n_roll.apps import RockNRollConfig

    class JazzManoucheConfig(RockNRollConfig):
        verbose_name = "Jazz Manouche"

    # anthology/settings.py

    INSTALLED_APPS = [
        'anthology.apps.JazzManoucheConfig',
        # ...
    ]

Again, defining project-specific configuration classes in a submodule called
``apps`` is a convention, not a requirement.

Application configuration
=========================

.. class:: AppConfig

    Application configuration objects store metadata for an application. Some
    attributes can be configured in :class:`~django.apps.AppConfig`
    subclasses. Others are set by Django and read-only.

Configurable attributes
-----------------------

.. attribute:: AppConfig.name

    Full Python path to the application, e.g. ``'django.contrib.admin'``.

    This attribute defines which application the configuration applies to. It
    must be set in all :class:`~django.apps.AppConfig` subclasses.

    It must be unique across a Django project.

.. attribute:: AppConfig.label

    Short name for the application, e.g. ``'admin'``

    This attribute allows relabeling an application when two applications
    have conflicting labels. It defaults to the last component of ``name``.
    It should be a valid Python identifier.

    It must be unique across a Django project.

.. attribute:: AppConfig.verbose_name

    Human-readable name for the application, e.g. "Administration".

    This attribute defaults to ``label.title()``.

.. attribute:: AppConfig.path

    Filesystem path to the application directory, e.g.
    ``'/usr/lib/python3.4/dist-packages/django/contrib/admin'``.

    In most cases, Django can automatically detect and set this, but you can
    also provide an explicit override as a class attribute on your
    :class:`~django.apps.AppConfig` subclass. In a few situations this is
    required; for instance if the app package is a `namespace package`_ with
    multiple paths.

Read-only attributes
--------------------

.. attribute:: AppConfig.module

    Root module for the application, e.g. ``<module 'django.contrib.admin' from
    'django/contrib/admin/__init__.pyc'>``.

.. attribute:: AppConfig.models_module

    Module containing the models, e.g. ``<module 'django.contrib.admin.models'
    from 'django/contrib/admin/models.pyc'>``.

    It may be ``None`` if the application doesn't contain a ``models`` module.
    Note that the database related signals such as
    :data:`~django.db.models.signals.pre_migrate` and
    :data:`~django.db.models.signals.post_migrate`
    are only emitted for applications that have a ``models`` module.

Methods
-------

.. method:: AppConfig.get_models()

    Returns an iterable of :class:`~django.db.models.Model` classes for this
    application.

.. method:: AppConfig.get_model(model_name)

    Returns the :class:`~django.db.models.Model` with the given
    ``model_name``. Raises :exc:`LookupError` if no such model exists in this
    application. ``model_name`` is case-insensitive.

.. method:: AppConfig.ready()

    Subclasses can override this method to perform initialization tasks such
    as registering signals. It is called as soon as the registry is fully
    populated.

    Although you can't import models at the module-level where
    :class:`~django.apps.AppConfig` classes are defined, you can import them in
    ``ready()``, using either an ``import`` statement or
    :meth:`~AppConfig.get_model`.

    If you're registering :mod:`model signals <django.db.models.signals>`, you
    can refer to the sender by its string label instead of using the model
    class itself.

    Example::

        from django.db.models.signals import pre_save

        def ready(self):
            # importing model classes
            from .models import MyModel  # or...
            MyModel = self.get_model('MyModel')

            # registering signals with the model's string label
            pre_save.connect(receiver, sender='app_label.MyModel')

    .. warning::

        Although you can access model classes as described above, avoid
        interacting with the database in your :meth:`ready()` implementation.
        This includes model methods that execute queries
        (:meth:`~django.db.models.Model.save()`,
        :meth:`~django.db.models.Model.delete()`, manager methods etc.), and
        also raw SQL queries via ``django.db.connection``. Your
        :meth:`ready()` method will run during startup of every management
        command. For example, even though the test database configuration is
        separate from the production settings, ``manage.py test`` would still
        execute some queries against your **production** database!

    .. note::

        In the usual initialization process, the ``ready`` method is only called
        once by Django. But in some corner cases, particularly in tests which
        are fiddling with installed applications, ``ready`` might be called more
        than once. In that case, either write idempotent methods, or put a flag
        on your ``AppConfig`` classes to prevent re-running code which should
        be executed exactly one time.

.. _namespace package:

Namespace packages as apps (Python 3.3+)
----------------------------------------

Python versions 3.3 and later support Python packages without an
``__init__.py`` file. These packages are known as "namespace packages" and may
be spread across multiple directories at different locations on ``sys.path``
(see :pep:`420`).

Django applications require a single base filesystem path where Django
(depending on configuration) will search for templates, static assets,
etc. Thus, namespace packages may only be Django applications if one of the
following is true:

1. The namespace package actually has only a single location (i.e. is not
   spread across more than one directory.)

2. The :class:`~django.apps.AppConfig` class used to configure the application
   has a :attr:`~django.apps.AppConfig.path` class attribute, which is the
   absolute directory path Django will use as the single base path for the
   application.

If neither of these conditions is met, Django will raise
:exc:`~django.core.exceptions.ImproperlyConfigured`.

Application registry
====================

.. data:: apps

    The application registry provides the following public API. Methods that
    aren't listed below are considered private and may change without notice.

.. attribute:: apps.ready

    Boolean attribute that is set to ``True`` after the registry is fully
    populated and all :meth:`AppConfig.ready` methods are called.

.. method:: apps.get_app_configs()

    Returns an iterable of :class:`~django.apps.AppConfig` instances.

.. method:: apps.get_app_config(app_label)

    Returns an :class:`~django.apps.AppConfig` for the application with the
    given ``app_label``. Raises :exc:`LookupError` if no such application
    exists.

.. method:: apps.is_installed(app_name)

    Checks whether an application with the given name exists in the registry.
    ``app_name`` is the full name of the app, e.g. ``'django.contrib.admin'``.

.. method:: apps.get_model(app_label, model_name)

    Returns the :class:`~django.db.models.Model` with the given ``app_label``
    and ``model_name``. As a shortcut, this method also accepts a single
    argument in the form ``app_label.model_name``. ``model_name`` is
    case-insensitive.

    Raises :exc:`LookupError` if no such application or model exists. Raises
    :exc:`ValueError` when called with a single argument that doesn't contain
    exactly one dot.

.. _app-loading-process:

Initialization process
======================

How applications are loaded
---------------------------

When Django starts, :func:`django.setup()` is responsible for populating the
application registry.

.. currentmodule:: django

.. function:: setup(set_prefix=True)

    Configures Django by:

    * Loading the settings.
    * Setting up logging.
    * If ``set_prefix`` is True, setting the URL resolver script prefix to
      :setting:`FORCE_SCRIPT_NAME` if defined, or ``/`` otherwise.
    * Initializing the application registry.

    .. versionchanged:: 1.10

        The ability to set the URL resolver script prefix is new.

    This function is called automatically:

    * When running an HTTP server via Django's WSGI support.
    * When invoking a management command.

    It must be called explicitly in other cases, for instance in plain Python
    scripts.

.. currentmodule:: django.apps

The application registry is initialized in three stages. At each stage, Django
processes all applications in the order of :setting:`INSTALLED_APPS`.

#. First Django imports each item in :setting:`INSTALLED_APPS`.

   If it's an application configuration class, Django imports the root package
   of the application, defined by its :attr:`~AppConfig.name` attribute. If
   it's a Python package, Django creates a default application configuration.

   *At this stage, your code shouldn't import any models!*

   In other words, your applications' root packages and the modules that
   define your application configuration classes shouldn't import any models,
   even indirectly.

   Strictly speaking, Django allows importing models once their application
   configuration is loaded. However, in order to avoid needless constraints on
   the order of :setting:`INSTALLED_APPS`, it's strongly recommended not
   import any models at this stage.

   Once this stage completes, APIs that operate on application configurations
   such as :meth:`~apps.get_app_config()` become usable.

#. Then Django attempts to import the ``models`` submodule of each application,
   if there is one.

   You must define or import all models in your application's ``models.py`` or
   ``models/__init__.py``. Otherwise, the application registry may not be fully
   populated at this point, which could cause the ORM to malfunction.

   Once this stage completes, APIs that operate on models such as
   :meth:`~apps.get_model()` become usable.

#. Finally Django runs the :meth:`~AppConfig.ready()` method of each application
   configuration.

.. _applications-troubleshooting:

Troubleshooting
---------------

Here are some common problems that you may encounter during initialization:

* :class:`~django.core.exceptions.AppRegistryNotReady`: This happens when
  importing an application configuration or a models module triggers code that
  depends on the app registry.

  For example, :func:`~django.utils.translation.ugettext()` uses the app
  registry to look up translation catalogs in applications. To translate at
  import time, you need :func:`~django.utils.translation.ugettext_lazy()`
  instead. (Using :func:`~django.utils.translation.ugettext()` would be a bug,
  because the translation would happen at import time, rather than at each
  request depending on the active language.)

  Executing database queries with the ORM at import time in models modules
  will also trigger this exception. The ORM cannot function properly until all
  models are available.

  Another common culprit is :func:`django.contrib.auth.get_user_model()`. Use
  the :setting:`AUTH_USER_MODEL` setting to reference the User model at import
  time.

  This exception also happens if you forget to call :func:`django.setup()` in
  a standalone Python script.

* ``ImportError: cannot import name ...`` This happens if the import sequence
  ends up in a loop.

  To eliminate such problems, you should minimize dependencies between your
  models modules and do as little work as possible at import time. To avoid
  executing code at import time, you can move it into a function and cache its
  results. The code will be executed when you first need its results. This
  concept is known as "lazy evaluation".

* ``django.contrib.admin`` automatically performs autodiscovery of ``admin``
  modules in installed applications. To prevent it, change your
  :setting:`INSTALLED_APPS` to contain
  ``'django.contrib.admin.apps.SimpleAdminConfig'`` instead of
  ``'django.contrib.admin'``.
