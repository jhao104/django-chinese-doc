========
Settings
========

.. contents::
    :local:
    :depth: 1

.. warning::

    当你修改配置时请注意, 特别是默认值为非空列表或字典的配置, 例如 :setting:`MIDDLEWARE_CLASSES`
    和 :setting:`STATICFILES_FINDERS`. 确保你保留的是你希望使用的Django功能所需的组件.

核心配置
=============

以下是一些Django的核心配置和其默认值. 下面列出了contrib应用提供的配置, 后面是核心配置的专题索引.
关于介绍性资料, 详见 :doc:`settings指南 </topics/settings>`.

.. setting:: ABSOLUTE_URL_OVERRIDES

``ABSOLUTE_URL_OVERRIDES``
--------------------------

默认值: ``{}`` (空字典)

它是一个将 ``"app_label.model_name"`` 字符串映射到接受模型对象并返回其URL的函数的字典.
这是一种预安装的插入或覆盖 ``get_absolute_url()`` 方法的方式. 例如::

    ABSOLUTE_URL_OVERRIDES = {
        'blogs.weblog': lambda o: "/blogs/%s/" % o.slug,
        'news.story': lambda o: "/stories/%s/%s/" % (o.pub_year, o.slug),
    }

注意上面配置中使用的模型对象名称一定要小写, 与模型类名的实际大小写情况无关.

.. setting:: ADMINS

``ADMINS``
----------

默认值: ``[]`` (空列表)

接收代码错误异常通知的人的列表. 当 ``DEBUG=False`` 且某个视图发生异常, Django会通过邮件方式将完整的错误信息发送给这些人.
列表中每个项是一个包含名字和邮箱的元组. 例如::

    [('John', 'john@example.com'), ('Mary', 'mary@example.com')]

注意, 无论何时发生错误, Django总会发邮件给配置中的 **所有** 人. 详见 :doc:`/howto/error-reporting`.

.. setting:: ALLOWED_HOSTS

``ALLOWED_HOSTS``
-----------------

默认值: ``[]`` (空列表)

允许访问Django站点的host/domain列表. 这是一个防止 :ref:`HTTP Host header 攻击
<host-headers-virtual-hosting>` 的安全措施.

如果列表中的值是完整的标准名称(例如. ``'www.example.com'``),
这种情况下它会和请求的 ``Host`` 首部完全匹配(不区分大小写, 不包括端口). 以英文句号开头的值可以用作子域通配符: ``'.example.com'`` 会
匹配到 ``example.com`` 和 ``www.example.com``, 以及 ``example.com`` 的所有子域名.
值 ``'*'`` 会匹配所有内容; 在这种情况下, 你有责任提供自己对 ``Host`` 首部的验证(可能在中间件中; 如果这样, 该中间件必须处在
:setting:`MIDDLEWARE` 中的第一位置).

Django还允许配置 `完全限定域名(FQDN)`_ .
某些浏览器在 ``Host`` 首部中包含了一个尾部的点, Django在执行验证时会将其去掉.

.. _`完全限定域名(FQDN)`: https://en.wikipedia.org/wiki/Fully_qualified_domain_name

如果 ``Host`` 首部(或者是 :setting:`USE_X_FORWARDED_HOST` 配置下的 ``X-Forwarded-Host``)没有匹配到该列表中的值,
:meth:`django.http.HttpRequest.get_host()` 方法会引发
:exc:`~django.core.exceptions.SuspiciousOperation` 异常.

如果 :setting:`DEBUG` 设置为 ``True`` 且 ``ALLOWED_HOSTS`` 为空, host将根据
``['localhost', '127.0.0.1', '[::1]']`` 进行验证.

这些验证仅通过 :meth:`~django.http.HttpRequest.get_host()` 来实现;
如果你的代码直接从 ``request.META`` 获取 ``Host`` 首部, 你将绕过此安全保护.

.. versionchanged:: 1.10.3

    在老版本中, 当 ``DEBUG=True`` 时 ``ALLOWED_HOSTS`` 不会验证.
    这也在Django 1.9.11 和 1.8.16 中有更改, 以防止DNS重新绑定攻击.

.. setting:: APPEND_SLASH

``APPEND_SLASH``
----------------

默认值: ``True``

设置为 ``True`` 时, 如果请求URL没有匹配到URLconf中的内容且没有以斜杠结尾, 将重定向到以斜杠结尾的相同URL.
需要注意的是重定向可能会导致POST请求中的数据丢失.

:setting:`APPEND_SLASH` 配置只有在使用了
:class:`~django.middleware.common.CommonMiddleware` 才会生效
(详见 :doc:`/topics/http/middleware`). 或 :setting:`PREPEND_WWW`.

.. setting:: CACHES

``CACHES``
----------

默认值::

    {
        'default': {
            'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        }
    }

Django缓存配置的字典. 它是一个嵌套字典, 包含缓存别名和其对应的缓存项.

:setting:`CACHES` 配置必须包含一个 ``default`` 缓存;
可以指定任何数量的其他缓存. 如果你使用了除本地内存缓存之外的缓存后端,
或者是你需要使用多个缓存, 你可能会使用到下面的缓存选项.

.. setting:: CACHES-BACKEND

``BACKEND``
~~~~~~~~~~~

默认值: ``''`` (空字符串)

Django内置的缓存后端:

* ``'django.core.cache.backends.db.DatabaseCache'``
* ``'django.core.cache.backends.dummy.DummyCache'``
* ``'django.core.cache.backends.filebased.FileBasedCache'``
* ``'django.core.cache.backends.locmem.LocMemCache'``
* ``'django.core.cache.backends.memcached.MemcachedCache'``
* ``'django.core.cache.backends.memcached.PyLibMCCache'``

要使用Django未提供的缓存后端, 可以将 :setting:`BACKEND <CACHES-BACKEND>` 设置为完全限定路径(如. ``mypackage.backends.whatever.WhateverCache``).

.. setting:: CACHES-KEY_FUNCTION

``KEY_FUNCTION``
~~~~~~~~~~~~~~~~

包含函数(或任何可调用)的点分隔路径的字符串, 该函数定义如何将前缀, 版本和键组合成最终缓存键. 默认实现相当于函数::

    def make_key(key, key_prefix, version):
        return ':'.join([key_prefix, str(version), key])

你可以使用任何具有相同参数的函数.

详见 :ref:`缓存文档 <cache_key_transformation>`.

.. setting:: CACHES-KEY_PREFIX

``KEY_PREFIX``
~~~~~~~~~~~~~~

默认值: ``''`` (空字符串)

一个自动包含在Django服务器使用的所有缓存键中的字符串(默认为前缀).

详见 :ref:`缓存文档 <cache_key_prefixing>`.

.. setting:: CACHES-LOCATION

``LOCATION``
~~~~~~~~~~~~

默认值: ``''`` (空字符串)

缓存位置. 它可以是缓存的文件系统目录, 内存缓存服务的host和port, 或者是本地内存缓存的标识名称. 例如::

    CACHES = {
        'default': {
            'BACKEND': 'django.core.cache.backends.filebased.FileBasedCache',
            'LOCATION': '/var/tmp/django_cache',
        }
    }

.. setting:: CACHES-OPTIONS

``OPTIONS``
~~~~~~~~~~~

默认值: ``None``

传递给缓存后端的额外参数. 可用的参数取决于你的缓存后端.

可用的参数信息请参见 :ref:`缓存参数 <cache_arguments>` 文档. 更多信息请查阅后端模块所属文档.

.. setting:: CACHES-TIMEOUT

``TIMEOUT``
~~~~~~~~~~~

默认值: ``300``

缓存过期时间. 如果该值为 ``None``, 则缓存将不会过期.

.. setting:: CACHES-VERSION

``VERSION``
~~~~~~~~~~~

默认值: ``1``

生成的缓存的默认版本号.

详见 :ref:`缓存文档 <cache_versioning>`.

.. setting:: CACHE_MIDDLEWARE_ALIAS

``CACHE_MIDDLEWARE_ALIAS``
--------------------------

默认值: ``default``

用于 :ref:`缓存中间件<the-per-site-cache>` 的缓存链接.

.. setting:: CACHE_MIDDLEWARE_KEY_PREFIX

``CACHE_MIDDLEWARE_KEY_PREFIX``
-------------------------------

默认值: ``''`` (空字符串)

:ref:`缓存中间件 <the-per-site-cache>` 生成缓存密钥的前缀字符串. 该前缀将和
:setting:`KEY_PREFIX <CACHES-KEY_PREFIX>` 配置组合在一起; 注意不是替换.

详见 :doc:`/topics/cache`.

.. setting:: CACHE_MIDDLEWARE_SECONDS

``CACHE_MIDDLEWARE_SECONDS``
----------------------------

默认值: ``600``

:ref:`缓存中间件<the-per-site-cache>` 缓存页面的默认秒数.

详见 :doc:`/topics/cache`.

.. _settings-csrf:

.. setting:: CSRF_COOKIE_AGE

``CSRF_COOKIE_AGE``
-------------------

默认值: ``31449600`` (约1年, 以秒为单位)

CSRF cookie有效期, 单位秒.

设置长有效期的原因是为了避免用户关闭浏览器或将页面存入书签, 然后从浏览器缓存加载该页面的情况下出现问题.
没有长有效期的cookie在这种情况下表单提交将失败.

某些浏览器(特别是Internet Explorer)禁止使用持久性cookie, 或可能使cookie jar的索引在磁盘上损坏,
从而导致CSRF保护检查(有时会间歇性地)失败. 将此配置更改为 ``None``,
使用基于会话的CSRF Cookie, 它将Cookie保存在内存中而不是磁盘.

.. setting:: CSRF_COOKIE_DOMAIN

``CSRF_COOKIE_DOMAIN``
----------------------

默认值: ``None``

设置CSRF cookie的域. 这对于允许跨子域请求被排除在正常的跨站点伪造请求保护之外是很有用的.
它必须是字符串, 例如 ``".example.com"`` 允许一个子域上的POST表单请求被另一个子域的视图接受.

请注意, 这个配置并不意味着Django的CSRF保护就是是安全的, 不会受到跨子域的攻击. 请参见 :ref:`CSRF限制 <csrf-limitations>` 部分.

.. setting:: CSRF_COOKIE_HTTPONLY

``CSRF_COOKIE_HTTPONLY``
------------------------

默认值: ``False``

无论是否设置 ``HttpOnly`` 标志. 只要该选项设置为 ``True``,
客户端的JavaScript都将无法访问CSRF cookie.

这有助于防止恶意的JavaScript绕过CSRF保护. 如果启用此功能并需要通过Ajax请求发送CSRF token的值,
则JavaScript将需要从页面上隐藏的CSRF token input表单中提取该值, 而不是从cookie中提取.

有关 ``HttpOnly`` 的详细信息, 见 :setting:`SESSION_COOKIE_HTTPONLY`.

.. setting:: CSRF_COOKIE_NAME

``CSRF_COOKIE_NAME``
--------------------

默认值: ``'csrftoken'``

用于CSRF身份验证令牌的cookie的名称. 这可以是任何你想要的名字(前提是它与应用程序中的其他cookie名字不重复). 详见 :doc:`/ref/csrf`.

.. setting:: CSRF_COOKIE_PATH

``CSRF_COOKIE_PATH``
--------------------

默认值: ``'/'``

设置在CSRF cookie上的路径. 这应该与你的Django安装的URL路径相匹配, 或者是该路径的父级.

如果你有多个Django实例在同一个主机名下运行, 这个功能将很有用. 它们可以使用不同的cookie路径, 而且每个实例只能看到自己的CSRFcookie.

.. setting:: CSRF_COOKIE_SECURE

``CSRF_COOKIE_SECURE``
----------------------

默认值: ``False``

是否为CSRF Cookie使用secure Cookie. 如果设置为 ``True``,
cookie将会标记为"secure", 这意味着浏览器可能会确保Cookie仅使用HTTPS连接发送.

.. setting:: CSRF_FAILURE_VIEW

``CSRF_FAILURE_VIEW``
---------------------

默认值: ``'django.views.csrf.csrf_failure'``

当传入的请求被 :doc:`CSRF保护 </ref/csrf>` 拒绝时, 要使用的视图函数的点分隔路径. 该函数应具有以下签名::

    def csrf_failure(request, reason=""):
        ...

其中 ``reason`` 是一个表示拒绝原因的短消息(针对开发者或日志, 而不是终端用户). 它需要返回一个 :class:`~django.http.HttpResponseForbidden`.

``django.views.csrf.csrf_failure()`` 还接收一个额外参数 ``template_name``,
默认为 ``'403_csrf.html'``. 如果传入的模板存在, 那么将用来渲染页面.

.. versionchanged:: 1.10

   ``csrf_failure()`` 新增 ``template_name`` 参数和搜索名为 ``403_csrf.html`` 模板的行为.

.. setting:: CSRF_HEADER_NAME

``CSRF_HEADER_NAME``
--------------------

.. versionadded:: 1.9

默认值: ``'HTTP_X_CSRFTOKEN'``

CSRF认证的请求头名称.

与 ``request.META`` 中的其他HTTP首部一样, 名称会将所有字符转换为大写字母, 用下划线代替连字符,
并添加 ``'HTTP_'`` 前缀. 例如, 如果要让客户端发送了一个 ``'X-XSRF-TOKEN'`` 首部, 应配置为 ``'HTTP_X_XSRF_TOKEN'``.

.. setting:: CSRF_TRUSTED_ORIGINS

``CSRF_TRUSTED_ORIGINS``
------------------------

.. versionadded:: 1.9

默认值: ``[]`` (空列表)

信任的非安全请求(例如. ``POST``)源列表.
对于 :meth:`secure <django.http.HttpRequest.is_secure>` 非安全请求,
Django的CSRF保护机制要求该请求的 ``Referer`` 首部必须与 ``Host`` 首部中的来源匹配.
这样可以阻止例如从 ``subdomain.example.com`` 对 ``api.example.com`` 的 ``POST`` 请求.
如果你需要通过HTTPS的跨源非安全请求, 可以将 ``"subdomain.example.com"`` 添加到这个列表.
该配置还支持子域, 例如你可以添加 ``".example.com"``, 来允许从 ``example.com`` 的所有子域访问.

.. setting:: DATABASES

``DATABASES``
-------------

默认值: ``{}`` (空字段)

一个包含所有数据库配置的字典. 它是一个嵌套字典, 包含数据库别名和其对应的数据库配置选项的字典.

:setting:`DATABASES` 配置必须包含一个 ``default`` 数据库; 除此之外可以指定任何数量的数据库.

最简单的配置是使用SQLite单个数据库配置. 可以通过如下设置::

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.sqlite3',
            'NAME': 'mydatabase',
        }
    }

