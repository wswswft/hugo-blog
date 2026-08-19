---
title: 踩坑记录——Django 的 Admin 页面的定制
description: 这段时间定制 Django Admin 页面遇到的一些坑以及解决方案
date: 2023-03-22 18:00:00
lastmod: 2023-03-22 18:00:00
image: cover.jpg
categories:
  - Codes
tags:
  - python
  - django
---

Django 提供了强大的 Admin 管理功能，然而它的官方文档对于如何针对已有的管理页面进行深度的定制并没有进行太多的讲解。很多时候，如果我们想要对这个已经相当完善的页面进行修改，会发现自己无从下手；而从网上搜索资料，可能又很难从各种零碎的内容中找到适合自己的解决方案。

我在这段时间一直在开发北京师范大学的心理学经典研究课程的小游戏，这也是我第一次较为深入接触 Django 的 Admin 功能。在感慨其强大之余，我也遇到了不少比较 tricky 的问题，因此就用本文来记录一下这些问题以及解决它们所使用的方案。

## 1 Admin 首页部分模块对staff用户不可见

默认情况下，Admin 首页上的**用户管理**等模块仅对 superuser 用户可见，而对于 staff 用户不可见。例如，我们在应用 `index` 中定义了一个模型 `Game`，并且我们想要让 staff 用户也能在 Admin 首页看到这个模型，该怎么做呢？

我们需要在 `index` 应用的 `admin.py` 文件中首先创建相应的 `GameAdmin` 类。后续我们对于 `Game` 这个模型在 Admin 页面上的管理，都要在这个类中完成：

```python
from django.contrib import admin

@admin.register(Game)
class GameAdmin(admin.ModelAdmin):
    pass
```

`GameAdmin` 继承自 `ModelAdmin`，而后者又继承自 `BaseAdmin`，你可以在这个类中找到很多用来定制 Admin 页面的方法，其中就包括我们当前这一部分用来将模块呈现在 Admin 首页上的 `has_module_permission` 方法。该方法的源代码为：

```python
def has_module_permission(self, request):
    """
    Return True if the given request has any permission in the given
    app label.

    Can be overridden by the user in subclasses. In such case it should
    return True if the given request has permission to view the module on
    the admin index page and access the module's index page. Overriding it
    does not restrict access to the add, change or delete views. Use
    `ModelAdmin.has_(add|change|delete)_permission` for that.
    """
    return request.user.has_module_perms(self.opts.app_label)
```

我们只需要在我们当前的 `GameAdmin` 类中重写这个方法即可。然而，需要注意一件事情：我们不能仅仅判断用户是否是 staff 或 superuser，因为我们还可以手动给一些用户赋予权限，而这些用户同样应该可以访问这一模块。所以，我的实现是这样的：

```python
@admin.register(Game)
class GameAdmin(admin.ModelAdmin):
    def has_module_permission(self, request):
        has_permission = super().has_module_permission(request)
        return request.user.is_staff or has_permission
```

这段代码相当于在原始的判断基础上，增加一个用户是否为 staff 的判断。

同理，BaseAdmin 中还提供了类似的 `has_add_permission` / `has_delete_permission` / `has_change_permission` / `has_view_permission` 等权限，用法和 `has_module_permission` 相同。

## 2 针对不同用户显示不同的字段 / 过滤器 / ……

在开发的过程中，我需要实现这样一个功能：将一部分字段设置为仅对 superuser 可见，对 staff 用户不可见。我们都知道，可以通过设置 `list_display` 属性来控制显示的字段，但是这种方式设置出来的可见字段对于所有能够访问 Admin 页面的用户都是可见的；而如果要更精细地控制哪些字段对哪些用户可见，就需要使用 `get_list_display`：

```python
@admin.register(Game)
class GameAdmin(admin.ModelAdmin):
    list_display = ['user', 'level']

    def get_list_display(self, request):
        if not request.user.is_superuser:
            return ['username', 'level', 'is_active']
        else:
            return self.list_display
```

同理，我们还可以用类似的方式，实现对于过滤器（`get_list_filter`）、只读字段（`get_readonly_fields`）、属性修改（`get_fieldsets`）等的控制。

## 3 根据外键的属性进行筛选

假定我们的 `Game` 模型中有一个外键 `user`，对应着一个用户模型，该模型有一个字段 `username`，现在如果我们想要对 `username` 设置一个 filter，该怎么操作呢？方法是：用双下划线 `__` 连接外键名和字段名：`user__username`。


```python
@admin.register(Game)
class GameAdmin(admin.ModelAdmin):
    list_filter = ['user__username']
```

这种方式在 `.filter` 方法中同样适用，例如：

```python
Game.objects.filter(user__username='Someone')
```

## 4 如何在字段中显示外键的属性

一般来说，我们的 Admin 页面只会显示模型中有的字段；如果使用了外键，一般显示的会是外键的模型的`__str__`方法返回的值；但是，如果我们希望额外用一个字段显示外键的某个属性，该怎么做呢？我们只需要再定义一个方法，让它返回相应的值，并将方法名添加到 `list_display` 即可。例如，下面的代码就额外将 `user` 的 `username` 属性显示了出来：

```python
@admin.register(Game)
class GameAdmin(admin.ModelAdmin):
    list_display = ['user', 'username']

    def username(self, obj):
        return obj.user.username
    username.short_description = '用户名'
```

请注意，`list_display` 中的 `'username'` 是和 `username` 这个方法同名，而不是和 `username` 字段同名。如果我们将 `username` 方法的名字进行了修改，就同样需要对 `list_display` 中的字段名称进行修改。

`username` 方法传入的 `obj`，你可以把它理解为当前管理的模型，因为 `Game` 模型有 `user` 字段，`user` 下有 `username` 字段，所以通过 `obj.user.username` 进行访问。至于 `short_description`，则是给字段一个自定义的名字。
