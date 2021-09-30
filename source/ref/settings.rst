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
:ref:`重定向 <ssl-redirect>` 到HTTPS (除了被
:setting:`SECURE_REDIRECT_EXEMPT` 中的正则表达式匹配到的URL).

.. note::

   如果你的网站部署在代理服务器之后, 且它无法判断那些请求是安全的那些是非安全的, 那么如果这个设置为 ``True``,
   则可能会导致无限重定向. 代理服务器需要设置一个首部来标记安全的请求, 你可以通过设置 :setting:`SECURE_PROXY_SSL_HEADER` 来解决这个问题.

.. setting:: SERIALIZATION_MODULES

``SERIALIZATION_MODULES``
-------------------------

默认值: 未定义

一个定义序列化器(字符串形式)的字典, 以序列化类型为键. 比如, 定义个YAML序列化器::

    SERIALIZATION_MODULES = {'yaml': 'path.to.yaml_serializer'}

.. setting:: SERVER_EMAIL

``SERVER_EMAIL``
----------------

默认值: ``'root@localhost'``

错误信息的发件地址, 例如发给 :setting:`ADMINS` 和 :setting:`MANAGERS` 的邮件.

.. admonition:: 为什么我的邮件是从其他地址发送的?

    该地址仅用于错误信息. 它 **不是** :meth:`~django.core.mail.send_mail()` 发送常规邮件的地址; 这个地址是 :setting:`DEFAULT_FROM_EMAIL`.

.. setting:: SHORT_DATE_FORMAT

``SHORT_DATE_FORMAT``
---------------------

默认值: ``'m/d/Y'`` (即 ``12/31/2003``)

在模板中显示日期字段的格式. 注意, 如果 :setting:`USE_L10N` 设置为 ``True``, 相应本地决定的格式具有更高的应用优先权.
见 :tfilter:`日期格式字符 <date>`.

另见 :setting:`DATE_FORMAT` 和 :setting:`SHORT_DATETIME_FORMAT`.

.. setting:: SHORT_DATETIME_FORMAT

``SHORT_DATETIME_FORMAT``
-------------------------

默认值: ``'m/d/Y P'`` (即 ``12/31/2003 4 p.m.``)

在模板中显示日期时间字段的格式. 注意, 如果 :setting:`USE_L10N` 设置为 ``True``, 相应本地决定的格式具有更高的应用优先权.
见 :tfilter:`日期格式字符 <date>`.

另见 :setting:`DATE_FORMAT` 和 :setting:`SHORT_DATE_FORMAT`.

.. setting:: SIGNING_BACKEND

``SIGNING_BACKEND``
-------------------

默认值: ``'django.core.signing.TimestampSigner'``

用来签名cookie和其他数据的backend.

见 :doc:`/topics/signing` 文档.

.. setting:: SILENCED_SYSTEM_CHECKS

``SILENCED_SYSTEM_CHECKS``
--------------------------

默认值: ``[]`` (空列表)

希望永久忽略的系统检查框架生成的消息标识符列表(如 ``["models.W001"]``). 静默检查不会被输出到控制台.

.. versionchanged:: 1.9

    在老版本中, ``ERROR`` 和更高级别的静默消息会被输出到控制台.

另见 :doc:`/ref/checks` 文档.

.. setting:: TEMPLATES

``TEMPLATES``
-------------

默认值: ``[]`` (空列表)

Django模板引擎的配置列表. 列表中的每项都是一个包含引擎配置的字典.

下面配置告诉Django模板引擎从每个安装的应用程序中的 ``templates`` 子目录中加载模板::

    TEMPLATES = [
        {
            'BACKEND': 'django.template.backends.django.DjangoTemplates',
            'APP_DIRS': True,
        },
    ]

以下选项适用于所有模板引擎.

.. setting:: TEMPLATES-BACKEND

``BACKEND``
~~~~~~~~~~~

默认值: 未定义

使用的模板引擎. 内建的模板引擎有:

* ``'django.template.backends.django.DjangoTemplates'``
* ``'django.template.backends.jinja2.Jinja2'``

如果要使用非Django内置的模板引擎, 可以将 ``BACKEND`` 设置为其完整路径 (例如 ``'mypackage.whatever.Backend'``).

.. setting:: TEMPLATES-NAME

``NAME``
~~~~~~~~

默认值: 见下文

设置模板引擎的别名. 它是一个标识符, 用来进行渲染时选择引擎. 别名在所有已配置的模板引擎中必须是唯一的.

