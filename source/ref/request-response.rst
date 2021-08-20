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

    包含所有可用HTTP部首的Python字典. 可用的部首取决了客户端和服务端, 下面是一些例子:

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

    Returns the originating port of the request using information from the
    ``HTTP_X_FORWARDED_PORT`` (if :setting:`USE_X_FORWARDED_PORT` is enabled)
    and ``SERVER_PORT`` ``META`` variables, in that order.

.. method:: HttpRequest.get_full_path()

    Returns the ``path``, plus an appended query string, if applicable.

    Example: ``"/music/bands/the_beatles/?print=true"``

.. method:: HttpRequest.build_absolute_uri(location)

    Returns the absolute URI form of ``location``. If no location is provided,
    the location will be set to ``request.get_full_path()``.

    If the location is already an absolute URI, it will not be altered.
    Otherwise the absolute URI is built using the server variables available in
    this request.

    Example: ``"https://example.com/music/bands/the_beatles/?print=true"``

    .. note::

        Mixing HTTP and HTTPS on the same site is discouraged, therefore
        :meth:`~HttpRequest.build_absolute_uri()` will always generate an
        absolute URI with the same scheme the current request has. If you need
        to redirect users to HTTPS, it's best to let your Web server redirect
        all HTTP traffic to HTTPS.

.. method:: HttpRequest.get_signed_cookie(key, default=RAISE_ERROR, salt='', max_age=None)

    Returns a cookie value for a signed cookie, or raises a
    ``django.core.signing.BadSignature`` exception if the signature is
    no longer valid. If you provide the ``default`` argument the exception
    will be suppressed and that default value will be returned instead.

    The optional ``salt`` argument can be used to provide extra protection
    against brute force attacks on your secret key. If supplied, the
    ``max_age`` argument will be checked against the signed timestamp
    attached to the cookie value to ensure the cookie is not older than
    ``max_age`` seconds.

    For example::

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

    See :doc:`cryptographic signing </topics/signing>` for more information.

.. method:: HttpRequest.is_secure()

    Returns ``True`` if the request is secure; that is, if it was made with
    HTTPS.

