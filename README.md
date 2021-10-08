>如果感兴趣， 欢迎加入进来一起翻译。Folk 项目:https://github.com/jhao104/django-chinese-doc 提交你的内容。翻译后的文档会自动更新到:http://django-chinese-doc.readthedocs.io/zh_CN/latest/

# Django中文文档
#### 当前版本 Django1.10
[![Documentation Status](https://readthedocs.org/projects/django-chinese-doc/badge/?version=latest)](http://django-chinese-doc.readthedocs.io/zh_CN/latest/?badge=latest)
[![](https://img.shields.io/badge/Powered%20by-@j_hao104-blue.svg)](http://www.spiderpy.cn/blog/)

　　你想知道的关于Django的一切。

## 文档组织方式

　　Django有很多的文档。一个好的组织结构能够帮助您在何处寻找你想要的内容：

* [入门教程](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/index.html) 一步步详解如何从零开始创建一个Web应用程序。如果您第一次使用Django或Web应用程序开发，那么请从这里开始。也就是下面的“[入门](#入门)”部分。

* [主题指南](http://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/index.html) 在比较高的层次上讨论某些话题和概念，并提供有用的背景信息和解释。

* [参考指南](http://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/index.html) 包含API和Django的机械等方面的技术参考。抓哟描述了它的工作原理以及如何使用它，但是这要求您对关键概念有基本的的了解。

* [方法指南](http://django-chinese-doc.readthedocs.io/zh_CN/latest/howto/index.html) 就像一种秘诀。他们将帮助您完成解决关键问题和用例的步骤。他们比教程更先进，并且还会帮你你了解Django的工作原理。

## <span id = "first_steps">入门：</span>

　　如果你是刚接触Django或编程，这是一个起点。

* **从零开始**: [概览](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/overview.html) | [安装](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/install.html)

* **入门教程**: [Part1:请求与响应](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial01.html) | [Part2:模型和管理站点](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial02.html) | [Part3:视图和模板](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial03.html) | [Part4:表单和通用视图](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial04.html) | [Part5:测试](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial05.html) | [Part6:静态文件](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial06.html) | [Part7:自定义管理站点](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial07.html)

* **进阶教程**: [开发可重用的应用](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/reusable-apps.html) | [开发Django补丁](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/contributing.html)

## 模型层

   Django提供一个抽象层（Models）以构建和操作你的web应用中的数据，通过以下内容了解更多：

* **模型**：[模型简介](http://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/db/models.html) |  [字段类型](http://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/models/fields.html) | [元选项](http://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/models/options.html) | [模型类](http://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/models/class.html)

* **查询集**: [执行查询](http://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/db/queries.html) |  [QuerySet方法](http://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/models/querysets.html) |  [查询表达式](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/models/lookups.html)

* **模型实例**: [实例方法](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/models/instances.html) |  [访问关联对象](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/models/relations.html)
 
* **迁移**: [迁移简介 (0%)](https://docs.djangoproject.com/en/1.10/topics/migrations/) | [操作参考 (0%)](https://docs.djangoproject.com/en/1.10/ref/migration-operations/) | [SchemaEditor](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/schema-editor.html) | [编写迁移 (0%)](https://docs.djangoproject.com/en/1.10/howto/writing-migrations/)

* **高级**: [管理器](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/db/managers.html) | [原始SQL](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/db/sql.html) [事务 (0%)](https://docs.djangoproject.com/en/1.10/topics/db/transactions/) | [聚合](http://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/db/aggregation.html) | [搜索](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/db/search.html) | [自定义字段](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/howto/custom-model-fields.html) | [多数据库](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/db/multi-db.html) |[自定义查询 (0%)](https://docs.djangoproject.com/en/1.10/howto/custom-lookups/)| [查询表达式](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/models/expressions.html) | [条件表达式](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/models/conditional-expressions.html) |[数据库函数 (0%)](https://docs.djangoproject.com/en/1.10/ref/models/conditional-expressions/)

* **其他**: [支持的数据库 (0%)](https://docs.djangoproject.com/en/1.10/ref/databases/)|[遗留的数据库 (0%)](https://docs.djangoproject.com/en/1.10/howto/legacy-databases/)|[提供初始数据 (0%)](https://docs.djangoproject.com/en/1.10/howto/initial-data/)|[优化数据库访问 (0%)](https://docs.djangoproject.com/en/1.10/topics/db/optimization/)|[PostgreSQL specific features (0%)](https://docs.djangoproject.com/en/1.10/ref/contrib/postgres/)

## 视图层

   Django使用"视图"这个概念, 负责处理用户请求并返回响应. 通过以下链接查找所有您需要知道的有关视图的信息：

* **The basics**: [URLconfs](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/http/urls.html) | [View functions](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/http/views.html) | [Shortcuts](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/http/shortcuts.html) | [Decorators](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/http/decorators.html)

* **参考**: [Built-in](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/views.html) | [请求/响应对象](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/request-response.html) | [TemplateResponse objects](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/template-response.html)

* **File uploads**: [Overview](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/http/file-uploads.html) | [File objects](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/files/file.html) | [Storage API](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/files/storage.html) | [Managing files](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/files.html) | [Custom storage](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/howto/custom-file-storage.html)

* **Class-based views**: [Overview](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/class-based-views/index.html) | [Built-in display views](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/class-based-views/generic-display.html) | [Built-in editing views](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/class-based-views/generic-editing.html) | [Using mixins](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/class-based-views/mixins.html) | [API reference](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/class-based-views/index.html) | [Flattened index](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/class-based-views/flattened-index.html)

* **Advanced**: [Generating CSV](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/howto/outputting-csv.html) | [Generating PDF](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/howto/outputting-pdf.html)

* **Middleware**: [Overview](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/http/middleware.html) | [Built-in middleware classes](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/middleware.html)

## The template layer

   The template layer provides a designer-friendly syntax for rendering the information to be presented to the user. Learn how this syntax can be used by designers and how it can be extended by programmers:

* **The basics**: [Overview](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/templates.html)

* **For designers**: [Language overview](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/templates/language.html) | [Built-in tags and filters](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/templates/builtins.html) | [Humanization](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/contrib/humanize.html)

* **For programmers**: [Template API](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/templates/api.html) | [Custom tags and filters](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/howto/custom-template-tags.html)


## Forms

   Django provides a rich framework to facilitate the creation of forms and the manipulation of form data.

* **The basics**: [Overview](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/forms/index.html) | [Form API](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/forms/api.html) | [Built-in fields](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/forms/fields.html) | [Built-in widgets](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/forms/widgets.html)

* **Advanced**: [Forms for models](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/forms/modelforms.html) | [Integrating media](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/forms/media.html) | [Formsets](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/forms/formsets.html) | [Customizing validation](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/forms/validation.html)

## The development process

   Learn about the various components and tools to help you in the development and testing of Django applications:

* **设置**: [Overview](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/topics/settings.html) | [配置列表](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/settings.html)

* **应用**: [应用](https://django-chinese-doc.readthedocs.io/zh_CN/1.10.x/ref/applications.html)

* **Exceptions**: [Overview](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/exceptions.html)

* **django-admin和manage.py**: [Overview](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/django-admin.html) | [Adding custom commands](https://django-chinese-doc.readthedocs.io/zh_CN/latest/howto/custom-management-commands.html)

* **Testing**: 
  [Introduction ](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/testing/index.html) | [Writing and running tests ](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/testing/overview.html) | [Included testing tools](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/testing/tools.html) | [Advanced topics](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/testing/advanced.html)

* **Deployment**:
  [Overview](https://django-chinese-doc.readthedocs.io/zh_CN/latest/howto/deployment/index.html) | [WSGI servers](https://django-chinese-doc.readthedocs.io/zh_CN/latest/howto/deployment/wsgi/index.html) | [Deploying static files ](https://django-chinese-doc.readthedocs.io/zh_CN/latest/howto/static-files/deployment.html) | [Tracking code errors by email](https://django-chinese-doc.readthedocs.io/zh_CN/latest/howto/error-reporting.html)

## The admin

   Find all you need to know about the automated admin interface, one of Django's most popular features:

* [Admin site](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/admin/index.html)
* [Admin actions](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/admin/actions.html)
* [Admin documentation generator](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/admin/admindocs.html)

## Security

   Security is a topic of paramount importance in the development of Web applications and Django provides multiple protection tools and mechanisms:

* [Security overview](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/security.html)
* [Disclosed security issues in Django](https://django-chinese-doc.readthedocs.io/zh_CN/latest/releases/security.html)
* [Clickjacking protection](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/clickjacking.html)
* [Cross Site Request Forgery protection](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/csrf.html)
* [Cryptographic signing](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/signing.html)
* [Security Middleware](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/middleware.html#security-middleware)

## Internationalization and localization

   Django offers a robust internationalization and localization framework to assist you in the development of applications for multiple languages and world regions:

* [Overview](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/i18n/index.html) |
  [Internationalization ](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/i18n/translation.html) |
  [Localization](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/i18n/translation.html#how-to-create-language-files) |
  [Localized Web UI formatting and form input](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/i18n/formatting.html)
* [Time zones](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/i18n/timezones.html)

## Performance and optimization

   There are a variety of techniques and tools that can help get your code running more efficiently - faster, and using fewer system resources.

* [Performance and optimization overview](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/performance.html)

## Python compatibility

   Django aims to be compatible with multiple different flavors and versions of Python:

* [Jython support](https://django-chinese-doc.readthedocs.io/zh_CN/latest/howto/jython.html)
* [Python 3 compatibility](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/python3.html)

## Geographic framework

   [GeoDjango](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/gis/index.html) intends to be a world-class geographic Web framework. Its goal is to make it as easy as possible to build GIS Web applications and harness the power of spatially enabled data.

## Common Web application tools

   Django offers multiple tools commonly needed in the development of Web applications:

* **Authentication**:
  [Overview](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/auth/index.html) | [Using the authentication system](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/auth/default.html) | [Password management](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/auth/passwords.html) | [Customizing authentication](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/auth/customizing.html) | [API Reference](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/auth.html)
* [Caching](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/cache.html)
* [Logging](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/logging.html)
* [Sending emails](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/email.html)
* [Syndication feeds (RSS/Atom)](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/syndication.html)
* [Pagination](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/pagination.html)
* [Messages framework](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/messages.html)
* [Serialization](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/serialization.html)
* [Sessions](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/http/sessions.html)
* [Sitemaps](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/sitemaps.html)
* [Static files management](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/staticfiles.html)
* [Data validation](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/validators.html)

## Other core functionalities

   Learn about some other core functionalities of the Django framework:

* [Conditional content processing](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/conditional-view-processing.html)
* [Content types and generic relations](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/contenttypes.html)
* [Flatpages](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/flatpages.html)
* [Redirects](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/redirects.html)
* [信号](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/signals.html)
* [System check framework](https://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/checks.html)
* [The sites framework](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/contrib/sites.html)
* [Unicode in Django](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/unicode.html)