没有设置时, 它默认是引擎类的模块名称, 即 :setting:`BACKEND <TEMPLATES-BACKEND>` 的后一段.
例如模板引擎为 ``'mypackage.whatever.Backend'`` 那么默认名称就是它中间的 ``'whatever'``.

.. setting:: TEMPLATES-DIRS

``DIRS``
~~~~~~~~

默认值: ``[]`` (空列表)

搜索路径的列表, 模板引擎会按照这个顺序查找模板文件文件.

.. setting:: TEMPLATES-APP_DIRS

``APP_DIRS``
~~~~~~~~~~~~

默认值: ``False``

模板引擎是否要在已安装的应用中查找模板源文件.

.. note::

    :djadmin:`django-admin startproject <startproject>` 创建的 :file:`settings.py` 文件中 ``'APP_DIRS': True``.

.. setting:: TEMPLATES-OPTIONS

``OPTIONS``
~~~~~~~~~~~

默认值: ``{}`` (空字典)

传给模板引擎的额外参数. 可用的参数取决于使用的模板引擎. 内置模板引擎参数见
:class:`~django.template.backends.django.DjangoTemplates` 和
:class:`~django.template.backends.jinja2.Jinja2`.

.. setting:: TEST_RUNNER

``TEST_RUNNER``
---------------

默认值: ``'django.test.runner.DiscoverRunner'``

用于启动测试用例类的名称. 见  :ref:`other-testing-frameworks`.

.. setting:: TEST_NON_SERIALIZED_APPS

``TEST_NON_SERIALIZED_APPS``
----------------------------

默认值: ``[]`` (空列表)

为了在 ``TransactionTestCase`` 和没有事务的数据库后端测试中恢复数据库状态,
Django在启动测试时会 :ref:`序列化所有应用的内容 <test-case-serialized-rollback>`, 这样就可以在运行需要的测试时, 从该副本重新加载.

这将拖慢运行测试的启动时间; 如果某些应用程序不需要这个功能, 可以设置成它们的全名(例如 ``'django.contrib.contenttypes'``),
将它们从这个序列化过程中排除.

.. setting:: THOUSAND_SEPARATOR

``THOUSAND_SEPARATOR``
----------------------

默认值: ``','`` (逗号)

格式化数字时默认的千位分隔符. 该配置只有在 :setting:`USE_THOUSAND_SEPARATOR` 设置为 ``True`` 且
:setting:`NUMBER_GROUPING` 大于 ``0`` 时生效.

注意, 如果 :setting:`USE_L10N` 设置为 ``True``, 相应本地决定的格式将更优先被使用.

另见 :setting:`NUMBER_GROUPING`, :setting:`DECIMAL_SEPARATOR` 和
:setting:`USE_THOUSAND_SEPARATOR`.

.. setting:: TIME_FORMAT

``TIME_FORMAT``
---------------

默认值: ``'P'`` (即 ``4 p.m.``)

在系统任何部分中显示时间字段的默认格式. 如果 :setting:`USE_L10N` 设置为 ``True``, 相应本地决定的格式将更优先被使用. 见
:tfilter:`allowed date format strings <date>`.

另见 :setting:`DATE_FORMAT` 和 :setting:`DATETIME_FORMAT`.

.. setting:: TIME_INPUT_FORMATS

``TIME_INPUT_FORMATS``
----------------------

默认值::

    [
        '%H:%M:%S',     # '14:30:59'
        '%H:%M:%S.%f',  # '14:30:59.000200'
        '%H:%M',        # '14:30'
    ]

时间字段输入数据时可接受的格式列表. 格式将按顺序依次被尝试, 使用第一个有效的格式.
注意这些格式字符使用 Python的 :ref:`datetime 模块语法
<strftime-strptime-behavior>`, 而不是 :tfilter:`date` 模板过滤器的格式字符.

当 :setting:`USE_L10N` 设置 ``True``, 相应本地决定的格式将更优先被使用.

另见 :setting:`DATE_INPUT_FORMATS` 和 :setting:`DATETIME_INPUT_FORMATS`.

.. setting:: TIME_ZONE

``TIME_ZONE``
-------------

默认值: ``'America/Chicago'``

当前时区字符串, 或者 ``None``. 见 `时区列表`_.

.. note::
    Django首次发布时 :setting:`TIME_ZONE` 设置为
    ``'America/Chicago'``, 为了向后兼容, 这个全局配置仍然保持为 ``'America/Chicago'`` (项目中的 ``settings.py`` 没有配置时).
    新项目模板默认为 ``'UTC'``.

注意, 它不一定要和服务器的时区一致. 例如, 一个服务器可上可能有多个Django站点, 每个站点都可以有一个单独的时区设置.

