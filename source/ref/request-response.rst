============================
请求和响应对象
============================

.. module:: django.http
   :synopsis: Classes dealing with HTTP requests and responses.

概述
==============

Django使用Request对象和Response对象在系统间传递状态.

当请求一个页面时, Django会创建一个包含请求元数据的 :class:`HttpRequest` 对象.
然后加载相应的视图, 将 :class:`HttpRequest` 作为视图函数的第一个参数传入.
视图函数负责返回一个 :class:`HttpResponse` 对象.

本文档对 :class:`HttpRequest` 和 :class:`HttpResponse` 对象的API进行说明, 这些API定义在 :mod:`django.http` 模块中.

``HttpRequest`` 对象
=======================

.. class:: HttpRequest

.. _httprequest-attributes:

属性
----------

除非另有说明, 否则所有属性都应视为只读.

.. attribute:: HttpRequest.scheme

    代表请求协议的字符串 (通常是 ``http`` 或 ``https``).

.. attribute:: HttpRequest.body

    表示原始HTTP请求正文的字节字符串. 它对处理非HTML表单数据非常有用: 二进制图片, XML等.
    如果要处理常规表单数据, 请使用 ``HttpRequest.POST``.

    还可以使用类似文件的接口从HttpRequest读取数据. 详见 :meth:`HttpRequest.read()`.

.. attribute:: HttpRequest.path

    请求页面完整路径, 不包括scheme和domain.

    例如: ``"/music/bands/the_beatles/"``

.. attribute:: HttpRequest.path_info

    在某些Web服务器配置下, 主机名后的URL被分成脚本前缀部分和路径信息部分.
    ``path_info`` 属性始终会包含路径信息部分, 不论使用什么Web服务器,
    使用它代替 :attr:`~HttpRequest.path` 可以使代码在测试和开发环境中更好地切换.

    例如, 如果应用的 ``WSGIScriptAlias`` 设置为 ``"/minfo"``, ``path`` 是 ``"/minfo/music/bands/the_beatles/"``
    时 ``path_info`` 是 ``"/music/bands/the_beatles/"``.

.. attribute:: HttpRequest.method

    一个字符串, 表示请求使用的HTTP方法. 大些字母形式. 例如::

        if request.method == 'GET':
            do_something()
        elif request.method == 'POST':
            do_something_else()

.. attribute:: HttpRequest.encoding

    一个字符串, 表示提交的数据的编码(如果为 ``None``, 表示使用 :setting:`DEFAULT_CHARSET` 设置).
    可以修改这个属性值来改变访问表单数据使用的编码. 修改后的属性访问(从 ``GET`` 或 ``POST`` 读取)都将使用新的 ``encoding`` 值.
    这对于访问数据不是 :setting:`DEFAULT_CHARSET` 编码时非常有用.

.. attribute:: HttpRequest.content_type

    .. versionadded:: 1.10

    从 ``CONTENT_TYPE`` 请求头中解析到的MIME类型字符串.

.. attribute:: HttpRequest.content_params

    .. versionadded:: 1.10

    ``CONTENT_TYPE`` 请求头中的键值参数字典.

.. attribute:: HttpRequest.GET

    包含所有HTTP GET请求参数的类字典对象. 详见 :class:`QueryDict`.

.. attribute:: HttpRequest.POST

    包含所有HTTP POST请求参数的类字典对象, 前提是请求包含了表单数据. 详见 :class:`QueryDict`.
    如果需要访问post提交的原始或非表单数据, 可以使用 :attr:`HttpRequest.body` 属性.

    有可能会有POST请求但是 ``POST`` 为空字典的情况 -- 比如, 不包含表单数据的POST请求.
    因此, 不要使用 ``if request.POST`` 来判断是否为POST请求; 应该使用 ``if request.method ==
    "POST"`` (见上文).

    注意: ``POST``  **不** 包含文件上传信息. 见 ``FILES``.

.. attribute:: HttpRequest.COOKIES

    包含所有cookie的Python字典. 键和值都是字符串.

.. attribute:: HttpRequest.FILES

    包含所有上传文件的类字典对象. ``FILES`` 中的键为 ``<input type="file" name="" />`` 中的 ``name``.
    ``FILES`` 中的值是一个 :class:`~django.core.files.uploadedfile.UploadedFile` 对象.

    详见 :doc:`/topics/files`.

    注意, 只有 ``<form>`` 中带有 ``enctype="multipart/form-data"`` 的POST请求, ``FILES`` 才会有数据, 否则将是一个空的类字典对象.

