+++
title = "odoo简化版refrence"
topics = [
  "topic 1",
]
description = "description"
author = "asdfsx"
draft = false
type = "post"
date = "2017-02-05T12:17:22+08:00"
tags = [
  "python",
  "odoo",
]
keywords = [
  "python",
  "odoo",
]

+++

原文地址：https://www.odoo.com/documentation/10.0/index.html

整理一下，捡重点的来弄。

* [Odoo Guidelines](#Odoo Guidelines)
  * [模块结构](#模块结构)
      * [目录结构](#目录结构) 
      * [命名规则](#命名规则)
  * [XML files](#XML files)
  * [Python](#Python)
      * [Idiomatics Python Programming](#Idiomatics Python Programming)
      * [Programming in Odoo](#Programming in Odoo)
* [Module](#Module)
* [Web Controllers](#Web Controllers)
* [ORM API](#ORM API)
  * [Recordsets](#Recordsets)
  * [Environment](#Environment)
  * [Common ORM methods](#Common ORM methods)
  * [Creating Models](#Creating Models)
  * [Compatibility between new API and old API](#Compatibility between new API and old API)
  * [Model Reference](#Model Reference)
  * [Method decorators](#Method decorators)
  * [Fields](#Fields)
* [Data Files](#Data Files)
* [QWeb](#QWeb)
* [Views](#Views)


<h1 id="Odoo Guidelines">Odoo Guidelines</h1>

<h2 id="模块结构">模块结构</h2>

<h3 id="目录结构">目录结构</h3>
主要目录：

* data/ : demo and data xml
* models/ : models definition
* controllers/ : contains controllers (HTTP routes).
* views/ : contains the views and templates
* static/ : contains the web assets, separated into css/, js/, img/, lib/, ...

可选目录：

* wizard/ : regroups the transient models (formerly osv_memory) and their views.
* report/ : contains the reports (RML report [deprecated], models based on SQL views (for reporting) and other complex reports). Python objects and XML views are included in this directory.
* tests/ : contains the Python/YML tests

<h3 id="命名规则">命名规则</h3>

* 对于视图，将后段的 views 与前端的模版拆成两个不同的文件。  

* 对于模型，将一个商业逻辑切分成为多组模型，每一组选择一个主模型，用主模型的名字来定义这个组。如果只有一个模型，那么这个模型的名字与模块相同。每组定义了<main_model>的模型会创建以下文件：

  * models/<main_model>.py
  * models/<inherited_main_model>.py
  * views/<main_model>_templates.xml
  * views/<main_model>_views.xml

  举例：sale 模块引入 sale_order 和 sale_order_line，其中 sale_order 是主模型。所以主模型的文件名为：models/sale\_order.py 和 views/sale\_order\_views.py。

* 对于数据，根据目的分为：demo 和 data。文件名是主模型名字加后缀 \_data.xml 或 \_demo.xml。

* 对于控制器，唯一的控制器被命名为 main.py。如果需要从其他模块中继承已有的控制器，则文件名为 \<module\_name\>.py。与模型不同，每个 controller 类要存放在单独的文件里。

* 对于静态文件，因为资源可能会在不同的context下使用（前端、后端、both），所以会把它们放在同一个 bundle 中。所以 CSS/Less，JavaScript 和 XML 会在文件末尾加bundle类型作为后缀。i.e.: im\_chat\_common.css，im\_chat\_common.js for 'assets\_common' bundle，and im\_chat\_backend.css, im\_chat\_backend.js for 'assets\_backend' bundle。如果模块中只有一个文件，习惯上使用 \<module\_name\>.ext (i.e.: project.js)。不要使用外部链接（图片、库）：不要使用图片的URL，而是把图片放到代码库中。

* 对于报表，
  * \<report\_name\_A\>\_report.py
  * \<report\_name\_A\>\_report\_views.py

* 对于可打印报表，
  * \<print\_report\_name\>\_reports.py (report actions, paperformat definition, ...)
  * \<print\_report\_name>\_templates.xml (xml report templates)

完整的目录结构：

```xml
addons/<my_module_name>/
|-- __init__.py
|-- __manifest__.py
|-- controllers/
|   |-- __init__.py
|   |-- <inherited_module_name>.py
|   `-- main.py
|-- data/
|   |-- <main_model>_data.xml
|   `-- <inherited_main_model>_demo.xml
|-- models/
|   |-- __init__.py
|   |-- <main_model>.py
|   `-- <inherited_main_model>.py
|-- report/
|   |-- __init__.py
|   |-- <main_stat_report_model>.py
|   |-- <main_stat_report_model>_views.xml
|   |-- <main_print_report>_reports.xml
|   `-- <main_print_report>_templates.xml
|-- security/
|   |-- ir.model.access.csv
|   `-- <main_model>_security.xml
|-- static/
|   |-- img/
|   |   |-- my_little_kitten.png
|   |   `-- troll.jpg
|   |-- lib/
|   |   `-- external_lib/
|   `-- src/
|       |-- js/
|       |   `-- <my_module_name>.js
|       |-- css/
|       |   `-- <my_module_name>.css
|       |-- less/
|       |   `-- <my_module_name>.less
|       `-- xml/
|           `-- <my_module_name>.xml
|-- views/
|   |-- <main_model>_templates.xml
|   |-- <main_model>_views.xml
|   |-- <inherited_main_model>_templates.xml
|   `-- <inherited_main_model>_views.xml
`-- wizard/
    |-- <main_transient_A>.py
    |-- <main_transient_A>_views.xml
    |-- <main_transient_B>.py
    `-- <main_transient_B>_views.xml
```

注意使用正确的文件权限：目录 755 ，文件 644

<h2 id="XML files">XML files</h2>

### Format
```xml
<record id="view_id" model="ir.ui.view">
    <field name="name">view.name</field>
    <field name="model">object_name</field>
    <field name="priority" eval="16"/>
    <field name="arch" type="xml">
        <tree>
            <field name="my_field_1"/>
            <field name="my_field_2" string="My Label" widget="statusbar" statusbar_visible="draft,sent,progress,done" />
        </tree>
    </field>
</record>
```
### Naming xml_id
#### Security, View and Action

```xml
<!-- views and menus -->
<record id="model_name_view_form" model="ir.ui.view">
    ...
</record>

<record id="model_name_view_kanban" model="ir.ui.view">
    ...
</record>

<menuitem
    id="model_name_menu_root"
    name="Main Menu"
    sequence="5"
/>
<menuitem
    id="model_name_menu_action"
    name="Sub Menu 1"
    parent="module_name.module_name_menu_root"
    action="model_name_action"
    sequence="10"
/>

<!-- actions -->
<record id="model_name_action" model="ir.actions.act_window">
    ...
</record>

<record id="model_name_action_child_list" model="ir.actions.act_window">
    ...
</record>

<!-- security -->
<record id="module_name_group_user" model="res.groups">
    ...
</record>

<record id="model_name_rule_public" model="ir.rule">
    ...
</record>

<record id="model_name_rule_company" model="ir.rule">
    ...
</record>
```

#### Inherited XML
继承一个 view 的命名方式：`<base_view>_inherit_<current_module_name>`

```xml
<record id="inherited_model_view_form_inherit_my_module" model="ir.ui.view">
    ...
</record>
```

<h2 id="Python">Python</h2>

### PEP8
### Imports
按照如下顺序引用库

```python
# 1 : imports of python lib
import base64
import re
import time
from datetime import datetime
# 2 :  imports of odoo
import odoo
from odoo import api, fields, models # alphabetically ordered
from odoo.tools.safe_eval import safe_eval as eval
from odoo.tools.translate import _
# 3 :  imports from odoo modules
from odoo.addons.website.models.website import slug
from odoo.addons.web.controllers.main import login_redirect
```

<h3 id="Idiomatics Python Programming">Idiomatics Python Programming</h3>

* 每个文件的第一行是`# -*- coding: utf-8 -*-`
* 可读性要比简洁、语言特性更重要
* 不要使用`clone`

  ```python
  # bad
  new_dict = my_dict.clone()
  new_list = old_list.clone()
  # good
  new_dict = dict(my_dict)
  new_list = list(old_list)
  ```
* python字典的创建和更新

  ```python
  # -- creation empty dict
  my_dict = {}
  my_dict2 = dict()

  # -- creation with values
  # bad
  my_dict = {}
  my_dict['foo'] = 3
  my_dict['bar'] = 4
  # good
  my_dict = {'foo': 3, 'bar': 4}

  # -- update dict
  # bad
  my_dict['foo'] = 3
  my_dict['bar'] = 4
  my_dict['baz'] = 5
  # good
  my_dict.update(foo=3, bar=4, baz=5)
  my_dict = dict(my_dict, **my_dict2)
  ```

* 命名变量／类／函数时，使用有意义的名字
* 无用的变量，临时变量可以让代码更明确，但是不意味着总要创建临时变量。

  ```python
  # pointless
  schema = kw['schema']
  params = {'schema': schema}
  # simpler
  params = {'schema': kw['schema']}
  ```
* 用多个 return 来让代码更简单

  ```python
  # a bit complex and with a redundant temp variable
  def axes(self, axis):
        axes = []
        if type(axis) == type([]):
                axes.extend(axis)
        else:
                axes.append(axis)
        return axes

  # clearer
  def axes(self, axis):
        if type(axis) == type([]):
                return list(axis) # clone the axis
        else:
                return [axis] # single-element list
  ```

* 熟悉内建函数

  ```python
  value = my_dict.get('key', None) # very very redundant
  value= my_dict.get('key') # good
  ```

* 列表表达式，它们会让代码更好读

  ```python
  # not very good
  cube = []
  for i in res:
        cube.append((i['id'],i['name']))
  # better
  cube = [(i['id'], i['name']) for i in res]
  ```
* 集合也可以作为布尔类型，空为 false，非空为 true。

  ```python
  bool([]) is False
  bool([1]) is True
  bool([False]) is True
  ```
  所以你可以写 `if some_collection:` 而不是 `if len(some_collection):`
  
* 迭代

  ```python
  # creates a temporary list and looks bar
  for key in my_dict.keys():
        "do something..."
  # better
  for key in my_dict:
        "do something..."
  # creates a temporary list
  for key, value in my_dict.items():
        "do something..."
  # only iterates
  for key, value in my_dict.iteritems():
        "do something..."
  ```
* 使用`dict.setdefault`

  ```python
  # longer.. harder to read
  values = {}
  for element in iterable:
    if element not in values:
        values[element] = []
    values[element].append(other_value)

  # better.. use dict.setdefault method
  values = {}
  for element in iterable:
    values.setdefault(element, []).append(other_value)
  ```
* 好的开发者，要做好文档工作。
* 更多的注意事项参看：http://python.net/~goodger/projects/pycon/2007/idiomatic/handout.html

<h3 id="Programming in Odoo">Programming in Odoo</h3>

* 不要创建生成器和装饰漆，只使用odoo api中提供的
* 使用`filtered`，`mapped`，`sorted`...等函数来提高可能性和性能

#### Make your method works in batch

#### Propagate the context

#### Do not bypass the ORM

```python
# very very wrong
self.env.cr.execute('SELECT id FROM auction_lots WHERE auction_id in (' + ','.join(map(str, ids))+') AND state=%s AND obj_price > 0', ('draft',))
auction_lots_ids = [x[0] for x in self.env.cr.fetchall()]

# no injection, but still wrong
self.env.cr.execute('SELECT id FROM auction_lots WHERE auction_id in %s '\
           'AND state=%s AND obj_price > 0', (tuple(ids), 'draft',))
auction_lots_ids = [x[0] for x in self.env.cr.fetchall()]

# better
auction_lots_ids = self.search([('auction_id','in',ids), ('state','=','draft'), ('obj_price','>',0)])
```

#### No SQL injections, please !

```python
# the following is very bad:
#   - it's a SQL injection vulnerability
#   - it's unreadable
#   - it's not your job to format the list of ids
self.env.cr.execute('SELECT distinct child_id FROM account_account_consol_rel ' +
           'WHERE parent_id IN ('+','.join(map(str, ids))+')')

# better
self.env.cr.execute('SELECT DISTINCT child_id '\
           'FROM account_account_consol_rel '\
           'WHERE parent_id IN %s',
           (tuple(ids),))
```

#### Keep your methods short/simple when possible

#### Never commit the transaction

#### Use translation method correctly

### Symbols and Conventions
* 模块名
* Odoo中的python类：驼峰命名

  ```
  class AccountInvoice(models.Model):
      ...
  ```
* 变量名
* `One2Many` 和 `Many2Many`字段需要有 \_ids 后缀，如：sale\_order\_line\_ids
* `Many2One` 字段需要有 \_id 后缀，如：partner\_id, user\_id, ...

<h1 id="Module">Module</h1>

描述文件用来描述将 odoo 的模块，并定义该模块的元数据。它的文件名是`__manifest__.py`，里边只包含一个 python 字典。

## Manifest

```python
{
    'name': "A Module",
    'version': '1.0',
    'depends': ['base'],
    'author': "Author Name",
    'category': 'Category',
    'description': """
    Description text
    """,
    # data files always loaded at installation
    'data': [
        'mymodule_view.xml',
    ],
    # data files containing optionally loaded demonstration data
    'demo': [
        'demo_data.xml',
    ],
}
```

<h1 id="Web Controllers">Web Controllers</h1>

## Routing
`odoo.http.route(route=None, **kw)`  
装饰器，用来装饰处理请求的函数，该函数，必须属于`Controller`的子类

## Request

request 的对象会在处理请求的时候自动判断并设置。

`class odoo.http.WebRequest(httprequest)`  
`class odoo.http.HttpRequest(*args)`  
`class odoo.http.JsonRequest(*args)`

## Response
`class odoo.http.Response(*args, **kw)`

## Controllers
控制器都继承了`class odoo.http.Controller`，处理请求的方法使用`route`来装饰

```python
class MyController(odoo.http.Controller):
    @route('/some_url', auth='public')
    def handler(self):
        return stuff()
```

覆盖一个控制器

```python
class Extension(MyController):
    @route()
    def handler(self):
        do_before()
        return super(Extension, self).handler()
```

* 使用`route`装饰是为了让方法可见：如果一个方法被重新定义了，但是没有装饰，那么它不可见
* 所有装饰过的方法都会整合在一起。如果方法的装饰器没有参数之前所有的定义都会被保留，任何新的参数都会覆盖之前的参数。

```python
class Restrict(MyController):
    @route(auth='user')
    def handler(self):
        return super(Restrict, self).handler()
```
会修改`/some_url`的认证方式，从 public 改为 user（需要登录）。

<h1 id="ORM API">ORM API</h1>

<h2 id="Recordsets">Recordsets</h2>

模型和记录之间的操作是通过 recordset 来进行的，同一个模型里排好序的记录集。定义在模型上的方法是通过 recordset 来执行的，他们的 self 就是 recordset。

```python
class AModel(models.Model):
    _name = 'a.model'
    def a_method(self):
        # self can be anywhere between 0 records and all records in the
        # database
        self.do_operation()
```
遍历一个 recordset 会返回一个单独的记录

```python
def do_operation(self):
    print self # => a.model(1, 2, 3, 4, 5)
    for record in self:
        print record # => a.model(1), then a.model(2), then a.model(3), ...
```

源码中的实现

```python
https://github.com/odoo/odoo/blob/59d1f9b564f1a4cec8a061e148725a3a9f7ac853/odoo/models.py#L5054

    def __iter__(self):
        """ Return an iterator over ``self``. """
        for id in self._ids:
            yield self._browse((id,), self.env, self._prefetch)
```

### Field access
Recordsets 提供一个接口可以直接访问模型的字段，设置一个值会触发更新数据库。

```python
>>> record.name
Example Name
>>> record.company_id.name
Company Name
>>> record.name = "Bob"
```
访问字段时，recordsets 里只能有一条记录，尝试从包含多条记录的 recordset 中读取模型字段会触发异常。

访问关系型字段（Many2one，One2many，Many2many）总会返回一个 recordset，

> 注意：  
> 每次设置字段都会触发一次数据库的更新，如果要同时设置多个字段，使用 `write`
>
> ```python
# 3 * len(records) database updates
for record in records:
    record.a = 1
    record.b = 2
    record.c = 3

# len(records) database updates
for record in records:
    record.write({'a': 1, 'b': 2, 'c': 3})
 
# 1 database update
records.write({'a': 1, 'b': 2, 'c': 3})
```



### Record cache and prefetching
Odoo 提供了缓存来保存记录的各个字段，所以不是每次对记录的访问都需要读取数据库。

```python
record.name             # first access reads value from database
record.name             # second access gets value from cache
```

### Set operations
`record in set` `record not in set`

`set1 <= set2` `set1 < set2`

`set1 >= set2` `set1 > set2`

`set1 | set2`

`set1 & set2`

`set1 - set2`

### Other recordset operations
`filtered()`

`sorted()`

`mapped()`


<h2 id="Environment">Environment</h2>

`Environment`中保存了大量 ORM 用到的上下文信息：数据库连接（用来查询数据库），当前用户（用来获取权限信息），当前的上下文（存储各种元数据）。而且里边还有缓存。

所有的 recordsets 中都有一个不可变的 enviroment，可以通过 `env` 获得，并可以通过它来获取当前用户（user），数据库连接（cr）和上下文（context）

```python
>>> records.env
<Environment object ...>
>>> records.env.user
res.user(3)
>>> records.env.cr
<Cursor object ...)
```

当通过一个 recordset 创建另一个 recordset 时，enviroment 会被继承下来。enviroment 可以用来获得另外一个模型的空 recordset ，然后进行查询

```python
>>> self.env['res.partner']
res.partner
>>> self.env['res.partner'].search([['is_company', '=', True], ['customer', '=', True]])
res.partner(7, 18, 12, 14, 17, 19, 8, 31, 26, 16, 13, 20, 30, 22, 29, 15, 23, 28, 74)
```

### Altering the environment
`sudo()`

```python
# create partner object as administrator
env['res.partner'].sudo().create({'name': "A Partner"})

# list partners visible by the "public" user
public = env.ref('base.public_user')
env['res.partner'].sudo(public).search([])
```
`with_context()`

```python
# look for partner, or create one with specified timezone if none is
# found
env['res.partner'].with_context(tz=a_tz).find_or_create(email_address)
```

`with_env()`

<h2 id="Common ORM methods">Common ORM methods</h2>

`search()`

```python
>>> # searches the current model
>>> self.search([('is_company', '=', True), ('customer', '=', True)])
res.partner(7, 18, 12, 14, 17, 19, 8, 31, 26, 16, 13, 20, 30, 22, 29, 15, 23, 28, 74)
>>> self.search([('is_company', '=', True)], limit=1).name
'Agrolait'
```
>注意：  
>如果只是想检查记录的条数，可以使用`search_count()`

`create()`

```python
>>> self.create({'name': "New Name"})
res.partner(78)
```

`write()`

```python
self.write({'name': "Newer Name"})
```

`browse()`

```python
>>> self.browse([7, 18, 12])
res.partner(7, 18, 12)
```

`exists()`

```python
if not record.exists():
    raise Exception("The record has been deleted")
    
records.may_remove_some()
# only keep records which were not deleted
records = records.exists()

```

`ref()`

```python
>>> env.ref('base.group_public')
res.groups(2)
```

`ensure_one()`

```python
records.ensure_one()
# is equivalent to but clearer than:
assert len(records) == 1, "Expected singleton"
```

<h2 id="Creating Models">Creating Models</h2>

模型的字段被定义为模型自身的属性。

```python
from odoo import models, fields
class AModel(models.Model):
    _name = 'a.model.name'

    field1 = fields.Char()
```

通过`string`属性，来定义字段的标签

```python
field2 = fields.Integer(string="an other field")
```

通过`default`属性，来定义字段的默认值

```python
a_field = fields.Char(default="a value")
```
 
也可以用一个方法来计算默认值，这个方法必须返回这个字段的值

```
def compute_default_value(self):
    return self.get_value()
a_field = fields.Char(default=compute_default_value)
```

### Computed fields
通过使用 compute 参数设置一个函数，可以计算字段的值（而不只是从数据库中读取）。__这个函数必须将计算好的值赋给对应的字段__。如果计算过程依赖了其它字段，要使用 depends() 来标明。

```python
from odoo import api
total = fields.Float(compute='_compute_total')

@api.depends('value', 'tax')
def _compute_total(self):
    for record in self:
        record.total = record.value + record.value * record.tax
```

* 依赖中可以引用子字段

    ```python
    @api.depends('line_ids.value')
    def _compute_total(self):
        for record in self:
          record.total = sum(line.value for line in record.line_ids)
    ```

* 可计算字段默认不会存储。设置`store=True`会将字段保存到数据库，并自动允许查询。
* 通过设置 search 参数，可以让可计算字段也能进行搜索

    ```
    upper_name = field.Char(compute='_compute_upper', search='_search_upper')

    def _search_upper(self, operator, value):
        if operator == 'like':
            operator = 'ilike'
        return [('name', operator, value)]
    ```   
* 通过设置 inverse 参数，可以向可计算字段赋值
    
    ```
    document = fields.Char(compute='_get_document', inverse='_set_document')

    def _get_document(self):
        for record in self:
            with open(record.get_document_path) as f:
                record.document = f.read()
    def _set_document(self):
        for record in self:
            if not record.document: continue
            with open(record.get_document_path()) as f:
                f.write(record.document)
    ```
* 设置了同一个函数的多个可计算字段，可以一起获得值

    ```
    discount_value = fields.Float(compute='_apply_discount')
total = fields.Float(compute='_apply_discount')

    @depends('value', 'discount')
    def _apply_discount(self):
        for record in self:
            # compute actual discount from discount percentage
            discount = record.value * record.discount
            record.discount_value = discount
            record.total = record.value - discount
    ```
    
##### Related fields

```
nickname = fields.Char(related='user_id.partner_id.name', store=True)
```

### onchange: updating UI on the fly

当用户修改表单中的值时，可以根据新的值自动修改其它字段的值。

* 可计算字段本身就可以检查并重新计算，他们不需要 onchange
* 不可计算字段，可以通过 onchange() 装饰器来获得此功能
 
    ```
    @api.onchange('field1', 'field2') # if these fields are changed, call method
    def check_change(self):
        if self.field1 < self.field2:
            self.field3 = True
    ```
* Both computed fields and new-API onchanges are automatically called by the client without having to add them in views
* It is possible to suppress the trigger from a specific field by adding on_change="0" in a view:

    ```
    <field name="name" on_change="0"/>
    ```
will not trigger any interface update when the field is edited by the user, even if there are function fields or explicit onchange depending on that field.

> onchange 不会修改数据库记录。

### Low-level SQL
enviroment 中的 cr 属性是当前连接数据库事务的一个游标，通过它可以直接执行 SQL 从而实现复杂的查询语句。

```
self.env.cr.execute("some_sql", param1, param2, param3)
```

因为其它模型也在使用同一个游标，而且 enviroment 中还保存了大量的缓存，当使用原始 SQL 修改数据库的时候，要让这些缓存都失效。在使用 CREATE，UPDATE，DELETE 后都应该清理缓存。

清除缓存可以使用 enviroment 的 invalidate_all() 方法。

<h2 id="Compatibility between new API and old API">Compatibility between new API and old API</h2>

Odoo 正在逐步从老的api过渡到新的。

新老 api 最大的区别是：

* 老 api 中，enviroment 中的值（cursor, user id and context）是显式传递的
* 老 api 中，记录信息（ids）是显式传递的
* 老 api 中，方法都是基于 ids 列表工作，而不是 recordset

```
>>> # method in the old API style
>>> def old_method(self, cr, uid, ids, context=None):
...    print ids

>>> # method in the new API style
>>> def new_method(self):
...     # system automatically infers how to call the old-style
...     # method from the new-style method
...     self.old_method()

>>> env[model].browse([1, 2, 3, 4]).new_method()
[1, 2, 3, 4]
```

<h2 id="Model Reference">Model Reference</h2>

`class odoo.models.Model(pool, cr)`

Odoo的模型都继承了这个类

```
class user(Model):
    ...
```
### Structural attributes
### CRUD
`create(vals) → record`  
`browse([ids]) → records`  
`unlink()`  
`write(vals)`  
`read([fields])`  
`read_group(domain, fields, groupby, offset=0, limit=None, orderby=False, lazy=True)`  

### Searching
`search(args[, offset=0][, limit=None][, order=None][, count=False])`  
`search_count(args) → int`  
`name_search(name='', args=None, operator='ilike', limit=100) → records`

### Recordset operations
`ids`  
`ensure_one()`
`exists() → records`
`filtered(func)`
`sorted(key=None, reverse=False)`
`mapped(func)`

### Environment swapping
`sudo([user=SUPERUSER])`  
`with_context([context][, **overrides]) → records`  
`with_env(env)`  

### Fields and views querying

### ???

### Automatic fields

### Reserved field names


<h2 id="Method decorators">Method decorators</h2>

`odoo.api.multi(method)`

`odoo.api.model(method)`

`odoo.api.depends(*args)`

`odoo.api.constrains(*args)`

`odoo.api.onchange(*args)`

`odoo.api.returns(model, downgrade=None, upgrade=None)`

`odoo.api.one(method)`

<h2 id="Fields">Fields</h2>

### Basic fields
`class odoo.fields.Field(string=<object object>, **kwargs)`  
所有字段类型的基础

##### Computed fields
##### Related fields
##### Company-dependent fields
##### Sparse fields
##### Incremental definition


### Relational fields
`class odoo.fields.Many2one(comodel_name=<object object>, string=<object object>, **kwargs)`

`class odoo.fields.One2many(comodel_name=<object object>, inverse_name=<object object>, string=<object object>, **kwargs)`

`class odoo.fields.Many2many(comodel_name=<object object>, relation=<object object>, column1=<object object>, column2=<object object>, string=<object object>, **kwargs)`


<h1 id="Data Files">Data Files</h1>

## Structure
Odoo中的数据文件之一是 XML，它的结构是
* 包含在 odoo 标签里的多个操作元素

```
<!-- the root elements of the data file -->
<odoo>
  <operation/>
  ...
</odoo>
```
## Core operations

#### record
定义或者更新数据库中的记录，有如下的属性：

* model（required）
* id
* context
* field
* ......

#### delete
删除数据库中的多条记录，有如下属性：

* model（required）
* id
* search

`id` 和 `search`二选一

#### function
#### workflow
这个标签会发送一个信号给指定的工作流。工作流的名字可以通过 ref 属性来指定。


## Shortcuts

有些重要的模型用起来比较麻烦，所以提供了一种简单的使用方式

#### menuitem
创建一个`ir.ui.menu`记录

#### template
创建一个QWeb模版

#### report
创建一个ir.actions.report.xml记录

## CSV data files
常用在设置模块的访问权限上

<h1 id="QWeb">QWeb</h1>

QWeb 是odoo使用的模版引擎。
模版命令是含有前缀 `t-` 的 xml 属性，例如`t-if`是条件表达式。
为了避免错误的代码生成，提供了一个标签 `<t>` ，这个标签会直接执行，而不生成任何的 html 代码。

例如：

```xml
<t t-if="condition">
    <p>Test</p>
</t>
```
会输出

```xml
<p>Test</p>
```
而

```xml
<div t-if="condition">
    <p>Test</p>
</div>
```
会输出

```xml
<div>
    <p>Test</p>
</div>
```

### data output
```
<p><t t-esc="value"/></p> 会对value里的值进行转码
<p><t t-raw="value"/></p> 输出value中的原始值
```

### conditionals
```
<div>
    <t t-if="condition">
        <p>ok</p>
    </t>
</div>

<div>
    <p t-if="user.birthday == today()">Happy bithday!</p>
    <p t-elif="user.login == 'root'">Welcome master!</p>
    <p t-else="">Welcome!</p>
</div>
```

### loops
```
<t t-foreach="[1, 2, 3]" t-as="i">
    <p><t t-esc="i"/></p>
</t>

<p t-foreach="[1, 2, 3]" t-as="i">
    <t t-esc="i"/>
</p>
```

### attributes

* `t-att-$name`

    ```
    <div t-att-a="42"/>
    ```
* `t-attf-$name`

    ```
    <t t-foreach="[1, 2, 3]" t-as="item">
        <li t-attf-class="row {{ item_parity }}"><t t-esc="item"/></li>
    </t>
    ```
* `t-att=mapping`
    
    ```
    <div t-att="{'a': 1, 'b': 2}"/>
    ```
* `t-att=pair`

    ```
    <div t-att="['a', 'b']"/>
    ```

### setting variables
```
<t t-set="foo" t-value="2 + 1"/>
<t t-esc="foo"/>

<t t-set="foo">
    <li>ok</li>
</t>
<t t-esc="foo"/>  会输出&lt;li&gt;ok&lt;/li&gt; 因为 t-esc 会进行转码
<t t-raw="foo"/>  会输出<li>ok</li>
```

### calling sub-templates
```
<t t-call="other-template">
    <t t-set="var" t-value="1"/>
</t>
<!-- "var" does not exist here -->
```

<h1 id="Views">Views</h1>

### Common Structure
视图对象有很多属性，它们都是可选的。

* name
* model
* priority
* arch
* ......



以下为Odoo中的视图种类

### Lists

创建列表视图的标签是 `<tree>`，它有如下属性：

* editable
* default_order
* create, edit, delete  
    通过对这些属性设置值`false`，可以禁止对应的事件
* on_write

可能出现的子元素

* button
    在表格上显示一个按钮
* field 
    用来定义模型的那些字段可以显示在列表里

### Forms

表单视图用来显示单条记录的数据。使用标签 `<form>`。

#### Structural components

结构组件用来提供展示结构。

* notebook
* group
* newline
* separator
* sheet
* header

#### Semantic components

Semantic 组件用来和 odoo 系统进行交互操作

* button
    同 list view 中的按钮类似
* field  
    显示当前记录中的一个字段，有以下属性
    * name
    * widget
    * options
    * class
    * groups
    * on_change
    * ......

#### Business Views guidelines

![](https://www.odoo.com/documentation/10.0/_images/oppreadonly.png)

一般来说业务视图由以下3部分组成：

1. 顶部的状态栏
2. 中间的表格
3. 底部的历史记录和评论

实现起来就像下边的 XML 一样
```
<form>
    <header> ... content of the status bar  ... </header>
    <sheet>  ... content of the sheet       ... </sheet>
    <div class="oe_chatter"> ... content of the bottom part ... </div>
</form>
```
##### 状态栏
状态栏用来显示当前记录的状态并提供一些操作按钮。

###### 按钮
按钮的顺序根据业务来决定。例如，对于销售来说，一般的顺序是：

1. 发报价单
2. 确认报价单
3. 创建最终的发货单
4. 发货

高亮的按钮用来提醒用户下一步是什么。一般放在最前边。而取消按钮（cancel）一般都是灰色的。
按钮的定义方式：

```
<button name="post" states="draft" string="Post" type="object" class="oe_highlight" groups="account.group_account_invoice"/>
<button name="%(action_view_account_move_reversal)d" states="posted" string="Reverse Entry" type="action" groups="account.group_account_invoice"/>
<button name="button_cancel" states="posted" string="Cancel Entry" type="object" groups="account.group_account_invoice"/>
```

###### 状态
使用 `statusbar` 插件，当前的状态显示红色。
显示的状态用属性 `statusbar_visible` 来定义。

```
<field name="state" widget="statusbar"
    statusbar_visible="draft,sent,progress,invoiced,done" />
```

#### 表格
##### 表头
##### 按钮区
##### 分组和标题



### Graphs

### Kanban

### Calendar

### Gantt

### Diagram

### Search
......