如果要使用其他数据库后端, 例如MySQL, Oracle和
PostgreSQL, 将需要额外的连接参数. 如何指定其他类型数据库请查看下面的 :setting:`ENGINE <DATABASE-ENGINE>` 设置.
下面是一个PostgreSQL例子::

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql',
            'NAME': 'mydatabase',
            'USER': 'mydatabaseuser',
            'PASSWORD': 'mypassword',
            'HOST': '127.0.0.1',
            'PORT': '5432',
        }
    }

更复杂的配置可能需要以下内部选项:

.. setting:: DATABASE-ATOMIC_REQUESTS

``ATOMIC_REQUESTS``
~~~~~~~~~~~~~~~~~~~

默认值: ``False``

如果设置为 ``True``, 同一个请求对应的所有sql都在一个事务中执行. 详见
:ref:`tying-transactions-to-http-requests`.

.. setting:: DATABASE-AUTOCOMMIT

``AUTOCOMMIT``
~~~~~~~~~~~~~~

默认值: ``True``

如果想 :ref:`禁用Django事务管理 <deactivate-transaction-management>` 并自己实现, 请将此设置为  ``False``.

.. setting:: DATABASE-ENGINE

``ENGINE``
~~~~~~~~~~

默认值: ``''`` (空字符串)

使用的数据库后端. 内置的数据库后端有:

* ``'django.db.backends.postgresql'``
* ``'django.db.backends.mysql'``
* ``'django.db.backends.sqlite3'``
* ``'django.db.backends.oracle'``

如果你不想使用Django的数据库后端, 可以为 ``ENGINE`` 配置自己数据库后端的完整路径 (例如. ``mypackage.backends.whatever``).

.. versionchanged:: 1.9

    在老的发行版中 ``django.db.backends.postgresql`` 也叫做
    ``django.db.backends.postgresql_psycopg2``. 为了向后兼容, 旧名称在新版本中仍然有效.

.. setting:: HOST

``HOST``
~~~~~~~~

默认值: ``''`` (空字符串)

连接目标数据库的host. 空字符串表示localhost. SQLite不需要此选项.

如果该值以斜杠(``'/'``)开头并且使用的是MySQL, MySQL将通过Unix套接字连接. 例如::

    "HOST": '/var/run/mysql'

如果你使用的是MySQL且该值 **不** 以斜杠开头, 那么该值应该是host.

如果你使用的是PostgreSQL, 默认情况下(空的 :setting:`HOST`), 与数据库的连接是通过 UNIX domain套接字 (
``pg_hba.conf`` 中的 'local' 行)完成的. 如果你的UNIX domain socket不在此位置, 则使用
``postgresql.conf`` 中的 ``unix_socket_directory``.
如果想使用TCP sockets连接, 设置 :setting:`HOST` 为 'localhost'
或者 '127.0.0.1' (``pg_hba.conf`` 中的 'host' 行).
在Windows上, 你必须设置 :setting:`HOST`, 因为UNIX domain sockets不可用.

.. setting:: NAME

``NAME``
~~~~~~~~

默认值: ``''`` (空字符串)

将使用的数据库名称. 对于SQLite, 它是完整的数据库文件路径. 路径中应使用斜线, 即便是Windows上也是如此
(例如. ``C:/homes/user/mysite/sqlite3.db``).

.. setting:: CONN_MAX_AGE

``CONN_MAX_AGE``
~~~~~~~~~~~~~~~~

默认值: ``0``

数据库连接寿命, 单位秒. ``0`` 表示在每次请求结束时关闭数据库连接 — Django的历史行为 —
``None`` 表示无限保持连接.

.. setting:: OPTIONS

``OPTIONS``
~~~~~~~~~~~

默认值: ``{}`` (空字典)

连接数据库时的额外参数. 可用的参数取决于使用的数据库后端.

可用的额外参数信息见 :doc:`数据库后端 </ref/databases>` 文档. 更多信息请查阅后端模块所属文档.

.. setting:: PASSWORD

``PASSWORD``
~~~~~~~~~~~~

默认值: ``''`` (空字符串)

连接数据库使用的密码. SQLite不需要此选项.

.. setting:: PORT

``PORT``
~~~~~~~~

默认值: ``''`` (空字符串)

连接数据库的端口. 空字符串表示默认端口. SQLite不需要此选项.

.. setting:: DATABASE-TIME_ZONE

``TIME_ZONE``
~~~~~~~~~~~~~

.. versionadded:: 1.9

默认值: ``None``

表示存储在此数据库中的日期时间的时区的字符串(假设数据库不支持时区)或 ``None``. 接受与常规 :setting:`TIME_ZONE` 设置中相同的值.

这允许与以本地时间而不是UTC存储日期时间的数据库进行交互. 为了避免DST更改带来的问题, 你不应该为Django管理的数据库设置此选项.

设置此选项需要安装 pytz_ 库.

当 :setting:`USE_TZ` 设置为 ``True`` 时, 数据库不支持时区(例如SQLite, MySQL, Oracle)时,
Django根据此选项(如果设置)以本地时间读取和写入日期时间, 如果未设置, 则以UTC时间读取和写入日期时间.

当 :setting:`USE_TZ` 设置为 ``True`` 时,  数据库支持时区(例如.
PostgreSQL)时, 设置此选项是错误的.

.. versionchanged:: 1.9

    在Django 1.9之前, PostgreSQL数据库后端接受未记录的 ``TIME_ZONE`` 选项, 这导致数据corruption.

当 :setting:`USE_TZ` 设置为 ``False``, 不应该设置此选项.

.. _pytz: http://pytz.sourceforge.net/

.. setting:: USER

``USER``
~~~~~~~~

默认值: ``''`` (空字符串)

连接数据库的用户名. SQLite不需要此选项.

.. setting:: DATABASE-TEST

``TEST``
~~~~~~~~

默认值: ``{}`` (空字典)

测试数据库的配置信息字典; 关于测试数据库的创建和使用的详细信息, 请参考 :ref:`the-test-database`.

下面是一个测试数据库的配置例子::

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql',
            'USER': 'mydatabaseuser',
            'NAME': 'mydatabase',
            'TEST': {
                'NAME': 'mytestdatabase',
            },
        },
    }

``TEST`` 字典可用的键如下:

.. setting:: TEST_CHARSET

``CHARSET``
^^^^^^^^^^^

默认值: ``None``

创建测试数据库使用的字符集编码. 该值是直接传递给数据库的, 所以它的格式取决于特定后端.

支持 PostgreSQL_ (``postgresql``) 和 MySQL_ (``mysql``) 后端.

.. _PostgreSQL: https://www.postgresql.org/docs/current/static/multibyte.html
.. _MySQL: https://dev.mysql.com/doc/refman/en/charset-database.html

.. setting:: TEST_COLLATION

``COLLATION``
^^^^^^^^^^^^^

默认值: ``None``

创建测试数据库要使用的字符顺序. 该值是直接传递给数据库的, 所以它的格式取决于特定后端.

仅支持 ``mysql`` 后端 (详见 `MySQL manual`_).

.. _MySQL manual: MySQL_

.. setting:: TEST_DEPENDENCIES

``DEPENDENCIES``
^^^^^^^^^^^^^^^^

默认值: ``['default']``, 对于除了 ``default`` 外的所有数据库, 没有依赖关系.

数据库的创建顺序依赖性. 详见 :ref:`控制测试数据库的创建顺序 <topics-testing-creation-dependencies>` 文档.

.. setting:: TEST_MIRROR

``MIRROR``
^^^^^^^^^^

默认值: ``None``

数据库在测试期间映射的数据库别名.

该配置允许测试多个数据库的主/副本(某些数据库称为主/从)配置. 有关详细信息,
请参阅 :ref:`测试 主/副 配置 <topics-testing-primaryreplica>` 文档.

.. setting:: TEST_NAME

``NAME``
^^^^^^^^

默认值: ``None``

测试时使用的数据库名称

如果SQLite数据库引擎使用默认值(``None``), 测试将使用内存数据库. 对于其他数据库引擎, 测试数据库将使用名称 ``'test_' + DATABASE_NAME``.

详见 :ref:`the-test-database`.

.. setting:: TEST_SERIALIZE

``SERIALIZE``
^^^^^^^^^^^^^

布尔值, 用于控制测试程序是否在运行之前将数据库序列化为内存中的JSON字符串(用于在没有事务的情况下在测试之间恢复数据库状态).
如果没有任何测试类 :ref:`serialized_rollback=True <test-case-serialized-rollback>`, 可以将其设置为 ``False`` 以加快创建时间.

.. setting:: TEST_CREATE

``CREATE_DB``
^^^^^^^^^^^^^

默认值: ``True``

这是一个Oracle特有配置.

如果设置为 ``False``, 测试表空间不会在测试开始时自动创建, 也不会在测试结束时删除.

.. setting:: TEST_USER_CREATE

``CREATE_USER``
^^^^^^^^^^^^^^^

默认值: ``True``

这是一个Oracle特有配置.

如果设置为 ``False``, 测试用户不会在测试开始时自动创建, 也不会在测试结束时删除.

.. setting:: TEST_USER

``USER``
^^^^^^^^

默认值: ``None``

这是一个Oracle特有配置.

连接Oracle数据库时使用的用户名. 如果没有特别设置, 将使用 ``'test_' + USER``.

.. setting:: TEST_PASSWD

``PASSWORD``
^^^^^^^^^^^^

默认值: ``None``

这是一个Oracle特有配置.

连接Oracle数据库时使用的密码. 如果没有特别设置, Django将生成随机密码.

.. versionchanged:: 1.10.3

    旧版本使用硬编码的默认密码. 这在1.10.3, 1.9.11和1.8.16也有变化, 以解决可能的安全隐患.

.. setting:: TEST_TBLSPACE

``TBLSPACE``
^^^^^^^^^^^^

默认值: ``None``

这是一个Oracle特有配置.

运行测试时使用的表空间的名称. 如果没有特别设置, Django将使用 ``'test_' + USER``.

.. setting:: TEST_TBLSPACE_TMP

``TBLSPACE_TMP``
^^^^^^^^^^^^^^^^

默认值: ``None``

这是一个Oracle特有配置.

运行测试时使用的临时表空间的名称. 如果没有特别设置, Django将使用 ``'test_' + USER + '_temp'``.

.. setting:: DATAFILE

``DATAFILE``
^^^^^^^^^^^^

默认值: ``None``

这是一个Oracle特有配置.

TBLSPACE使用的数据文件名. 如果没有特别设置, Django将使用 ``TBLSPACE + '.dbf'``.

.. setting:: DATAFILE_TMP

``DATAFILE_TMP``
^^^^^^^^^^^^^^^^

默认值: ``None``

这是一个Oracle特有配置.

TBLSPACE_TMP使用的数据文件名. 如果没有特别设置, Django将使用 ``TBLSPACE_TMP + '.dbf'``.

.. setting:: DATAFILE_MAXSIZE

``DATAFILE_MAXSIZE``
^^^^^^^^^^^^^^^^^^^^

默认值: ``'500M'``

这是一个Oracle特有配置.

DATAFILE允许的最大大小.

.. setting:: DATAFILE_TMP_MAXSIZE

``DATAFILE_TMP_MAXSIZE``
^^^^^^^^^^^^^^^^^^^^^^^^

默认值: ``'500M'``

这是一个Oracle特有配置.

DATAFILE_TMP允许的最大大小.

.. setting:: DATA_UPLOAD_MAX_MEMORY_SIZE

DATA_UPLOAD_MAX_MEMORY_SIZE
---------------------------

.. versionadded:: 1.10

默认值: ``2621440`` (i.e. 2.5 MB).

请求正文最大字节大小, 超出将引发 :exc:`~django.core.exceptions.SuspiciousOperation` (``RequestDataTooBig``).
该检查在访问 ``request.body`` 或 ``request.POST`` 时进行, 根据总请求大小(不包括文件上传数据)计算.
可以将其设置为 ``None`` 以禁用此检查. 希望接收超大的表单请求的应用应该调整此配置.

请求的数据量与处理请求和填充GET和POST字典所需的内存容量有关. 如果不检查, 超大请求可以用作拒绝服务攻击载体.
由于web服务器通常不会执行深层的请求检查, 因此不可能在该级别执行类似的检查.