.. method:: HttpRequest.is_ajax()

    Returns ``True`` if the request was made via an ``XMLHttpRequest``, by
    checking the ``HTTP_X_REQUESTED_WITH`` header for the string
    ``'XMLHttpRequest'``. Most modern JavaScript libraries send this header.
    If you write your own XMLHttpRequest call (on the browser side), you'll
    have to set this header manually if you want ``is_ajax()`` to work.

    If a response varies on whether or not it's requested via AJAX and you are
    using some form of caching like Django's :mod:`cache middleware
    <django.middleware.cache>`, you should decorate the view with
    :func:`vary_on_headers('X-Requested-With')
    <django.views.decorators.vary.vary_on_headers>` so that the responses are
    properly cached.

.. method:: HttpRequest.read(size=None)
.. method:: HttpRequest.readline()
.. method:: HttpRequest.readlines()
.. method:: HttpRequest.xreadlines()
.. method:: HttpRequest.__iter__()

    Methods implementing a file-like interface for reading from an
    HttpRequest instance. This makes it possible to consume an incoming
    request in a streaming fashion. A common use-case would be to process a
    big XML payload with an iterative parser without constructing a whole
    XML tree in memory.

    Given this standard interface, an HttpRequest instance can be
    passed directly to an XML parser such as ElementTree::

        import xml.etree.ElementTree as ET
        for element in ET.iterparse(request):
            process(element)


``QueryDict`` objects
=====================

.. class:: QueryDict

In an :class:`HttpRequest` object, the ``GET`` and ``POST`` attributes are
instances of ``django.http.QueryDict``, a dictionary-like class customized to
deal with multiple values for the same key. This is necessary because some HTML
form elements, notably ``<select multiple>``, pass multiple values for the same
key.

The ``QueryDict``\ s at ``request.POST`` and ``request.GET`` will be immutable
when accessed in a normal request/response cycle. To get a mutable version you
need to use ``.copy()``.

Methods
-------

:class:`QueryDict` implements all the standard dictionary methods because it's
a subclass of dictionary. Exceptions are outlined here:

.. method:: QueryDict.__init__(query_string=None, mutable=False, encoding=None)

    Instantiates a ``QueryDict`` object based on ``query_string``.

    >>> QueryDict('a=1&a=2&c=3')
    <QueryDict: {'a': ['1', '2'], 'c': ['3']}>

    If ``query_string`` is not passed in, the resulting ``QueryDict`` will be
    empty (it will have no keys or values).

    Most ``QueryDict``\ s you encounter, and in particular those at
    ``request.POST`` and ``request.GET``, will be immutable. If you are
    instantiating one yourself, you can make it mutable by passing
    ``mutable=True`` to its ``__init__()``.

    Strings for setting both keys and values will be converted from ``encoding``
    to unicode. If encoding is not set, it defaults to :setting:`DEFAULT_CHARSET`.

.. method:: QueryDict.__getitem__(key)

    Returns the value for the given key. If the key has more than one value,
    ``__getitem__()`` returns the last value. Raises
    ``django.utils.datastructures.MultiValueDictKeyError`` if the key does not
    exist. (This is a subclass of Python's standard ``KeyError``, so you can
    stick to catching ``KeyError``.)

.. method:: QueryDict.__setitem__(key, value)

    Sets the given key to ``[value]`` (a Python list whose single element is
    ``value``). Note that this, as other dictionary functions that have side
    effects, can only be called on a mutable ``QueryDict`` (such as one that
    was created via ``copy()``).

.. method:: QueryDict.__contains__(key)

    Returns ``True`` if the given key is set. This lets you do, e.g., ``if "foo"
    in request.GET``.

.. method:: QueryDict.get(key, default=None)

    Uses the same logic as ``__getitem__()`` above, with a hook for returning a
    default value if the key doesn't exist.

.. method:: QueryDict.setdefault(key, default=None)

    Just like the standard dictionary ``setdefault()`` method, except it uses
    ``__setitem__()`` internally.

.. method:: QueryDict.update(other_dict)

    Takes either a ``QueryDict`` or standard dictionary. Just like the standard
    dictionary ``update()`` method, except it *appends* to the current
    dictionary items rather than replacing them. For example::

        >>> q = QueryDict('a=1', mutable=True)
        >>> q.update({'a': '2'})
        >>> q.getlist('a')
        ['1', '2']
        >>> q['a'] # returns the last
        '2'

.. method:: QueryDict.items()

    Just like the standard dictionary ``items()`` method, except this uses the
    same last-value logic as ``__getitem__()``. For example::

        >>> q = QueryDict('a=1&a=2&a=3')
        >>> q.items()
        [('a', '3')]

.. method:: QueryDict.iteritems()

    Just like the standard dictionary ``iteritems()`` method. Like
    :meth:`QueryDict.items()` this uses the same last-value logic as
    :meth:`QueryDict.__getitem__()`.

    Available only on Python 2.

.. method:: QueryDict.iterlists()

    Like :meth:`QueryDict.iteritems()` except it includes all values, as a list,
    for each member of the dictionary.

    Available only on Python 2.

.. method:: QueryDict.values()

    Just like the standard dictionary ``values()`` method, except this uses the
    same last-value logic as ``__getitem__()``. For example::

        >>> q = QueryDict('a=1&a=2&a=3')
        >>> q.values()
        ['3']

.. method:: QueryDict.itervalues()

    Just like :meth:`QueryDict.values()`, except an iterator.

    Available only on Python 2.

In addition, ``QueryDict`` has the following methods:

.. method:: QueryDict.copy()

    Returns a copy of the object, using ``copy.deepcopy()`` from the Python
    standard library. This copy will be mutable even if the original was not.

.. method:: QueryDict.getlist(key, default=None)

    Returns the data with the requested key, as a Python list. Returns an
    empty list if the key doesn't exist and no default value was provided.
    It's guaranteed to return a list of some sort unless the default value
    provided is not a list.

.. method:: QueryDict.setlist(key, list_)

    Sets the given key to ``list_`` (unlike ``__setitem__()``).

.. method:: QueryDict.appendlist(key, item)

    Appends an item to the internal list associated with key.

.. method:: QueryDict.setlistdefault(key, default_list=None)

    Just like ``setdefault``, except it takes a list of values instead of a
    single value.

.. method:: QueryDict.lists()

    Like :meth:`items()`, except it includes all values, as a list, for each
    member of the dictionary. For example::

        >>> q = QueryDict('a=1&a=2&a=3')
        >>> q.lists()
        [('a', ['1', '2', '3'])]

.. method:: QueryDict.pop(key)

    Returns a list of values for the given key and removes them from the
    dictionary. Raises ``KeyError`` if the key does not exist. For example::

        >>> q = QueryDict('a=1&a=2&a=3', mutable=True)
        >>> q.pop('a')
        ['1', '2', '3']

.. method:: QueryDict.popitem()

    Removes an arbitrary member of the dictionary (since there's no concept
    of ordering), and returns a two value tuple containing the key and a list
    of all values for the key. Raises ``KeyError`` when called on an empty
    dictionary. For example::

        >>> q = QueryDict('a=1&a=2&a=3', mutable=True)
        >>> q.popitem()
        ('a', ['1', '2', '3'])

.. method:: QueryDict.dict()

    Returns ``dict`` representation of ``QueryDict``. For every (key, list)
    pair in ``QueryDict``, ``dict`` will have (key, item), where item is one
    element of the list, using same logic as :meth:`QueryDict.__getitem__()`::

        >>> q = QueryDict('a=1&a=3&a=5')
        >>> q.dict()
        {'a': '5'}

.. method:: QueryDict.urlencode(safe=None)

    Returns a string of the data in query-string format. Example::

        >>> q = QueryDict('a=2&b=3&b=5')
        >>> q.urlencode()
        'a=2&b=3&b=5'

    Optionally, urlencode can be passed characters which
    do not require encoding. For example::

        >>> q = QueryDict(mutable=True)
        >>> q['next'] = '/a&b/'
        >>> q.urlencode(safe='/')
        'next=/a%26b/'

``HttpResponse`` objects
========================

.. class:: HttpResponse

In contrast to :class:`HttpRequest` objects, which are created automatically by
Django, :class:`HttpResponse` objects are your responsibility. Each view you
write is responsible for instantiating, populating and returning an
:class:`HttpResponse`.

The :class:`HttpResponse` class lives in the :mod:`django.http` module.

Usage
-----

Passing strings
~~~~~~~~~~~~~~~

Typical usage is to pass the contents of the page, as a string, to the
:class:`HttpResponse` constructor::

    >>> from django.http import HttpResponse
    >>> response = HttpResponse("Here's the text of the Web page.")
    >>> response = HttpResponse("Text only, please.", content_type="text/plain")

But if you want to add content incrementally, you can use ``response`` as a
file-like object::

    >>> response = HttpResponse()
    >>> response.write("<p>Here's the text of the Web page.</p>")
    >>> response.write("<p>Here's another paragraph.</p>")

Passing iterators
~~~~~~~~~~~~~~~~~

Finally, you can pass ``HttpResponse`` an iterator rather than strings.
``HttpResponse`` will consume the iterator immediately, store its content as a
string, and discard it. Objects with a ``close()`` method such as files and
generators are immediately closed.

If you need the response to be streamed from the iterator to the client, you
must use the :class:`StreamingHttpResponse` class instead.

.. versionchanged:: 1.10

    Objects with a ``close()`` method used to be closed when the WSGI server
    called ``close()`` on the response.

Setting header fields
~~~~~~~~~~~~~~~~~~~~~

To set or remove a header field in your response, treat it like a dictionary::

    >>> response = HttpResponse()
    >>> response['Age'] = 120
    >>> del response['Age']

Note that unlike a dictionary, ``del`` doesn't raise ``KeyError`` if the header
field doesn't exist.

For setting the ``Cache-Control`` and ``Vary`` header fields, it is recommended
to use the :func:`~django.utils.cache.patch_cache_control` and
:func:`~django.utils.cache.patch_vary_headers` methods from
:mod:`django.utils.cache`, since these fields can have multiple, comma-separated
values. The "patch" methods ensure that other values, e.g. added by a
middleware, are not removed.

HTTP header fields cannot contain newlines. An attempt to set a header field
containing a newline character (CR or LF) will raise ``BadHeaderError``

Telling the browser to treat the response as a file attachment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To tell the browser to treat the response as a file attachment, use the
``content_type`` argument and set the ``Content-Disposition`` header. For example,
this is how you might return a Microsoft Excel spreadsheet::

    >>> response = HttpResponse(my_data, content_type='application/vnd.ms-excel')
    >>> response['Content-Disposition'] = 'attachment; filename="foo.xls"'

There's nothing Django-specific about the ``Content-Disposition`` header, but
it's easy to forget the syntax, so we've included it here.

Attributes
----------

.. attribute:: HttpResponse.content

    A bytestring representing the content, encoded from a Unicode
    object if necessary.

.. attribute:: HttpResponse.charset

    A string denoting the charset in which the response will be encoded. If not
    given at ``HttpResponse`` instantiation time, it will be extracted from
    ``content_type`` and if that is unsuccessful, the
    :setting:`DEFAULT_CHARSET` setting will be used.

.. attribute:: HttpResponse.status_code

    The :rfc:`HTTP status code <7231#section-6>` for the response.

    .. versionchanged:: 1.9

        Unless :attr:`reason_phrase` is explicitly set, modifying the value of
        ``status_code`` outside the constructor will also modify the value of
        ``reason_phrase``.

