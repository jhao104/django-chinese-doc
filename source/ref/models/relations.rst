=========================
关联对象参考
=========================

.. currentmodule:: django.db.models.fields.related

.. class:: RelatedManager

    "关联管理器" 用于处理一对多和多对多关系的管理器. 它应用于下面两种情况:

    * :class:`~django.db.models.ForeignKey` 关系的 "另一边".
      即::

            from django.db import models

            class Reporter(models.Model):
                # ...
                pass

            class Article(models.Model):
                reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)

      上面代码例子中, ``reporter.article_set`` 将具有下文中的管理器方法.

    * :class:`~django.db.models.ManyToManyField` 关系的两边::

            class Topping(models.Model):
                # ...
                pass

            class Pizza(models.Model):
                toppings = models.ManyToManyField(Topping)

      这个例子中, ``topping.pizza_set`` 和 ``pizza.toppings`` 都将具有下面的管理器方法.

    .. method:: add(*objs, bulk=True)

        添加给定模型对象到关联对象集.

        例如::

            >>> b = Blog.objects.get(id=1)
            >>> e = Entry.objects.get(id=234)
            >>> b.entry_set.add(e) # 将 Entry e 与 Blog b 关联.

        下上面例子中, 由于 :class:`~django.db.models.ForeignKey` 关系使用
        :meth:`QuerySet.update() <django.db.models.query.QuerySet.update>` 方法来保存更新.
        所以这要求关联对象已经 save 到数据中.

        可以使用 ``bulk=False`` 参数来指定管理器通过 ``e.save()`` 来执行更新.

        在多对多关系中 ``add()`` 方法不会调用 ``save()`` 方法, 而是使用 :meth:`QuerySet.bulk_create()
        <django.db.models.query.QuerySet.bulk_create>` 来更新关联关系. 如果在创建关联关系时需要添加一些自定义逻辑,
        可以通过监听 :data:`~django.db.models.signals.m2m_changed` 信号来实现.

        .. versionchanged:: 1.9

            新增 ``bulk`` 参数. 在此之间版本外键都是使用 ``save()`` 更新, 使用 ``bulk=False`` 参数来使用此行为.

    .. method:: create(**kwargs)

        创建并保存一个新对象添加至关联对象集. 返回新创建的对象::

            >>> b = Blog.objects.get(id=1)
            >>> e = b.entry_set.create(
            ...     headline='Hello',
            ...     body_text='Hi',
            ...     pub_date=datetime.date(2005, 1, 1)
            ... )

            # 重点 不用再调用 e.save() 来保存.

        这等价于 (但更加简洁)::

            >>> b = Blog.objects.get(id=1)
            >>> e = Entry(
            ...     blog=b,
            ...     headline='Hello',
            ...     body_text='Hi',
            ...     pub_date=datetime.date(2005, 1, 1)
            ... )
            >>> e.save(force_insert=True)

        注意, 并不需要传入模型中的关联关系参数. 在上面例子中并没有给 ``create()`` 传入 ``blog`` 参数, Django会自动将
        ``Entry`` 对象的 ``blog`` 字段设置为 ``b``.

    .. method:: remove(*objs)

        从关联对象集中删除给的模型对象::

            >>> b = Blog.objects.get(id=1)
            >>> e = Entry.objects.get(id=234)
            >>> b.entry_set.remove(e) # 解除 Entry e 和 Blog b 关联.

        和 :meth:`add()` 方法一样, 上面例子中是调用 ``e.save()`` 来执行更新.
        但是 ``remove()`` 在删除多对多关系时是使用
        :meth:`QuerySet.delete()<django.db.models.query.QuerySet.delete>` 方法而不会调用 ``save()``
        方法, 因此要在删除关系时执行一些自定义操作请监听
        :data:`~django.db.models.signals.m2m_changed` 信号.

        对于 :class:`~django.db.models.ForeignKey` 关联对象, remove方法仅在设置了 ``null=True`` 时可以使用.
        因为如果关联字段不能设置为 ``None`` (``NULL``), 在对象添加了另一个关联对象之前不能删除现有关联对象.
        在上面例子中. 从 ``b.entry_set()`` 移除 ``e`` 就意味着设置 ``e.blog = None``,
        但是如果 ``blog`` 的 :class:`~django.db.models.ForeignKey` 没有设置 ``null=True`` 时这样做是不行的.

        对于 :class:`~django.db.models.ForeignKey` 关联对象, 该方法接收一个
        ``bulk`` 参数用于控制其如何执行更新操作. 如果设置为 ``True`` (默认值) 则会使用 ``QuerySet.update()`` .
        如果设置为 ``bulk=False`` 则是调用每个实例的 ``save()`` 方法, 这个发送
        :data:`~django.db.models.signals.pre_save` 信号和
        :data:`~django.db.models.signals.post_save` 信号, 这比前者多一定性能损耗.

    .. method:: clear()

        从关联对象集中删除所有对象::

            >>> b = Blog.objects.get(id=1)
            >>> b.entry_set.clear()

        注意这并不会删除关联的对象, 这是解除关联.

        和 ``remove()`` 方法一样, ``clear()`` 方法仅在
        :class:`~django.db.models.ForeignKey` 设置了 ``null=True`` 时才可用, 而且它也接收关键字参数 ``bulk``.

    .. method:: set(objs, bulk=True, clear=False)

        .. versionadded:: 1.9

        替换关联对象集::

            >>> new_list = [obj1, obj2, obj3]
            >>> e.related_set.set(new_list)

        该方法接收一个 ``clear`` 参数用于控制如何执行替换操作. 如果设置为 ``False`` (默认值) 则是 ``remove()`` 新集合中不存在的值, 然后添加新值.
        如果设置 ``clear=True`` 则是先调用 ``clear()`` 方法然后一次添加所有值.

        参数 ``bulk`` 用于透传给 :meth:`add` 方法.

        注意由于 ``set()`` 是一个复合操作, 所以它也会受竞争条件影响, 比如在调用 ``clear()`` 之后 ``add()`` 之前又有新值被插入数据库.

    .. note::

       注意 ``add()``, ``create()``, ``remove()``, ``clear()`` 和
       ``set()`` 都会对所有类型的关联字段立即更新至数据库. 换句话说在两边都不需要再调用 ``save()``.

       如果使用了 :ref:`an intermediate model
       <intermediary-manytomany>` for a many-to-many relationship,
       ``add()``, ``create()``, ``remove()``, 和 ``set()`` 方法都会清除预取的缓存.

直接赋值
=================

通过赋值一个新的可迭代的对象, 关联对象集可以被整体替换掉::

    >>> new_list = [obj1, obj2, obj3]
    >>> e.related_set = new_list

如果外键关系设置``null=True``, 则在添加 ``new_list`` 的内容之前, 关联管理器将首先解除关联集中的现有的关联对象, 然后再添加新列表的内容.
否则, 新列表中的对象将直接添加到现有的关联对象集中.

.. versionchanged:: 1.9

    在此之前版本中, 直接赋值是执行 ``clear()`` 然后 ``add()``. 现在是执行带有 ``clear=False`` 参数的 ``set()`` 方法.

.. deprecated:: 1.10

    不再建议直接赋值而是使用
    :meth:`~django.db.models.fields.related.RelatedManager.set` 方法::

        >>> e.related_set.set([obj1, obj2, obj3])

    这可以防止直接赋值带来的隐式保存带来的困惑.