详见 :setting:`FILE_UPLOAD_MAX_MEMORY_SIZE`.

.. setting:: DATA_UPLOAD_MAX_NUMBER_FIELDS

DATA_UPLOAD_MAX_NUMBER_FIELDS
-----------------------------

.. versionadded:: 1.10

默认值: ``1000``

允许的最多参数数量, 超出将引发 :exc:`~django.core.exceptions.SuspiciousOperation` (``TooManyFields``) 异常.
可以将其设置为 ``None`` 以禁用此检查. 预计会收到超多表单字段的应用程序应该调整这个配置.

请求的数据量与处理请求和填充GET和POST字典所需的内存容量有关. 如果不检查, 超大请求可以用作拒绝服务攻击载体.
由于web服务器通常不会执行深层的请求检查, 因此不可能在该级别执行类似的检查.

.. setting:: DATABASE_ROUTERS

``DATABASE_ROUTERS``
--------------------

默认值: ``[]`` (空列表)

用于在执行数据库查询时确定要使用哪个数据库的路由列表.

详见 :ref:`多数据库配置中的自动数据库路由 <topics-db-multi-db-routing>` 文档.

.. setting:: DATE_FORMAT

``DATE_FORMAT``
---------------

默认值: ``'N j, Y'`` (e.g. ``Feb. 4, 2003``)

在系统的任何显示日期字段部分中使用的默认格式. 注意, 如果 :setting:`USE_L10N` 设置为 ``True``, 那么本地设置的格式具有更高的优先权, 并将被应用.
详见 :tfilter:`允许的日期格式字符串 <date>`.

另见 :setting:`DATETIME_FORMAT`, :setting:`TIME_FORMAT` 和 :setting:`SHORT_DATE_FORMAT`.

.. setting:: DATE_INPUT_FORMATS

``DATE_INPUT_FORMATS``
----------------------

默认值::

    [
        '%Y-%m-%d', '%m/%d/%Y', '%m/%d/%y', # '2006-10-25', '10/25/2006', '10/25/06'
        '%b %d %Y', '%b %d, %Y',            # 'Oct 25 2006', 'Oct 25, 2006'
        '%d %b %Y', '%d %b, %Y',            # '25 Oct 2006', '25 Oct, 2006'
        '%B %d %Y', '%B %d, %Y',            # 'October 25 2006', 'October 25, 2006'
        '%d %B %Y', '%d %B, %Y',            # '25 October 2006', '25 October, 2006'
    ]

日期字段输入数据时接受的格式列表. 将按顺序尝试格式, 使用第一个有效的格式.
注意这些格式字符串使用Python的是 :ref:`datetime模块语法
<strftime-strptime-behavior>`, 而不是 :tfilter:`date` 模板过滤器语法.

当 :setting:`USE_L10N` 设置为 ``True`` 时, 将采用本地设置的格式, 具有更高的优先级.

另见 :setting:`DATETIME_INPUT_FORMATS` 和 :setting:`TIME_INPUT_FORMATS`.

.. setting:: DATETIME_FORMAT

``DATETIME_FORMAT``
-------------------

默认值: ``'N j, Y, P'`` (e.g. ``Feb. 4, 2003, 4 p.m.``)

在系统的显示datetime字段部分中使用的默认格式. 注意, 如果 :setting:`USE_L10N` 设置为 ``True``, 那么本地设置的格式具有更高的优先权, 并将被应用.
详见 :tfilter:`允许的日期格式字符串 <date>`.

另见 :setting:`DATE_FORMAT`, :setting:`TIME_FORMAT` 和 :setting:`SHORT_DATETIME_FORMAT`.

.. setting:: DATETIME_INPUT_FORMATS

``DATETIME_INPUT_FORMATS``
--------------------------

默认值::

    [
        '%Y-%m-%d %H:%M:%S',     # '2006-10-25 14:30:59'
        '%Y-%m-%d %H:%M:%S.%f',  # '2006-10-25 14:30:59.000200'
        '%Y-%m-%d %H:%M',        # '2006-10-25 14:30'
        '%Y-%m-%d',              # '2006-10-25'
        '%m/%d/%Y %H:%M:%S',     # '10/25/2006 14:30:59'
        '%m/%d/%Y %H:%M:%S.%f',  # '10/25/2006 14:30:59.000200'
        '%m/%d/%Y %H:%M',        # '10/25/2006 14:30'
        '%m/%d/%Y',              # '10/25/2006'
        '%m/%d/%y %H:%M:%S',     # '10/25/06 14:30:59'
        '%m/%d/%y %H:%M:%S.%f',  # '10/25/06 14:30:59.000200'
        '%m/%d/%y %H:%M',        # '10/25/06 14:30'
        '%m/%d/%y',              # '10/25/06'
    ]

datetime字段输入数据时接受的格式列表. 将按顺序尝试格式, 使用第一个有效的格式.
注意这些格式字符串使用Python的是 :ref:`datetime模块语法
<strftime-strptime-behavior>`, 而不是 :tfilter:`date` 模板过滤器语法.

当 :setting:`USE_L10N` 设置为 ``True`` 时, 将采用本地设置的格式, 具有更高的优先级.


另见 :setting:`DATE_INPUT_FORMATS` 和 :setting:`TIME_INPUT_FORMATS`.

.. setting:: DEBUG

``DEBUG``
---------

默认值: ``False``

打开/关闭调试模式的布尔值.

千万不在生产部署时开启 :setting:`DEBUG`.

重要事情说三遍, 千万不在生产部署时开启 :setting:`DEBUG`.

调试模式的重要功能之一是显示详细的错误页面.
当 :setting:`DEBUG` 为 ``True`` 且你的应用发生异常时,
Django会显示追溯细节, 包括你环境的元数据, 比如所有Django当前设置(``settings.py`` 中).

为了安全, Django **不会** 显示一些敏感设置, 比如 :setting:`SECRET_KEY`. 特别是名字中包含下面单词的设置:

* ``'API'``
* ``'KEY'``
* ``'PASS'``
* ``'SECRET'``
* ``'SIGNATURE'``
* ``'TOKEN'``

注意, 这里使用的是 **部分** 匹配. 比如 ``'PASS'`` 会匹配到PASSWORD,
``'TOKEN'`` 会匹配到TOKENIZED等等.

尽管如此, 调试输出中还是会有一部分内容是不适合公开的. 比如文件路径, 配置选项等, 这些都会给攻击者提供关于你服务器的额外信息.

同样重要的是, 当 :setting:`DEBUG` 开启时, Django会记住它执行的每个SQL查询. 这在调试时非常有用, 但这会消耗运行服务器的大量内存资源.

最后, 当 :setting:`DEBUG` 设置为 ``False`` 时, 你必须要设置
:setting:`ALLOWED_HOSTS` 选项. 否则所有的请求都会返回"Bad Request (400)".

.. note::

    为方便起见 :djadmin:`django-admin
    startproject <startproject>` 创建的默认 :file:`settings.py` 文件中,  ``DEBUG = True``.

.. _django/views/debug.py: https://github.com/django/django/blob/master/django/views/debug.py

.. setting:: DEBUG_PROPAGATE_EXCEPTIONS

``DEBUG_PROPAGATE_EXCEPTIONS``
------------------------------

默认值: ``False``

如果设置为True, Django对视图函数的正常异常将被跳过, 异常将向上传播. 这对于某些测试设置非常有用, 不要在实时站点上使用.

.. setting:: DECIMAL_SEPARATOR

``DECIMAL_SEPARATOR``
---------------------

默认值: ``'.'`` (点号)

格式化十进制数时使用的默认分隔符.

注意, 如果 :setting:`USE_L10N` 设置为 ``True``, 那么将采用本地设置的格式, 具有更高的优先级别.

另见 :setting:`NUMBER_GROUPING`, :setting:`THOUSAND_SEPARATOR` 和
:setting:`USE_THOUSAND_SEPARATOR`.


.. setting:: DEFAULT_CHARSET

``DEFAULT_CHARSET``
-------------------

默认值: ``'utf-8'``

没有指定MIME类型时, ``HttpResponse`` 对象使用的编码字符集. 与 :setting:`DEFAULT_CONTENT_TYPE` 配合使用构造 ``Content-Type`` 首部.

.. setting:: DEFAULT_CONTENT_TYPE

``DEFAULT_CONTENT_TYPE``
------------------------

默认值: ``'text/html'``

没有指定MIME类型时, ``HttpResponse`` 对象的默认内容类型. 与 :setting:`DEFAULT_CHARSET` 配合使用构造 ``Content-Type`` 首部.

.. setting:: DEFAULT_EXCEPTION_REPORTER_FILTER

``DEFAULT_EXCEPTION_REPORTER_FILTER``
-------------------------------------

默认值: ``'``:class:`django.views.debug.SafeExceptionReporterFilter`\ ``'``

没有为 :class:`~django.http.HttpRequest` 实例分配异常报告过滤类时, 默认的异常报告过滤类.
见 :ref:`Filtering error reports<filtering-error-reports>`.

.. setting:: DEFAULT_FILE_STORAGE

``DEFAULT_FILE_STORAGE``
------------------------

默认值: ``'``:class:`django.core.files.storage.FileSystemStorage`\ ``'``

默认文件存储类, 用于不指定特定存储系统的文件相关的操作. 见 :doc:`/topics/files`.

.. setting:: DEFAULT_FROM_EMAIL

``DEFAULT_FROM_EMAIL``
----------------------

默认值: ``'webmaster@localhost'``

站点管理员的各种自动通信的默认电子邮件地址. 不包括发送到 :setting:`ADMINS`
和 :setting:`MANAGERS` 的错误信息; 详见 :setting:`SERVER_EMAIL`.

.. setting:: DEFAULT_INDEX_TABLESPACE

``DEFAULT_INDEX_TABLESPACE``
----------------------------

默认值: ``''`` (空字符串)

没有指定索引的字段使用的默认表空间, 需要数据库引擎支持(见 :doc:`/topics/db/tablespaces`).

.. setting:: DEFAULT_TABLESPACE

``DEFAULT_TABLESPACE``
----------------------

默认值: ``''`` (空字符串)

没有指定表空间的模型使用的默认表空间, 需要数据库引擎支持(见 :doc:`/topics/db/tablespaces`).

.. setting:: DISALLOWED_USER_AGENTS

``DISALLOWED_USER_AGENTS``
--------------------------

默认值: ``[]`` (空列表)

编译后的正则表达式对象列表, 代表不允许访问任何页面的User-Agent字符串.
用于robots/crawlers. 只有在安装了 ``CommonMiddleware`` 的情况下才会使用(
见 :doc:`/topics/http/middleware`).

.. setting:: EMAIL_BACKEND

``EMAIL_BACKEND``
-----------------

默认值: ``'``:class:`django.core.mail.backends.smtp.EmailBackend`\ ``'``

用于发送邮件的引擎. 关于可用的引擎列表 :doc:`/topics/email`.

.. setting:: EMAIL_FILE_PATH

``EMAIL_FILE_PATH``
-------------------

默认值: 无默认值

``file`` 类型的邮件引擎保存输出文件时使用的目录.

.. setting:: EMAIL_HOST

``EMAIL_HOST``
--------------

默认值: ``'localhost'``

发送邮件的host.

另见 :setting:`EMAIL_PORT`.

.. setting:: EMAIL_HOST_PASSWORD

``EMAIL_HOST_PASSWORD``
-----------------------

默认值: ``''`` (空字符串)

:setting:`EMAIL_HOST` 定义的SMTP服务器使用的密码.
该配置和 :setting:`EMAIL_HOST_USER` 一起用于SMTP服务器的认证. 如果两个中有一个为空, Django则不会进行认证.

另见 :setting:`EMAIL_HOST_USER`.

.. setting:: EMAIL_HOST_USER

``EMAIL_HOST_USER``
-------------------

默认值: ``''`` (空字符串)

:setting:`EMAIL_HOST` 定义的SMTP服务器使用的用户名. 如果为空Django不会进行认证.

另见 :setting:`EMAIL_HOST_PASSWORD`.

.. setting:: EMAIL_PORT

``EMAIL_PORT``
--------------

默认值: ``25``

:setting:`EMAIL_HOST` 定义的SMTP服务器使用的端口.

.. setting:: EMAIL_SUBJECT_PREFIX

``EMAIL_SUBJECT_PREFIX``
------------------------

默认值: ``'[Django] '``

``django.core.mail.mail_admins`` 和 ``django.core.mail.mail_managers`` 发送邮件的主题前缀. 最好以空格结尾.

.. setting:: EMAIL_USE_TLS

``EMAIL_USE_TLS``
-----------------

默认值: ``False``

是否使用TLS(更安全)连接SMTP服务. 这用于显示的TLS连接, 通常端口为587.
如果你遇到挂起的连接, 请查看隐式TLS配置
:setting:`EMAIL_USE_SSL`.

.. setting:: EMAIL_USE_SSL

``EMAIL_USE_SSL``
-----------------

默认值: ``False``

是否使用隐式TLS(更安全)连接SMTP服务. 在大多数电子邮件文档中, 该类型的TLS连接也被称为SSL.
它通常使用465. 如果遇到问题可以尝试显式的TLS配置 :setting:`EMAIL_USE_TLS`.

注意 :setting:`EMAIL_USE_TLS`/:setting:`EMAIL_USE_SSL` 是互斥的, 因此它们只能有一个设置为 ``True``.