当 :setting:`USE_TZ` 设置为 ``False`` 时, 它将成为Django存储所有日期和时间时使用的时区.
当 :setting:`USE_TZ` 设置为 ``True`` 时, 它是Django模板以及表单中显示日期和时间使用的默认时区.

Django将 ``os.environ['TZ']`` 变量设置为你在 :setting:`TIME_ZONE`  配置中指定的时区.
因此, 所有视图和模型都将自动再此时区下运行. 但是, Django不会在以下条件下设置 ``TZ`` 环境变量:

* 如果你使用手动设置配置中所述的
  :ref:`手动设置配置
  <settings-without-django-settings-module>`.

* 如果设置 ``TIME_ZONE = None``. 这会让Django使用系统时区, 但是如果 :setting:`USE_TZ
  = True <USE_TZ>` 则不鼓励这样做, 因为这样会降低本地时间和UTC之间的转换可靠性.

如果Django没有设置 ``TZ`` 环境变量, 则你应该来确保所有流程在正确的环境中运行.

.. note::
    Django无法在Windows环境中使用交替时区. 如果在Windows上运行Django, 则必须将 :setting:`TIME_ZONE` 设置为正确的系统时区.

.. _时区列表: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones

.. setting:: USE_ETAGS

``USE_ETAGS``
-------------

默认值: ``False``

是否输出 ``ETag`` 首部的布尔值. 这种方式会节省带宽但是会降低性能. 它在
:class:`~django.middleware.common.CommonMiddleware` 和 :doc:`cache
framework </topics/cache>` 中使用.

.. setting:: USE_I18N

``USE_I18N``
------------

默认值: ``True``

一个布尔值, 它指定Django的翻译系统是否被启用. 它提供了一种简单的方式去关闭翻译系统.
如果设置为 ``False``, Django会做一些优化不去加载翻译机制.

另见 :setting:`LANGUAGE_CODE`, :setting:`USE_L10N` 和 :setting:`USE_TZ`.

.. note::

    为方便起见, :djadmin:`django-admin
    startproject <startproject>` 创建的 :file:`settings.py` 默认包含 ``USE_I18N = True`` 设置.

.. setting:: USE_L10N

``USE_L10N``
------------

默认值: ``False``

一个布尔值, 用于决定是否默认进行日期格式本地化. 例如, 如果此设置为 ``True``,
Django将使用当前语言环境的格式显示数字和日期.

另见 :setting:`LANGUAGE_CODE`, :setting:`USE_I18N` 和 :setting:`USE_TZ`.

.. note::

    为方便起见, :djadmin:`django-admin
    startproject <startproject>` 创建的 :file:`settings.py` 默认包含 ``USE_L10N = True`` 设置.

.. setting:: USE_THOUSAND_SEPARATOR

``USE_THOUSAND_SEPARATOR``
--------------------------

默认值: ``False``

一个布尔值, 用于决定是否显示数字的千分位分隔符.
当 :setting:`USE_L10N` 设置为 ``True`` 且该配置也为
``True``, Django 将使用 :setting:`THOUSAND_SEPARATOR` 和
:setting:`NUMBER_GROUPING` 的配置来格式化数字, 除非语言环境已经有一个现有的千位分隔符.
如果在本地设置格式中有千分位分隔符,它将具有更高的优先级.

另见 :setting:`DECIMAL_SEPARATOR`, :setting:`NUMBER_GROUPING` 和
:setting:`THOUSAND_SEPARATOR`.

.. setting:: USE_TZ

``USE_TZ``
----------

默认值: ``False``

一个布尔值, 指定在默认情况下是否使用时区感知. 如果设置为 ``True``, Django将在内部使用时区感知日期时间. 否则, Django将在本地时间中使用时区感知的日期.

另见 :setting:`TIME_ZONE`, :setting:`USE_I18N` 和 :setting:`USE_L10N`.

.. note::

    为方便起见, :djadmin:`django-admin
    startproject <startproject>` 创建的 :file:`settings.py` 默认包含 ``USE_TZ = True`` 设置.

.. setting:: USE_X_FORWARDED_HOST

``USE_X_FORWARDED_HOST``
------------------------

默认值: ``False``

一个布尔值, 指定是否优先使用 ``X-Forwarded-Host`` 首部而不是 ``Host`` 首部. 仅当代理有设置此首部时才应启用此配置.

此配置优先于 :setting:`USE_X_FORWARDED_PORT`. 依据
:rfc:`7239#page-7`, ``X-Forwarded-Host`` 首部可以包括端口号, 在这种情况下不需要使用 :setting:`USE_X_FORWARDED_PORT`.