.. attribute:: HttpRequest.META

    包含所有可用HTTP首部的Python字典. 可用的首部取决了客户端和服务端, 下面是一些例子:

    * ``CONTENT_LENGTH`` -- 请求体的长度 (字符串).
    * ``CONTENT_TYPE`` -- 请求体的 MIME 类型.
    * ``HTTP_ACCEPT`` -- 可接受的响应类型.
    * ``HTTP_ACCEPT_ENCODING`` -- 可接受的响应编码.
    * ``HTTP_ACCEPT_LANGUAGE`` -- 可接受的响应语言.
    * ``HTTP_HOST`` -- 客户端发送的HTTP HOST首部.
    * ``HTTP_REFERER`` -- referrer页面, 如果有的话.
    * ``HTTP_USER_AGENT`` -- 客户端的user-agent字符串.
    * ``QUERY_STRING`` -- 查询字符串, 单独(未解析)的一个字符串.
    * ``REMOTE_ADDR`` -- 客户端的IP地址.
    * ``REMOTE_HOST`` -- 客户端的hostname.
    * ``REMOTE_USER`` -- Web服务器认证的用户, 如果有的话.
    * ``REQUEST_METHOD`` -- ``"GET"`` 或 ``"POST"`` 等字符串.
    * ``SERVER_NAME`` -- 服务端的hostname.
    * ``SERVER_PORT`` --服务端的端口(字符串).

    从上面可以看出, 除了 ``CONTENT_LENGTH`` 和 ``CONTENT_TYPE`` 之外, 请求中的HTTP首部都会被转换为 ``META`` 中的键,
    它将所有字符转换为大写用下划线连接, 并加上 ``HTTP_`` 前缀, 例如名为 ``X-Bender`` 的首部将被映射到 ``META`` 键
    ``HTTP_X_BENDER``.

    注意, :djadmin:`runserver` 会取消所有名称中带有下划线的请求头, 因此您不会在 ``META`` 中看到它们.
    这可以防止基于下划线和破折号之间的歧义的报文欺骗, 这两种符号在WSGI环境变量中都被规范化为下划线.
    它与Nginx和apache2.4+等Web服务器的行为相匹配.

.. attribute:: HttpRequest.resolver_match

    代表解析后URL的 :class:`~django.urls.ResolverMatch` 实例. 该属性只有在URL解析后才会被设置,
    因此它在视图中是可用的, 但是在URL解析之前执行的中间件中不可用(不过可以在 :meth:`process_view` 中使用).

应用程序代码设置的属性
----------------------------------

Django不会自己设置这些属性, 而是由应用程序设置使用它们.

.. attribute:: HttpRequest.current_app

    :ttag:`url` 模板标签使用它的值作为 :func:`~django.urls.reverse()` 的 ``current_app`` 参数.

.. attribute:: HttpRequest.urlconf

    这将作为当前请求的根URLconf, 覆盖 :setting:`ROOT_URLCONF` 设置. 详见 :ref:`how-django-processes-a-request`.

    ``urlconf`` 可以设置为 ``None`` 用来重置在之前中间件中所做的修改, 并返回使用 :setting:`ROOT_URLCONF`.

    .. versionchanged:: 1.9

        在之前版本中, 设置 ``urlconf=None`` 会引发 :exc:`~django.core.exceptions.ImproperlyConfigured` 异常.

中间件设置的属性
----------------------------

Django的contrib应用中的一些中间件会在请求上设置属性.
如果在请求中没有见到这些属性, 请求确保 :setting:`MIDDLEWARE` 中使用了正确的中间件.

.. attribute:: HttpRequest.session

    出自 :class:`~django.contrib.sessions.middleware.SessionMiddleware`: 一个可读写的代表当前session的类字典对象.

.. attribute:: HttpRequest.site

    出自 :class:`~django.contrib.sites.middleware.CurrentSiteMiddleware`:
    :func:`~django.contrib.sites.shortcuts.get_current_site()` 返回的
    :class:`~django.contrib.sites.models.Site` 或者
    :class:`~django.contrib.sites.requests.RequestSite` 实例, 代表当前站点.

.. attribute:: HttpRequest.user

    出自 :class:`~django.contrib.auth.middleware.AuthenticationMiddleware`:
    代表当前登录用户的 :setting:`AUTH_USER_MODEL` 实例. 如果用户没有登录, ``user`` 会返回
    一个 :class:`~django.contrib.auth.models.AnonymousUser` 实例. 可以使用
    :attr:`~django.contrib.auth.models.User.is_authenticated` 来区分它们, 例如::

        if request.user.is_authenticated:
            ... # Do something for logged-in users.
        else:
            ... # Do something for anonymous users.

方法
-------

