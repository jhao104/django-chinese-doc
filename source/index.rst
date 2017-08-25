====================
Django1.10 中文文档
====================

.. rubric:: Django的百科全书.

.. note::

    本项目正在进行中，未翻译完成的部分仍然是英文显示，如果你有兴趣可以在github上加入我们。

获取帮助
============

如果您有问题，我们很乐意帮助您解决!

* Try the :doc:`FAQ <faq/index>` -- it's got answers to many common questions.

* Looking for specific information? Try the :ref:`genindex`, :ref:`modindex` or
  the :doc:`detailed table of contents <contents>`.

* Search for information in the archives of the |django-users| mailing list, or
  `post a question`_.

* Ask a question in the `#django IRC channel`_, or search the `IRC logs`_ to see
  if it's been asked before.

* Report bugs with Django in our `ticket tracker`_.

.. _archives: https://groups.google.com/group/django-users/
.. _post a question: https://groups.google.com/d/forum/django-users
.. _#django IRC channel: irc://irc.freenode.net/django
.. _IRC logs: http://django-irc-logs.com/
.. _ticket tracker: https://code.djangoproject.com/

文档结构
========

这有很多关于Django的文档。一个好的组织结构能够帮助您在何处寻找你想要的内容：

* :doc:`入门教程 </intro/index>` 一步步详解如何从零开始创建一个Web应用程序。如果您第一次使用Django或Web应用程序开发，那么请从这里开始。也就是下面的“:ref:`index-first-steps`”部分。

* :doc:`主题指南 </topics/index>` 在比较高的层次上讨论某些话题和概念，并提供有用的背景信息和解释。

* :doc:`参考指南 </ref/index>` 包含API和Django等方面的技术参考。专注描述了它的工作原理以及如何使用它，
  但是这要求您对关键概念有基本的的了解。

* :doc:`使用指南 </howto/index>` 就像一种秘诀。他们将帮助您完成解决关键问题和用例的步骤。他们比教程更先进，并且还会帮你了解Django的工作原理。

.. _index-first-steps:

入门
===========

Are you new to Django or to programming? This is the place to start!

* **从零开始:**
  :doc:`概览 <intro/overview>` |
  :doc:`安装 <intro/install>`

* **入门教程:**
  :doc:`Part 1: 请求和响应 <intro/tutorial01>` |
  :doc:`Part 2: 模型和管理站点 <intro/tutorial02>` |
  :doc:`Part 3: 视图和模板 <intro/tutorial03>` |
  :doc:`Part 4: 表单和通用视图 <intro/tutorial04>` |
  :doc:`Part 5: 测试 <intro/tutorial05>` |
  :doc:`Part 6: 静态文件 <intro/tutorial06>` |
  :doc:`Part 7: 自定义管理站点 <intro/tutorial07>`

* **进阶教程:**
  :doc:`如何开发可重用的应用 <intro/reusable-apps>` |
  :doc:`开发Django补丁 <intro/contributing>`

模型层
=======

Django提供一个抽象层（Models）以构建和操作你的web应用中的数据，通过以下内容了解更多：

* **模型:**
  :doc:`模型简介 <topics/db/models>` |
  :doc:`Field types <ref/models/fields>` |
  :doc:`Meta options <ref/models/options>` |
  :doc:`Model class <ref/models/class>`

* **QuerySets:**
  :doc:`Executing queries <topics/db/queries>` |
  :doc:`QuerySet method reference <ref/models/querysets>` |
  :doc:`Lookup expressions <ref/models/lookups>`

* **Model instances:**
  :doc:`Instance methods <ref/models/instances>` |
  :doc:`Accessing related objects <ref/models/relations>`

* **Migrations:**
  :doc:`Introduction to Migrations<topics/migrations>` |
  :doc:`Operations reference <ref/migration-operations>` |
  :doc:`SchemaEditor <ref/schema-editor>` |
  :doc:`Writing migrations <howto/writing-migrations>`

* **Advanced:**
  :doc:`Managers <topics/db/managers>` |
  :doc:`Raw SQL <topics/db/sql>` |
  :doc:`Transactions <topics/db/transactions>` |
  :doc:`Aggregation <topics/db/aggregation>` |
  :doc:`Search <topics/db/search>` |
  :doc:`Custom fields <howto/custom-model-fields>` |
  :doc:`Multiple databases <topics/db/multi-db>` |
  :doc:`Custom lookups <howto/custom-lookups>` |
  :doc:`Query Expressions <ref/models/expressions>` |
  :doc:`Conditional Expressions <ref/models/conditional-expressions>` |
  :doc:`Database Functions <ref/models/database-functions>`