.. setting:: EMAIL_SSL_CERTFILE

``EMAIL_SSL_CERTFILE``
----------------------

默认值: ``None``

:setting:`EMAIL_USE_SSL` 或 :setting:`EMAIL_USE_TLS` 设置为 ``True`` 时, 用于指定PEM格式的证书链文件路径的可选配置, 用于SSL连接.

.. setting:: EMAIL_SSL_KEYFILE

``EMAIL_SSL_KEYFILE``
---------------------

默认值: ``None``

:setting:`EMAIL_USE_SSL` 或 :setting:`EMAIL_USE_TLS` 设置为 ``True`` 时, 用于指定PEM格式的私钥文件路径的可选配置, 用于SSL连接.

注意, 配置 :setting:`EMAIL_SSL_CERTFILE` 和 :setting:`EMAIL_SSL_KEYFILE` 后不会做相应的证书检查, 它们会直接被传递给底层的SSL连接.
请参考Python的
:func:`python:ssl.wrap_socket` 函数, 了解如何正确使用证书链文件和私钥文件.

.. setting:: EMAIL_TIMEOUT

``EMAIL_TIMEOUT``
-----------------

默认值: ``None``

指定尝试连接操作的超时时间来中断连接, 单位秒.

.. setting:: FILE_CHARSET

``FILE_CHARSET``
----------------

默认值: ``'utf-8'``

用于解码从磁盘读取的文件时使用的字符编码. 它包括模板文件和初始SQL数据文件.

.. setting:: FILE_UPLOAD_HANDLERS

``FILE_UPLOAD_HANDLERS``
------------------------

默认值::

    [
        'django.core.files.uploadhandler.MemoryFileUploadHandler',
        'django.core.files.uploadhandler.TemporaryFileUploadHandler',
    ]

用户上传操作的处理程序列表. 更改此配置可以完全自定义 - 甚至替换Django的上传处理.

详见 :doc:`/topics/files`.

.. setting:: FILE_UPLOAD_MAX_MEMORY_SIZE

``FILE_UPLOAD_MAX_MEMORY_SIZE``
-------------------------------

默认值: ``2621440`` (即 2.5 MB).

上传到文件系统之前文件的最大大小(单位字节). 详见 :doc:`/topics/files`.

另见 :setting:`DATA_UPLOAD_MAX_MEMORY_SIZE`.

.. setting:: FILE_UPLOAD_DIRECTORY_PERMISSIONS

``FILE_UPLOAD_DIRECTORY_PERMISSIONS``
-------------------------------------

默认值: ``None``

用于上传文件时创建的目录权限的数字模式.

此配置还会使用在 :djadmin:`collectstatic` 管理命令收集的静态目录的默认权限. 如果要覆盖它见 :djadmin:`collectstatic`.

该值反映了 :setting:`FILE_UPLOAD_PERMISSIONS` 配置的功能的注意事项.

.. setting:: FILE_UPLOAD_PERMISSIONS

``FILE_UPLOAD_PERMISSIONS``
---------------------------

默认值: ``None``

新上传文件权限的数字模式 (例如 ``0o644``). 有关这些模式的含义请参见 :func:`os.chmod`.

如果没有设置或者设置为 ``None``, 这依赖于操作系统行为. 在大多数平台上, 临时文件模式为 ``0o600``,
从内存中保存的文件使用系统标准的umask保存.

出于安全考虑, 这些权限不会应用在储存在 :setting:`FILE_UPLOAD_TEMP_DIR` 的临时文件.

该配置也会影响使用 :djadmin:`collectstatic` 管理命令收集的静态文件的默认权限. 详见 :djadmin:`collectstatic`.

.. warning::

    **一定要在模式前加上 0.**

    如果你不熟悉文件模式, 一定要注意前缀的 ``0`` 非常重要: 它表示一个八进制数,
    这是模式必须指定的. 如果你尝试使用 ``644``, 这是完全错误的行为.

.. setting:: FILE_UPLOAD_TEMP_DIR

``FILE_UPLOAD_TEMP_DIR``
------------------------

默认值: ``None``

上传文件时(通常大于 :setting:`FILE_UPLOAD_MAX_MEMORY_SIZE` 的文件)储存数据的临时目录.
如果设置为 ``None``, Django将使用操作系统的默认临时目录. 例如, 类 \*nix 风格的操作系统上将使用 ``/tmp`` 目录.

详见 :doc:`/topics/files`.

.. setting:: FIRST_DAY_OF_WEEK

``FIRST_DAY_OF_WEEK``
---------------------

默认值: ``0`` (星期日)

代表一周第一天的数字. 这在显示日历时特别有用. 该值仅在不使用格式国际化或找不到当前语言环境的格式时使用.

该配置的值必须为0到6的整数, 0代表星期日, 1代表星期一, 以此类推.

.. setting:: FIXTURE_DIRS

``FIXTURE_DIRS``
-----------------

默认值: ``[]`` (空列表)

除每个应用程序的 ``fixtures`` 目录外, 按搜索顺序搜索 ``fixtures`` 文件的目录列表.

注意, 该路径应该使用Unix风格的斜线, 即使在Windows上也是如此.

详见 :ref:`initial-data-via-fixtures` 和 :ref:`topics-testing-fixtures`.

.. setting:: FORCE_SCRIPT_NAME

``FORCE_SCRIPT_NAME``
---------------------

默认值: ``None``

如果不为 ``None``, 将作为所有HTTP请求中 ``SCRIPT_NAME`` 环境变量的值. 这个配置可以用来覆盖服务器的 ``SCRIPT_NAME`` 值,
这个值可以是首选值的重写, 也可以直接不设置. 它还被用于 :func:`django.setup()` 在请求/响应周期外的URL前缀(例如. 管理命令和独立脚本)
以便在 ``SCRIPT_NAME`` 不为 ``/`` 时生成正确的URL.

.. versionchanged:: 1.10

    新增该配置在 :func:`django.setup()` 中的应用.

.. setting:: FORMAT_MODULE_PATH

``FORMAT_MODULE_PATH``
----------------------

默认值: ``None``

Python包的完整Python路径, 其中包含项目语言环境的格式定义. 如果不为 ``None``,
Django将在名为当前语言环境的目录下检查 ``formats.py`` 文件, 并使用此文件中定义的格式.

例如, 如果 :setting:`FORMAT_MODULE_PATH` 设置为 ``mysite.formats``, 并且当前语言环境为 ``en`` (英语),
Django需要这样一个目录树::

    mysite/
        formats/
            __init__.py
            en/
                __init__.py
                formats.py

也可以使用列表设置多个Python路径, 例如::

    FORMAT_MODULE_PATH = [
        'mysite.formats',
        'some_app.formats',
    ]

Django搜索某个格式时, 它会遍历所有给出的Python路径, 直接找到实际定义该格式的模块.
这意味着在列表中靠前的包中定义的格式将优先于靠后的包中的相同格式.

可用格式有 :setting:`DATE_FORMAT`, :setting:`TIME_FORMAT`,
:setting:`DATETIME_FORMAT`, :setting:`YEAR_MONTH_FORMAT`,
:setting:`MONTH_DAY_FORMAT`, :setting:`SHORT_DATE_FORMAT`,
:setting:`SHORT_DATETIME_FORMAT`, :setting:`FIRST_DAY_OF_WEEK`,
:setting:`DECIMAL_SEPARATOR`, :setting:`THOUSAND_SEPARATOR` 和
:setting:`NUMBER_GROUPING`.

.. setting:: IGNORABLE_404_URLS

``IGNORABLE_404_URLS``
----------------------

默认值: ``[]`` (空列表)

编译的正则表达式对象列表, 表示电子邮件报告HTTP404错误时应该被忽略的URL(见
:doc:`/howto/error-reporting`). 正则表达式与
:meth:`请求的完整路径 <django.http.HttpRequest.get_full_path>` (包含查询字符串)匹配.
如果你的网站没有提供常用的请求文件, 例如 ``favicon.ico`` 和 ``robots.txt``, 请使用此方法.

只有在启用
:class:`~django.middleware.common.BrokenLinkEmailsMiddleware` 时才能使用此功能(见
:doc:`/topics/http/middleware`).

.. setting:: INSTALLED_APPS

``INSTALLED_APPS``
------------------

默认值: ``[]`` (空列表)

字符串列表, 表示项目中所有启用的应用. 每一个字符串都是Python的点分隔路径:

* 应用程序配置类(首选), 或
* 包含应用程序的包.

:doc:`更多相关应用配置 </ref/applications>`.

.. admonition:: 使用应用程序注册进行自我检查

    你的代码不应该直接访问 :setting:`INSTALLED_APPS`. 请改用 :attr:`django.apps.apps`.

.. admonition:: :setting:`INSTALLED_APPS` 中的应用名称和label必须是唯一的

    应用的 :attr:`names <django.apps.AppConfig.name>` — 应用程序包的点分隔Python路径必须是唯一的.
    没有办法包含两个相同的应用程序, 除非用另一个名称并复制它的代码.

    应用的 :attr:`labels <django.apps.AppConfig.label>` — 默认情况下名称的后面部分 — 也必须是唯一的.
    例如, 你不可以同时包含 ``django.contrib.auth`` 和 ``myproject.auth``. 但是,
    你可以使用定义不同 :attr:`~django.apps.AppConfig.label` 的自定义配置重新标记应用程序.

    无论 :setting:`INSTALLED_APPS` 引用的是应用配置类还是应用程序包, 这些规则都适用.

当多个应用程序提供相同资源(模板, 静态文件, 管理命令, 翻译)的不同版本时,
:setting:`INSTALLED_APPS` 中排在第一位的应用程序具有优先权.

.. setting:: INTERNAL_IPS

``INTERNAL_IPS``
----------------

默认值: ``[]`` (空列表)

IP字符串的列表, 它:

* 允许 :func:`~django.template.context_processors.debug` 上下文处理器向模板上下文添加一些变量.
* 即使不以员工用户身份登录, 也可以使用 :ref:`管理文档书签 <admindocs-bookmarklets>`.
* 在 :class:`~django.utils.log.AdminEmailHandler` 邮件中被标记为 "internal" (相对"EXTERNAL").

.. setting:: LANGUAGE_CODE

``LANGUAGE_CODE``
-----------------

默认值: ``'en-us'``

表示安装的语言代码的字符串. 它必须是标准的 :term:`语言ID格式 <language code>`. 例如, U.S. English
是 ``"en-us"``. 详见 `语言标识符列表`_ 和
:doc:`/topics/i18n/index`.

:setting:`USE_I18N` 配置必须是启用状态该设置才会生效.

它有两个作用:

* 如果没有使用locale中间件, 它决定向用户提供哪种翻译.
* 如果locale中间件是启用的, 它提供了一个后备语言, 以防用户的首选语言无法确定或网站不支持. 当用户的首选语言不存在给定字词的翻译时, 它也会提供后备翻译.

详见 :ref:`how-django-discovers-language-preference`.

.. _语言标识符列表: http://www.i18nguy.com/unicode/language-identifiers.html

.. setting:: LANGUAGE_COOKIE_AGE

``LANGUAGE_COOKIE_AGE``
-----------------------

默认值: ``None`` (浏览器关闭时失效)

语言cookie的有效期, 单位秒.

.. setting:: LANGUAGE_COOKIE_DOMAIN

``LANGUAGE_COOKIE_DOMAIN``
--------------------------

默认值: ``None``

语言cookie的域. 对于跨域cookie, 将其设置为
``".example.com"`` 之类的字符串(注意开头的点号!), 或者对于标准域cookie使用 ``None``.

在生产环境的网站上更新此配置时要谨慎. 如果你更新此配置, 在以前使用标准域cookie的网站上启用跨域cookie,
则现有的具有旧域的用户cookie将不会被更新. 这将导致网站用户无法切换语言,
只要这些cookie持续存在. 执行切换的唯一安全可靠的方案是永久更改语言cookie名称(通过 :setting:`LANGUAGE_COOKIE_NAME` 设置),
并添加一个中间件, 将旧cookie的值复制到新cookie中, 然后删除旧cookie.

.. setting:: LANGUAGE_COOKIE_NAME

``LANGUAGE_COOKIE_NAME``
------------------------

默认值: ``'django_language'``

用于语言cookie的名称. 这可以是任何你想要的(只要它与你的应用程序中的其他cookie名称不同). 见 :doc:`/topics/i18n/index`.

.. setting:: LANGUAGE_COOKIE_PATH

``LANGUAGE_COOKIE_PATH``
------------------------

默认值: ``'/'``

语言cookie的路径. 这个路径应该与Django安装的URL路径相匹配, 或者是该路径的父路径.

当你有多个Django实例在同一个主机下运行时, 这个功能很适用.
只要它们使用不同的cookie路径, 每个实例就只能看到自己的语言.

在生产环境的网站上更新此配置时要谨慎. 如果你更新此配置, 使用比以前下级的路径,
则现有的用户cookie的旧路径将不会被更新. 这将导致网站用户无法切换语言, 只要这些cookie持续存在.
执行切换的唯一安全可靠的方案是永久更改语言cookie的名称(通过 :setting:`LANGUAGE_COOKIE_NAME` 配置),
并添加一个中间件, 将旧cookie的值复制到新的cookie中, 然后删除这个cookie.

.. setting:: LANGUAGES

``LANGUAGES``
-------------

默认值: 所有可用语言的列表. 这个列表在不断的增加, 如果在这里有具体配置, 那么不可避免的会很快过时. 你可以在
``django/conf/global_settings.py`` (或查看 `在线源`_) 中查看当前翻译语言列表.