.. method:: HttpRequest.get_host()

    根据 ``HTTP_X_FORWARDED_HOST`` (如果启用了 :setting:`USE_X_FORWARDED_HOST`)
    和 ``HTTP_HOST`` 首部按顺序返回请求原始host. 如果没有提供响应的值,
    则使用 ``SERVER_NAME`` 和 ``SERVER_PORT`` 组合, 详见 :pep:`3333`.

    例如: ``"127.0.0.1:8000"``

    .. note:: 当使用了多重代理时 :meth:`~HttpRequest.get_host()` 方法会失败. 有个解决方法是使用中间件重新代理首部, 例如::

            from django.utils.deprecation import MiddlewareMixin

            class MultipleProxyMiddleware(MiddlewareMixin):
                FORWARDED_FOR_FIELDS = [
                    'HTTP_X_FORWARDED_FOR',
                    'HTTP_X_FORWARDED_HOST',
                    'HTTP_X_FORWARDED_SERVER',
                ]

                def process_request(self, request):
                    """
                    Rewrites the proxy headers so that only the most
                    recent proxy is used.
                    """
                    for field in self.FORWARDED_FOR_FIELDS:
                        if field in request.META:
                            if ',' in request.META[field]:
                                parts = request.META[field].split(',')
                                request.META[field] = parts[-1].strip()

        该中间件应该放在所有依赖 :meth:`~HttpRequest.get_host()` 的中间件之前 -- 例如,
        :class:`~django.middleware.common.CommonMiddleware` 或者
        :class:`~django.middleware.csrf.CsrfViewMiddleware`.

.. method:: HttpRequest.get_port()

    .. versionadded:: 1.9

    根据 ``HTTP_X_FORWARDED_PORT`` (如果启用了 :setting:`USE_X_FORWARDED_PORT` 设置)
    和 ``SERVER_PORT`` ``META`` 变量, 按顺序返回请求的原始端口.

.. method:: HttpRequest.get_full_path()

    返回带有查询字符串的 ``path`` (如果有的话).

    例如: ``"/music/bands/the_beatles/?print=true"``

.. method:: HttpRequest.build_absolute_uri(location)

    返回 ``location`` 的绝对URI. 如果没有则设置为 ``request.get_full_path()``.

    如果URI已经是一个绝对的URI则不会修改, 否则, 使用请求中的服务器相关的变量构建绝对URI.

    例如: ``"https://example.com/music/bands/the_beatles/?print=true"``

    .. note::

        不鼓励在同一站点中混用HTTP和HTTPS, 因此,
        :meth:`~HttpRequest.build_absolute_uri()` 会始终返回与当前请求相同协议的绝对URI.
        如果你想将用户重定向到HTTPS, 最好让Web服务器将所有HTTP请求重定向到HTTPS.

.. method:: HttpRequest.get_signed_cookie(key, default=RAISE_ERROR, salt='', max_age=None)

    返回签名的cookie值, 如果签名不再有效则引发 ``django.core.signing.BadSignature`` 异常.
    如果提供了 ``default`` 参数, 则不会引发异常而是返回default值.

    可选参数 ``salt`` 用来提供额外的保护以防止对秘钥的暴力攻击. ``max_age`` 参数用于检查cookie的签名时间戳, 确保不超过 ``max_age`` 秒.

    例如::

        >>> request.get_signed_cookie('name')
        'Tony'
        >>> request.get_signed_cookie('name', salt='name-salt')
        'Tony' # assuming cookie was set using the same salt
        >>> request.get_signed_cookie('non-existing-cookie')
        ...
        KeyError: 'non-existing-cookie'
        >>> request.get_signed_cookie('non-existing-cookie', False)
        False
        >>> request.get_signed_cookie('cookie-that-was-tampered-with')
        ...
        BadSignature: ...
        >>> request.get_signed_cookie('name', max_age=60)
        ...
        SignatureExpired: Signature age 1677.3839159 > 60 seconds
        >>> request.get_signed_cookie('name', False, max_age=60)
        False

    详见 :doc:`加密签名 </topics/signing>` .

.. method:: HttpRequest.is_secure()

    如果请求是安全的则返回 ``True`` , 即使用HTTPS.

.. method:: HttpRequest.is_ajax()

    如果请求是通过 ``XMLHttpRequest`` 发起的则返回 ``True``, 方法是检查 ``HTTP_X_REQUESTED_WITH`` 首部中是否包含 ``'XMLHttpRequest'``.
    大部分现在的JavaScript库都会发送这个首部. 如果你自定义了XMLHttpRequest调用(浏览器端), 想要 ``is_ajax()`` 正常工作的话需要手动设置这个首部.

    如果响应根据是否通过AJAX请求而有所不同, 并且您正在使用某种形式的缓存, 如Django的
    :mod:`缓存中间件<django.middleware.cache>`, 则应使用 :func:`vary_on_headers('X-Requested-With')
    <django.views.decorators.vary.vary_on_headers>` 来装饰视图以便正确缓存响应.

.. method:: HttpRequest.read(size=None)
.. method:: HttpRequest.readline()
.. method:: HttpRequest.readlines()
.. method:: HttpRequest.xreadlines()
.. method:: HttpRequest.__iter__()

    实现从HttpRequest实例类似文件读取的接口. 这使得可以以流的方式读取请求.
    常见的用例是使用迭代解析器处理大型XML负载, 而无需在内存中构建整个XML树.

    给定此标准接口, HttpRequest实例可以直接传递给XML解析器, 如ElementTree::

        import xml.etree.ElementTree as ET
        for element in ET.iterparse(request):
            process(element)


