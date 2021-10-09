=======
信号
=======

Django发送的所有信号. 所有内置信号都是通过 :meth:`~django.dispatch.Signal.send` 方法发送.

.. seealso::

    有关如何注册和接收信号的信息, 请参见 :doc:`信号收发 </topics/signals>` 文档.

    :doc:`认证框架 </topics/auth/index>` 会发送 :ref:`用户登录和登出信号 <topics-auth-signals>`.

模型信号
=============

.. module:: django.db.models.signals
   :synopsis: Signals sent by the model system.

:mod:`django.db.models.signals` 模块定义了一系列由模型系统发送的信号.

.. warning::

    很多信号是通过模型方法如
    ``__init__()`` 或者 :meth:`~django.db.models.Model.save` 发送的, 你可以在代码中重写这一行为.

    如果你在模型中重写了这些方法, 则必须调用父类的方法才能发送这些信号.

    还需要注意的是, 信号处理程序在Django中默认为弱引用, 所以如果你的处理程序是一个本地函数, 它可能会被垃圾回收.
    为了防止这种情况, 在你调用信号的 :meth:`~django.dispatch.Signal.connect` 时, 传入 ``weak=False``.

.. note::

    连接接收器时通过指定其完整的应用标签, 延迟引用模型信号 ``sender`` 模型. 例如, ``polls`` 应用程序中定义的 ``Answer`` 模型可以通过 ``'polls.Answer'`` 引用.
    这在处理循环导入依赖和可交换模型时非常方便.

``pre_init``
------------

.. attribute:: django.db.models.signals.pre_init
   :module:

.. ^^^^^^^ this :module: hack keeps Sphinx from prepending the module.

每当实例化Django模型时, 该信号都会在模型的 ``__init__()`` 方法之前发出.

该信号发送的参数有:

``sender``
    刚刚创建模型实例的类.

``args``
    传入 ``__init__()`` 中的位置参数列表:

``kwargs``
    传入 ``__init__()`` 中的关键字参数的字典:

例如, 在 :doc:`教程 </intro/tutorial01>` 中有这样一行::

    p = Poll(question="What's up?", pub_date=datetime.now())

发送给 :data:`pre_init` 处理程序的参数是:

==========  ===============================================================
参数         值
==========  ===============================================================
``sender``  ``Poll`` (类本身)

``args``    ``[]`` (空列表, 因为 ``__init__()`` 中没有传入位置参数.)

``kwargs``  ``{'question': "What's up?", 'pub_date': datetime.now()}``
==========  ===============================================================

``post_init``
-------------

.. data:: django.db.models.signals.post_init
   :module:

类似pre_init, 在 ``__init__()`` 方法完成后发出的信号.

该信号发送的参数有:

``sender``
    同上: 刚刚创建模型实例的类.

``instance``
    刚刚创建的模型的实例.

``pre_save``
------------

.. data:: django.db.models.signals.pre_save
   :module:

在模型的 :meth:`~django.db.models.Model.save` 方法开始时发出的信号.

该信号发送的参数有:

``sender``
    模型类.

``instance``
    正被保存的模型实例.

``raw``
    布尔值; 如果模型完全按照显示的方式保存(即加载固定数据时)则为 ``True``. 不应查询和修改数据库中的其他记录, 因为数据库中可能尚未处于一致状态.

``using``
    使用的数据库别名.

``update_fields``
    传递给 :meth:`.Model.save` 方法的要更新的字段集,
    如果 ``save()`` 没有传递 ``update_fields`` 参数则为 ``None``.

``post_save``
-------------

.. data:: django.db.models.signals.post_save
   :module:

类似 :data:`pre_save`, 在模型
:meth:`~django.db.models.Model.save` 方法结束后发送的信号.

该信号发送的参数有:

``sender``
    模型类.

``instance``
    刚被保存的模型实例.

``created``
    布尔值; 创建了新记录则为 ``True``.

``raw``
    布尔值; 如果模型完全按照显示的方式保存(即加载固定数据时)则为 ``True``. 不应查询和修改数据库中的其他记录, 因为数据库中可能尚未处于一致状态.

``using``
    使用的数据库别名.

``update_fields``
    传递给 :meth:`.Model.save` 方法的要更新的字段集,
    如果 ``save()`` 没有传递 ``update_fields`` 参数则为 ``None``.

``pre_delete``
--------------

.. data:: django.db.models.signals.pre_delete
   :module:

在模型的 :meth:`~django.db.models.Model.delete` 方法 和
查询集的 :meth:`~django.db.models.query.QuerySet.delete` 方法开始时发送的信号.

该信号发送的参数有:

``sender``
    模型类.

``instance``
    正被删除的模型实例.

``using``
    使用的数据库别名.

``post_delete``
---------------

.. data:: django.db.models.signals.post_delete
   :module:

类似 :data:`pre_delete`, 在模型的
:meth:`~django.db.models.Model.delete` 方法和查询集的
:meth:`~django.db.models.query.QuerySet.delete` 方法结束后发送的信号.

该信号发送的参数有:

``sender``
    模型类.

``instance``
    被删除的模型实例.

    注意,该对象已经不存在于数据库中, 处理这个实例时要小心这一点.

``using``
    使用的数据库别名.

``m2m_changed``
---------------

.. data:: django.db.models.signals.m2m_changed
   :module:

模型实例的 :class:`~django.db.models.ManyToManyField` 改变时发出的信号.
严格的讲, 它不是一个模型信号, 因为它是由
:class:`~django.db.models.ManyToManyField` 发出, 但是由于当涉及到跟踪模型的变化时, 它是对
:data:`pre_save`/:data:`post_save` 和 :data:`pre_delete`/:data:`post_delete` 的补充,
所以它被放到了这里.

该信号发送的参数有:

``sender``
    用来表示 :class:`~django.db.models.ManyToManyField` 的中间类. 在定义了多对多关系时会自动创建这个类;
    可以通过多对多字段的 ``through`` 属性来访问它.

``instance``
    更新了多对多关系的实例. 它可以是 ``sender`` 实例, 或者
    :class:`~django.db.models.ManyToManyField` 关联的类的实例.

``action``
    表示更新类型的字符串. 可以是以下类型之一:

    ``"pre_add"``
        在添加一个或多个对象关联 **之前** 发送.
    ``"post_add"``
        在添加一个或多个对象关联 **之后** 发送.
    ``"pre_remove"``
        在移除一个或多个对象关联 **之前** 发送.
    ``"post_remove"``
        在移除一个或多个对象关联 **之后** 发送.
    ``"pre_clear"``
        在清除关联 **之前** 发送.
    ``"post_clear"``
        在清除关联 **之后** 发送.

``reverse``
    表示关联的哪一侧被更新(即被修改的是正向关联还是反向关联).

``model``
    从关联中添加, 移除或清除的对象的类.

``pk_set``
    对于 ``pre_add``, ``post_add``, ``pre_remove`` 和 ``post_remove`` 动作,
    它是被添加或移除关联的主键值的集合.

    对于 ``pre_clear`` 和 ``post_clear`` 动作, 该值为 ``None``.

``using``
    使用的数据库别名.

举个例子, 假如 ``Pizza`` 有多个 ``Topping`` 对象, 模型如下::

    class Topping(models.Model):
        # ...
        pass

    class Pizza(models.Model):
        # ...
        toppings = models.ManyToManyField(Topping)

如果连接一个这样的处理程序::

    from django.db.models.signals import m2m_changed

    def toppings_changed(sender, **kwargs):
        # Do something
        pass

    m2m_changed.connect(toppings_changed, sender=Pizza.toppings.through)

然后做了这样的操作::

    >>> p = Pizza.objects.create(...)
    >>> t = Topping.objects.create(...)
    >>> p.toppings.add(t)

那么发送给 :data:`m2m_changed` 处理程序(上面例子中的 ``toppings_changed``)的参数将是:

==============  ============================================================
参数             值
==============  ============================================================
``sender``      ``Pizza.toppings.through`` (m2m的中间类)

``instance``    ``p`` (被修改 ``Pizza`` 实例)

``action``      ``"pre_add"`` (之后是 ``"post_add"`` 信号)

``reverse``     ``False`` (``Pizza`` 包含
                :class:`~django.db.models.ManyToManyField`, 所以这个调用是更新正向关系)

``model``       ``Topping`` (添加到 ``Pizza`` 的对象)

``pk_set``      ``{t.id}`` (因为只有 ``Topping t`` 被添加到关联中)

``using``       ``"default"`` (使用默认值)
==============  ============================================================

如果再做这样的操作::

    >>> t.pizza_set.remove(p)

那么发送给 :data:`m2m_changed` 处理程序的参数将是:

==============  ============================================================
参数             值
==============  ============================================================
``sender``      ``Pizza.toppings.through`` (m2m的中间类)