.. _在线源: https://github.com/django/django/blob/master/django/conf/global_settings.py

该列表是(:term:`language code<language code>`, ``language name``)这样的二元元组组成 -- 例如, ``('ja', 'Japanese')``.
这里列出了哪些可以选择的语言. 见 :doc:`/topics/i18n/index`.

一般来说, 使用默认值就可以了. 只有当你想将语言限制在指定的语言子集时才需要设置此配置.

如果定义了自定义的 :setting:`LANGUAGES` 设置, 可以使用 :func:`~django.utils.translation.ugettext_lazy` 函数将语言名称标记为翻译字符串.

下面是一个示例配置文件::

    from django.utils.translation import ugettext_lazy as _

    LANGUAGES = [
        ('de', _('German')),
        ('en', _('English')),
    ]

.. setting:: LOCALE_PATHS

``LOCALE_PATHS``
----------------

默认值: ``[]`` (空列表)

Django搜索翻译文件的目录列表.
见 :ref:`how-django-discovers-translations`.

例如::

    LOCALE_PATHS = [
        '/home/www/project/common_files/locale',
        '/var/local/translations/locale',
    ]

Django将在这些路径中搜索包含翻译文件的 ``<locale_code>/LC_MESSAGES`` 目录.

.. setting:: LOGGING

``LOGGING``
-----------

默认值: 日志配置字典.

一个包含配置信息的数据结构. 该数据结构的内容将作为参数传递给 :setting:`LOGGING_CONFIG` 中提到的配置方法.

其中, 当 :setting:`DEBUG` 为 ``False`` 时, 默认的日志配置会将HTTP500错误传递给电子邮件日志处理程序. 见 :ref:`configuring-logging`.

默认日志配置见
``django/utils/log.py`` (或在线 `查看`__).

__ https://github.com/django/django/blob/master/django/utils/log.py

.. setting:: LOGGING_CONFIG

``LOGGING_CONFIG``
------------------

默认值: ``'logging.config.dictConfig'``

用于配置日志的可调用路径. 默认指向Python的 :ref:`dictConfig
<logging-config-dictschema>` 配置方法的实例.

如果将 :setting:`LOGGING_CONFIG` 设置为 ``None``, 将跳过日志配置过程.

.. setting:: MANAGERS

``MANAGERS``
------------

默认值: ``[]`` (空列表)

一个与 :setting:`ADMINS` 格式相同的列表, 用于指定当启用
:class:`~django.middleware.common.BrokenLinkEmailsMiddleware` 时, 谁应该收到断链通知.

.. setting:: MEDIA_ROOT

``MEDIA_ROOT``
--------------

默认值: ``''`` (空字符串)

保存 :doc:`用户上传文件 </topics/files>` 的绝对文件系统路径.

例如: ``"/var/www/example.com/media/"``

另见 :setting:`MEDIA_URL`.

.. warning::

    :setting:`MEDIA_ROOT` 和 :setting:`STATIC_ROOT` 必须设置不同的值.
    在引入 :setting:`STATIC_ROOT` 之前, 通常依靠或回溯 :setting:`MEDIA_ROOT` 来提供静态文件,
    但是, 由于这可能会导致严重的安全隐患, 因此有这个检查来防止这种情况.

.. setting:: MEDIA_URL

``MEDIA_URL``
-------------

默认值: ``''`` (空字符串)

处理 :setting:`MEDIA_ROOT` 提供的多媒体URL, 用于 :doc:`管理存储的文件 </topics/files>`.
如果设置为非空值, 则必须以斜杠结尾. 在开发和生产环境中, 你都需要 :ref:`配置这些文件服务
<serving-uploaded-files-in-development>`.

如果要在模板中使用 ``{{ MEDIA_URL }}``, 需要在 :setting:`TEMPLATES` 的 ``'context_processors'`` 设置中添加
``'django.template.context_processors.media'``.

例如: ``"http://media.example.com/"``

.. warning::

    接收非授信用户的上传内容会有安全隐患! 详见 :ref:`user-uploaded-content-security`.

.. warning::

    :setting:`MEDIA_URL` 和 :setting:`STATIC_URL` 必须设置不同的值. 详见 :setting:`MEDIA_ROOT`.

.. setting:: MIDDLEWARE

``MIDDLEWARE``
--------------

.. versionadded:: 1.10

默认值:: ``None``

使用的中间件列表. 见 :doc:`/topics/http/middleware`.

.. setting:: MIDDLEWARE_CLASSES

``MIDDLEWARE_CLASSES``
----------------------

.. deprecated:: 1.10

    已经不建议再使用 ``settings.MIDDLEWARE_CLASSES`` 这种旧式中间件方式. :ref:`调整旧的, 自定义中间件 <upgrading-middleware>` 并
    使用 :setting:`MIDDLEWARE` 配置.

默认值::

    [
        'django.middleware.common.CommonMiddleware',
        'django.middleware.csrf.CsrfViewMiddleware',
    ]

使用的中间件列表. 这是Django 1.9及更早版本中使用的默认配置. Django 1.10引入了一种新风格的中间件.
如果较早项目中使用了此配置, 您应该 :ref:`将自己编写的所有中间件更新为新样式 <upgrading-middleware>` 然后使用 :setting:`MIDDLEWARE` 配置.

.. setting:: MIGRATION_MODULES

``MIGRATION_MODULES``
---------------------

默认值: ``{}`` (空字典)

指定每个应用迁移模块包位置的字典. 该配置的默认值是一个空字典, 迁移模块的默认包名为 ``migrations``.

例如::

    {'blog': 'blog.db_migrations'}

在此配置下, ``blog`` 应用相关的迁移都会存在于 ``blog.db_migrations`` 包中.

如果提供了 ``app_label`` , 包不存在时 :djadmin:`makemigrations` 将自动创建它.

.. versionadded:: 1.9

如果为某个应用设置``None``, Django将会视为此应用没有迁移, 而不考虑是否存在 ``migrations`` 子模块.
例如, 在测试配置文件中, 这可以用来在测试时跳过迁移(仍然会为应用程序的模型创建表).
如果这在常规项目配置中使用, 想为应用程序创建表, 请使用 :option:`migrate --run-syncdb` 选项.

.. setting:: MONTH_DAY_FORMAT

``MONTH_DAY_FORMAT``
--------------------

默认值: ``'F j'``

Django admin change-list页面上日期字段的默认格式 -- 可能还包括系统的其他部分 -- 在只显示月和日的情况下.

例如, 在Django admin change-list页面使用日期过滤时, 对目标日期的月和日在不同地区显示不同的格式.
例如, U.S. English 地区显示"January 1," 西班牙语地区显示 "1 Enero."

注意, 如果 :setting:`USE_L10N` 设置为 ``True``, 则对应的地区设置格式将具有更高优先级.

见 :tfilter:`允许的日期格式字符 <date>`. 也见
:setting:`DATE_FORMAT`, :setting:`DATETIME_FORMAT`,
:setting:`TIME_FORMAT` 和 :setting:`YEAR_MONTH_FORMAT`.

.. setting:: NUMBER_GROUPING

``NUMBER_GROUPING``
--------------------

默认值: ``0``

整数分割位数.

常见的用途是显示千位分隔符. 如果设置为 ``0`` 则不会进行分隔. 如果设置的大于
``0`` 则会使用 :setting:`THOUSAND_SEPARATOR` 分隔数字.

注意, 如果 :setting:`USE_L10N` 设置为 ``True``, 则对应的区域设置格式将具有更高优先级.

见 :setting:`DECIMAL_SEPARATOR`, :setting:`THOUSAND_SEPARATOR` 和
:setting:`USE_THOUSAND_SEPARATOR`.

.. setting:: PREPEND_WWW

``PREPEND_WWW``
---------------

默认值: ``False``

是否自动给没有 "www." 子域加上该前缀. 只有在使用了 :class:`~django.middleware.common.CommonMiddleware` 才会生效
(见 :doc:`/topics/http/middleware`). 也见 :setting:`APPEND_SLASH`.

.. setting:: ROOT_URLCONF

``ROOT_URLCONF``
----------------

默认值: 未定义

表示根URLconf的完整Python import路径. 例如: ``"mydjangoapps.urls"``.
可以通过设置请求的 ``HttpRequest`` 的 ``urlconf`` 属性来覆盖它. 详见 :ref:`how-django-processes-a-request`.

.. setting:: SECRET_KEY

``SECRET_KEY``
--------------

默认值: ``''`` (空字符串)

设置Django安装密钥. 用于提供 :doc:`加密签名 </topics/signing>`, 应设置为一个唯一的, 复杂的值.

:djadmin:`django-admin startproject <startproject>` 会自动向每个新项目添加一个随机生成的 ``SECRET_KEY``.

使用密钥时不应该直接认定它就是文本或字节. 每次使用都应该用
:func:`~django.utils.encoding.force_text` 或
:func:`~django.utils.encoding.force_bytes` 将其强转为对应类型.

如果没有设置 :setting:`SECRET_KEY` Django将无法启动.

.. warning::

    **保密该值.**

    使用泄露的 :setting:`SECRET_KEY` 运行Django会致使许多Django的安全保护措施失效,
    并可能导致越权操作和远程代码执行漏洞.

安全密钥用于:

* 所有 :doc:`sessions </topics/http/sessions>`, 如果使用了 ``django.contrib.sessions.backends.cache`` 以外的后端,
  或者使用默认的
  :meth:`~django.contrib.auth.models.AbstractBaseUser.get_session_auth_hash()`.
* 所有 :doc:`messages </ref/contrib/messages>`, 如果你使用了
  :class:`~django.contrib.messages.storage.cookie.CookieStorage` 或
  :class:`~django.contrib.messages.storage.fallback.FallbackStorage`.
* 所有 :func:`~django.contrib.auth.views.password_reset` token.
* 除了使用其他密钥的所有 :doc:`加密签名 </topics/signing>`.

如果你更换了密钥, 那么上述所有内容都将失效. 但是密钥不用于用户密码, 更好密钥不会影响用户密码.

.. note::

    为方便起见, 由 :djadmin:`django-admin startproject <startproject>` 创建的默认 :file:`settings.py` 自带一个唯一的 ``SECRET_KEY``.

.. setting:: SECURE_BROWSER_XSS_FILTER

``SECURE_BROWSER_XSS_FILTER``
-----------------------------

默认值: ``False``

如果设置为 ``True``, 则 :class:`~django.middleware.security.SecurityMiddleware` 会给所有响应加上 :ref:`x-xss-protection` 首部.

.. setting:: SECURE_CONTENT_TYPE_NOSNIFF

``SECURE_CONTENT_TYPE_NOSNIFF``
-------------------------------

默认值: ``False``

如果设置为 ``True``, 则 :class:`~django.middleware.security.SecurityMiddleware` 会给所有响应加上 :ref:`x-content-type-options` 首部.

.. setting:: SECURE_HSTS_INCLUDE_SUBDOMAINS

``SECURE_HSTS_INCLUDE_SUBDOMAINS``
----------------------------------

默认值: ``False``

如果设置为 ``True``, 则 :class:`~django.middleware.security.SecurityMiddleware` 会添加
``includeSubDomains`` 指令到 :ref:`http-strict-transport-security` 首部.
只有 :setting:`SECURE_HSTS_SECONDS` 设置为非0值值才会生效.

.. warning::
    如果 :setting:`SECURE_HSTS_SECONDS` 设置不正确可能会不可逆地破坏你的网站. 请先阅读
    :ref:`http-strict-transport-security` 文档.

.. setting:: SECURE_HSTS_SECONDS

``SECURE_HSTS_SECONDS``
-----------------------

默认值: ``0``

如果设置为一个非零的整数值, 则
:class:`~django.middleware.security.SecurityMiddleware` 会给所有响应加上
:ref:`http-strict-transport-security` 首部.

.. warning::
    如果设置不正确的值可能会不可逆地破坏你的网站.
    请先阅读 :ref:`http-strict-transport-security` 文档.

.. setting:: SECURE_PROXY_SSL_HEADER

``SECURE_PROXY_SSL_HEADER``
---------------------------

默认值: ``None``

用于表示请求是安全的HTTP首部/值组合的元组. 这直接影响请求对象 ``is_secure()`` 方法的结果.

这里需要解释下. 默认情况下, ``is_secure()`` 是通过查看请求地址中是否使用 "https://" 来确定请求是否安全.
这对于Django的CSRF保护很重要, 而且或许你自己的代码或第三方应用也会使用到.

如果在你Django应用服务之前还有个代理服务器, 那么这个代理服务器可能会"吞掉"请求使用HTTPS的情况,
比如代理服务器和Django之间通过非HTTPS连接, 那么 ``is_secure()`` 会始终返回 ``False`` -- 即使客户端是通过HTTPS请求.

这种情况下, 你需要配置代理服务器自定义HTTP首部, 来告诉Django请求是否是通过HTTPS发起, 同时还需要设置
``SECURE_PROXY_SSL_HEADER`` 首部以告诉Django应该从哪个首部找到它.

设置一个包含两个元素的元组 -- 要查找的首部名称和所需的值. 例如::

    SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

这样, 就等于告诉Django通过 ``X-Forwarded-Proto`` 首部来判断, 只要该值为 ``'https'`` 就认为请求是安全的(即. 该请求是通过HTTPS发起).
显然, 如果你可以控制你的代理服务或者有其保证, 你可以 **只** 设置这个配置, 它可以适当地设置/移除这个首部.

