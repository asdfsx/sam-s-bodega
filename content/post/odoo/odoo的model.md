+++
title = "odoo的模型 Model"
topics = [
  "topic 1",
]
keywords = [
  "python",
  "odoo",
]
description = "odoo的模型 Model"
draft = false
type = "post"
date = "2016-12-30T14:31:31+08:00"
tags = [
  "python",
  "odoo",
]
author = "asdfsx"

+++

`model`是`odoo`中最重要的部分之一。主要负责各种功能的实现，`crm`之类的业务模块中的功能姑且不论，页面渲染、工作流引擎、定时任务等核心的功能，也都是基于模型来实现。

### 元类的基础：odoo.api.Meta
检查要创建的类中的所有函数，然后根据各函数`_api`属性进行特殊处理。处理完成后再创建该类型。

```
class Meta(type):
    """ Metaclass that automatically decorates traditional-style methods by
        guessing their API. It also implements the inheritance of the
        :func:`returns` decorators.
    """

    def __new__(meta, name, bases, attrs):
        # dummy parent class to catch overridden methods decorated with 'returns'
        parent = type.__new__(meta, name, bases, {})

        for key, value in attrs.items():
            if not key.startswith('__') and callable(value):
                # make the method inherit from decorators
                value = propagate(getattr(parent, key, None), value)

                # guess calling convention if none is given
                if not hasattr(value, '_api'):
                    try:
                        value = guess(value)
                    except TypeError:
                        pass

                if (getattr(value, '_api', None) or '').startswith('cr'):
                    _logger.warning("Deprecated method %s.%s in module %s", name, key, attrs.get('__module__'))

                attrs[key] = value

        return type.__new__(meta, name, bases, attrs)
        
```

### 元类：odoo.models.MetaModel
所有的模型的元类，继承了`odoo.api.Meta`，可以自动记录模型属于哪个模块。在Python的数据结构定义了`__new__`和`__init__`的区别。这里也可以发现，`Meta`中`__new__`负责创建类型对象，`MetaModel`中`__init__`负责初始化类型对象。

```
class MetaModel(api.Meta):
    module_to_models = defaultdict(list)
    def __init__(self, name, bases, attrs):
        ...
        if not self._custom:
            self.module_to_models[self._module].append(self)
        ...
```

### 基类的基类：odoo.models.BaseModel

这个基类不再是鸡肋了。它是所有模型的基类。它的作用大致如下：

* 定义了一系列模型的属性
* 根据类属性，用`_build_model`创建新模型类（要处理模型之间的继承关系），新模型类会存放在`registry`中。（`pool`变量其实就是`registry`）

    ```
    @classmethod
    def _build_model(cls, pool, cr):
        ...
        ModelClass.pool = pool
        pool[name] = ModelClass
            
        model = object.__new__(ModelClass)
        model.__init__(pool, cr)
        ...
    ```
* 根据模型属性，用`_setup_base`为新的模型类添加父模型的字段、删除不需要的字段。最重要的添加魔法字段`_add_magic_fields`，其中包括模型的主键`id`。
* 根据模型属性，用`_setup_fields`为新的模型类添加字段。

* 当新的模型类安装配置好以后，用`_auto_init`来进行初始化。
   
    * 根据`_fields`属性，在数据库中的`ir_model`等表中记录模型中的字段；
        
        ```
        @api.model_cr_context
        def _field_create(self):
            ...
            cr.execute(""" INSERT INTO ir_model (model, name, info, state, transient)
                           VALUES (%(model)s, %(name)s, %(info)s, %(state)s, %(transient)s)
                           RETURNING id """, params)
            ...
        ```
    * 根据`_table`属性，在数据库建表（建立一个只有主键的表）；

        ```
        @api.model_cr
        def _create_table(self):
            self._cr.execute('CREATE TABLE "%s" (id SERIAL NOT NULL, PRIMARY KEY(id))' % (self._table,))
            self._cr.execute("COMMENT ON TABLE \"%s\" IS %%s" % self._table, (self._description,))
            _schema.debug("Table '%s': created", self._table)
        ```

    * 根据`_fields`属性，修改之前建的表（用字段填充之间建的没有字段的表，或者改字段名，或者修改字段类型......）

        ```
        @api.model_cr_context
        def _auto_init(self):
           ...
           else :
               # the column doesn't exist in database, create it
               cr.execute('ALTER TABLE "%s" ADD COLUMN "%s" %s' % (self._table, name, field.column_type[1]))
               cr.execute("COMMENT ON COLUMN %s.\"%s\" IS %%s" % (self._table, name), (field.string,))
               ...
           ...
    ```


    * 处理模型的依赖关系（可能反映到数据库上就是一个个外键约束）

        ```
        @api.model_cr
        def _add_sql_constraints(self):
            ...
            def add(name, definition):
                query = 'ALTER TABLE "%s" ADD CONSTRAINT "%s" %s' % (self._table, name, definition)
            ...
        ```
    
