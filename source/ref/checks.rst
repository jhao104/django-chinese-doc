=============
系统检查框架
=============

.. currentmodule:: django.core.checks

系统检查框架是一组用于验证Django项目的静态检查。
它检测常见问题并提供如何修复它们的提示。框架是可扩展的，因此您可以轻松添加自定义的检查。

有关如何添加自定义检查并将其与Django的系统检查集成的详细信息，请参阅 :doc:`System check topic guide </topics/checks>`。

API参考
=========

``CheckMessage``
-----------------

.. class:: CheckMessage(level, msg, hint=None, obj=None, id=None)

系统检查引起的警告和错误必须是 ``CheckMessage`` 的实例。
这个实例封装了一个单一的可重复使用的错误和警告。
它还提供了用于消息的上下文和提示，以及用于过滤的惟一标识符。

构造函数的参数:

``level``
    信息的严重性。使用一个预定义值: ``DEBUG``,
    ``INFO``, ``WARNING``, ``ERROR``, ``CRITICAL``. 如果级别大于或等于 ``ERROR``，
    则Django将阻止管理命令执行。如果消息等级下雨 ``ERROR`` (i.e. warnings) 将报告给控制台, 但不做其他处理。

``msg``
    对问题简短描述的字符串 (少于 80 个字符)，字符串不能包含换行符。

``hint``
    为解决问题提供提示的单行字符串。如果没有提供任何提示，
    或者提示从错误消息中可以明显看出，则提示可以省略，或者用``None``。

``obj``
    可选。为消息提供上下文的对象(例如，发现问题的模型)。对象应该是一个模型、字段、管理器或定义
    了 ``__str__`` 方法的任何其他对象(在Python 2中，您需要定义 ``__unicode__`` 方法)。
    在报告所有消息时使用该方法。

``id``
    可选字符串，问题的唯一标示。标识符应该遵循这种模式
    ``applabel.X001`` 。``X`` 是一个 ``CEWID`` 的一个字母，表示消息严重程度(
    ``C`` 代表 ``CRITICAL``, ``E`` 代表 ``ERROR`` 等。) 这个数字可以由应用程序任意分配，但在应用程序中应该是唯一的。

有子类可以使创建具有公共级别的消息更加容易。在使用它们时，您可以省略 ``level`` 参数，因为它是由类名隐含的

.. class:: Debug(msg, hint=None, obj=None, id=None)
.. class:: Info(msg, hint=None, obj=None, id=None)
.. class:: Warning(msg, hint=None obj=None, id=None)
.. class:: Error(msg, hint=None, obj=None, id=None)
.. class:: Critical(msg, hint=None, obj=None, id=None)

内建检查
=========

.. _system-check-builtin-tags:

内建tags
------------

Django的系统检查使用以下标记:

* ``models``: 检查管理模型，字段和管理器定义。
* ``signals``: 检查信号声明和处理程序注册。
* ``admin``: 检查所有管理网站声明。
* ``compatibility``: 标记版本升级的潜在问题。
* ``security``: 检查安全相关的配置。
* ``templates``: 检查模板相关配置。
* ``caches``: 检查缓存相关配置。
* ``urls``: 检查路由相关配置。
* ``database``: 检查数据库配置相关问题。数据库检查并不是以默认方式运行，因为它们比静态的代码分析做的更多。
  它们只由 :djadmin:`migrate` 命令运行，或者在调用 :djadmin:`check` 命令时指定数据库标记。

.. versionadded:: 1.10

     ``database`` tag 在再1.10版本开始加入的。

某些检查可能会向多个标签注册。

核心系统检查
------------

Models
~~~~~~