``QueryDict`` 对象
=====================

.. class:: QueryDict

在 :class:`HttpRequest` 对象中, ``GET`` 和 ``POST`` 属性都是 ``django.http.QueryDict`` 实例,
这是一个类似字典的类, 用来处理一个键的多个值. 这很有用, 因为有些HTML表单, 尤其是 ``<select multiple>``, 会传递一个键的多个值.

``QueryDict`` 在 ``request.POST`` 和 ``request.GET`` 的一个正常请求/响应周期中是不可变的. 如果有修改需要请使用 ``.copy()``.

方法
-------

:class:`QueryDict` 实现了字典的所有标准方法, 因为它是字典的子类. 下面列出了例外情况:

.. method:: QueryDict.__init__(query_string=None, mutable=False, encoding=None)

    基于 ``query_string`` 实例化的 ``QueryDict`` 对象.

    >>> QueryDict('a=1&a=2&c=3')
    <QueryDict: {'a': ['1', '2'], 'c': ['3']}>

    如果没有传入 ``query_string``, 返回的 ``QueryDict`` 为空(没有键和值).

    大多数的 ``QueryDict``, 特别是 ``request.POST`` 和 ``request.GET`` 中的都是不可变的. 如果你在自己实例化时, 可以在 ``__init__()``
    中传入 ``mutable=True`` 来创建可变的实例.

    设置的键和值都会使用 ``encoding`` 编码方式编码. 如果没有设置编码方式, 则使用默认的 :setting:`DEFAULT_CHARSET`.

.. method:: QueryDict.__getitem__(key)

    返回给定键的值. 如果该键有多个值, ``__getitem__()`` 返回最后一个值.
    如果键不存在则会引发 ``django.utils.datastructures.MultiValueDictKeyError``.
    (它是Python ``KeyError`` 异常的一个子类, 所以可以使用 ``KeyError`` 捕获该异常.)

.. method:: QueryDict.__setitem__(key, value)

    为指定键设置值 ``[value]`` (只有一个 ``value`` 元素的Python列表).
    注意, 和其他带有限制的字典函数一样, 它仅可以在可变的 ``QueryDict`` 上调用(比如通过 ``copy()`` 创建的副本).

.. method:: QueryDict.__contains__(key)

    如果设置了给定的键则返回 ``True`` . 这使你能够 ``if "foo"
    in request.GET`` 这样.

.. method:: QueryDict.get(key, default=None)

    使用和上面 ``__getitem__()`` 一样逻辑, 如果键不存在则返回默认值.

.. method:: QueryDict.setdefault(key, default=None)

    和普通字典的 ``setdefault()`` 方法一样, 只是它内部使用的是 ``__setitem__()``.

.. method:: QueryDict.update(other_dict)

    接收一个 ``QueryDict`` 或普通字典. 和普通字典的 ``update()`` 方法类似, 除了当键存在时它是 **添加** 新值而不是替换. 例如::

        >>> q = QueryDict('a=1', mutable=True)
        >>> q.update({'a': '2'})
        >>> q.getlist('a')
        ['1', '2']
        >>> q['a'] # returns the last
        '2'

.. method:: QueryDict.items()

    类似普通字典的 ``items()`` 方法, 除了它使用与 ``__getitem__()`` 相同逻辑取最后一个值. 例如::

        >>> q = QueryDict('a=1&a=2&a=3')
        >>> q.items()
        [('a', '3')]

.. method:: QueryDict.iteritems()

    类似普通字典的 ``iteritems()`` 方法. 和 :meth:`QueryDict.items()` 一样使用与 :meth:`QueryDict.__getitem__()` 相同的逻辑取最后一个值.

    仅适用于Python2.

.. method:: QueryDict.iterlists()

    类似 :meth:`QueryDict.iteritems()`, 只是它以列表形式返回字典成员的所有值.

    仅适用于Python2.

.. method:: QueryDict.values()

    类似普通字典的 ``values()`` 方法, 除了它使用与 ``__getitem__()`` 相同的逻辑返回最后一个值. 例如::

        >>> q = QueryDict('a=1&a=2&a=3')
        >>> q.values()
        ['3']

.. method:: QueryDict.itervalues()

    类似 :meth:`QueryDict.values()`, 只是它返回的是迭代器.

    仅适用于Python2.

此外, ``QueryDict`` 还有如下方法:

.. method:: QueryDict.copy()

    使用Python标准库 ``copy.deepcopy()`` 返回对象的副本. 返回的对象是可变的即使原始对象不可变.

.. method:: QueryDict.getlist(key, default=None)

    以列表形式返回给定键的数据, 如果键不存在且没有提供默认值则返回空列表.
    除了默认值不是列表的情况, 它的返回值一定是列表.

