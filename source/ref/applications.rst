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

对于应用使用者
---------------

如果你在 ``anthology`` 项目中使用"Rock ’n’ rol"，但您希望显示成 ``"Gypsy jazz"``，你可以修改你自己的配置︰::

    # anthology/apps.py

    from rock_n_roll.apps import RockNRollConfig

    class JazzManoucheConfig(RockNRollConfig):
        verbose_name = "Jazz Manouche"

    # anthology/settings.py

    INSTALLED_APPS = [
        'anthology.apps.JazzManoucheConfig',
        # ...
    ]

再说一次，在 ``apps`` 子模块定义特定于项目的配置类是一种习惯，并不强制要求。

Application 配置
=================

.. class:: AppConfig

    Application configuration对象存储应用程序的元数据。某些属性配置在 :class:`~django.apps.AppConfig` 子类中。
    其他由Django 设置且是只读的

可配置的属性
-------------

.. attribute:: AppConfig.name

    完整的Python路径, e.g. ``'django.contrib.admin'``.

    他的属性定义了该配置适用于哪个applications. 而且必须所有的 :class:`~django.apps.AppConfig` 的子类都要设置。

    在整个Django项目中必须是唯一的。

.. attribute:: AppConfig.label

    application的缩写名, e.g. ``'admin'``

    此属性可以重新标记应用，当两个应用程序有冲突的标签。它默认为 ``name`` 的最后一部分。它是一个有效的 Python 标识符。

    在整个Django项目中必须是唯一的。

.. attribute:: AppConfig.verbose_name

    应用的适合阅读的名称, e.g. "Administration".

    默认是 ``label.title()``.

.. attribute:: AppConfig.path

    应用目录的文件系统路径, e.g.
    ``'/usr/lib/python3.4/dist-packages/django/contrib/admin'``.

    在大多数情况下，Django 可以自动检测并设置它，你也可以在 :class:`~django.apps.AppConfig`
    子类上提供一个显式的类属性以覆盖它 。

    在有些情况下是需要这样的；例如，如果应用的包是一个具有多个路径的 `namespace package`_ 。

只读属性
---------

.. attribute:: AppConfig.module

    应用的根模块, e.g. ``<module 'django.contrib.admin' from
    'django/contrib/admin/__init__.pyc'>``.

.. attribute:: AppConfig.models_module

    包含 ``models`` 的模块, e.g. ``<module 'django.contrib.admin.models'
    from 'django/contrib/admin/models.pyc'>``.

    如果应用不包含 ``models`` 模块，它可能为 ``None``。
    注意与数据库相关的信号，如 :data:`~django.db.models.signals.pre_migrate` 和
    :data:`~django.db.models.signals.post_migrate` 只有在应用具有 ``models`` 模块时才发出。

方法
-----

.. method:: AppConfig.get_models()

    返回本应用的所有 :class:`~django.db.models.Model` 类的一个可迭代对象。

.. method:: AppConfig.get_model(model_name)

    根据 ``model_name`` 返回对应 :class:`~django.db.models.Model`.
    如果模型不存在，则抛出一个 :exc:`LookupError` 异常， ``model_name`` 不区分大小写。

.. method:: AppConfig.ready()

    子类可以重写此方法以执行初始化任务，如注册信号。它在注册表填充完之后立即调用。

    虽然不能在定义 :class:`~django.apps.AppConfig` 类的模块级别导入模型，
    但是可以使用 ``import`` 语句或 :meth:`~AppConfig.get_model` 将它们导入到 ``ready()`` 中。

    如果您注册的是 :mod:`model signals <django.db.models.signals>` ，
    您可以通过它的字符串标签来引用发送方，而不是使用模型类本身。

    例如::

        from django.db.models.signals import pre_save

        def ready(self):
            # importing model classes
            from .models import MyModel  # or...
            MyModel = self.get_model('MyModel')

            # registering signals with the model's string label
            pre_save.connect(receiver, sender='app_label.MyModel')

    .. warning::

        尽管可以像上面描述的那样访问模型类，但要避免在 :meth:`ready()` 中实现与数据库交互。
        包括执行查询( :meth:`~django.db.models.Model.save()` 、
        :meth:`~django.db.models.Model.delete()` 、manager方法等)的模型方法，
        以及通过 ``django.db.connection`` 的原始SQL查询。
        您的 :meth:`ready()` 方法将在任意管理命令启动时运行。例如:
        即使测试数据库配置与生产环境设置是分离的, ``manage.py test`` 仍会执行一些针对您的 **生产环境** 数据库的查询!

    .. note::

        在常规的初始化过程中，``ready`` 方法只由Django 调用一次。但在一些极端情况下，
        特别是在那些摆弄安装应用程序的测试中，``ready`` 可能被调用不止一次。在这种情况下，编写幂等方法，
        或者在 ``AppConfig`` 类上放置一个标识来防止应该执行一次的代码重复运行。

.. _namespace package:

命名空间包作为应用程序 (Python 3.3+)
------------------------------------

Python3.3以及更高版本支持不包含
``__init__.py`` 文件的Python包。 这些包分布在 ``sys.path``
(see :pep:`420`)上的不同位置的多个目录中。

Django应用程序需要一个单一的基本文件系统路径，其中Django(取决于配置)将搜索模板、静态资源。
因此，如果符合以下条件，则名称空间包可能只是Django应用程序。

1. 命名空间包实际上只有一个位置（即不会分布在多个目录中）。

2. 用于配置应用程序的 :class:`~django.apps.AppConfig` 类具有 :attr:`~django.apps.AppConfig.path` 类属性，这是Django将用作应用程序的唯一基本路径的绝对目录路径。