* **models.E001**: ``<swappable>`` 格式不是 ``app_label.app_name``.
* **models.E002**: ``<SETTING>`` 引用的 ``<model>`` 没有被 installed,或者是抽象的。
* **models.E003**: 该模型通过中介模型 ``<app_label>.<model>`` 有两个多对多关系。
* **models.E004**: ``id`` 只能用于设置了 ``primary_key=True`` 的字段名称。
* **models.E005**:  ``<model>`` 中的字段 ``<field name>`` 有冲突。
* **models.E006**: 该字段和模型 ``<model>`` 中的字段 ``<field name>`` 存在冲突。
* **models.E007**: 字段 ``<field name>`` 的列名 ``<column name>`` 已经被其他字段使用了。
* **models.E008**: ``index_together`` 必须是列表或者元组。
* **models.E009**: 所有 ``index_together`` 必须是列表或者元组。
* **models.E010**: ``unique_together`` 必须是列表或者元组。
* **models.E011**: 所有 ``unique_together`` 必须是列表或者元组。
* **models.E012**: ``index_together/unique_together`` 关联到了不存在的字段名 ``<field name>``。
* **models.E013**: ``index_together/unique_together`` 关联了
  ``ManyToManyField`` ``<field name>``, 但是 ``ManyToManyField`` 不支持该选项。
* **models.E014**: ``ordering`` 必须是列表或者元组 ( 即使你只想按一个字段排序)。
* **models.E015**: ``ordering`` 关联到了一个存在的 ``<field name>``。
* **models.E016**: ``index_together/unique_together`` 关联的字段 ``<field_name>`` 不在本地模型 ``<model>`` 中。
* **models.E017**: 代理模型 ``<model>`` 不能有模型字段。
* **models.E018**: 字段 ``<field>`` 的自动生成列名过长。数据库 ``<alias>`` 中的最大长度是 ``<maximum length>``。
* **models.E019**: M2M字段 ``<M2M field>`` 的自动生成列名过长。 数据库 ``<alias>`` 中的最大长度是 ``<maximum length>``。
* **models.E020**:  ``<model>.check()`` 类方法当前被覆盖。
* **models.E021**: ``ordering`` 和 ``order_with_respect_to`` 不能同时使用。
* **models.E022**: ``<function>`` 包含了 ``<app label>.<model>`` 的惰性引用,
  但是应用 ``<app label>`` 没有install或者没有模型 ``<model>`` 。

字段
~~~~~~