请求注意, 首部的格式需要和 ``request.META`` 使用的格式一样 --
全大写, 而且以 ``HTTP_`` 开头. (记住, Django会自动在x-header名称上加上 ``'HTTP_'`` 前缀, 然后才在 ``request.META`` 中使用.)

.. warning::

    **修改此设置可能会危及你网站的安全。在修改之前, 请确保你完全了解你的配置.**

    在设置之前, 请确保以下所有条件都成立(假设上面例子中的值):

    * Django应用在代理服务器之后.
    * 代理服务器需要移除所有请求中的 ``X-Forwarded-Proto`` 首部, 换句话说, 如果终端用户在其请求中包括该首部, 代理服务器将移除它.
    * 代理服务器只会对HTTPS请求加上 ``X-Forwarded-Proto`` 首部, 然后再转发给Django.

    如果上述条件有任何一个不成立, 那么你应该将此配置保持为 ``None``. 使用其他的方法来确认是否是通过HTTPS, 比如通过定义中间件.

.. setting:: SECURE_REDIRECT_EXEMPT

``SECURE_REDIRECT_EXEMPT``
--------------------------

默认值: ``[]`` (空列表)

如果请求的URL路径匹配到该列表中的正则表达式, 那么该请求就不会被重定向成HTTPS.
如果 :setting:`SECURE_SSL_REDIRECT` 设置为 ``False``, 该配置失效.

.. setting:: SECURE_SSL_HOST

``SECURE_SSL_HOST``
-------------------

默认值: ``None``

如果是一个字符串(例如. ``secure.example.com``), 所有SSL重定向将重定向到这个域, 而不是请求的原始域
(例如. ``www.example.com``). 如果 :setting:`SECURE_SSL_REDIRECT` 设置为 ``False``, 改配置失效.

.. setting:: SECURE_SSL_REDIRECT

``SECURE_SSL_REDIRECT``
-----------------------

默认值: ``False``

如果设置为 ``True``, :class:`~django.middleware.security.SecurityMiddleware` 会将所有非HTTPS请求
:ref:`重定向 <ssl-redirect>` 到HTTPS (except for
those URLs matching a regular expression listed in
:setting:`SECURE_REDIRECT_EXEMPT`).

.. note::

   If turning this to ``True`` causes infinite redirects, it probably means
   your site is running behind a proxy and can't tell which requests are secure
   and which are not. Your proxy likely sets a header to indicate secure
   requests; you can correct the problem by finding out what that header is and
   configuring the :setting:`SECURE_PROXY_SSL_HEADER` setting accordingly.

.. setting:: SERIALIZATION_MODULES

``SERIALIZATION_MODULES``
-------------------------

Default: Not defined

A dictionary of modules containing serializer definitions (provided as
strings), keyed by a string identifier for that serialization type. For
example, to define a YAML serializer, use::

    SERIALIZATION_MODULES = {'yaml': 'path.to.yaml_serializer'}

.. setting:: SERVER_EMAIL

``SERVER_EMAIL``
----------------

Default: ``'root@localhost'``

The email address that error messages come from, such as those sent to
:setting:`ADMINS` and :setting:`MANAGERS`.

.. admonition:: Why are my emails sent from a different address?

    This address is used only for error messages. It is *not* the address that
    regular email messages sent with :meth:`~django.core.mail.send_mail()`
    come from; for that, see :setting:`DEFAULT_FROM_EMAIL`.

.. setting:: SHORT_DATE_FORMAT

``SHORT_DATE_FORMAT``
---------------------

Default: ``'m/d/Y'`` (e.g. ``12/31/2003``)

An available formatting that can be used for displaying date fields on
templates. Note that if :setting:`USE_L10N` is set to ``True``, then the
corresponding locale-dictated format has higher precedence and will be applied.
See :tfilter:`allowed date format strings <date>`.

See also :setting:`DATE_FORMAT` and :setting:`SHORT_DATETIME_FORMAT`.

.. setting:: SHORT_DATETIME_FORMAT

``SHORT_DATETIME_FORMAT``
-------------------------

Default: ``'m/d/Y P'`` (e.g. ``12/31/2003 4 p.m.``)

An available formatting that can be used for displaying datetime fields on
templates. Note that if :setting:`USE_L10N` is set to ``True``, then the
corresponding locale-dictated format has higher precedence and will be applied.
See :tfilter:`allowed date format strings <date>`.

See also :setting:`DATE_FORMAT` and :setting:`SHORT_DATE_FORMAT`.

.. setting:: SIGNING_BACKEND

``SIGNING_BACKEND``
-------------------

Default: ``'django.core.signing.TimestampSigner'``

The backend used for signing cookies and other data.

See also the :doc:`/topics/signing` documentation.

.. setting:: SILENCED_SYSTEM_CHECKS

``SILENCED_SYSTEM_CHECKS``
--------------------------

Default: ``[]`` (Empty list)

A list of identifiers of messages generated by the system check framework
(i.e. ``["models.W001"]``) that you wish to permanently acknowledge and ignore.
Silenced checks will not be output to the console.

.. versionchanged:: 1.9

    In older versions, silenced messages of ``ERROR`` level or higher were
    printed to the console.

See also the :doc:`/ref/checks` documentation.

.. setting:: TEMPLATES

``TEMPLATES``
-------------

Default: ``[]`` (Empty list)

A list containing the settings for all template engines to be used with
Django. Each item of the list is a dictionary containing the options for an
individual engine.

Here's a simple setup that tells the Django template engine to load templates
from the ``templates`` subdirectory inside each installed application::

    TEMPLATES = [
        {
            'BACKEND': 'django.template.backends.django.DjangoTemplates',
            'APP_DIRS': True,
        },
    ]

The following options are available for all backends.

.. setting:: TEMPLATES-BACKEND

``BACKEND``
~~~~~~~~~~~

Default: Not defined

The template backend to use. The built-in template backends are:

* ``'django.template.backends.django.DjangoTemplates'``
* ``'django.template.backends.jinja2.Jinja2'``

You can use a template backend that doesn't ship with Django by setting
``BACKEND`` to a fully-qualified path (i.e. ``'mypackage.whatever.Backend'``).

.. setting:: TEMPLATES-NAME

``NAME``
~~~~~~~~

Default: see below

The alias for this particular template engine. It's an identifier that allows
selecting an engine for rendering. Aliases must be unique across all
configured template engines.

It defaults to the name of the module defining the engine class, i.e. the
next to last piece of :setting:`BACKEND <TEMPLATES-BACKEND>`, when it isn't
provided. For example if the backend is ``'mypackage.whatever.Backend'`` then
its default name is ``'whatever'``.

.. setting:: TEMPLATES-DIRS

``DIRS``
~~~~~~~~

Default: ``[]`` (Empty list)

Directories where the engine should look for template source files, in search
order.

.. setting:: TEMPLATES-APP_DIRS

``APP_DIRS``
~~~~~~~~~~~~

Default: ``False``

Whether the engine should look for template source files inside installed
applications.

.. note::

    The default :file:`settings.py` file created by :djadmin:`django-admin
    startproject <startproject>` sets ``'APP_DIRS': True``.

.. setting:: TEMPLATES-OPTIONS

``OPTIONS``
~~~~~~~~~~~

Default: ``{}`` (Empty dict)

Extra parameters to pass to the template backend. Available parameters vary
depending on the template backend. See
:class:`~django.template.backends.django.DjangoTemplates` and
:class:`~django.template.backends.jinja2.Jinja2` for the options of the
built-in backends.

.. setting:: TEST_RUNNER

``TEST_RUNNER``
---------------

Default: ``'django.test.runner.DiscoverRunner'``

The name of the class to use for starting the test suite. See
:ref:`other-testing-frameworks`.

.. setting:: TEST_NON_SERIALIZED_APPS

``TEST_NON_SERIALIZED_APPS``
----------------------------

Default: ``[]`` (Empty list)

In order to restore the database state between tests for
``TransactionTestCase``\s and database backends without transactions, Django
will :ref:`serialize the contents of all apps <test-case-serialized-rollback>`
when it starts the test run so it can then reload from that copy before running
tests that need it.

This slows down the startup time of the test runner; if you have apps that
you know don't need this feature, you can add their full names in here (e.g.
``'django.contrib.contenttypes'``) to exclude them from this serialization
process.

.. setting:: THOUSAND_SEPARATOR

``THOUSAND_SEPARATOR``
----------------------

Default: ``','`` (Comma)

Default thousand separator used when formatting numbers. This setting is
used only when :setting:`USE_THOUSAND_SEPARATOR` is ``True`` and
:setting:`NUMBER_GROUPING` is greater than ``0``.

Note that if :setting:`USE_L10N` is set to ``True``, then the locale-dictated
format has higher precedence and will be applied instead.

See also :setting:`NUMBER_GROUPING`, :setting:`DECIMAL_SEPARATOR` and
:setting:`USE_THOUSAND_SEPARATOR`.

.. setting:: TIME_FORMAT

``TIME_FORMAT``
---------------

Default: ``'P'`` (e.g. ``4 p.m.``)

The default formatting to use for displaying time fields in any part of the
system. Note that if :setting:`USE_L10N` is set to ``True``, then the
locale-dictated format has higher precedence and will be applied instead. See
:tfilter:`allowed date format strings <date>`.

See also :setting:`DATE_FORMAT` and :setting:`DATETIME_FORMAT`.

.. setting:: TIME_INPUT_FORMATS

``TIME_INPUT_FORMATS``
----------------------

Default::

    [
        '%H:%M:%S',     # '14:30:59'
        '%H:%M:%S.%f',  # '14:30:59.000200'
        '%H:%M',        # '14:30'
    ]

A list of formats that will be accepted when inputting data on a time field.
Formats will be tried in order, using the first valid one. Note that these
format strings use Python's :ref:`datetime module syntax
<strftime-strptime-behavior>`, not the format strings from the :tfilter:`date`
template filter.

When :setting:`USE_L10N` is ``True``, the locale-dictated format has higher
precedence and will be applied instead.

See also :setting:`DATE_INPUT_FORMATS` and :setting:`DATETIME_INPUT_FORMATS`.

.. setting:: TIME_ZONE

``TIME_ZONE``
-------------

Default: ``'America/Chicago'``

A string representing the time zone for this installation, or ``None``. See
the `list of time zones`_.

.. note::
    Since Django was first released with the :setting:`TIME_ZONE` set to
    ``'America/Chicago'``, the global setting (used if nothing is defined in
    your project's ``settings.py``) remains ``'America/Chicago'`` for backwards
    compatibility. New project templates default to ``'UTC'``.

Note that this isn't necessarily the time zone of the server. For example, one
server may serve multiple Django-powered sites, each with a separate time zone
setting.

When :setting:`USE_TZ` is ``False``, this is the time zone in which Django
will store all datetimes. When :setting:`USE_TZ` is ``True``, this is the
default time zone that Django will use to display datetimes in templates and
to interpret datetimes entered in forms.

Django sets the ``os.environ['TZ']`` variable to the time zone you specify in
the :setting:`TIME_ZONE` setting. Thus, all your views and models will
automatically operate in this time zone. However, Django won't set the ``TZ``
environment variable under the following conditions:

* If you're using the manual configuration option as described in
  :ref:`manually configuring settings
  <settings-without-django-settings-module>`, or

* If you specify ``TIME_ZONE = None``. This will cause Django to fall back to
  using the system timezone. However, this is discouraged when :setting:`USE_TZ
  = True <USE_TZ>`, because it makes conversions between local time and UTC
  less reliable.

If Django doesn't set the ``TZ`` environment variable, it's up to you
to ensure your processes are running in the correct environment.

.. note::
    Django cannot reliably use alternate time zones in a Windows environment.
    If you're running Django on Windows, :setting:`TIME_ZONE` must be set to
    match the system time zone.

.. _list of time zones: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones

.. setting:: USE_ETAGS

``USE_ETAGS``
-------------

Default: ``False``

A boolean that specifies whether to output the ``ETag`` header. This saves
bandwidth but slows down performance. This is used by the
:class:`~django.middleware.common.CommonMiddleware` and in the :doc:`cache
framework </topics/cache>`.

.. setting:: USE_I18N

``USE_I18N``
------------

Default: ``True``

A boolean that specifies whether Django's translation system should be enabled.
This provides an easy way to turn it off, for performance. If this is set to
``False``, Django will make some optimizations so as not to load the
translation machinery.

See also :setting:`LANGUAGE_CODE`, :setting:`USE_L10N` and :setting:`USE_TZ`.

.. note::

    The default :file:`settings.py` file created by :djadmin:`django-admin
    startproject <startproject>` includes ``USE_I18N = True`` for convenience.

.. setting:: USE_L10N

``USE_L10N``
------------

Default: ``False``

A boolean that specifies if localized formatting of data will be enabled by
default or not. If this is set to ``True``, e.g. Django will display numbers and
dates using the format of the current locale.

See also :setting:`LANGUAGE_CODE`, :setting:`USE_I18N` and :setting:`USE_TZ`.

.. note::

    The default :file:`settings.py` file created by :djadmin:`django-admin
    startproject <startproject>` includes ``USE_L10N = True`` for convenience.

.. setting:: USE_THOUSAND_SEPARATOR

``USE_THOUSAND_SEPARATOR``
--------------------------

Default: ``False``

A boolean that specifies whether to display numbers using a thousand separator.
When :setting:`USE_L10N` is set to ``True`` and if this is also set to
``True``, Django will use the values of :setting:`THOUSAND_SEPARATOR` and
:setting:`NUMBER_GROUPING` to format numbers unless the locale already has an
existing thousands separator. If there is a thousands separator in the locale
format, it will have higher precedence and will be applied instead.

See also :setting:`DECIMAL_SEPARATOR`, :setting:`NUMBER_GROUPING` and
:setting:`THOUSAND_SEPARATOR`.

.. setting:: USE_TZ

``USE_TZ``
----------

Default: ``False``

A boolean that specifies if datetimes will be timezone-aware by default or not.
If this is set to ``True``, Django will use timezone-aware datetimes internally.
Otherwise, Django will use naive datetimes in local time.

See also :setting:`TIME_ZONE`, :setting:`USE_I18N` and :setting:`USE_L10N`.

.. note::

    The default :file:`settings.py` file created by
    :djadmin:`django-admin startproject <startproject>` includes
    ``USE_TZ = True`` for convenience.

.. setting:: USE_X_FORWARDED_HOST

``USE_X_FORWARDED_HOST``
------------------------

Default: ``False``

A boolean that specifies whether to use the ``X-Forwarded-Host`` header in
preference to the ``Host`` header. This should only be enabled if a proxy
which sets this header is in use.

This setting takes priority over :setting:`USE_X_FORWARDED_PORT`. Per
:rfc:`7239#page-7`, the ``X-Forwarded-Host`` header can include the port
number, in which case you shouldn't use :setting:`USE_X_FORWARDED_PORT`.