* 处理模型的继承关系。  
    例如：在`_build_model`过程中，子类的`cls._table`属性可能直接使用了基类的`cls._table`

    ```
    for base in reversed(cls.__bases__):
        if not getattr(base, 'pool', None):
        ...
            cls._table = base._table or cls._table
        ...
    ```
   
* 创建模型对象`_create`，并返回一个模型类的新对象。  
    这个应该处理创建全新的模型记录。它会将新的模型记录存放到数据库中。
    
    ```
    @api.model
    def _create(self, vals):
        ...
        query = """INSERT INTO "%s" (%s) VALUES(%s) RETURNING id""" % (
                self._table,
                ', '.join('"%s"' % u[0] for u in updates),
                ', '.join(u[1] for u in updates),
            )
        cr.execute(query, tuple(u[2] for u in updates if len(u) > 2))

        # from now on, self is the new record
        id_new, = cr.fetchone()
        self = self.browse(id_new)
        ...
    ```
    
* 创建模型对象`_browse`。  
    可以看到利用`object.__new__(cls)`创建了对象，然后对对象里的属性进行设定。个人感觉，这里创建的模型对象里只有最基本的id，没有模型的其它属性。如果需要其它属性的话，还需要通过`search`来进行查询。这个函数应该用来处理系统中已存在的模型记录。

    ```
    @classmethod
    def _browse(cls, ids, env, prefetch=None):
        records = object.__new__(cls)
        records.env = env
        records._ids = ids
        if prefetch is None:
            prefetch = defaultdict(set)         # {model_name: set(ids)}
        records._prefetch = prefetch
        prefetch[cls._name].update(ids)
        return records
    ```
    
* 模型数据的读取。  
    在基类中还定义了一些特殊的函数`__getitem__`、`__setitem__`。通过这些函数，可以像访问字典一样访问模型对象中的数据。
* 模型数据的查询。

    ```
    @api.model
    def _search(self, args, offset=0, limit=None, order=None, count=False, access_rights_uid=None):
        ...
        query_str = 'SELECT "%s".id FROM ' % self._table + from_clause + where_str + order_by + limit_str + offset_str
        self._cr.execute(query_str, where_clause_params)
        res = self._cr.fetchall()
        ...
    ```


```
class BaseModel(object):
    __metaclass__ = MetaModel
    _auto = False               # don't create any database backend
    _register = False           # not visible in ORM registry
    _abstract = True            # whether model is abstract
    _transient = False          # whether model is transient
    
    _name = None                # the model name
    _table = None               # SQL table name used by model
    ......
    
    @api.model_cr_context
    def _field_create(self):
        ...
    
    @classmethod
    def _build_model(cls, pool, cr):
        ...
        
    @api.model_cr_context
    def _auto_init(self):
        """ Instantiate a given model in the registry.
        ...
```

### 基类：odoo.models.Model
模型大部分是继承这个类的。继承这个类的模型都会自动创建表。也就意味着会有数据写入表中。

```
AbstractModel = BaseModel

class Model(AbstractModel):
    _auto = True                # automatically create database backend
    _register = False           # not visible in ORM registry, meant to be python-inherited only
    _abstract = False           # not abstract
    _transient = False          # not transient
```

### 例子：odoo.addons.base.ir.ir\_ui\_view.View
在之前研究请求处理的时候，看到了这个类。它主要是负责根据模版生成页面的。