.. setting:: USE_X_FORWARDED_PORT

``USE_X_FORWARDED_PORT``
------------------------

.. versionadded:: 1.9

默认值: ``False``

一个布尔值, 指定是否优先使用 ``X-Forwarded-Port`` 首部而不是 ``SERVER_PORT`` ``META`` 变量.
仅当代理有设置此首部时才应启用此配置.

:setting:`USE_X_FORWARDED_HOST` 优先于此配置.

.. setting:: WSGI_APPLICATION

``WSGI_APPLICATION``
--------------------

默认值: ``None``

Django内置服务器(例如 :djadmin:`runserver`)将使用的WSGI应用对象的完整Python路径.
管理命令 :djadmin:`django-admin startproject <startproject>` 会创建一个简单的 ``wsgi.py`` 文件,
它包含一个可调用的 ``application``, 并将此配置指向该 ``application``.

如果没有设置, 将使用 ``django.core.wsgi.get_wsgi_application()`` 的返回值.
在这种情况下, :djadmin:`runserver` 的行为将与之前的Django版本相同.

.. setting:: YEAR_MONTH_FORMAT

``YEAR_MONTH_FORMAT``
---------------------

默认值: ``'F Y'``

在Django admin change-list 页面中, 当只显示年和月的时候, 默认的日期字段的格式, 也可以被系统的其他部分使用.

例如, 在Django管理站点的change-list页面使用日期过滤时, 指定月份的标题显示月份和年份. 不同的区域设置具有不同的格式.
例如, U.S. English 地区显示"January 2006," 西班牙语地区显示 "2006/January."

如果 :setting:`USE_L10N` 设置为 ``True``, 相应本地决定的格式将更优先被使用.

见 :tfilter:`allowed date format strings <date>`. 另见
:setting:`DATE_FORMAT`, :setting:`DATETIME_FORMAT`, :setting:`TIME_FORMAT`
和 :setting:`MONTH_DAY_FORMAT`.

.. setting:: X_FRAME_OPTIONS

``X_FRAME_OPTIONS``
-------------------

默认值: ``'SAMEORIGIN'``

:class:`~django.middleware.clickjacking.XFrameOptionsMiddleware` 使用的X-Frame-Options首部的默认值. 见
:doc:`clickjacking protection </ref/clickjacking/>` 文档.


Auth
====

:mod:`django.contrib.auth` 的配置.

.. setting:: AUTHENTICATION_BACKENDS

``AUTHENTICATION_BACKENDS``
---------------------------

默认值: ``['django.contrib.auth.backends.ModelBackend']``

认证用户的认证引擎类(字符串形式)的列表. 详见 :ref:`认证引擎文档
<authentication-backends>`.

.. setting:: AUTH_USER_MODEL

``AUTH_USER_MODEL``
-------------------

默认值: ``'auth.User'``

用户模型. 见 :ref:`auth-custom-user`.

.. warning::
    在项目的生命周期内(即生成并迁移依赖于它的模型之后), 你无法不费吹灰之力的更改AUTH_USER_MODEL配置,
    它应在项目开始时设置, 并且它所引用的模型必须在它所在的应用程序的第一次迁移中可用.
    详见 :ref:`auth-custom-user`.

.. setting:: LOGIN_REDIRECT_URL

``LOGIN_REDIRECT_URL``
----------------------

默认值: ``'/accounts/profile/'``

登录后 ``contrib.auth.login`` 视图找不到 ``next`` 参数时. 请求将被重定向到的URL.

例如, :func:`~django.contrib.auth.decorators.login_required` 装饰器就使用了这一点.

此设置还接受可以用于减少配置重复的 :ref:`URL命名模式 <naming-url-patterns>` .
因为你不必在两个位置(``settings`` 和 URLconf)中定义URL.

.. setting:: LOGIN_URL

``LOGIN_URL``
-------------

默认值: ``'/accounts/login/'``

重定向请求以进行登录的URL, 尤其是在使用 :func:`~django.contrib.auth.decorators.login_required` 的视图.

此设置还接受可以用于减少配置重复的 :ref:`URL命名模式 <naming-url-patterns>` .
因为你不必在两个位置(``settings`` 和 URLconf)中定义URL.

.. setting:: LOGOUT_REDIRECT_URL

``LOGOUT_REDIRECT_URL``
-----------------------

.. versionadded:: 1.10

默认值: ``None``

使用 :func:`~django.contrib.auth.views.logout` 退出登录后, 请求被重定向的URL(如果视图没有找到 ``next_page`` 参数).