.. setting:: USE_X_FORWARDED_PORT

``USE_X_FORWARDED_PORT``
------------------------

.. versionadded:: 1.9

Default: ``False``

A boolean that specifies whether to use the ``X-Forwarded-Port`` header in
preference to the ``SERVER_PORT`` ``META`` variable. This should only be
enabled if a proxy which sets this header is in use.

:setting:`USE_X_FORWARDED_HOST` takes priority over this setting.

.. setting:: WSGI_APPLICATION

``WSGI_APPLICATION``
--------------------

Default: ``None``

The full Python path of the WSGI application object that Django's built-in
servers (e.g. :djadmin:`runserver`) will use. The :djadmin:`django-admin
startproject <startproject>` management command will create a simple
``wsgi.py`` file with an ``application`` callable in it, and point this setting
to that ``application``.

If not set, the return value of ``django.core.wsgi.get_wsgi_application()``
will be used. In this case, the behavior of :djadmin:`runserver` will be
identical to previous Django versions.

.. setting:: YEAR_MONTH_FORMAT

``YEAR_MONTH_FORMAT``
---------------------

Default: ``'F Y'``

The default formatting to use for date fields on Django admin change-list
pages -- and, possibly, by other parts of the system -- in cases when only the
year and month are displayed.

For example, when a Django admin change-list page is being filtered by a date
drilldown, the header for a given month displays the month and the year.
Different locales have different formats. For example, U.S. English would say
"January 2006," whereas another locale might say "2006/January."

Note that if :setting:`USE_L10N` is set to ``True``, then the corresponding
locale-dictated format has higher precedence and will be applied.

See :tfilter:`allowed date format strings <date>`. See also
:setting:`DATE_FORMAT`, :setting:`DATETIME_FORMAT`, :setting:`TIME_FORMAT`
and :setting:`MONTH_DAY_FORMAT`.

.. setting:: X_FRAME_OPTIONS

``X_FRAME_OPTIONS``
-------------------

Default: ``'SAMEORIGIN'``

The default value for the X-Frame-Options header used by
:class:`~django.middleware.clickjacking.XFrameOptionsMiddleware`. See the
:doc:`clickjacking protection </ref/clickjacking/>` documentation.


Auth
====

Settings for :mod:`django.contrib.auth`.

.. setting:: AUTHENTICATION_BACKENDS

``AUTHENTICATION_BACKENDS``
---------------------------

Default: ``['django.contrib.auth.backends.ModelBackend']``

A list of authentication backend classes (as strings) to use when attempting to
authenticate a user. See the :ref:`authentication backends documentation
<authentication-backends>` for details.

.. setting:: AUTH_USER_MODEL

``AUTH_USER_MODEL``
-------------------

Default: ``'auth.User'``

The model to use to represent a User. See :ref:`auth-custom-user`.

.. warning::
    You cannot change the AUTH_USER_MODEL setting during the lifetime of
    a project (i.e. once you have made and migrated models that depend on it)
    without serious effort. It is intended to be set at the project start,
    and the model it refers to must be available in the first migration of
    the app that it lives in.
    See :ref:`auth-custom-user` for more details.

.. setting:: LOGIN_REDIRECT_URL

``LOGIN_REDIRECT_URL``
----------------------

Default: ``'/accounts/profile/'``

The URL where requests are redirected after login when the
``contrib.auth.login`` view gets no ``next`` parameter.

This is used by the :func:`~django.contrib.auth.decorators.login_required`
decorator, for example.

This setting also accepts :ref:`named URL patterns <naming-url-patterns>` which
can be used to reduce configuration duplication since you don't have to define
the URL in two places (``settings`` and URLconf).

.. setting:: LOGIN_URL

``LOGIN_URL``
-------------

Default: ``'/accounts/login/'``

The URL where requests are redirected for login, especially when using the
:func:`~django.contrib.auth.decorators.login_required` decorator.

This setting also accepts :ref:`named URL patterns <naming-url-patterns>` which
can be used to reduce configuration duplication since you don't have to define
the URL in two places (``settings`` and URLconf).

.. setting:: LOGOUT_REDIRECT_URL

``LOGOUT_REDIRECT_URL``
-----------------------

.. versionadded:: 1.10

Default: ``None``

The URL where requests are redirected after a user logs out using the
:func:`~django.contrib.auth.views.logout` view (if the view doesn't get a
``next_page`` argument).

If ``None``, no redirect will be performed and the logout view will be
rendered.

This setting also accepts :ref:`named URL patterns <naming-url-patterns>` which
can be used to reduce configuration duplication since you don't have to define
the URL in two places (``settings`` and URLconf).

.. setting:: PASSWORD_RESET_TIMEOUT_DAYS

``PASSWORD_RESET_TIMEOUT_DAYS``
-------------------------------

Default: ``3``

The number of days a password reset link is valid for. Used by the
:mod:`django.contrib.auth` password reset mechanism.

.. setting:: PASSWORD_HASHERS

``PASSWORD_HASHERS``
--------------------

See :ref:`auth_password_storage`.

Default::

    [
        'django.contrib.auth.hashers.PBKDF2PasswordHasher',
        'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
        'django.contrib.auth.hashers.Argon2PasswordHasher',
        'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
        'django.contrib.auth.hashers.BCryptPasswordHasher',
    ]

.. versionchanged:: 1.10

    The following hashers were removed from the defaults::

        'django.contrib.auth.hashers.SHA1PasswordHasher'
        'django.contrib.auth.hashers.MD5PasswordHasher'
        'django.contrib.auth.hashers.UnsaltedSHA1PasswordHasher'
        'django.contrib.auth.hashers.UnsaltedMD5PasswordHasher'
        'django.contrib.auth.hashers.CryptPasswordHasher'

    Consider using a :ref:`wrapped password hasher <wrapping-password-hashers>`
    to strengthen the hashes in your database. If that's not feasible, add this
    setting to your project and add back any hashers that you need.

    Also, the ``Argon2PasswordHasher`` was added.

.. setting:: AUTH_PASSWORD_VALIDATORS

``AUTH_PASSWORD_VALIDATORS``
----------------------------

.. versionadded:: 1.9

Default: ``[]`` (Empty list)

The list of validators that are used to check the strength of user's passwords.
See :ref:`password-validation` for more details. By default, no validation is
performed and all passwords are accepted.

.. _settings-messages:

Messages
========

Settings for :mod:`django.contrib.messages`.

.. setting:: MESSAGE_LEVEL

``MESSAGE_LEVEL``
-----------------

Default: ``messages.INFO``

Sets the minimum message level that will be recorded by the messages
framework. See :ref:`message levels <message-level>` for more details.

.. admonition:: Important

   If you override ``MESSAGE_LEVEL`` in your settings file and rely on any of
   the built-in constants, you must import the constants module directly to
   avoid the potential for circular imports, e.g.::

       from django.contrib.messages import constants as message_constants
       MESSAGE_LEVEL = message_constants.DEBUG

   If desired, you may specify the numeric values for the constants directly
   according to the values in the above :ref:`constants table
   <message-level-constants>`.

.. setting:: MESSAGE_STORAGE

``MESSAGE_STORAGE``
-------------------

Default: ``'django.contrib.messages.storage.fallback.FallbackStorage'``

Controls where Django stores message data. Valid values are:

* ``'django.contrib.messages.storage.fallback.FallbackStorage'``
* ``'django.contrib.messages.storage.session.SessionStorage'``
* ``'django.contrib.messages.storage.cookie.CookieStorage'``

See :ref:`message storage backends <message-storage-backends>` for more details.

The backends that use cookies --
:class:`~django.contrib.messages.storage.cookie.CookieStorage` and
:class:`~django.contrib.messages.storage.fallback.FallbackStorage` --
use the value of :setting:`SESSION_COOKIE_DOMAIN`, :setting:`SESSION_COOKIE_SECURE`
and :setting:`SESSION_COOKIE_HTTPONLY` when setting their cookies.

.. setting:: MESSAGE_TAGS

``MESSAGE_TAGS``
----------------

Default::

    {
        messages.DEBUG: 'debug',
        messages.INFO: 'info',
        messages.SUCCESS: 'success',
        messages.WARNING: 'warning',
        messages.ERROR: 'error',
    }

This sets the mapping of message level to message tag, which is typically
rendered as a CSS class in HTML. If you specify a value, it will extend
the default. This means you only have to specify those values which you need
to override. See :ref:`message-displaying` above for more details.

.. admonition:: Important

   If you override ``MESSAGE_TAGS`` in your settings file and rely on any of
   the built-in constants, you must import the ``constants`` module directly to
   avoid the potential for circular imports, e.g.::

       from django.contrib.messages import constants as message_constants
       MESSAGE_TAGS = {message_constants.INFO: ''}

   If desired, you may specify the numeric values for the constants directly
   according to the values in the above :ref:`constants table
   <message-level-constants>`.

.. _settings-sessions:

Sessions
========

Settings for :mod:`django.contrib.sessions`.

.. setting:: SESSION_CACHE_ALIAS

``SESSION_CACHE_ALIAS``
-----------------------

Default: ``'default'``

If you're using :ref:`cache-based session storage <cached-sessions-backend>`,
this selects the cache to use.

.. setting:: SESSION_COOKIE_AGE

``SESSION_COOKIE_AGE``
----------------------

Default: ``1209600`` (2 weeks, in seconds)

The age of session cookies, in seconds.

.. setting:: SESSION_COOKIE_DOMAIN

``SESSION_COOKIE_DOMAIN``
-------------------------

Default: ``None``

The domain to use for session cookies. Set this to a string such as
``".example.com"`` (note the leading dot!) for cross-domain cookies, or use
``None`` for a standard domain cookie.

Be cautious when updating this setting on a production site. If you update
this setting to enable cross-domain cookies on a site that previously used
standard domain cookies, existing user cookies will be set to the old
domain. This may result in them being unable to log in as long as these cookies
persist.

This setting also affects cookies set by :mod:`django.contrib.messages`.

.. setting:: SESSION_COOKIE_HTTPONLY

``SESSION_COOKIE_HTTPONLY``
---------------------------

Default: ``True``

Whether to use ``HTTPOnly`` flag on the session cookie. If this is set to
``True``, client-side JavaScript will not to be able to access the
session cookie.

HTTPOnly_ is a flag included in a Set-Cookie HTTP response header. It
is not part of the :rfc:`2109` standard for cookies, and it isn't honored
consistently by all browsers. However, when it is honored, it can be a
useful way to mitigate the risk of a client side script accessing the
protected cookie data.

Turning it on makes it less trivial for an attacker to escalate a cross-site
scripting vulnerability into full hijacking of a user's session. There's not
much excuse for leaving this off, either: if your code depends on reading
session cookies from JavaScript, you're probably doing it wrong.

.. _HTTPOnly: https://www.owasp.org/index.php/HTTPOnly

.. setting:: SESSION_COOKIE_NAME

``SESSION_COOKIE_NAME``
-----------------------

Default: ``'sessionid'``

