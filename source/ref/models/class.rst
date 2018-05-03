===========
模型类参考
===========

.. currentmodule:: django.db.models

这篇文档覆盖 :class:`~django.db.models.Model` 类的特性. 关于模型的更多信息，参见 :doc:`模型参考指南的完整列表 </ref/models/index>`

属性
=====

``objects``
-----------

.. attribute:: Model.objects

    每个非抽象 :class:`~django.db.models.Model` 类必须添加一个
    :class:`~django.db.models.Manager` 实例.
    Django 会确保模型中至少有一个模型的 ``Manager`` . 如果没有添加自己的 ``Manager``,
    Django会添加一个包含 :class:`~django.db.models.Manager` 实例的 ``objects`` 属性.
    如果已经添加了 :class:`~django.db.models.Manager` 实例属性, 则不会有默认管理器.
    例如下面的例子::

        from django.db import models

        class Person(models.Model):
            # Add manager with another name
            people = models.Manager()

    关于模型管理器的更多信息, 参考 :doc:`管理器 </topics/db/managers>`
    和 :ref:`对象查询 <retrieving-objects>`.