如果设置为 ``None``, 则在登出后不进行重定向.

此设置还接受可以用于减少配置重复的 :ref:`URL命名模式 <naming-url-patterns>` .
因为你不必在两个位置(``settings`` 和 URLconf)中定义URL.

.. setting:: PASSWORD_RESET_TIMEOUT_DAYS

``PASSWORD_RESET_TIMEOUT_DAYS``
-------------------------------

默认值: ``3``

重置密码链接的有效天数. 用于 :mod:`django.contrib.auth` 的密码重置功能.

.. setting:: PASSWORD_HASHERS

``PASSWORD_HASHERS``
--------------------

见 :ref:`auth_password_storage`.

默认值::

    [
        'django.contrib.auth.hashers.PBKDF2PasswordHasher',
        'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
        'django.contrib.auth.hashers.Argon2PasswordHasher',
        'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
        'django.contrib.auth.hashers.BCryptPasswordHasher',
    ]

.. versionchanged:: 1.10

    以下hashers从默认值中移除::

        'django.contrib.auth.hashers.SHA1PasswordHasher'
        'django.contrib.auth.hashers.MD5PasswordHasher'
        'django.contrib.auth.hashers.UnsaltedSHA1PasswordHasher'
        'django.contrib.auth.hashers.UnsaltedMD5PasswordHasher'
        'django.contrib.auth.hashers.CryptPasswordHasher'

    请考虑使用 :ref:`wrapped password hasher <wrapping-password-hashers>` 来加强数据库中的哈希值.
    如果这不可行, 请将此过程添加到代码中.

    此外, 新增了 ``Argon2PasswordHasher``.

.. setting:: AUTH_PASSWORD_VALIDATORS

``AUTH_PASSWORD_VALIDATORS``
----------------------------

.. versionadded:: 1.9

默认值: ``[]`` (空列表)

用于检查用户密码强度的验证器列表. 有关详细信息, 见 :ref:`password-validation`. 默认情况下, 不执行任何验证接受所有密码.

.. _settings-messages:

Messages
========

:mod:`django.contrib.messages` 中的配置.

.. setting:: MESSAGE_LEVEL

``MESSAGE_LEVEL``
-----------------

默认值: ``messages.INFO``

设置消息框架记录消息的最小级别. 详见 :ref:`message levels <message-level>`.

.. admonition:: Important

   如果你在配置文件中覆盖了 ``MESSAGE_LEVEL``, 并依赖内置的常量,
   你必须直接导入常量模块以避免循环引用的可能性, 例如::

       from django.contrib.messages import constants as message_constants
       MESSAGE_LEVEL = message_constants.DEBUG

   如果需要, 可以直接根据上面 :ref:`常量表
   <message-level-constants>` 中的数值来指定常量.

.. setting:: MESSAGE_STORAGE

``MESSAGE_STORAGE``
-------------------

默认值: ``'django.contrib.messages.storage.fallback.FallbackStorage'``

控制Django存储消息数据的地方.有效值有:

* ``'django.contrib.messages.storage.fallback.FallbackStorage'``
* ``'django.contrib.messages.storage.session.SessionStorage'``
* ``'django.contrib.messages.storage.cookie.CookieStorage'``

详见 :ref:`消息储存后端 <message-storage-backends>`.

使用cookie后端 --
:class:`~django.contrib.messages.storage.cookie.CookieStorage` 和
:class:`~django.contrib.messages.storage.fallback.FallbackStorage` --
在设置它们的cookie时, 使用 :setting:`SESSION_COOKIE_DOMAIN`, :setting:`SESSION_COOKIE_SECURE`
和 :setting:`SESSION_COOKIE_HTTPONLY`.

.. setting:: MESSAGE_TAGS

``MESSAGE_TAGS``
----------------

默认值::

    {
        messages.DEBUG: 'debug',
        messages.INFO: 'info',
        messages.SUCCESS: 'success',
        messages.WARNING: 'warning',
        messages.ERROR: 'error',
    }

设置消息级别和消息标签的映射, 通常在HTML中以CSS类的形式呈现.
如果你指定了一个值, 它将扩展默认值. 这意味着你只需要指定需要覆盖的那些值. 详见 :ref:`message-displaying`.

.. admonition:: Important

   如果你在配置文件中覆盖了 ``MESSAGE_TAGS``, 并依赖内置的常量, 你必须导入 ``constants`` 模块, 以避免循环导入的可能性, 例如::

       from django.contrib.messages import constants as message_constants
       MESSAGE_TAGS = {message_constants.INFO: ''}

   如果需要, 可以直接根据上面 :ref:`常量表
   <message-level-constants>` 中的数值来指定常量.