.. method:: QueryDict.setlist(key, list_)

    为指定键设置值 ``list_`` (与 ``__setitem__()`` 不同).

.. method:: QueryDict.appendlist(key, item)

    将项追加到与键相关联的列表中.

.. method:: QueryDict.setlistdefault(key, default_list=None)

    类似 ``setdefault``, 只是它接收一个列表而不是单个值.

.. method:: QueryDict.lists()

    类似 :meth:`items()`, 它以列表形式返回字典成员的所有值. 例如::

        >>> q = QueryDict('a=1&a=2&a=3')
        >>> q.lists()
        [('a', ['1', '2', '3'])]

.. method:: QueryDict.pop(key)

    移除并返回字典中给定键对应值的列表. 如果键不存在会引发 ``KeyError`` 异常. 例如::

        >>> q = QueryDict('a=1&a=2&a=3', mutable=True)
        >>> q.pop('a')
        ['1', '2', '3']

.. method:: QueryDict.popitem()

    删除并返回字典的任意成员(因为没有顺序的概念), 返回一个元组, 包含键和所有值的列表. 如果对空字典调用会引发 ``KeyError`` 异常. 例如::

        >>> q = QueryDict('a=1&a=2&a=3', mutable=True)
        >>> q.popitem()
        ('a', ['1', '2', '3'])

.. method:: QueryDict.dict()

    返回 ``QueryDict`` 的 ``dict`` 形式. 对于 ``QueryDict`` 中的每个(键, 列表), ``dict`` 是(键, item), item是列表中的一个元素,
    使用与 :meth:`QueryDict.__getitem__()` 相同逻辑返回::

        >>> q = QueryDict('a=1&a=3&a=5')
        >>> q.dict()
        {'a': '5'}

.. method:: QueryDict.urlencode(safe=None)

    返回query-string格式字符串. 例如::

        >>> q = QueryDict('a=2&b=3&b=5')
        >>> q.urlencode()
        'a=2&b=3&b=5'

    或者, 可以向urlencode传递不需要编码的字符. 例如::

        >>> q = QueryDict(mutable=True)
        >>> q['next'] = '/a&b/'
        >>> q.urlencode(safe='/')
        'next=/a%26b/'

``HttpResponse`` 对象
========================

.. class:: HttpResponse

与自动创建的 :class:`HttpRequest` 对象不同, :class:`HttpResponse` 是你需要在每个视图里实例化, 填充和返回的.

:class:`HttpResponse` 类定义在 :mod:`django.http` 模块中.

用例
-----

传递字符串
~~~~~~~~~~~~~~~

典型的用法是将页面内容以字符串的形式传递给 :class:`HttpResponse`::

    >>> from django.http import HttpResponse
    >>> response = HttpResponse("Here's the text of the Web page.")
    >>> response = HttpResponse("Text only, please.", content_type="text/plain")

如果想要添加内容, ``response`` 可以像文件对象那样::

    >>> response = HttpResponse()
    >>> response.write("<p>Here's the text of the Web page.</p>")
    >>> response.write("<p>Here's another paragraph.</p>")

传递迭代器
~~~~~~~~~~~~~~~~~

你也可以传递一个迭代器给 ``HttpResponse``. ``HttpResponse`` 会立即消费这个迭代器, 然后将其内容储存为字符串.
如果对象带有 ``close()`` 方法, 例如文件或迭代器, 会立即关闭.

如果想以流的形式返回给客户端, 请使用 :class:`StreamingHttpResponse` 类.

.. versionchanged:: 1.10

    WSGI服务器响应时调用 ``close()`` 方法, 具有 ``close()`` 方法的对象将会被close.

设置首部字段
~~~~~~~~~~~~~~~~~~~~~

可以将它当做一个字典来设置或删除response首部字段::

    >>> response = HttpResponse()
    >>> response['Age'] = 120
    >>> del response['Age']

注意与字典不同的是, ``del`` 删除不存在的字段时不会引发 ``KeyError`` 异常.

如果要设置 ``Cache-Control`` 和 ``Vary`` 首部字段, 建议使用 :mod:`django.utils.cache` 中的
:func:`~django.utils.cache.patch_cache_control` 和 :func:`~django.utils.cache.patch_vary_headers`
方法, 因为这两个字段是以逗号分隔的多个值. 而这些"patch"方法确保不会丢失其他值, 例如中间件添加的值.

HTTP首部字段不能包含换行. 如果尝试为首部字段设置包含换行符(CR或LF)的字符串会引发 ``BadHeaderError``.

告诉浏览器将响应作为文件处理
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

要告诉浏览器将响应作为文件处理, 需要使用 ``content_type`` 参数并设置 ``Content-Disposition`` 首部.
例如, 以Microsoft Excel电子表格的形式响应::

    >>> response = HttpResponse(my_data, content_type='application/vnd.ms-excel')
    >>> response['Content-Disposition'] = 'attachment; filename="foo.xls"'