``instance``    ``t`` (被修改的 ``Topping`` 实例)

``action``      ``"pre_remove"`` (之后是 ``"post_remove"`` 信号)

``reverse``     ``True`` (``Pizza`` 包含
                :class:`~django.db.models.ManyToManyField`, 所以这个调用是更新反向关系)

``model``       ``Pizza`` (从 ``Topping`` 中移除的对象)

``pk_set``      ``{p.id}`` (因为只删除 ``Pizza p`` 的关联)

``using``       ``"default"`` (使用默认值)
==============  ============================================================

``class_prepared``
------------------

.. data:: django.db.models.signals.class_prepared
   :module:

当“准备好”一个模型类 —— 即模型被定义好并在Django的模型系统中注册时, 就会发出这个信号.
这是Django内部使用的信号, 第三方应用中一般不使用.

由于这个信号是在应用注册表填充期间中发出的, 而 :meth:`AppConfig.ready() <django.apps.AppConfig.ready>` 是在应用注册表完全填充后运行的,
所以不能在该方法中连接接收器. 一种方法是用 ``AppConfig.__init__()`` 来代替连接它们, 注意不要导入模型或触发对应用注册表的调用.

该信号发送的参数有:

``sender``
    刚刚准备好的模型类.

管理信号
==================

由 :doc:`django-admin </ref/django-admin>` 发出的信号.

``pre_migrate``
---------------

.. data:: django.db.models.signals.pre_migrate
   :module:

在 :djadmin:`migrate` 命令安装应用之前发出的信号. 没有 ``models`` 模块的应用不会发出此信号.

该信号发送的参数有:

``sender``
    迁移/同步的应用的 :class:`~django.apps.AppConfig` 实例.

``app_config``
    同 ``sender``.

``verbosity``
    表示manage.py打印到屏幕上的信息量. 详见 :option:`--verbosity`.

    监听 :data:`pre_migrate` 的函数应该根据这个参数的值来调整它们向屏幕输出的内容.

``interactive``
    如果 ``interactive`` 为 ``True``, 则可以安全地提示用户在命令行上输入东西. 如果 ``interactive`` 为 ``False``, 则监听该信号的函数不应提示任何内容.

    例如 :mod:`django.contrib.auth` 应用只有在 ``interactive`` 为 ``True`` 时才会提示创建超级用户.

``using``
    命令将在其上运行的数据库的别名.

``plan``
    .. versionadded:: 1.10

    用于运行迁移的迁移计划. 迁移计划不是公开的API, 但在极少数情况下有必要知道计划.
    计划是一个由二元元组组成的列表, 第一项是迁移类的实例, 第二项显示迁移是否被回滚(``True``)或应用(``False``).

``apps``
    .. versionadded:: 1.10

    运行迁移前含有项目状态的 :data:`Apps <django.apps>` 实例. 应该使用它来代替注册的全局 :attr:`apps <django.apps.apps>` 来检索要执行操作的模型.

``post_migrate``
----------------

.. data:: django.db.models.signals.post_migrate
   :module:

在 :djadmin:`migrate` (即使没有运行迁移) 和 :djadmin:`flush` 命令结束之后发出的信号. 没有 ``models`` 模块的应用不会发出此信号.

该信号的处理者不得更改数据库模式, 因为如果在 :djadmin:`migrate` 命令期间运行 :djadmin:`flush` 命令, 则可能会导致 :djadmin:`flush`  命令失败.

该信号发送的参数有:

``sender``
    刚刚安装的应用的 :class:`~django.apps.AppConfig` 实例.

``app_config``
    同 ``sender``.

``verbosity``
    表示manage.py打印到屏幕上的信息量. 详见 :option:`--verbosity`.

    监听 :data:`pre_migrate` 的函数应该根据这个参数的值来调整它们向屏幕输出的内容.

``interactive``
    如果 ``interactive`` 为 ``True``, 则可以安全地提示用户在命令行上输入东西. 如果 ``interactive`` 为 ``False``, 则监听该信号的函数不应提示任何内容.

    例如 :mod:`django.contrib.auth` 应用只有在 ``interactive`` 为 ``True`` 时才会提示创建超级用户.

``using``
    用于同步的数据库别名. 默认为 ``default`` 数据库.

``plan``
    .. versionadded:: 1.10

    用于运行迁移的迁移计划. 迁移计划不是公开的API, 但在极少数情况下有必要知道计划.
    计划是一个由二元元组组成的列表, 第一项是迁移类的实例, 第二项显示迁移是否被回滚(``True``)或应用(``False``).