.. _settings-sessions:

Sessions
========

:mod:`django.contrib.sessions` 的配置.

.. setting:: SESSION_CACHE_ALIAS

``SESSION_CACHE_ALIAS``
-----------------------

默认值: ``'default'``

使用 :ref:`基于缓存的会话储存 <cached-sessions-backend>` 时, 要使用的缓存.

.. setting:: SESSION_COOKIE_AGE

``SESSION_COOKIE_AGE``
----------------------

默认值: ``1209600`` (单位秒, 即2周)

会话cookie的有效期, 单位秒.

.. setting:: SESSION_COOKIE_DOMAIN

``SESSION_COOKIE_DOMAIN``
-------------------------

默认值: ``None``

会话cookie的域. 如果设置类似
``".example.com"`` (注意前面的点号!)的字符串可用于跨域cookie,
``None`` 用于标准域.

在生产环境网站上更新此配置时要谨慎. 如果你更新此配置, 在以前使用标准域cookie的网站上启用跨域cookie,
现有用户cookie将被设置为旧域. 只要这些cookie持续存在, 这会导致他们无法登录.

此配置也会影响 :mod:`django.contrib.messages` 设置的cookie.

.. setting:: SESSION_COOKIE_HTTPONLY

``SESSION_COOKIE_HTTPONLY``
---------------------------

默认值: ``True``

会话cookie是否使用 ``HTTPOnly`` 标志. 如果设置
``True``, 客户端的JavaScript将无法访问会话cookie.

HTTPOnly_ 是HTTP响应的Set-Cookie首部中包含的标志. 它是 :rfc:`2109` 规范中cookie的一部分,
可以作为一种用来降低客户端脚本访问受保护Cookie数据风险的方式.

这使得攻击者将跨站脚本漏洞升级为完全劫持用户的会话变得不那么简单. 没有什么缘由关闭它, 除非: 你的代码依赖于从JavaScript中读取会话cookie, 你可能是做错了.

.. _HTTPOnly: https://www.owasp.org/index.php/HTTPOnly

.. setting:: SESSION_COOKIE_NAME

``SESSION_COOKIE_NAME``
-----------------------

默认值: ``'sessionid'``

会话cookie的名称. 这可以是任何值(只要它与应用中其他的cookie名称不同).

.. setting:: SESSION_COOKIE_PATH

``SESSION_COOKIE_PATH``
-----------------------

默认值: ``'/'``

会话cookie设置的路径. 这个路径应该与你Django的URL路径相匹配, 或者是该路径的父路径.

如果你有多个Django实例在同一个主机名下运行, 这个功能很有用. 他们可以使用不同的cookie路径, 而且每个实例只能看到自己的会话cookie.

.. setting:: SESSION_COOKIE_SECURE

``SESSION_COOKIE_SECURE``
-------------------------

默认值: ``False``

是否使用安全会话cookie. 如果设置为 ``True``, cookie将被标记为"安全", 这意味着浏览器可以确保cookie只在HTTPS连接下发送.

由于如果会话cookie未加密发送, 数据包嗅探器(例如 `Firesheep`_ )就能劫持用户的会话, 这是很常见的,
真的没有什么好的缘由关闭它. 它会阻止你使用不安全的会话请求, 这是一件好事.

.. _Firesheep: http://codebutler.com/firesheep

.. setting:: SESSION_ENGINE

``SESSION_ENGINE``
------------------

默认值: ``'django.contrib.sessions.backends.db'``

控制Django存储会话数据的地方. 包括的引擎有:

* ``'django.contrib.sessions.backends.db'``
* ``'django.contrib.sessions.backends.file'``
* ``'django.contrib.sessions.backends.cache'``
* ``'django.contrib.sessions.backends.cached_db'``
* ``'django.contrib.sessions.backends.signed_cookies'``

详见 :ref:`configuring-sessions`.

.. setting:: SESSION_EXPIRE_AT_BROWSER_CLOSE

``SESSION_EXPIRE_AT_BROWSER_CLOSE``
-----------------------------------

默认值: ``False``

用户关闭浏览器时是否结束会话. 见
:ref:`browser-length-vs-persistent-sessions`.

.. setting:: SESSION_FILE_PATH

``SESSION_FILE_PATH``
---------------------

默认值: ``None``

使用基于文件的会话存储情况下, 设置Django存储会话数据的目录. 使用默认值(``None``)时, Django将使用系统的临时目录.


.. setting:: SESSION_SAVE_EVERY_REQUEST

``SESSION_SAVE_EVERY_REQUEST``
------------------------------

