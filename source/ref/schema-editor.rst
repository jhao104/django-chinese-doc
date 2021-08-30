================
``SchemaEditor``
================

.. module:: django.db.backends.base.schema

.. class:: BaseDatabaseSchemaEditor

Django的迁移系统分为两部分: 计算和存储需要运行哪些操作的逻辑(``django.db.migrations``),
以及将“创建模型”或“删除字段”之类的数据库抽象层转换为SQL这是 ``SchemaEditor`` 负责.

作为Django的普通开发人员, 可能不太会直接与 ``SchemaEditor`` 交互, 但是如果您想编写自己的迁移系统, 或者有更高级的需求, 那么它比直接编写SQL要好得多.

Django的每个数据库后端都有自己版本的 ``SchemaEditor``, 它可以通过 ``connection.schema_editor()`` 上下文管理器访问::

    with connection.schema_editor() as schema_editor:
        schema_editor.delete_model(MyModel)

它必须通过上下文管理器来使用, 因为这样可以管理一些类似于事务和延迟SQL(比如创建 ``ForeignKey`` 约束).

它提供了所有可能的操作方法, 这些方法应该按照执行修改的顺序调用. 可能某些操作或类型修改并不可用于所有的数据库. 例如, MyISAM不支持外键约束.

如果在Django中使用和维护了第三方数据库后端, 那么需要提供 ``SchemaEditor`` 实现1.7的迁移功能.
但是, 只要你的数据库在SQL的使用和关系设计上遵循标准, 你应该只需要将 Django 内置的 ``SchemaEditor`` 类子类化,
然后简单调整一下语法. 还请注意, 有一些新的数据库特性是迁移所需要的: ``can_rollback_ddl`` 和 ``supports_combined_alters`` 都很重要.

方法
=======

``execute()``
-------------

.. method:: BaseDatabaseSchemaEditor.execute(sql, params=[])

执行传入的SQL语句, 如果提供了参数则会带上它们. 这是一个对通数据库游标的简单封装, 如果用户需要, 它可以从 ``.sql`` 文件中读取SQL.

``create_model()``
------------------

.. method:: BaseDatabaseSchemaEditor.create_model(model)

在数据库中为给定模型创建表, 以及它所需要的所有唯一约束和索引.

``delete_model()``
------------------

.. method:: BaseDatabaseSchemaEditor.delete_model(model)

删除数据库中给定模型的表以及所有唯一约束和索引.

``alter_unique_together()``
---------------------------

.. method:: BaseDatabaseSchemaEditor.alter_unique_together(model, old_unique_together, new_unique_together)

修改模型的 :attr:`~django.db.models.Options.unique_together` 值; 这将添加或删除模型表的唯一约束, 直到它们与新的值相匹配.

``alter_index_together()``
--------------------------

.. method:: BaseDatabaseSchemaEditor.alter_index_together(model, old_index_together, new_index_together)

修改模型的 :attr:`~django.db.models.Options.index_together` 值; 这将添加或删除模型表中的索引, 直到它们与新值相匹配.

``alter_db_table()``
--------------------

.. method:: BaseDatabaseSchemaEditor.alter_db_table(model, old_db_table, new_db_table)

将模型的表名从 ``old_db_table`` 改为 ``new_db_table``.

``alter_db_tablespace()``
-------------------------

.. method:: BaseDatabaseSchemaEditor.alter_db_tablespace(model, old_db_tablespace, new_db_tablespace)

将模型的表从一个表空间移动到另一个表空间.

``add_field()``
---------------

.. method:: BaseDatabaseSchemaEditor.add_field(model, field)

添加模型字段的列(可以多列). 如果字段设置了 ``db_index=True`` 或 ``unique=True`` 就会创建索引和唯一索引.

如果 ``ManyToManyField`` 字段没有设置 ``through``, 它会创建一个表来表示关联关系而不是一个字段. 如果提供了 ``through`` 则什么都不做.

如果该字段为 ``ForeignKey``, 则会为该字段添加外键约束.

``remove_field()``
------------------

.. method:: BaseDatabaseSchemaEditor.remove_field(model, field)

删除模型字段代表的列(多列), 以及唯一约束, 外键约束和索引.

如果ManyToManyField缺少 ``through`` 值, 它会移除创建用来记录关系的表. 如果没有提供 ``through`` 则什么都不做.

``alter_field()``
-----------------

.. method:: BaseDatabaseSchemaEditor.alter_field(model, old_field, new_field, strict=False)

将模型的旧字段转换为新字段. 这包括修改列名(:attr:`~django.db.models.Field.db_column` 属性), 修改字段类型(如果更改了字段类),
修改字段 ``NULL`` 状态, 新增或删除只属于字段的唯一约束和索引, 修改主键, 修改 ``ForeignKey`` 约束目标.

一个不能做的常见操作是将一个 ``ManyToManyField`` 转换为普通字段, 反之亦然, Django无法在不丢失数据的情况下执行此操作,
因此它将拒绝这样做. 作为代替, 应该分别调用 :meth:`.remove_field` 和 :meth:`.add_field`.

如果数据库有 ``supports_combined_alters``, Django会尝试在一次数据库调用中尽可能多地进行这些操作; 否则, 它会为每一个变化发出单独的ALTER语句, 但不会在不需要变化的地方发出ALTER.

属性
==========

除非另有说明, 所有属性都是只读的.

``connection``
--------------

.. attribute:: SchemaEditor.connection

数据库的连接对象. ``alias`` 是connection的一个实用的属性, 它可以用来确定被访问的数据库的名称.

这在 :ref:`多数据库迁移 <data-migrations-and-multiple-databases>` 进行数据迁移时很有用.