关于 ``Content-Disposition`` 首部没有Django相关的内容, 很容易忘记语法, 所以我们将其放在这里.

属性
----------

.. attribute:: HttpResponse.content

    表单内容的字节字符串, 必要时由Unicode编码.

.. attribute:: HttpResponse.charset

    表示响应的编码的字符串. 如果 ``HttpResponse`` 在实例化时没有指定, 将从
    ``content_type`` 中提取, 如果不成功将使用 :setting:`DEFAULT_CHARSET` 设置.

.. attribute:: HttpResponse.status_code

    响应的 :rfc:`HTTP状态码 <7231#section-6>`.

    .. versionchanged:: 1.9

        除非明确设置了 :attr:`reason_phrase` , 否则在构造函数之外修改
        ``status_code`` 的值也会修改 ``reason_phrase``.

.. attribute:: HttpResponse.reason_phrase

    响应的HTTP原因短语.

    .. versionchanged:: 1.9

        ``reason_phrase`` 不再默认为所有大写字母. 它现在使用 :rfc:`HTTP标准 <7231#section-6.1>` 默认原因短语.

        除非显式设置, ``reason_phrase`` 由 :attr:`status_code` 值决定.

.. attribute:: HttpResponse.streaming

    此选项总为 ``False``.

    此属性的存在是为了让中间件能够将流式响应与常规响应区别对待.

.. attribute:: HttpResponse.closed

    如果响应已关闭则为 ``True``.

方法
-------

.. method:: HttpResponse.__init__(content='', content_type=None, status=200, reason=None, charset=None)

    使用页面内容和内容类型来实例化一个 ``HttpResponse`` 对象.

    ``content`` 应该是一个迭代器或字符串. 如果是迭代器, 它应该返回字符串, 然后这些字符串连接在一起形成响应的内容.
    如果它不是迭代器或字符串, 将在访问时转换为字符串.

    ``content_type`` 是可选地由字符集编码的MIME类型, 用于填充HTTP的 ``Content-Type`` 首部.
    如果没有指定, 则由 :setting:`DEFAULT_CONTENT_TYPE` 和
    :setting:`DEFAULT_CHARSET` 设置组成, 默认为: "`text/html; charset=utf-8`".

    ``status`` 是响应的 :rfc:`HTTP状态码<7231#section-6>`.

    ``reason`` 是HTTP响应短语. 如果没有指定则使用默认值.

    ``charset`` 是响应的编码字符集. 如果没有设置, 将从 ``content_type`` 中提取, 如果不成功, 将使用 :setting:`DEFAULT_CHARSET` 配置.

.. method:: HttpResponse.__setitem__(header, value)

    设置给定首部的值. ``header`` 和 ``value`` 都必须是字符串.

.. method:: HttpResponse.__delitem__(header)

    删除给定名字的首部. 如果给定首部不存在不会引发异常. 大小写不敏感.

.. method:: HttpResponse.__getitem__(header)

    返回给定名字的首部值. 大小写不敏感.

.. method:: HttpResponse.has_header(header)

    检查首部是否包含给定的名字, 返回 ``True`` 或者 ``False``. 大小写不敏感.

.. method:: HttpResponse.setdefault(header, value)

    设置首部, 除非它已经存在.

.. method:: HttpResponse.set_cookie(key, value='', max_age=None, expires=None, path='/', domain=None, secure=None, httponly=False)

    设置Cookie. 参数与Python标准库 :class:`~http.cookies.Morsel` 中的cookie对象一样.

    * ``max_age`` 应为秒数, 如果cookie的持续时间与客户端浏览器会话的持续时间相同,
      则为 ``None`` (默认值). 如果未指定 ``expires``, 则会通过计算得到.
    * ``expires`` should either be a string in the format
      ``"Wdy, DD-Mon-YY HH:MM:SS GMT"`` or a ``datetime.datetime`` object
      in UTC. If ``expires`` is a ``datetime`` object, the ``max_age``
      will be calculated.
    * Use ``domain`` if you want to set a cross-domain cookie. For example,
      ``domain=".lawrence.com"`` will set a cookie that is readable by
      the domains www.lawrence.com, blogs.lawrence.com and
      calendars.lawrence.com. Otherwise, a cookie will only be readable by
      the domain that set it.
    * Use ``httponly=True`` if you want to prevent client-side
      JavaScript from having access to the cookie.

      HTTPOnly_ is a flag included in a Set-Cookie HTTP response
      header. It is not part of the :rfc:`2109` standard for cookies,
      and it isn't honored consistently by all browsers. However,
      when it is honored, it can be a useful way to mitigate the
      risk of a client-side script from accessing the protected cookie
      data.

    .. _HTTPOnly: https://www.owasp.org/index.php/HTTPOnly

    .. warning::

        Both :rfc:`2109` and :rfc:`6265` state that user agents should support
        cookies of at least 4096 bytes. For many browsers this is also the
        maximum size. Django will not raise an exception if there's an attempt
        to store a cookie of more than 4096 bytes, but many browsers will not
        set the cookie correctly.