默认值: ``False``

是否每个请求都保存一次会话数据. 如果设置为 ``False`` (默认值), 那么会话数据只有在被修改时才被保存 --
即它的字典值被赋值或删除. 即使此设置处于活动状态, 也不会创建空会话.

.. setting:: SESSION_SERIALIZER

``SESSION_SERIALIZER``
----------------------

默认值: ``'django.contrib.sessions.serializers.JSONSerializer'``

序列化会话数据的序列化类的完整路径. 包括:

* ``'django.contrib.sessions.serializers.PickleSerializer'``
* ``'django.contrib.sessions.serializers.JSONSerializer'``

详见 :ref:`session_serialization`, 包括当使用
:class:`~django.contrib.sessions.serializers.PickleSerializer`  时可能会出现远程代码执行的警告.

Sites
=====

:mod:`django.contrib.sites` 的配置.

.. setting:: SITE_ID

``SITE_ID``
-----------

默认值: 未定义

当前网站在 ``django_site`` 数据库表中的ID, 整数. 这样, 应用数据就可以挂到特定的站点上, 一个数据库可以管理多个站点的内容.


.. _settings-staticfiles:

Static Files
============

Settings for :mod:`django.contrib.staticfiles`.

.. setting:: STATIC_ROOT

``STATIC_ROOT``
---------------

默认值: ``None``

:djadmin:`collectstatic` 收集静态文件部署的目录的绝对路径.

例如: ``"/var/www/example.com/static/"``

在配置了 :doc:`staticfiles</ref/contrib/staticfiles>` 的应用(如默认的项目模板), :djadmin:`collectstatic`
管理命令会将静态文件收集到该目录. 详细应用见
:doc:`管理静态文件 </howto/static-files/index>`.

.. warning::

    这应该是一个初始为空的目标目录, 用于将你的静态文件从各个位置收集到一个目录中, 以方便部署.
    它 **不是** 永久存储静态文件的地方. 你应该在那些会被
    :doc:`staticfiles</ref/contrib/staticfiles>` 的
    :setting:`finders<STATICFILES_FINDERS>` 找到的目录中进行, 默认情况下, 这些目录是你应用的
    ``'static/'`` 子目录和
    :setting:`STATICFILES_DIRS` 包含的目录.

.. setting:: STATIC_URL

``STATIC_URL``
--------------

默认值: ``None``

引用位于 :setting:`STATIC_ROOT` 中的静态文件时的URL.

例如: ``"/static/"`` 或 ``"http://static.example.com/"``

如果不为 ``None``, 它将被用作
:ref:`资源定义<form-asset-paths>` (``Media`` 类) 和
:doc:`staticfiles app</ref/contrib/staticfiles>`.

如果设置为非空值必须以斜线结束.

您可能需要 :ref:`在开发中配置这些文件
<serving-static-files-in-development>`, 并且在生产中必须要这样做.

.. setting:: STATICFILES_DIRS

``STATICFILES_DIRS``
--------------------

默认值: ``[]`` (空列表)

定义在启用了 ``FileSystemFinder`` 时静态文件应用将遍历的额外目录, 例如, 使用 :djadmin:`collectstatic` 或 :djadmin:`findstatic` 管理命令或使用静态文件服务视图时.

应将此设置为包含其他文件目录的完整路径的字符串列表, 例如::

    STATICFILES_DIRS = [
        "/home/special.polls.com/polls/static",
        "/home/polls.com/polls/static",
        "/opt/webfiles/common",
    ]

请注意, 这些路径应该使用Unix风格的斜线, 即使在Windows上也是如此
(例如 ``"C:/Users/user/mysite/extra_static_content"``).

前缀 (可选)
~~~~~~~~~~~~~~~~~~~

如果想用一个额外的命名空间来引用其中一个位置的文件, 可以 **可选地** 设置一个前缀类似 ``(prefix, path)``
的元组, 例如.::

    STATICFILES_DIRS = [
        # ...
        ("downloads", "/opt/webfiles/stats"),
    ]

例如, 假如你 :setting:`STATIC_URL` 设置为 ``'/static/'``,
:djadmin:`collectstatic` 管理命令将收集 :setting:`STATIC_ROOT` 的 ``'downloads'`` 子目录的"stats"文件.

这可以使你在模板中用
``'/opt/webfiles/stats/polls_20101022.tar.gz'`` 引用本地文件
``'/static/downloads/polls_20101022.tar.gz'`` 例如:

.. code-block:: html+django

    <a href="{% static "downloads/polls_20101022.tar.gz" %}">

.. setting:: STATICFILES_STORAGE