The name of the cookie to use for sessions. This can be whatever you want
(as long as it's different from the other cookie names in your application).

.. setting:: SESSION_COOKIE_PATH

``SESSION_COOKIE_PATH``
-----------------------

Default: ``'/'``

The path set on the session cookie. This should either match the URL path of your
Django installation or be parent of that path.

This is useful if you have multiple Django instances running under the same
hostname. They can use different cookie paths, and each instance will only see
its own session cookie.

.. setting:: SESSION_COOKIE_SECURE

``SESSION_COOKIE_SECURE``
-------------------------

Default: ``False``

Whether to use a secure cookie for the session cookie. If this is set to
``True``, the cookie will be marked as "secure," which means browsers may
ensure that the cookie is only sent under an HTTPS connection.

Since it's trivial for a packet sniffer (e.g. `Firesheep`_) to hijack a user's
session if the session cookie is sent unencrypted, there's really no good
excuse to leave this off. It will prevent you from using sessions on insecure
requests and that's a good thing.

.. _Firesheep: http://codebutler.com/firesheep

.. setting:: SESSION_ENGINE

``SESSION_ENGINE``
------------------

Default: ``'django.contrib.sessions.backends.db'``

Controls where Django stores session data. Included engines are:

* ``'django.contrib.sessions.backends.db'``
* ``'django.contrib.sessions.backends.file'``
* ``'django.contrib.sessions.backends.cache'``
* ``'django.contrib.sessions.backends.cached_db'``
* ``'django.contrib.sessions.backends.signed_cookies'``

See :ref:`configuring-sessions` for more details.

.. setting:: SESSION_EXPIRE_AT_BROWSER_CLOSE

``SESSION_EXPIRE_AT_BROWSER_CLOSE``
-----------------------------------

Default: ``False``

Whether to expire the session when the user closes their browser. See
:ref:`browser-length-vs-persistent-sessions`.

.. setting:: SESSION_FILE_PATH

``SESSION_FILE_PATH``
---------------------

Default: ``None``

If you're using file-based session storage, this sets the directory in
which Django will store session data. When the default value (``None``) is
used, Django will use the standard temporary directory for the system.


.. setting:: SESSION_SAVE_EVERY_REQUEST

``SESSION_SAVE_EVERY_REQUEST``
------------------------------

Default: ``False``

Whether to save the session data on every request. If this is ``False``
(default), then the session data will only be saved if it has been modified --
that is, if any of its dictionary values have been assigned or deleted. Empty
sessions won't be created, even if this setting is active.

.. setting:: SESSION_SERIALIZER

``SESSION_SERIALIZER``
----------------------

Default: ``'django.contrib.sessions.serializers.JSONSerializer'``

Full import path of a serializer class to use for serializing session data.
Included serializers are:

* ``'django.contrib.sessions.serializers.PickleSerializer'``
* ``'django.contrib.sessions.serializers.JSONSerializer'``

See :ref:`session_serialization` for details, including a warning regarding
possible remote code execution when using
:class:`~django.contrib.sessions.serializers.PickleSerializer`.

Sites
=====

Settings for :mod:`django.contrib.sites`.

.. setting:: SITE_ID

``SITE_ID``
-----------

Default: Not defined

The ID, as an integer, of the current site in the ``django_site`` database
table. This is used so that application data can hook into specific sites
and a single database can manage content for multiple sites.


.. _settings-staticfiles:

Static Files
============

Settings for :mod:`django.contrib.staticfiles`.

.. setting:: STATIC_ROOT

``STATIC_ROOT``
---------------

Default: ``None``

The absolute path to the directory where :djadmin:`collectstatic` will collect
static files for deployment.

Example: ``"/var/www/example.com/static/"``

If the :doc:`staticfiles</ref/contrib/staticfiles>` contrib app is enabled
(as in the default project template), the :djadmin:`collectstatic` management
command will collect static files into this directory. See the how-to on
:doc:`managing static files</howto/static-files/index>` for more details about
usage.

.. warning::

    This should be an initially empty destination directory for collecting
    your static files from their permanent locations into one directory for
    ease of deployment; it is **not** a place to store your static files
    permanently. You should do that in directories that will be found by
    :doc:`staticfiles</ref/contrib/staticfiles>`’s
    :setting:`finders<STATICFILES_FINDERS>`, which by default, are
    ``'static/'`` app sub-directories and any directories you include in
    :setting:`STATICFILES_DIRS`).

.. setting:: STATIC_URL

``STATIC_URL``
--------------

Default: ``None``

URL to use when referring to static files located in :setting:`STATIC_ROOT`.

Example: ``"/static/"`` or ``"http://static.example.com/"``

If not ``None``, this will be used as the base path for
:ref:`asset definitions<form-asset-paths>` (the ``Media`` class) and the
:doc:`staticfiles app</ref/contrib/staticfiles>`.

It must end in a slash if set to a non-empty value.

You may need to :ref:`configure these files to be served in development
<serving-static-files-in-development>` and will definitely need to do so
:doc:`in production </howto/static-files/deployment>`.

.. setting:: STATICFILES_DIRS

``STATICFILES_DIRS``
--------------------

Default: ``[]`` (Empty list)

This setting defines the additional locations the staticfiles app will traverse
if the ``FileSystemFinder`` finder is enabled, e.g. if you use the
:djadmin:`collectstatic` or :djadmin:`findstatic` management command or use the
static file serving view.

This should be set to a list of strings that contain full paths to
your additional files directory(ies) e.g.::

    STATICFILES_DIRS = [
        "/home/special.polls.com/polls/static",
        "/home/polls.com/polls/static",
        "/opt/webfiles/common",
    ]

Note that these paths should use Unix-style forward slashes, even on Windows
(e.g. ``"C:/Users/user/mysite/extra_static_content"``).

Prefixes (optional)
~~~~~~~~~~~~~~~~~~~

In case you want to refer to files in one of the locations with an additional
namespace, you can **optionally** provide a prefix as ``(prefix, path)``
tuples, e.g.::

    STATICFILES_DIRS = [
        # ...
        ("downloads", "/opt/webfiles/stats"),
    ]

For example, assuming you have :setting:`STATIC_URL` set to ``'/static/'``, the
:djadmin:`collectstatic` management command would collect the "stats" files
in a ``'downloads'`` subdirectory of :setting:`STATIC_ROOT`.

This would allow you to refer to the local file
``'/opt/webfiles/stats/polls_20101022.tar.gz'`` with
``'/static/downloads/polls_20101022.tar.gz'`` in your templates, e.g.:

.. code-block:: html+django

    <a href="{% static "downloads/polls_20101022.tar.gz" %}">

.. setting:: STATICFILES_STORAGE

``STATICFILES_STORAGE``
-----------------------

Default: ``'django.contrib.staticfiles.storage.StaticFilesStorage'``

The file storage engine to use when collecting static files with the
:djadmin:`collectstatic` management command.

A ready-to-use instance of the storage backend defined in this setting
can be found at ``django.contrib.staticfiles.storage.staticfiles_storage``.

For an example, see :ref:`staticfiles-from-cdn`.

.. setting:: STATICFILES_FINDERS

``STATICFILES_FINDERS``
-----------------------

Default::

    [
        'django.contrib.staticfiles.finders.FileSystemFinder',
        'django.contrib.staticfiles.finders.AppDirectoriesFinder',
    ]

The list of finder backends that know how to find static files in
various locations.

The default will find files stored in the :setting:`STATICFILES_DIRS` setting
(using ``django.contrib.staticfiles.finders.FileSystemFinder``) and in a
``static`` subdirectory of each app (using
``django.contrib.staticfiles.finders.AppDirectoriesFinder``). If multiple
files with the same name are present, the first file that is found will be
used.

One finder is disabled by default:
``django.contrib.staticfiles.finders.DefaultStorageFinder``. If added to
your :setting:`STATICFILES_FINDERS` setting, it will look for static files in
the default file storage as defined by the :setting:`DEFAULT_FILE_STORAGE`
setting.

.. note::

    When using the ``AppDirectoriesFinder`` finder, make sure your apps
    can be found by staticfiles. Simply add the app to the
    :setting:`INSTALLED_APPS` setting of your site.

Static file finders are currently considered a private interface, and this
interface is thus undocumented.

Core Settings Topical Index
===========================

Cache
-----
* :setting:`CACHES`
* :setting:`CACHE_MIDDLEWARE_ALIAS`
* :setting:`CACHE_MIDDLEWARE_KEY_PREFIX`
* :setting:`CACHE_MIDDLEWARE_SECONDS`

Database
--------
* :setting:`DATABASES`
* :setting:`DATABASE_ROUTERS`
* :setting:`DEFAULT_INDEX_TABLESPACE`
* :setting:`DEFAULT_TABLESPACE`

Debugging
---------
* :setting:`DEBUG`
* :setting:`DEBUG_PROPAGATE_EXCEPTIONS`

Email
-----
* :setting:`ADMINS`
* :setting:`DEFAULT_CHARSET`
* :setting:`DEFAULT_FROM_EMAIL`
* :setting:`EMAIL_BACKEND`
* :setting:`EMAIL_FILE_PATH`
* :setting:`EMAIL_HOST`
* :setting:`EMAIL_HOST_PASSWORD`
* :setting:`EMAIL_HOST_USER`
* :setting:`EMAIL_PORT`
* :setting:`EMAIL_SSL_CERTFILE`
* :setting:`EMAIL_SSL_KEYFILE`
* :setting:`EMAIL_SUBJECT_PREFIX`
* :setting:`EMAIL_TIMEOUT`
* :setting:`EMAIL_USE_TLS`
* :setting:`MANAGERS`
* :setting:`SERVER_EMAIL`

Error reporting
---------------
* :setting:`DEFAULT_EXCEPTION_REPORTER_FILTER`
* :setting:`IGNORABLE_404_URLS`
* :setting:`MANAGERS`
* :setting:`SILENCED_SYSTEM_CHECKS`

.. _file-upload-settings:

File uploads
------------
* :setting:`DEFAULT_FILE_STORAGE`
* :setting:`FILE_CHARSET`
* :setting:`FILE_UPLOAD_HANDLERS`
* :setting:`FILE_UPLOAD_MAX_MEMORY_SIZE`
* :setting:`FILE_UPLOAD_PERMISSIONS`
* :setting:`FILE_UPLOAD_TEMP_DIR`
* :setting:`MEDIA_ROOT`
* :setting:`MEDIA_URL`

Globalization (``i18n``/``l10n``)
---------------------------------
* :setting:`DATE_FORMAT`
* :setting:`DATE_INPUT_FORMATS`
* :setting:`DATETIME_FORMAT`
* :setting:`DATETIME_INPUT_FORMATS`
* :setting:`DECIMAL_SEPARATOR`
* :setting:`FIRST_DAY_OF_WEEK`
* :setting:`FORMAT_MODULE_PATH`
* :setting:`LANGUAGE_CODE`
* :setting:`LANGUAGE_COOKIE_AGE`
* :setting:`LANGUAGE_COOKIE_DOMAIN`
* :setting:`LANGUAGE_COOKIE_NAME`
* :setting:`LANGUAGE_COOKIE_PATH`
* :setting:`LANGUAGES`
* :setting:`LOCALE_PATHS`
* :setting:`MONTH_DAY_FORMAT`
* :setting:`NUMBER_GROUPING`
* :setting:`SHORT_DATE_FORMAT`
* :setting:`SHORT_DATETIME_FORMAT`
* :setting:`THOUSAND_SEPARATOR`
* :setting:`TIME_FORMAT`
* :setting:`TIME_INPUT_FORMATS`
* :setting:`TIME_ZONE`
* :setting:`USE_I18N`
* :setting:`USE_L10N`
* :setting:`USE_THOUSAND_SEPARATOR`
* :setting:`USE_TZ`
* :setting:`YEAR_MONTH_FORMAT`

HTTP
----
* :setting:`DATA_UPLOAD_MAX_MEMORY_SIZE`
* :setting:`DATA_UPLOAD_MAX_NUMBER_FIELDS`
* :setting:`DEFAULT_CHARSET`
* :setting:`DEFAULT_CONTENT_TYPE`
* :setting:`DISALLOWED_USER_AGENTS`
* :setting:`FORCE_SCRIPT_NAME`
* :setting:`INTERNAL_IPS`
* :setting:`MIDDLEWARE`
* :setting:`MIDDLEWARE_CLASSES`
* Security

  * :setting:`SECURE_BROWSER_XSS_FILTER`
  * :setting:`SECURE_CONTENT_TYPE_NOSNIFF`
  * :setting:`SECURE_HSTS_INCLUDE_SUBDOMAINS`
  * :setting:`SECURE_HSTS_SECONDS`
  * :setting:`SECURE_PROXY_SSL_HEADER`
  * :setting:`SECURE_REDIRECT_EXEMPT`
  * :setting:`SECURE_SSL_HOST`
  * :setting:`SECURE_SSL_REDIRECT`
* :setting:`SIGNING_BACKEND`
* :setting:`USE_ETAGS`
* :setting:`USE_X_FORWARDED_HOST`
* :setting:`USE_X_FORWARDED_PORT`
* :setting:`WSGI_APPLICATION`

Logging
-------
* :setting:`LOGGING`
* :setting:`LOGGING_CONFIG`

Models
------
* :setting:`ABSOLUTE_URL_OVERRIDES`
* :setting:`FIXTURE_DIRS`
* :setting:`INSTALLED_APPS`

Security
--------
* Cross Site Request Forgery Protection

  * :setting:`CSRF_COOKIE_DOMAIN`
  * :setting:`CSRF_COOKIE_NAME`
  * :setting:`CSRF_COOKIE_PATH`
  * :setting:`CSRF_COOKIE_SECURE`
  * :setting:`CSRF_FAILURE_VIEW`
  * :setting:`CSRF_HEADER_NAME`
  * :setting:`CSRF_TRUSTED_ORIGINS`

* :setting:`SECRET_KEY`
* :setting:`X_FRAME_OPTIONS`

Serialization
-------------
* :setting:`DEFAULT_CHARSET`
* :setting:`SERIALIZATION_MODULES`

Templates
---------
* :setting:`TEMPLATES`

Testing
-------
* Database: :setting:`TEST <DATABASE-TEST>`
* :setting:`TEST_NON_SERIALIZED_APPS`
* :setting:`TEST_RUNNER`

URLs
----
* :setting:`APPEND_SLASH`
* :setting:`PREPEND_WWW`
* :setting:`ROOT_URLCONF`