``apps``
    .. versionadded:: 1.10

    运行迁移前含有项目状态的 :data:`Apps <django.apps>` 实例. 应该使用它来代替注册的全局 :attr:`apps <django.apps.apps>` 来检索要执行操作的模型.

举个例子, 在
:class:`~django.apps.AppConfig` 中注册一个回调::

    from django.apps import AppConfig
    from django.db.models.signals import post_migrate

    def my_callback(sender, **kwargs):
        # Your specific logic here
        pass

    class MyAppConfig(AppConfig):
        ...

        def ready(self):
            post_migrate.connect(my_callback, sender=self)

.. note::

    如果你使用了 :class:`~django.apps.AppConfig` 实例作为sender的参数, 请确保该信号注册在
    :meth:`~django.apps.AppConfig.ready` 中.
    使用修改过的 :setting:`INSTALLED_APPS` (例如当配置被修改时)集合运行测试时,
    ``AppConfig`` 会被重新创建, 这种信号应该为每个新的 ``AppConfig`` 实例连接e.

请求/响应信号
========================

.. module:: django.core.signals
   :synopsis: Core signals sent by the request/response system.

核心框架处理请求时发出的信号.

``request_started``
-------------------

.. data:: django.core.signals.request_started
   :module:

HTTP请求在开始被处理时发送的信号.

该信号发送的参数有:

``sender``
    处理类 -- 例如. ``django.core.handlers.wsgi.WsgiHandler`` -- 处理该请求.
``environ``
    提供给请求的 ``environ`` 字典.

``request_finished``
--------------------

.. data:: django.core.signals.request_finished
   :module:

当Django向客户端发送HTTP响应完成时发送的信号.

.. note::

    一些WSGI服务器和中间件在处理请求后并不总会对响应对象调用 ``close``, 最明显的是uWSGI1.2.6之前的版本和Sentry的错误报告中间件2.0.7之前的版本.

该信号发送的参数有:

``sender``
    处理类, 同上.

``got_request_exception``
-------------------------

.. data:: django.core.signals.got_request_exception
   :module:

每当Django在处理HTTP请求遇到异常时就会发出这个信号.

该信号发送的参数有

``sender``
    处理类, 同上.

``request``
    :class:`~django.http.HttpRequest` 对象.

测试信号
============

.. module:: django.test.signals
   :synopsis: Signals sent during testing.

:ref:`运行测试 <running-tests>` 时发出的信号.

``setting_changed``
-------------------

.. data:: django.test.signals.setting_changed
   :module:

在使用 ``django.test.TestCase.settings()`` 上下文管理器或者
:func:`django.test.override_settings` 装饰器/上下文管理器修改配置值时发出的信号.

它实际上会发送两次: 应用新值("setup")和恢复原始值("teardown"). 使用 ``enter`` 参数来区别这两种情况.

为避免在非测试情况下从 ``django.test`` 中引入, 你可以从 ``django.core.signals`` 引入该信号.

该信号发送的参数有:

``sender``
    配置处理器.

``setting``
    配置的名称.

``value``
    更改后的配置值. 对于最初不存在的配置, 在"teardown"阶段, ``value`` 为 ``None``.

``enter``
    布尔值; 应用新值为 ``True``, 还原为 ``False``.

``template_rendered``
---------------------

.. data:: django.test.signals.template_rendered
   :module:

当测试系统渲染模板时发出的信号. 该信号在Django服务器平时运行时不会发出, 只有在测试时才会发出.

该信号发送的参数有:

``sender``
    被渲染的 :class:`~django.template.Template` 对象.

``template``
    同sender

``context``
    渲染模板的 :class:`~django.template.Context`.

Database Wrappers
=================

.. module:: django.db.backends
   :synopsis: Core signals sent by the database wrapper.

当数据库连接启动时, database wrapper发出的信号.

``connection_created``
----------------------

.. data:: django.db.backends.signals.connection_created
   :module:

当database wrapper与数据库进行初始连接时发送的信号. 这对于向SQL后端发送连接后的命令特别有用.

该信号发送的参数有:

``sender``
    database wrapper类 -- 例如.
    ``django.db.backends.postgresql.DatabaseWrapper`` 或
    ``django.db.backends.mysql.DatabaseWrapper``, 等.

``connection``
    打开的数据库连接. 这可以在多数据库配置中区分来自不同数据库的连接信号.