* 在`registry`可以通过它的名字`ir.ui.view`来获取这个模型的类。
* 数据库里会有一个表叫做`ir_ui_view`。所有`View`的子类，都会将相关的信息写入这个表。
* `render_template`用来生成页面
    * 使用`get_view_id`，获取`ir.model.data`对象，然后从`ir_model_data`表中根据模版名查找对应的
     
        ```
        select * from ir_model_data where module='web' and name='webclient_bootstrap'
        ```

    * 通过`browse`创建模型`ir.ui.View`的对象，模型对象里存放模版的id
    * 使用模型的`render`函数，制作页面
     
        ```
        class View(models.Model):
            _name = 'ir.ui.view'
            _order = "priority,name,id"

            # Holds the RNG schema
            _relaxng_validator = None
    
            name = fields.Char(string='View Name', required=True)
            model = fields.Char(index=True)
            key = fields.Char()
            priority = fields.Integer(string='Sequence', default=16, required=True)
            type = fields.Selection([('tree', 'Tree'),
                                     ('form', 'Form'),
                                     ('graph', 'Graph'),
                                     ('pivot', 'Pivot'),
                                     ('calendar', 'Calendar'),
                                     ('diagram', 'Diagram'),
                                     ('gantt', 'Gantt'),
                                     ('kanban', 'Kanban'),
                                     ('search', 'Search'),
                                     ('qweb', 'QWeb')], string='View Type')
            ......
    
            @api.model
            def get_view_id(self, template):
                ...
                return self.env['ir.model.data'].xmlid_to_res_id(template, raise_if_not_found=True)
        
            ...
    
            @api.model
            def render_template(self, template, values=None, engine='ir.qweb'):
                return self.browse(self.get_view_id(template)).render(values, engine)
        
             @api.multi
            def render(self, values=None, engine='ir.qweb'):
                assert isinstance(self.id, (int, long))

                qcontext = dict(
                    env=self.env,
                    keep_query=keep_query,
                    request=request, # might be unbound if we're not in an httprequest context
                    debug=request.debug if request else False,
                    json=json,
                    quote_plus=werkzeug.url_quote_plus,
                    time=time,
                    datetime=datetime,
                    relativedelta=relativedelta,
                    xmlid=self.key,
                )
                qcontext.update(values or {})

                return self.env[engine].render(self.id, qcontext)
```

    * 首先获取模版引擎，默认引擎是`ir.qweb`
    * 使用模版引擎的`render`，生成页面

        ```
        class IrQWeb(models.AbstractModel, QWeb):
            _name = 'ir.qweb'
            
            @api.model
            def render(self, id_or_xml_id, values=None, **options):
                ...
                return super(IrQWeb, self).render(id_or_xml_id, values=values, **context)
                
            ......
        
        class QWeb(object):
            ...
            def render(self, template, values=None, **options):
            
            ...
            
            def compile(self, template, options):
            
            ...
        ```
        
### 例子：odoo.addons.base.res.res_user
由`base`模块提供的一个保存`odoo`用户的一个模型。应该有很多模型都利用了这个模型。比如`crm`模块中的`odoo.addons.crm.models.res_user.User`，这个模型就继承并扩展了这个`base`模块中的模型。当安装`crm`模块的时候，会将`crm`中扩展的字段添加到`base`中已经建好的`res_user`表中。

```
下边为base模块中的模型

class Users(models.Model):
    _name = "res.users"
    _description = 'Users'
    _inherits = {'res.partner': 'partner_id'}
    _order = 'name, login'
    __uid_cache = defaultdict(dict)
    
    login = fields.Char(required=True, help="Used to log into the system")
    password = fields.Char(default='', invisible=True, copy=False,
    
.....

下边是crm模块中的模型    

class Users(models.Model):

    _inherit = 'res.users'

    target_sales_won = fields.Integer('Won in Opportunities Target')
    target_sales_done = fields.Integer('Activities Done Target')
```

### 不算总结的总结
大概了解了一下模型的作用。还有遗漏的以后再补。这里遗留了一个问题，就是模型中的字段。
看了一下模型中的字段也算是`odoo`自己实现的`orm`的一部分，包括了元类、基类、一堆扩展。单开一章吧。`view`在`odoo`中也是很大的一块，也单开一章介绍吧。