``STATICFILES_STORAGE``
-----------------------

默认值: ``'django.contrib.staticfiles.storage.StaticFilesStorage'``

:djadmin:`collectstatic` 命令收集静态文件时使用的文件存储引擎.

在此配置中定义的存储后端的即用例可以在 ``django.contrib.staticfiles.storage.staticfiles_storage`` 中找到.

例见 :ref:`staticfiles-from-cdn`.

.. setting:: STATICFILES_FINDERS

``STATICFILES_FINDERS``
-----------------------

默认值::

    [
        'django.contrib.staticfiles.finders.FileSystemFinder',
        'django.contrib.staticfiles.finders.AppDirectoriesFinder',
    ]

如何找到不同位置的静态文件的查找器引擎列表.

默认情况下, 将查找存储在 :setting:`STATICFILES_DIRS` 配置中的文件
(使用 ``django.contrib.staticfiles.finders.FileSystemFinder``) 和每个应用的 ``static`` 子目录中的文件
(使用
``django.contrib.staticfiles.finders.AppDirectoriesFinder``). 如果存在多个同名文件, 将使用第一个找到的文件.

有一个查找器是默认禁用的:
``django.contrib.staticfiles.finders.DefaultStorageFinder``. 如果添加到你的
:setting:`STATICFILES_FINDERS` 配置中, 它将在 :setting:`DEFAULT_FILE_STORAGE`
配置所定义的默认文件存储中查找静态文件.

.. note::

    在使用 ``AppDirectoriesFinder`` 查找器时, 需要将应用添加到
    :setting:`INSTALLED_APPS` 配置中, 以确保你的应用程序可以通过staticfiles找到.

静态文件查找器目前被认为是一个私有接口, 因此这个接口是没有文档的.

核心配置索引
===========================

缓存
-----
* :setting:`CACHES`
* :setting:`CACHE_MIDDLEWARE_ALIAS`
* :setting:`CACHE_MIDDLEWARE_KEY_PREFIX`
* :setting:`CACHE_MIDDLEWARE_SECONDS`

数据库
--------
* :setting:`DATABASES`
* :setting:`DATABASE_ROUTERS`
* :setting:`DEFAULT_INDEX_TABLESPACE`
* :setting:`DEFAULT_TABLESPACE`

调式
---------
* :setting:`DEBUG`
* :setting:`DEBUG_PROPAGATE_EXCEPTIONS`

邮件
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

错误报告
---------------
* :setting:`DEFAULT_EXCEPTION_REPORTER_FILTER`
* :setting:`IGNORABLE_404_URLS`
* :setting:`MANAGERS`
* :setting:`SILENCED_SYSTEM_CHECKS`

.. _file-upload-settings:

文件上传
------------
* :setting:`DEFAULT_FILE_STORAGE`
* :setting:`FILE_CHARSET`
* :setting:`FILE_UPLOAD_HANDLERS`
* :setting:`FILE_UPLOAD_MAX_MEMORY_SIZE`
* :setting:`FILE_UPLOAD_PERMISSIONS`
* :setting:`FILE_UPLOAD_TEMP_DIR`
* :setting:`MEDIA_ROOT`
* :setting:`MEDIA_URL`

国际化 (``i18n``/``l10n``)
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

日志
-------
* :setting:`LOGGING`
* :setting:`LOGGING_CONFIG`

模型
------
* :setting:`ABSOLUTE_URL_OVERRIDES`
* :setting:`FIXTURE_DIRS`
* :setting:`INSTALLED_APPS`

安全
--------
* 跨站请求保护

  * :setting:`CSRF_COOKIE_DOMAIN`
  * :setting:`CSRF_COOKIE_NAME`
  * :setting:`CSRF_COOKIE_PATH`
  * :setting:`CSRF_COOKIE_SECURE`
  * :setting:`CSRF_FAILURE_VIEW`
  * :setting:`CSRF_HEADER_NAME`
  * :setting:`CSRF_TRUSTED_ORIGINS`

* :setting:`SECRET_KEY`
* :setting:`X_FRAME_OPTIONS`

序列化
-------------
* :setting:`DEFAULT_CHARSET`
* :setting:`SERIALIZATION_MODULES`

模板
---------
* :setting:`TEMPLATES`

测试
-------
* Database: :setting:`TEST <DATABASE-TEST>`
* :setting:`TEST_NON_SERIALIZED_APPS`
* :setting:`TEST_RUNNER`

URLs
----
* :setting:`APPEND_SLASH`
* :setting:`PREPEND_WWW`
* :setting:`ROOT_URLCONF`