* **Other:**
  :doc:`Supported databases <ref/databases>` |
  :doc:`Legacy databases <howto/legacy-databases>` |
  :doc:`Providing initial data <howto/initial-data>` |
  :doc:`Optimize database access <topics/db/optimization>` |
  :doc:`PostgreSQL specific features <ref/contrib/postgres/index>`

The view layer
==============

Django has the concept of "views" to encapsulate the logic responsible for
processing a user's request and for returning the response. Find all you need
to know about views via the links below:

* **The basics:**
  :doc:`URLconfs <topics/http/urls>` |
  :doc:`View functions <topics/http/views>` |
  :doc:`Shortcuts <topics/http/shortcuts>` |
  :doc:`Decorators <topics/http/decorators>`

* **Reference:**
  :doc:`Built-in Views <ref/views>` |
  :doc:`Request/response objects <ref/request-response>` |
  :doc:`TemplateResponse objects <ref/template-response>`

* **File uploads:**
  :doc:`Overview <topics/http/file-uploads>` |
  :doc:`File objects <ref/files/file>` |
  :doc:`Storage API <ref/files/storage>` |
  :doc:`Managing files <topics/files>` |
  :doc:`Custom storage <howto/custom-file-storage>`

* **Class-based views:**
  :doc:`Overview <topics/class-based-views/index>` |
  :doc:`Built-in display views <topics/class-based-views/generic-display>` |
  :doc:`Built-in editing views <topics/class-based-views/generic-editing>` |
  :doc:`Using mixins <topics/class-based-views/mixins>` |
  :doc:`API reference <ref/class-based-views/index>` |
  :doc:`Flattened index<ref/class-based-views/flattened-index>`

* **Advanced:**
  :doc:`Generating CSV <howto/outputting-csv>` |
  :doc:`Generating PDF <howto/outputting-pdf>`

* **Middleware:**
  :doc:`Overview <topics/http/middleware>` |
  :doc:`Built-in middleware classes <ref/middleware>`

The template layer
==================

The template layer provides a designer-friendly syntax for rendering the
information to be presented to the user. Learn how this syntax can be used by
designers and how it can be extended by programmers:

* **The basics:**
  :doc:`Overview <topics/templates>`

* **For designers:**
  :doc:`Language overview <ref/templates/language>` |
  :doc:`Built-in tags and filters <ref/templates/builtins>` |
  :doc:`Humanization <ref/contrib/humanize>`

* **For programmers:**
  :doc:`Template API <ref/templates/api>` |
  :doc:`Custom tags and filters <howto/custom-template-tags>`

Forms
=====

Django provides a rich framework to facilitate the creation of forms and the
manipulation of form data.

* **The basics:**
  :doc:`Overview <topics/forms/index>` |
  :doc:`Form API <ref/forms/api>` |
  :doc:`Built-in fields <ref/forms/fields>` |
  :doc:`Built-in widgets <ref/forms/widgets>`

* **Advanced:**
  :doc:`Forms for models <topics/forms/modelforms>` |
  :doc:`Integrating media <topics/forms/media>` |
  :doc:`Formsets <topics/forms/formsets>` |
  :doc:`Customizing validation <ref/forms/validation>`

The development process
=======================

Learn about the various components and tools to help you in the development and
testing of Django applications:

* **Settings:**
  :doc:`Overview <topics/settings>` |
  :doc:`Full list of settings <ref/settings>`

* **Applications:**
  :doc:`Overview <ref/applications>`

* **Exceptions:**
  :doc:`Overview <ref/exceptions>`

* **django-admin and manage.py:**
  :doc:`Overview <ref/django-admin>` |
  :doc:`Adding custom commands <howto/custom-management-commands>`

* **Testing:**
  :doc:`Introduction <topics/testing/index>` |
  :doc:`Writing and running tests <topics/testing/overview>` |
  :doc:`Included testing tools <topics/testing/tools>` |
  :doc:`Advanced topics <topics/testing/advanced>`

* **Deployment:**
  :doc:`Overview <howto/deployment/index>` |
  :doc:`WSGI servers <howto/deployment/wsgi/index>` |
  :doc:`Deploying static files <howto/static-files/deployment>` |
  :doc:`Tracking code errors by email <howto/error-reporting>`

The admin
=========

Find all you need to know about the automated admin interface, one of Django's
most popular features:

* :doc:`Admin site <ref/contrib/admin/index>`
* :doc:`Admin actions <ref/contrib/admin/actions>`
* :doc:`Admin documentation generator<ref/contrib/admin/admindocs>`

Security
========

Security is a topic of paramount importance in the development of Web
applications and Django provides multiple protection tools and mechanisms:

* :doc:`Security overview <topics/security>`
* :doc:`Disclosed security issues in Django <releases/security>`
* :doc:`Clickjacking protection <ref/clickjacking>`
* :doc:`Cross Site Request Forgery protection <ref/csrf>`
* :doc:`Cryptographic signing <topics/signing>`
* :ref:`Security Middleware <security-middleware>`

Internationalization and localization
=====================================

Django offers a robust internationalization and localization framework to
assist you in the development of applications for multiple languages and world
regions:

* :doc:`Overview <topics/i18n/index>` |
  :doc:`Internationalization <topics/i18n/translation>` |
  :ref:`Localization <how-to-create-language-files>` |
  :doc:`Localized Web UI formatting and form input <topics/i18n/formatting>`
* :doc:`Time zones </topics/i18n/timezones>`

Performance and optimization
============================

There are a variety of techniques and tools that can help get your code running
more efficiently - faster, and using fewer system resources.

* :doc:`Performance and optimization overview <topics/performance>`

Python compatibility
====================

Django aims to be compatible with multiple different flavors and versions of
Python:

* :doc:`Jython support <howto/jython>`
* :doc:`Python 3 compatibility <topics/python3>`

Geographic framework
====================

:doc:`GeoDjango <ref/contrib/gis/index>` intends to be a world-class geographic
Web framework. Its goal is to make it as easy as possible to build GIS Web
applications and harness the power of spatially enabled data.

Common Web application tools
============================

Django offers multiple tools commonly needed in the development of Web
applications:

* **Authentication:**
  :doc:`Overview <topics/auth/index>` |
  :doc:`Using the authentication system <topics/auth/default>` |
  :doc:`Password management <topics/auth/passwords>` |
  :doc:`Customizing authentication <topics/auth/customizing>` |
  :doc:`API Reference <ref/contrib/auth>`
* :doc:`Caching <topics/cache>`
* :doc:`Logging <topics/logging>`
* :doc:`Sending emails <topics/email>`
* :doc:`Syndication feeds (RSS/Atom) <ref/contrib/syndication>`
* :doc:`Pagination <topics/pagination>`
* :doc:`Messages framework <ref/contrib/messages>`
* :doc:`Serialization <topics/serialization>`
* :doc:`Sessions <topics/http/sessions>`
* :doc:`Sitemaps <ref/contrib/sitemaps>`
* :doc:`Static files management <ref/contrib/staticfiles>`
* :doc:`Data validation <ref/validators>`

Other core functionalities
==========================

Learn about some other core functionalities of the Django framework:

* :doc:`Conditional content processing <topics/conditional-view-processing>`
* :doc:`Content types and generic relations <ref/contrib/contenttypes>`
* :doc:`Flatpages <ref/contrib/flatpages>`
* :doc:`Redirects <ref/contrib/redirects>`
* :doc:`Signals <topics/signals>`
* :doc:`System check framework <topics/checks>`
* :doc:`The sites framework <ref/contrib/sites>`
* :doc:`Unicode in Django <ref/unicode>`

The Django open-source project
==============================

Learn about the development process for the Django project itself and about how
you can contribute:

* **Community:**
  :doc:`How to get involved <internals/contributing/index>` |
  :doc:`The release process <internals/release-process>` |
  :doc:`Team organization <internals/organization>` |
  :doc:`Meet the team <internals/team>` |
  :doc:`Current roles <internals/roles>` |
  :doc:`The Django source code repository <internals/git>` |
  :doc:`Security policies <internals/security>` |
  :doc:`Mailing lists <internals/mailing-lists>`

* **Design philosophies:**
  :doc:`Overview <misc/design-philosophies>`

* **Documentation:**
  :doc:`About this documentation <internals/contributing/writing-documentation>`

* **Third-party distributions:**
  :doc:`Overview <misc/distributions>`

* **Django over time:**
  :doc:`API stability <misc/api-stability>` |
  :doc:`Release notes and upgrading instructions <releases/index>` |
  :doc:`Deprecation Timeline <internals/deprecation>`

.. toctree::
   :maxdepth: 2
   :hidden:

   intro/index
   topics/index
   ref/index
   howto/index