.. method:: HttpResponse.set_signed_cookie(key, value, salt='', max_age=None, expires=None, path='/', domain=None, secure=None, httponly=True)

    Like :meth:`~HttpResponse.set_cookie()`, but
    :doc:`cryptographic signing </topics/signing>` the cookie before setting
    it. Use in conjunction with :meth:`HttpRequest.get_signed_cookie`.
    You can use the optional ``salt`` argument for added key strength, but
    you will need to remember to pass it to the corresponding
    :meth:`HttpRequest.get_signed_cookie` call.

.. method:: HttpResponse.delete_cookie(key, path='/', domain=None)

    Deletes the cookie with the given key. Fails silently if the key doesn't
    exist.

    Due to the way cookies work, ``path`` and ``domain`` should be the same
    values you used in ``set_cookie()`` -- otherwise the cookie may not be
    deleted.

.. method:: HttpResponse.write(content)

    This method makes an :class:`HttpResponse` instance a file-like object.

.. method:: HttpResponse.flush()

    This method makes an :class:`HttpResponse` instance a file-like object.

.. method:: HttpResponse.tell()

    This method makes an :class:`HttpResponse` instance a file-like object.

.. method:: HttpResponse.getvalue()

    Returns the value of :attr:`HttpResponse.content`. This method makes
    an :class:`HttpResponse` instance a stream-like object.

.. method:: HttpResponse.readable()

   .. versionadded:: 1.10

    Always ``False``. This method makes an :class:`HttpResponse` instance a
    stream-like object.

.. method:: HttpResponse.seekable()

   .. versionadded:: 1.10

    Always ``False``. This method makes an :class:`HttpResponse` instance a
    stream-like object.

.. method:: HttpResponse.writable()

    Always ``True``. This method makes an :class:`HttpResponse` instance a
    stream-like object.

.. method:: HttpResponse.writelines(lines)

    Writes a list of lines to the response. Line separators are not added. This
    method makes an :class:`HttpResponse` instance a stream-like object.

.. _ref-httpresponse-subclasses:

``HttpResponse`` subclasses
---------------------------

Django includes a number of ``HttpResponse`` subclasses that handle different
types of HTTP responses. Like ``HttpResponse``, these subclasses live in
:mod:`django.http`.

.. class:: HttpResponseRedirect

    The first argument to the constructor is required -- the path to redirect
    to. This can be a fully qualified URL
    (e.g. ``'https://www.yahoo.com/search/'``), an absolute path with no domain
    (e.g. ``'/search/'``), or even a relative path (e.g. ``'search/'``). In that
    last case, the client browser will reconstruct the full URL itself
    according to the current path. See :class:`HttpResponse` for other optional
    constructor arguments. Note that this returns an HTTP status code 302.

    .. attribute:: HttpResponseRedirect.url

        This read-only attribute represents the URL the response will redirect
        to (equivalent to the ``Location`` response header).

.. class:: HttpResponsePermanentRedirect

    Like :class:`HttpResponseRedirect`, but it returns a permanent redirect
    (HTTP status code 301) instead of a "found" redirect (status code 302).

.. class:: HttpResponseNotModified

    The constructor doesn't take any arguments and no content should be added
    to this response. Use this to designate that a page hasn't been modified
    since the user's last request (status code 304).

.. class:: HttpResponseBadRequest

    Acts just like :class:`HttpResponse` but uses a 400 status code.

.. class:: HttpResponseNotFound

    Acts just like :class:`HttpResponse` but uses a 404 status code.

.. class:: HttpResponseForbidden

    Acts just like :class:`HttpResponse` but uses a 403 status code.

.. class:: HttpResponseNotAllowed

    Like :class:`HttpResponse`, but uses a 405 status code. The first argument
    to the constructor is required: a list of permitted methods (e.g.
    ``['GET', 'POST']``).

.. class:: HttpResponseGone

    Acts just like :class:`HttpResponse` but uses a 410 status code.

.. class:: HttpResponseServerError

    Acts just like :class:`HttpResponse` but uses a 500 status code.

.. note::

    If a custom subclass of :class:`HttpResponse` implements a ``render``
    method, Django will treat it as emulating a
    :class:`~django.template.response.SimpleTemplateResponse`, and the
    ``render`` method must itself return a valid response object.

``JsonResponse`` objects
========================