如果这些条件都不满足，Django将抛出 :exc:`~django.core.exceptions.ImproperlyConfigured` 异常。

应用注册表
==========

.. data:: apps

    应用程序注册中心提供以下公共API。以下未列出的方法被认为是私有的，可能会在没有通知的情况下发生变化。

.. attribute:: apps.ready

    布尔属性，在注册中心完全填充后或所有 :meth:`AppConfig.ready` 方法被调用后设置为 ``True``。

.. method:: apps.get_app_configs()

    返回一个所有 :class:`~django.apps.AppConfig` 的可迭代对象。

.. method:: apps.get_app_config(app_label)

    根据 ``app_label`` 返回对应的 :class:`~django.apps.AppConfig` ，如果没有对应的应用，则抛出一个 :exc:`LookupError` 异常。

.. method:: apps.is_installed(app_name)

    检查注册表中是否存在具有给定名称的应用。
    ``app_name`` 是应用的完整名称, e.g. ``'django.contrib.admin'``.

.. method:: apps.get_model(app_label, model_name)

    返回给定 ``app_label`` 和 ``model_name`` 对应的 :class:`~django.db.models.Model`
    作为快捷方式，此方法还接受 ``app_label.model_name`` 形式的一个单一参数。``model_name`` 不区分大小写。

    如果没有这种模型存在，抛出 :exc:`LookupError` 。使用不包含点号的单个参数调用时将引发 ``ValueError`` 异常。

.. _app-loading-process:

初始化过程
===========

如何加载应用
-------------

当Django 启动时，:func:`django.setup()` 负责填充应用注册表。

.. currentmodule:: django

.. function:: setup(set_prefix=True)

    配置Django:

    * 加载设置。
    * 设置日志。
    * 如果 ``set_prefix`` 为 True, 那么将URL解析器脚本前缀设置为 :setting:`FORCE_SCRIPT_NAME` (如果定义)，
      或者为 ``/``。

    * 初始化应用注册表。

    .. versionchanged:: 1.10

        设置URL解析器脚本前缀的功能是1.10中新加的。

    此函数在这些情况自动调用:

    * 当运行一个通过Django 的WSGI 支持 的HTTP 服务器。
    * 当调用管理命令。

    在其他情况下，必须显式地调用它，例如在普通的Python脚本中。

.. currentmodule:: django.apps

应用程序注册表初始化分为三个阶段。在每一个阶段，Django都按照 :setting:`INSTALLED_APPS` 中的顺序处理所有应用程序。

#. 首先Django先导入 :setting:`INSTALLED_APPS` 中的每个应用程序。

   如果它是一个应用配置类，Django 导入应用的root包，通过其 :attr:`~AppConfig.name` 属性。
   如果它是一个Python 包，Django 创建应用的一个默认配置。

   *在这个阶段，你的代码不应该将任何模型导入！*

   换句话说，您的应用程序的根包和定义您的应用程序配置类的模块不应该导入任何模型，甚至是间接的。

   严格地讲，Django 允许应用配置加载后导入模型。然而，为了避免 :setting:`INSTALLED_APPS` 的顺序带来不必要的约束，
   强烈推荐在这一阶段不导入任何模型

   这一阶段完成后，操作应用配置的API 开始变得可以使用了，例如 :meth:`~apps.get_app_config()`。

#. 然后 Django 将导入每个应用的models 子模块(如果存在的话)。

   你必须在应用的 ``models.py`` 或 ``models/__init__.py`` 中定义或导入所有模型。否则，
   应用注册表在此时可能不会完全填充，这可能导致ORM 出现故障

   一旦完成该步骤,  :meth:`~apps.get_model()` 之类的 model API 可以使用了。

#. 最后，Django 运行每个应用程序配置的 :meth:`~AppConfig.ready()` 方法。

.. _applications-troubleshooting:

故障排除
-----------

下面是在初始化过程中可能遇到的一些常见问题:

* :class:`~django.core.exceptions.AppRegistryNotReady`: 导致这种情况是由于导入应用程序配置或模型模块触发了依赖应用程序注册表的代码。

  例如, :func:`~django.utils.translation.ugettext()` 使用应用注册表来查找应用中的翻译目录。
  若要在导入时翻译，你需要 :func:`~django.utils.translation.ugettext_lazy()` (使用 :func:`~django.utils.translation.ugettext()` would be a bug,
  将是一个bug，因为翻译发生在导入的时候，而不是取决于每个请求的当前语言。)

  模型模块中在导入时使用ORM 执行数据库查询也会引发此异常。ORM 要直到所有的模型都可用时才能正常工作。

  另一个常见的罪魁祸首 :func:`django.contrib.auth.get_user_model()`。
  请在导入时使用 :setting:`AUTH_USER_MODEL` 设置来引用用户模型。

  如果在一个独立的 Python 脚本中你忘了调用 :func:`django.setup()` 也会发生此异常。

* ``ImportError: cannot import name ...`` 如果出现循环导入，就会反正这种情况。

  要消除这种问题，应尽量减少模型模块之间的依赖项，并在导入时尽可能少做工作。
  为了避免在导入时执行代码，你可以移动它到一个函数和缓存其结果。当你第一次需要其结果时，
  将执行该代码。这一概念被称为"惰性求值"。

* ``django.contrib.admin`` 在安装的应用中自动发现 ``admin``，
  要阻止它，请更改你的
  :setting:`INSTALLED_APPS` 以包含
  ``'django.contrib.admin.apps.SimpleAdminConfig'`` 而不是
  ``'django.contrib.admin'``.