.. attribute:: HttpResponse.reason_phrase

    The HTTP reason phrase for the response.

    .. versionchanged:: 1.9

        ``reason_phrase`` no longer defaults to all capital letters. It now
        uses the :rfc:`HTTP standard's <7231#section-6.1>` default reason
        phrases.

        Unless explicitly set, ``reason_phrase`` is determined by the current
        value of :attr:`status_code`.

.. attribute:: HttpResponse.streaming

    This is always ``False``.

    This attribute exists so middleware can treat streaming responses
    differently from regular responses.

.. attribute:: HttpResponse.closed

    ``True`` if the response has been closed.

Methods
-------

.. method:: HttpResponse.__init__(content='', content_type=None, status=200, reason=None, charset=None)

    Instantiates an ``HttpResponse`` object with the given page content and
    content type.

    ``content`` should be an iterator or a string. If it's an
    iterator, it should return strings, and those strings will be
    joined together to form the content of the response. If it is not
    an iterator or a string, it will be converted to a string when
    accessed.

    ``content_type`` is the MIME type optionally completed by a character set
    encoding and is used to fill the HTTP ``Content-Type`` header. If not
    specified, it is formed by the :setting:`DEFAULT_CONTENT_TYPE` and
    :setting:`DEFAULT_CHARSET` settings, by default: "`text/html; charset=utf-8`".

    ``status`` is the :rfc:`HTTP status code <7231#section-6>` for the response.

    ``reason`` is the HTTP response phrase. If not provided, a default phrase
    will be used.

    ``charset`` is the charset in which the response will be encoded. If not
    given it will be extracted from ``content_type``, and if that
    is unsuccessful, the :setting:`DEFAULT_CHARSET` setting will be used.

.. method:: HttpResponse.__setitem__(header, value)

    Sets the given header name to the given value. Both ``header`` and
    ``value`` should be strings.

.. method:: HttpResponse.__delitem__(header)

    Deletes the header with the given name. Fails silently if the header
    doesn't exist. Case-insensitive.

.. method:: HttpResponse.__getitem__(header)

    Returns the value for the given header name. Case-insensitive.

.. method:: HttpResponse.has_header(header)

    Returns ``True`` or ``False`` based on a case-insensitive check for a
    header with the given name.

.. method:: HttpResponse.setdefault(header, value)

    Sets a header unless it has already been set.

.. method:: HttpResponse.set_cookie(key, value='', max_age=None, expires=None, path='/', domain=None, secure=None, httponly=False)

    Sets a cookie. The parameters are the same as in the
    :class:`~http.cookies.Morsel` cookie object in the Python standard library.

    * ``max_age`` should be a number of seconds, or ``None`` (default) if
      the cookie should last only as long as the client's browser session.
      If ``expires`` is not specified, it will be calculated.
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
