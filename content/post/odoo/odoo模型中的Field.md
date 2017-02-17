+++
description = "odoo模型中的Field"
author = "asdfsx"
type = "post"
date = "2017-01-05T12:39:46+08:00"
title = "odoo模型中的Field"
tags = [
  "python",
  "odoo",
]
topics = [
  "topic 1",
]
keywords = [
  "python",
  "odoo",
]
draft = false

+++

`Odoo`的定义了自己的一套`ORM`系统。其中的一个重要组成部分就是字段的处理。这部分的内容都在`odoo.field`中。和其他模块一样，也大量使用了python的特殊语法，如：元类、`__slots__`、特殊函数，等。

### 元类：odoo.fields.MetaField

所有的字段类型的元类。如果一个字段类型使用了这个类，那么在创建这个类型时元类会扫描类中是否有`_slots`属性。如果有的话，会将`_slots`中的东西放到`__slots__`中。然后在初始化这个类型的时候，会把这个新创建的类型放在`MetaField`的`by_type`字典中。

注：`__slots__`的作用是用来存放类实例中的属性。默认，python中的实例是存放在`__dict__`中的；如果声明了`__slots__`就不会创建`__dict__`；`__slots__`应该比
`__dict__`节省空间。

```
class MetaField(type):
    """ Metaclass for field classes. """
    by_type = {}
    
    def __new__(meta, name, bases, attrs):
        """ Combine the ``_slots`` dict from parent classes, and determine
        ``__slots__`` for them on the new class.
        """
        base_slots = {}
        for base in reversed(bases):
            base_slots.update(getattr(base, '_slots', ()))

        slots = dict(base_slots)
        slots.update(attrs.get('_slots', ()))

        attrs['__slots__'] = set(slots) - set(base_slots)
        attrs['_slots'] = slots
        return type.__new__(meta, name, bases, attrs)

    def __init__(cls, name, bases, attrs):
        super(MetaField, cls).__init__(name, bases, attrs)
        if cls.type and cls.type not in MetaField.by_type:
            MetaField.by_type[cls.type] = cls

        # compute class attributes to avoid calling dir() on fields
        cls.related_attrs = []
        cls.description_attrs = []
        for attr in dir(cls):
            if attr.startswith('_related_'):
                cls.related_attrs.append((attr[9:], attr))
            elif attr.startswith('_description_'):
                cls.description_attrs.append((attr[13:], attr))
```

### 基类：odoo.fields.Field
所有字段类的基类。定义了一个长长的`_slots`；定义了字段用到的大部分方法；定义了类型转换的方法`convert_to_read`、`convert_to_column`等，以便子类扩展。

```
class Field(object):
    __metaclass__ = MetaField
    type = None                         # type of the field (string)
    relational = False                  # whether the field is a relational one
    translate = False                   # whether the field is translated

    column_type = None                  # database column type (ident, spec)
    column_format = '%s'                # placeholder for value in queries
    ......
```

### 布尔型：odoo.fields.Boolean

布尔类型是最简单的一个实现。可以看到通过覆盖`convert_to_column`函数，定义了如何将数据库中读出的内容，转换为python中的布尔型。

```
class Boolean(Field):
    type = 'boolean'
    column_type = ('bool', 'bool')

    def convert_to_column(self, value, record):
        return bool(value)

    def convert_to_cache(self, value, record, validate=True):
        return bool(value)

    def convert_to_export(self, value, record):
        if record._context.get('export_raw_data'):
            return value
        return ustr(value)
```

其它类似得类还有很多：

* odoo.fields.Integer
* odoo.fields.Float
* odoo.fields.Float
* odoo.fields.Monetary
* odoo.fields._String
* odoo.fields.Char
* odoo.fields.Text
* odoo.fields.Html
* odoo.fields.Date
* odoo.fields.Datetime
* odoo.fields.Binary
* odoo.fields.Selection
* odoo.fields.Reference

### 关系类：odoo.fields._Relational
还有一些特殊的字段用来处理一对多或多对一的关系，它们的父类是`odoo.fields._Relational`

```
class _Relational(Field):
    """ Abstract class for relational fields. """
    relational = True
    _slots = {
        'domain': [],                   # domain for searching values
        'context': {},                  # context for searching values
    }
    ...
```

这个类型有如下子类：

* odoo.fields.Many2one
* odoo.fields.One2many
* odoo.fields.Many2many


### 主键类：odoo.fields.Id

每个模型类中都会有的字段，从这里可以看到，主键与`_ids`的关系。（曾经非常疑惑为什么通过`browse`获得模型类以后，可以访问`id`属性。从`__get__`函数里可以基本找到答案了。）

```
class Id(Field):
""" Special case for field 'id'. """
    type = 'integer'
    column_type = ('int4', 'int4')
    _slots = {
        'string': 'ID',
        'store': True,
        'readonly': True,
    }

    def __get__(self, record, owner):
        if record is None:
            return self         # the field is accessed through the class owner
        if not record:
            return False
        return record.ensure_one()._ids[0]

    def __set__(self, record, value):
        raise TypeError("field 'id' cannot be assigned")
```

### 例子：odoo.addons.base.ir.res.res_users.Users:
依然以`users`为例子

```
class Users(models.Model):
    _name = "res.users"
    _description = 'Users'
    _inherits = {'res.partner': 'partner_id'}
    _order = 'name, login'
    __uid_cache = defaultdict(dict)
    
    partner_id = fields.Many2one('res.partner', required=True, ondelete='restrict', auto_join=True,
        string='Related Partner', help='Partner-related data of the user')
    login = fields.Char(required=True, help="Used to log into the system")
    password = fields.Char(default='', invisible=True, copy=False,
        help="Keep empty if you don't want the user to be able to connect on the system.")
    new_password = fields.Char(string='Set Password',
        compute='_compute_password', inverse='_inverse_password',
        help="Specify a value only when creating a user or if you're "\
             "changing the user's password, otherwise leave empty. After "\
             "a change of password, the user has to login again.")
    signature = fields.Html()
    active = fields.Boolean(default=True)
    action_id = fields.Many2one('ir.actions.actions', string='Home Action',
        help="If specified, this action will be opened at log on for this user, in addition to the standard menu.")
    groups_id = fields.Many2many('res.groups', 'res_groups_users_rel', 'uid', 'gid', string='Groups', default=_default_groups)
    log_ids = fields.One2many('res.users.log', 'create_uid', string='User log entries')
    login_date = fields.Datetime(related='log_ids.create_date', string='Latest connection')
    share = fields.Boolean(compute='_compute_share', compute_sudo=True, string='Share User', store=True,
         help="External user with limited access, created only for the purpose of sharing data.")
    companies_count = fields.Integer(compute='_compute_companies_count', string="Number of Companies", default=_companies_count)
    tz_offset = fields.Char(compute='_compute_tz_offset', string='Timezone offset', invisible=True)

```


### 不算总结的总结
通过自己定的`Field`类型。`Odoo`实现了将数据库中读到的数据转化成python中的数据类型。但是仍然有些没有太搞清楚，如`many2one`、`one2many`这种类型。之后在使用中再研究它们。