.. class:: JsonResponse(data, encoder=DjangoJSONEncoder, safe=True, json_dumps_params=None, **kwargs)

    An :class:`HttpResponse` subclass that helps to create a JSON-encoded
    response. It inherits most behavior from its superclass with a couple
    differences:

    Its default ``Content-Type`` header is set to ``application/json``.

    The first parameter, ``data``, should be a ``dict`` instance. If the
    ``safe`` parameter is set to ``False`` (see below) it can be any
    JSON-serializable object.

    The ``encoder``, which defaults to
    :class:`django.core.serializers.json.DjangoJSONEncoder`, will be used to
    serialize the data. See :ref:`JSON serialization
    <serialization-formats-json>` for more details about this serializer.

    The ``safe`` boolean parameter defaults to ``True``. If it's set to
    ``False``, any object can be passed for serialization (otherwise only
    ``dict`` instances are allowed). If ``safe`` is ``True`` and a non-``dict``
    object is passed as the first argument, a :exc:`TypeError` will be raised.

    The ``json_dumps_params`` parameter is a dictionary of keyword arguments
    to pass to the ``json.dumps()`` call used to generate the response.

    .. versionchanged:: 1.9

        The ``json_dumps_params`` argument was added.

Usage
-----

Typical usage could look like::

    >>> from django.http import JsonResponse
    >>> response = JsonResponse({'foo': 'bar'})
    >>> response.content
    b'{"foo": "bar"}'


Serializing non-dictionary objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In order to serialize objects other than ``dict`` you must set the ``safe``
parameter to ``False``::

    >>> response = JsonResponse([1, 2, 3], safe=False)

Without passing ``safe=False``, a :exc:`TypeError` will be raised.

.. warning::

    Before the `5th edition of ECMAScript
    <http://www.ecma-international.org/ecma-262/5.1/index.html#sec-11.1.4>`_
    it was possible to poison the JavaScript ``Array`` constructor. For this
    reason, Django does not allow passing non-dict objects to the
    :class:`~django.http.JsonResponse` constructor by default.  However, most
    modern browsers implement EcmaScript 5 which removes this attack vector.
    Therefore it is possible to disable this security precaution.

Changing the default JSON encoder
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you need to use a different JSON encoder class you can pass the ``encoder``
parameter to the constructor method::

    >>> response = JsonResponse(data, encoder=MyJSONEncoder)

.. _httpresponse-streaming:

``StreamingHttpResponse`` objects
=================================

.. class:: StreamingHttpResponse

The :class:`StreamingHttpResponse` class is used to stream a response from
Django to the browser. You might want to do this if generating the response
takes too long or uses too much memory. For instance, it's useful for
:ref:`generating large CSV files <streaming-csv-files>`.

.. admonition:: Performance considerations

    Django is designed for short-lived requests. Streaming responses will tie
    a worker process for the entire duration of the response. This may result
    in poor performance.

    Generally speaking, you should perform expensive tasks outside of the
    request-response cycle, rather than resorting to a streamed response.

The :class:`StreamingHttpResponse` is not a subclass of :class:`HttpResponse`,
because it features a slightly different API. However, it is almost identical,
with the following notable differences:

* It should be given an iterator that yields strings as content.

* You cannot access its content, except by iterating the response object
  itself. This should only occur when the response is returned to the client.

* It has no ``content`` attribute. Instead, it has a
  :attr:`~StreamingHttpResponse.streaming_content` attribute.

* You cannot use the file-like object ``tell()`` or ``write()`` methods.
  Doing so will raise an exception.

:class:`StreamingHttpResponse` should only be used in situations where it is
absolutely required that the whole content isn't iterated before transferring
the data to the client. Because the content can't be accessed, many
middlewares can't function normally. For example the ``ETag`` and
``Content-Length`` headers can't be generated for streaming responses.

Attributes
----------

.. attribute:: StreamingHttpResponse.streaming_content

    An iterator of strings representing the content.

.. attribute:: StreamingHttpResponse.status_code

    The :rfc:`HTTP status code <7231#section-6>` for the response.

    .. versionchanged:: 1.9

        Unless :attr:`reason_phrase` is explicitly set, modifying the value of
        ``status_code`` outside the constructor will also modify the value of
        ``reason_phrase``.

.. attribute:: StreamingHttpResponse.reason_phrase

    The HTTP reason phrase for the response.

    .. versionchanged:: 1.9

        ``reason_phrase`` no longer defaults to all capital letters. It now
        uses the :rfc:`HTTP standard's <7231#section-6.1>` default reason
        phrases.

        Unless explicitly set, ``reason_phrase`` is determined by the current
        value of :attr:`status_code`.

.. attribute:: StreamingHttpResponse.streaming

    This is always ``True``.

``FileResponse`` objects
========================

.. class:: FileResponse

:class:`FileResponse` is a subclass of :class:`StreamingHttpResponse` optimized
for binary files. It uses `wsgi.file_wrapper`_ if provided by the wsgi server,
otherwise it streams the file out in small chunks.

.. _wsgi.file_wrapper: https://www.python.org/dev/peps/pep-3333/#optional-platform-specific-file-handling

``FileResponse`` expects a file open in binary mode like so::

    >>> from django.http import FileResponse
    >>> response = FileResponse(open('myfile.png', 'rb'))