* **fields.E001**: 字段名称不能以下划线结尾。
* **fields.E002**: 字段名称不能包含``"__"`` 。
* **fields.E003**: ``pk`` 是不能用作字段名称的保留字。
* **fields.E004**: ``choices`` 必须是可迭代的 (e.g., 元组或者列表).
* **fields.E005**: ``choices`` 必须是可迭代的返回 ``(实际值,易读值)`` 元组。
* **fields.E006**: ``db_index`` 必须是 ``None``, ``True`` 或者 ``False``。
* **fields.E007**: 主键必须设置 ``null=True``。
* **fields.E100**: ``AutoField`` 必须设置 primary_key=True.
* **fields.E110**: ``BooleanField`` 不接受null。
* **fields.E120**: ``CharField`` 必须定义 ``max_length`` 属性。
* **fields.E121**: ``max_length`` 必须是正整数。
* **fields.W122**: 使用 ``IntegerField`` 时可忽略 ``max_length`` 。
* **fields.E130**: ``DecimalField`` 必须定义 ``decimal_places`` 属性。
* **fields.E131**: ``decimal_places`` 必须是非负整数。
* **fields.E132**: ``DecimalField` 必须定义 ``max_digits`` 属性。
* **fields.E133**: ``max_digits`` 必须是非负整数。
* **fields.E134**: ``max_digits`` 必须大于等于 ``decimal_places``。
* **fields.E140**: ``FilePathField`` 必须设置 ``allow_files`` 或者 ``allow_folders`` 为True。
* **fields.E150**: ``GenericIPAddressField`` 如果不允许空值, 则不能接受空值，，因为空值存储为null。
* **fields.E160**: 选项 ``auto_now``, ``auto_now_add`` 和 ``default``
  是互斥的。这些选项中只能有一个存在。
* **fields.W161**: 提供固定的默认值。
* **fields.E900**: ``IPAddressField`` 已被删除，仅在历史迁移中支持。
* **fields.W900**: ``IPAddressField`` 已被弃用。对它的支持（除了历史迁移）将在Django 1.9中删除。
  * 这个检查只在  Django 1.7 和 1.8 中* 。
* **fields.W901**: ``CommaSeparatedIntegerField`` 已弃用。对它的支持(除了在历史迁移)将在Django 2.0中删除。

文件字段
~~~~~~~~~

* **fields.E200**: ``unique`` 不是 ``FileField`` 的合法参数。
* **fields.E201**: ``primary_key`` 不是 ``FileField`` 的合法参数。
* **fields.E210**: 由于 Pillow 没有安装，所以  ``ImageField`` 无法使用。

关系字段
~~~~~~~~~

* **fields.E300**: Field defines a relation with model ``<model>``, which is
  either not installed, or is abstract.
* **fields.E301**: Field defines a relation with the model ``<model>`` which
  has been swapped out.
* **fields.E302**: Accessor for field ``<field name>`` clashes with field
  ``<field name>``.
* **fields.E303**: Reverse query name for field ``<field name>`` clashes with
  field ``<field name>``.
* **fields.E304**: Field name ``<field name>`` clashes with accessor for
  ``<field name>``.
* **fields.E305**: Field name ``<field name>`` clashes with reverse query name
  for ``<field name>``.
* **fields.E306**: Related name must be a valid Python identifier or end with
  a ``'+'``.
* **fields.E307**: The field ``<app label>.<model>.<field name>`` was declared
  with a lazy reference to ``<app label>.<model>``, but app ``<app label>``
  isn't installed or doesn't provide model ``<model>``.
* **fields.E310**: No subset of the fields ``<field1>``, ``<field2>``, ... on
  model ``<model>`` is unique. Add ``unique=True`` on any of those fields or
  add at least a subset of them to a unique_together constraint.
* **fields.E311**: ``<model>`` must set ``unique=True`` because it is
  referenced by a ``ForeignKey``.
* **fields.E320**: Field specifies ``on_delete=SET_NULL``, but cannot be null.
* **fields.E321**: The field specifies ``on_delete=SET_DEFAULT``, but has no
  default value.
* **fields.E330**: ``ManyToManyField``\s cannot be unique.
* **fields.E331**: Field specifies a many-to-many relation through model
  ``<model>``, which has not been installed.
* **fields.E332**: Many-to-many fields with intermediate tables must not be
  symmetrical.
* **fields.E333**: The model is used as an intermediate model by ``<model>``,
  but it has more than two foreign keys to ``<model>``, which is ambiguous.
  You must specify which two foreign keys Django should use via the
  ``through_fields`` keyword argument.
* **fields.E334**: The model is used as an intermediate model by ``<model>``,
  but it has more than one foreign key from ``<model>``, which is ambiguous.
  You must specify which foreign key Django should use via the
  ``through_fields`` keyword argument.
* **fields.E335**: The model is used as an intermediate model by ``<model>``,
  but it has more than one foreign key to ``<model>``, which is ambiguous.
  You must specify which foreign key Django should use via the
  ``through_fields`` keyword argument.
* **fields.E336**: The model is used as an intermediary model by ``<model>``,
  but it does not have foreign key to ``<model>`` or ``<model>``.
* **fields.E337**: Field specifies ``through_fields`` but does not provide the
  names of the two link fields that should be used for the relation through
  ``<model>``.
* **fields.E338**: The intermediary model ``<through model>`` has no field
  ``<field name>``.
* **fields.E339**: ``<model>.<field name>`` is not a foreign key to ``<model>``.
* **fields.W340**: ``null`` has no effect on ``ManyToManyField``.
* **fields.W341**: ``ManyToManyField`` does not support ``validators``.
* **fields.W342**: Setting ``unique=True`` on a ``ForeignKey`` has the same
  effect as using a ``OneToOneField``.

Signals
~~~~~~~

* **signals.E001**: ``<handler>`` was connected to the ``<signal>`` signal with
  a lazy reference to the sender ``<app label>.<model>``, but app ``<app label>``
  isn't installed or doesn't provide model ``<model>``.

Backwards Compatibility
~~~~~~~~~~~~~~~~~~~~~~~

The following checks are performed to warn the user of any potential problems
that might occur as a result of a version upgrade.

* **1_6.W001**: Some project unit tests may not execute as expected. *This
  check was removed in Django 1.8 due to false positives*.
* **1_6.W002**: ``BooleanField`` does not have a default value. *This
  check was removed in Django 1.8 due to false positives*.
* **1_7.W001**:  Django 1.7 changed the global defaults for the
  ``MIDDLEWARE_CLASSES.``
  ``django.contrib.sessions.middleware.SessionMiddleware``,
  ``django.contrib.auth.middleware.AuthenticationMiddleware``, and
  ``django.contrib.messages.middleware.MessageMiddleware`` were removed from
  the defaults. If your project needs these middleware then you should
  configure this setting. *This check was removed in Django 1.9*.
* **1_8.W001**: The standalone ``TEMPLATE_*`` settings were deprecated in
  Django 1.8 and the :setting:`TEMPLATES` dictionary takes precedence. You must
  put the values of the following settings into your defaults ``TEMPLATES``
  dict: ``TEMPLATE_DIRS``, ``TEMPLATE_CONTEXT_PROCESSORS``, ``TEMPLATE_DEBUG``,
  ``TEMPLATE_LOADERS``, ``TEMPLATE_STRING_IF_INVALID``.
* **1_10.W001**: The ``MIDDLEWARE_CLASSES`` setting is deprecated in Django
  1.10  and the :setting:`MIDDLEWARE` setting takes precedence. Since you've
  set ``MIDDLEWARE``, the value of ``MIDDLEWARE_CLASSES`` is ignored.

Admin
-----

Admin checks are all performed as part of the ``admin`` tag.

The following checks are performed on any
:class:`~django.contrib.admin.ModelAdmin` (or subclass) that is registered
with the admin site:

* **admin.E001**: The value of ``raw_id_fields`` must be a list or tuple.
* **admin.E002**: The value of ``raw_id_fields[n]`` refers to ``<field name>``,
  which is not an attribute of ``<model>``.
* **admin.E003**: The value of ``raw_id_fields[n]`` must be a foreign key or
  a many-to-many field.
* **admin.E004**: The value of ``fields`` must be a list or tuple.
* **admin.E005**: Both ``fieldsets`` and ``fields`` are specified.
* **admin.E006**: The value of ``fields`` contains duplicate field(s).
* **admin.E007**: The value of ``fieldsets`` must be a list or tuple.
* **admin.E008**: The value of ``fieldsets[n]`` must be a list or tuple.
* **admin.E009**: The value of ``fieldsets[n]`` must be of length 2.
* **admin.E010**: The value of ``fieldsets[n][1]`` must be a dictionary.
* **admin.E011**: The value of ``fieldsets[n][1]`` must contain the key
  ``fields``.
* **admin.E012**: There are duplicate field(s) in ``fieldsets[n][1]``.
* **admin.E013**: ``fields[n]/fieldsets[n][m]`` cannot include the
  ``ManyToManyField`` ``<field name>``, because that field manually specifies a
  relationship model.
* **admin.E014**: The value of ``exclude`` must be a list or tuple.
* **admin.E015**: The value of ``exclude`` contains duplicate field(s).
* **admin.E016**: The value of ``form`` must inherit from ``BaseModelForm``.
* **admin.E017**: The value of ``filter_vertical`` must be a list or tuple.
* **admin.E018**: The value of ``filter_horizontal`` must be a list or tuple.
* **admin.E019**: The value of ``filter_vertical[n]/filter_vertical[n]`` refers
  to ``<field name>``, which is not an attribute of ``<model>``.
* **admin.E020**: The value of ``filter_vertical[n]/filter_vertical[n]`` must
  be a many-to-many field.
* **admin.E021**: The value of ``radio_fields`` must be a dictionary.
* **admin.E022**: The value of ``radio_fields`` refers to ``<field name>``,
  which is not an attribute of ``<model>``.
* **admin.E023**: The value of ``radio_fields`` refers to ``<field name>``,
  which is not a ``ForeignKey``, and does not have a ``choices`` definition.
* **admin.E024**: The value of ``radio_fields[<field name>]`` must be either
  ``admin.HORIZONTAL`` or ``admin.VERTICAL``.
* **admin.E025**: The value of ``view_on_site`` must be either a callable or a
  boolean value.
* **admin.E026**: The value of ``prepopulated_fields`` must be a dictionary.
* **admin.E027**: The value of ``prepopulated_fields`` refers to
  ``<field name>``, which is not an attribute of ``<model>``.
* **admin.E028**: The value of ``prepopulated_fields`` refers to
  ``<field name>``, which must not be a ``DateTimeField``, a ``ForeignKey``, or a
  ``ManyToManyField`` field.
* **admin.E029**: The value of ``prepopulated_fields[<field name>]`` must be a
  list or tuple.
* **admin.E030**: The value of ``prepopulated_fields`` refers to
  ``<field name>``, which is not an attribute of ``<model>``.
* **admin.E031**: The value of ``ordering`` must be a list or tuple.
* **admin.E032**: The value of ``ordering`` has the random ordering marker
  ``?``, but contains other fields as well.
* **admin.E033**: The value of ``ordering`` refers to ``<field name>``, which
  is not an attribute of ``<model>``.
* **admin.E034**: The value of ``readonly_fields`` must be a list or tuple.
* **admin.E035**: The value of ``readonly_fields[n]`` is not a callable, an
  attribute of ``<ModelAdmin class>``, or an attribute of ``<model>``.

``ModelAdmin``
~~~~~~~~~~~~~~

The following checks are performed on any
:class:`~django.contrib.admin.ModelAdmin` that is registered
with the admin site:

* **admin.E101**: The value of ``save_as`` must be a boolean.
* **admin.E102**: The value of ``save_on_top`` must be a boolean.
* **admin.E103**: The value of ``inlines`` must be a list or tuple.
* **admin.E104**: ``<InlineModelAdmin class>`` must inherit from
  ``BaseModelAdmin``.
* **admin.E105**: ``<InlineModelAdmin class>`` must have a ``model`` attribute.
* **admin.E106**: The value of ``<InlineModelAdmin class>.model`` must be a
  ``Model``.
* **admin.E107**: The value of ``list_display`` must be a list or tuple.
* **admin.E108**: The value of ``list_display[n]`` refers to ``<label>``,
  which is not a callable, an attribute of ``<ModelAdmin class>``, or an
  attribute or method on ``<model>``.
* **admin.E109**: The value of ``list_display[n]`` must not be a
  ``ManyToManyField`` field.
* **admin.E110**: The value of ``list_display_links`` must be a list, a tuple,
  or ``None``.
* **admin.E111**: The value of ``list_display_links[n]`` refers to ``<label>``,
  which is not defined in ``list_display``.
* **admin.E112**: The value of ``list_filter`` must be a list or tuple.
* **admin.E113**: The value of ``list_filter[n]`` must inherit from
  ``ListFilter``.
* **admin.E114**: The value of ``list_filter[n]`` must not inherit from
  ``FieldListFilter``.
* **admin.E115**: The value of ``list_filter[n][1]`` must inherit from
  ``FieldListFilter``.
* **admin.E116**: The value of ``list_filter[n]`` refers to ``<label>``,
  which does not refer to a Field.
* **admin.E117**: The value of ``list_select_related`` must be a boolean,
  tuple or list.
* **admin.E118**: The value of ``list_per_page`` must be an integer.
* **admin.E119**: The value of ``list_max_show_all`` must be an integer.
* **admin.E120**: The value of ``list_editable`` must be a list or tuple.
* **admin.E121**: The value of ``list_editable[n]`` refers to ``<label>``,
  which is not an attribute of ``<model>``.
* **admin.E122**: The value of ``list_editable[n]`` refers to ``<label>``,
  which is not contained in ``list_display``.
* **admin.E123**: The value of ``list_editable[n]`` cannot be in both
  ``list_editable`` and ``list_display_links``.
* **admin.E124**: The value of ``list_editable[n]`` refers to the first field
  in ``list_display`` (``<label>``), which cannot be used unless
  ``list_display_links`` is set.
* **admin.E125**: The value of ``list_editable[n]`` refers to ``<field name>``,
  which is not editable through the admin.
* **admin.E126**: The value of ``search_fields`` must be a list or tuple.
* **admin.E127**: The value of ``date_hierarchy`` refers to ``<field name>``,
  which is not an attribute of ``<model>``.
* **admin.E128**: The value of ``date_hierarchy`` must be a ``DateField`` or
  ``DateTimeField``.

``InlineModelAdmin``
~~~~~~~~~~~~~~~~~~~~

The following checks are performed on any
:class:`~django.contrib.admin.InlineModelAdmin` that is registered as an
inline on a :class:`~django.contrib.admin.ModelAdmin`.

* **admin.E201**: Cannot exclude the field ``<field name>``, because it is the
  foreign key to the parent model ``<app_label>.<model>``.
* **admin.E202**: ``<model>`` has no ``ForeignKey`` to ``<parent model>``./
  ``<model>`` has more than one ``ForeignKey`` to ``<parent model>``.
* **admin.E203**: The value of ``extra`` must be an integer.
* **admin.E204**: The value of ``max_num`` must be an integer.
* **admin.E205**: The value of ``min_num`` must be an integer.
* **admin.E206**: The value of ``formset`` must inherit from
  ``BaseModelFormSet``.

``GenericInlineModelAdmin``
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following checks are performed on any
:class:`~django.contrib.contenttypes.admin.GenericInlineModelAdmin` that is
registered as an inline on a :class:`~django.contrib.admin.ModelAdmin`.

* **admin.E301**: ``'ct_field'`` references ``<label>``, which is not a field
  on ``<model>``.
* **admin.E302**: ``'ct_fk_field'`` references ``<label>``, which is not a
  field on ``<model>``.
* **admin.E303**: ``<model>`` has no ``GenericForeignKey``.
* **admin.E304**: ``<model>`` has no ``GenericForeignKey`` using content type
  field ``<field name>`` and object ID field ``<field name>``.

``AdminSite``
~~~~~~~~~~~~~

The following checks are performed on the default
:class:`~django.contrib.admin.AdminSite`:

* **admin.E401**: :mod:`django.contrib.contenttypes` must be in
  :setting:`INSTALLED_APPS` in order to use the admin application.
* **admin.E402**: :mod:`django.contrib.auth.context_processors.auth`
  must be in :setting:`TEMPLATES` in order to use the admin application.

Auth
----

* **auth.E001**: ``REQUIRED_FIELDS`` must be a list or tuple.
* **auth.E002**: The field named as the ``USERNAME_FIELD`` for a custom user
  model must not be included in ``REQUIRED_FIELDS``.
* **auth.E003**: ``<field>`` must be unique because it is named as the
  ``USERNAME_FIELD``.
* **auth.W004**: ``<field>`` is named as the ``USERNAME_FIELD``, but it is not
  unique.
* **auth.E005**: The permission codenamed ``<codename>`` clashes with a builtin
  permission for model ``<model>``.
* **auth.E006**: The permission codenamed ``<codename>`` is duplicated for model
  ``<model>``.
* **auth.E007**: The :attr:`verbose_name
  <django.db.models.Options.verbose_name>` of model ``<model>`` must be at most
  244 characters for its builtin permission names
  to be at most 255 characters.
* **auth.E008**: The permission named ``<name>`` of model ``<model>`` is longer
  than 255 characters.
* **auth.C009**: ``<User model>.is_anonymous`` must be an attribute or property
  rather than a method. Ignoring this is a security issue as anonymous users
  will be treated as authenticated!
* **auth.C010**: ``<User model>.is_authenticated`` must be an attribute or
  property rather than a method. Ignoring this is a security issue as anonymous
  users will be treated as authenticated!


Content Types
-------------

The following checks are performed when a model contains a
:class:`~django.contrib.contenttypes.fields.GenericForeignKey` or
:class:`~django.contrib.contenttypes.fields.GenericRelation`:

* **contenttypes.E001**: The ``GenericForeignKey`` object ID references the
  non-existent field ``<field>``.
* **contenttypes.E002**: The ``GenericForeignKey`` content type references the
  non-existent field ``<field>``.
* **contenttypes.E003**: ``<field>`` is not a ``ForeignKey``.
* **contenttypes.E004**: ``<field>`` is not a ``ForeignKey`` to
  ``contenttypes.ContentType``.

Security
--------

The security checks do not make your site secure. They do not audit code, do
intrusion detection, or do anything particularly complex. Rather, they help
perform an automated, low-hanging-fruit checklist. They help you remember the
simple things that improve your site's security.

Some of these checks may not be appropriate for your particular deployment
configuration. For instance, if you do your HTTP to HTTPS redirection in a load
balancer, it'd be irritating to be constantly warned about not having enabled
:setting:`SECURE_SSL_REDIRECT`. Use :setting:`SILENCED_SYSTEM_CHECKS` to
silence unneeded checks.

The following checks are run if you use the :option:`check --deploy` option:

* **security.W001**: You do not have
  :class:`django.middleware.security.SecurityMiddleware` in your
  :setting:`MIDDLEWARE`/:setting:`MIDDLEWARE_CLASSES` so the :setting:`SECURE_HSTS_SECONDS`,
  :setting:`SECURE_CONTENT_TYPE_NOSNIFF`, :setting:`SECURE_BROWSER_XSS_FILTER`,
  and :setting:`SECURE_SSL_REDIRECT` settings will have no effect.
* **security.W002**: You do not have
  :class:`django.middleware.clickjacking.XFrameOptionsMiddleware` in your
  :setting:`MIDDLEWARE`/:setting:`MIDDLEWARE_CLASSES`, so your pages will not be served with an
  ``'x-frame-options'`` header. Unless there is a good reason for your
  site to be served in a frame, you should consider enabling this
  header to help prevent clickjacking attacks.
* **security.W003**: You don't appear to be using Django's built-in cross-site
  request forgery protection via the middleware
  (:class:`django.middleware.csrf.CsrfViewMiddleware` is not in your
  :setting:`MIDDLEWARE`/:setting:`MIDDLEWARE_CLASSES`). Enabling the middleware is the safest
  approach to ensure you don't leave any holes.
* **security.W004**: You have not set a value for the
  :setting:`SECURE_HSTS_SECONDS` setting. If your entire site is served only
  over SSL, you may want to consider setting a value and enabling :ref:`HTTP
  Strict Transport Security <http-strict-transport-security>`. Be sure to read
  the documentation first; enabling HSTS carelessly can cause serious,
  irreversible problems.
* **security.W005**: You have not set the
  :setting:`SECURE_HSTS_INCLUDE_SUBDOMAINS` setting to ``True``. Without this,
  your site is potentially vulnerable to attack via an insecure connection to a
  subdomain. Only set this to ``True`` if you are certain that all subdomains of
  your domain should be served exclusively via SSL.
* **security.W006**: Your :setting:`SECURE_CONTENT_TYPE_NOSNIFF` setting is not
  set to ``True``, so your pages will not be served with an
  ``'x-content-type-options: nosniff'`` header. You should consider enabling
  this header to prevent the browser from identifying content types incorrectly.
* **security.W007**: Your :setting:`SECURE_BROWSER_XSS_FILTER` setting is not
  set to ``True``, so your pages will not be served with an
  ``'x-xss-protection: 1; mode=block'`` header. You should consider enabling
  this header to activate the browser's XSS filtering and help prevent XSS
  attacks.
* **security.W008**: Your :setting:`SECURE_SSL_REDIRECT` setting is not set to
  ``True``. Unless your site should be available over both SSL and non-SSL
  connections, you may want to either set this setting to ``True`` or configure
  a load balancer or reverse-proxy server  to redirect all connections to HTTPS.
* **security.W009**: Your :setting:`SECRET_KEY` has less than 50 characters or
  less than 5 unique characters. Please generate a long and random
  ``SECRET_KEY``, otherwise many of Django's security-critical features will be
  vulnerable to attack.
* **security.W010**: You have :mod:`django.contrib.sessions` in your
  :setting:`INSTALLED_APPS` but you have not set
  :setting:`SESSION_COOKIE_SECURE` to ``True``. Using a secure-only session
  cookie makes it more difficult for network traffic sniffers to hijack user
  sessions.
* **security.W011**: You have
  :class:`django.contrib.sessions.middleware.SessionMiddleware` in your
  :setting:`MIDDLEWARE`/:setting:`MIDDLEWARE_CLASSES`, but you have not set
  :setting:`SESSION_COOKIE_SECURE` to ``True``. Using a secure-only session
  cookie makes it more difficult for network traffic sniffers to hijack user
  sessions.
* **security.W012**: :setting:`SESSION_COOKIE_SECURE` is not set to ``True``.
  Using a secure-only session cookie makes it more difficult for network traffic
  sniffers to hijack user sessions.
* **security.W013**: You have :mod:`django.contrib.sessions` in your
  :setting:`INSTALLED_APPS`, but you have not set
  :setting:`SESSION_COOKIE_HTTPONLY` to ``True``. Using an ``HttpOnly`` session
  cookie makes it more difficult for cross-site scripting attacks to hijack user
  sessions.
* **security.W014**: You have
  :class:`django.contrib.sessions.middleware.SessionMiddleware` in your
  :setting:`MIDDLEWARE`/:setting:`MIDDLEWARE_CLASSES`, but you have not set
  :setting:`SESSION_COOKIE_HTTPONLY` to ``True``. Using an ``HttpOnly`` session
  cookie makes it more difficult for cross-site scripting attacks to hijack user
  sessions.
* **security.W015**: :setting:`SESSION_COOKIE_HTTPONLY` is not set to ``True``.
  Using an ``HttpOnly`` session cookie makes it more difficult for cross-site
  scripting attacks to hijack user sessions.
* **security.W016**: :setting:`CSRF_COOKIE_SECURE` is not set to ``True``.
  Using a secure-only CSRF cookie makes it more difficult for network traffic
  sniffers to steal the CSRF token.
* **security.W017**: :setting:`CSRF_COOKIE_HTTPONLY` is not set to ``True``.
  Using an ``HttpOnly`` CSRF cookie makes it more difficult for cross-site
  scripting attacks to steal the CSRF token.
* **security.W018**: You should not have :setting:`DEBUG` set to ``True`` in
  deployment.
* **security.W019**: You have
  :class:`django.middleware.clickjacking.XFrameOptionsMiddleware` in your
  :setting:`MIDDLEWARE`/:setting:`MIDDLEWARE_CLASSES`, but :setting:`X_FRAME_OPTIONS` is not set to
  ``'DENY'``. The default is ``'SAMEORIGIN'``, but unless there is a good reason
  for your site to serve other parts of itself in a frame, you should change
  it to ``'DENY'``.
* **security.W020**: :setting:`ALLOWED_HOSTS` must not be empty in deployment.

Sites
-----

The following checks are performed on any model using a
:class:`~django.contrib.sites.managers.CurrentSiteManager`:

* **sites.E001**: ``CurrentSiteManager`` could not find a field named
  ``<field name>``.
* **sites.E002**: ``CurrentSiteManager`` cannot use ``<field>`` as it is not a
  foreign key or a many-to-many field.

Database
--------

MySQL
~~~~~

If you're using MySQL, the following checks will be performed:

* **mysql.E001**: MySQL does not allow unique ``CharField``\s to have a
  ``max_length`` > 255.
* **mysql.W002**: MySQL Strict Mode is not set for database connection
  '<alias>'. See also :ref:`mysql-sql-mode`.

Templates
---------

The following checks verify that your :setting:`TEMPLATES` setting is correctly
configured:

* **templates.E001**: You have ``'APP_DIRS': True`` in your
  :setting:`TEMPLATES` but also specify ``'loaders'`` in ``OPTIONS``. Either
  remove ``APP_DIRS`` or remove the ``'loaders'`` option.
* **templates.E002**: ``string_if_invalid`` in :setting:`TEMPLATES`
  :setting:`OPTIONS <TEMPLATES-OPTIONS>` must be a string but got: ``{value}``
  (``{type}``).

Caches
------

The following checks verify that your :setting:`CACHES` setting is correctly
configured:

* **caches.E001**: You must define a ``'default'`` cache in your
  :setting:`CACHES` setting.

URLs
----

The following checks are performed on your URL configuration:

* **urls.W001**: Your URL pattern ``<pattern>`` uses
  :func:`~django.conf.urls.include` with a ``regex`` ending with a
  ``$``. Remove the dollar from the ``regex`` to avoid problems
  including URLs.
* **urls.W002**: Your URL pattern ``<pattern>`` has a ``regex``
  beginning with a ``/``. Remove this slash as it is unnecessary.
  If this pattern is targeted in an :func:`~django.conf.urls.include`, ensure
  the :func:`~django.conf.urls.include` pattern has a trailing ``/``.
* **urls.W003**: Your URL pattern ``<pattern>`` has a ``name``
  including a ``:``. Remove the colon, to avoid ambiguous namespace
  references.
* **urls.E004**: Your URL pattern ``<pattern>`` is invalid. Ensure that
  ``urlpatterns`` is a list of :func:`~django.conf.urls.url()` instances.
