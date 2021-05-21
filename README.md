>如果感兴趣， 欢迎加入进来一起翻译。Folk 项目:https://github.com/jhao104/django-chinese-doc 提交你的内容。翻译后的文档会自动更新到:http://django-chinese-doc.readthedocs.io/zh_CN/latest/

# Django中文文档
#### 当前版本 Django1.10
[![Documentation Status](https://readthedocs.org/projects/django-chinese-doc/badge/?version=latest)](http://django-chinese-doc.readthedocs.io/zh_CN/latest/?badge=latest)
[![](https://img.shields.io/badge/Powered%20by-@j_hao104-blue.svg)](http://www.spiderpy.cn/blog/)

　　你想知道的关于Django的一切。

## 文档组织方式

　　Django有很多的文档。一个好的组织结构能够帮助您在何处寻找你想要的内容：

* [入门教程](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/index.html)一步步详解如何从零开始创建一个Web应用程序。如果您第一次使用Django或Web应用程序开发，那么请从这里开始。也就是下面的“[入门](#入门)”部分。

* [主题指南](http://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/index.html)在比较高的层次上讨论某些话题和概念，并提供有用的背景信息和解释。

* [参考指南](http://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/index.html)包含API和Django的机械等方面的技术参考。抓哟描述了它的工作原理以及如何使用它，但是这要求您对关键概念有基本的的了解。

* [方法指南](http://django-chinese-doc.readthedocs.io/zh_CN/latest/howto/index.html)就像一种秘诀。他们将帮助您完成解决关键问题和用例的步骤。他们比教程更先进，并且还会帮你你了解Django的工作原理。

## <span id = "first_steps">入门：</span>

　　如果你是刚接触Django或编程，这是一个起点。

* **从零开始**: [概览](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/overview.html) | [安装](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/install.html)

* **入门教程**: [Part1:请求与响应](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial01.html) | [Part2:模型和管理站点](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial02.html) | [Part3:视图和模板](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial03.html) | [Part4:表单和通用视图](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial04.html) | [Part5:测试](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial05.html) | [Part6:静态文件](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial06.html) | [Part7:自定义管理站点](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/tutorial07.html)

* **进阶教程**: [开发可重用的应用](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/reusable-apps.html) | [开发Django补丁](http://django-chinese-doc.readthedocs.io/zh_CN/latest/intro/contributing.html)

## 模型层

　　Django提供一个抽象层（Models）以构建和操作你的web应用中的数据，通过以下内容了解更多：

* **模型**：[模型简介](http://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/db/models.html)|[字段类型](http://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/models/fields.html)|[元选项](http://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/models/options.html)|[模型类](http://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/models/class.html)

* **查询集**: [执行查询](http://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/db/queries.html)|[QuerySet方法](http://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/models/querysets.html)|[查询表达式](https://django-chinese-doc.readthedocs.io/zh_CN/latest/ref/models/lookups.html)

* **模型实例**: [实例方法(0%)](https://docs.djangoproject.com/en/1.11/ref/models/instances/)|[访问关联对象(0%)](http://python.usyiyi.cn/documents/django_182/ref/models/relations.html)
 
* **迁移**: [迁移简介 (0%)](https://docs.djangoproject.com/en/1.10/topics/migrations/)|[操作参考 (0%)](https://docs.djangoproject.com/en/1.10/ref/migration-operations/)|[模式编辑器 (0%)](https://docs.djangoproject.com/en/1.10/ref/schema-editor/)|[编写迁移 (0%)](https://docs.djangoproject.com/en/1.10/howto/writing-migrations/)

* **高级**: [管理器 (0%)](https://docs.djangoproject.com/en/1.10/topics/db/managers/)|[原始SQL (0%)](https://docs.djangoproject.com/en/1.10/topics/db/sql/)|[事务 (0%)](https://docs.djangoproject.com/en/1.10/topics/db/transactions/)|[聚合](http://django-chinese-doc.readthedocs.io/zh_CN/latest/topics/db/aggregation.html)|[查询 (0%)](https://docs.djangoproject.com/en/1.10/topics/db/search/)|[自定义字段 (0%)](https://docs.djangoproject.com/en/1.10/howto/custom-model-fields/)|[多数据库 (0%)](https://docs.djangoproject.com/en/1.10/topics/db/multi-db/)|[自定义查询 (0%)](https://docs.djangoproject.com/en/1.10/howto/custom-lookups/)|[查询表达式 (0%)](https://docs.djangoproject.com/en/1.10/ref/models/expressions/)|[条件表达式 (0%)](https://docs.djangoproject.com/en/1.10/ref/models/conditional-expressions/)|[数据库函数 (0%)](https://docs.djangoproject.com/en/1.10/ref/models/conditional-expressions/)

* **其他**: [支持的数据库 (0%)](https://docs.djangoproject.com/en/1.10/ref/databases/)|[遗留的数据库 (0%)](https://docs.djangoproject.com/en/1.10/howto/legacy-databases/)|[提供初始数据 (0%)](https://docs.djangoproject.com/en/1.10/howto/initial-data/)|[优化数据库访问 (0%)](https://docs.djangoproject.com/en/1.10/topics/db/optimization/)|[PostgreSQL specific features (0%)](https://docs.djangoproject.com/en/1.10/ref/contrib/postgres/)
