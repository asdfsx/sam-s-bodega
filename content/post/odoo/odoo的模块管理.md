+++
date = "2016-12-27T18:05:15+08:00"
title = "odoo的模块管理"
author = "asdfsx"
keywords = [
  "odoo",
  "python",
]
description = "odoo的模块管理"
draft = false
type = "post"
tags = [
  "odoo",
  "python",
]
topics = [
  "topic 1",
]

+++

之前一篇记录了odoo web server的大概情况，以及简单的启动流程、模块加载的情况。深入研究会发现odoo的所有功能都是基于模块制作的，所以本篇开始研究odoo的模块。首先研究下模块是如何进行管理的。负责管理模块的代码主要存放在`odoo.modules`包里。

### 模型的注册机 odoo.modules.registry.Registry

`Registry`的用途是存放模型名、模型类对应关系。是一个深度定制化的类。注意，`model`、`module`的区别。

* 此类继承了`collections.Mapping`，因此它的对像可以按照字典方式来使用
* 这个类也是一个自产自销的类：创建自己的实例，然后将实例放到自己的类属性中。
* 类函数／属性`registries`用来存放已创建的`Registry`。
    装饰器`lazy_classproperty`是一个神器的东西，它把一个函数变成了一个属性：当函数第一次执行时，获得函数的返回值，然后将返回值设置为类中的一个属性（注意那个`setattr`函数）。`__get__`方法会在Registry.registry的时候执行。
    
    ```
    class lazy_property(object):
        def __init__(self, fget):
            self.fget = fget
            
    class lazy_classproperty(lazy_property):
        def __get__(self, obj, cls):
            val = self.fget(cls)
            setattr(cls, self.fget.__name__, val)
            return val
    ```
* 使用`__new__`而不是`__init__`来创建对象：首先尝试从`registries`中获取已经生成的对象，失败后创建新对象。之前的python的数据结构中曾经探讨过`__init__`和`__new__`的区别。
* `Registry`对象在`new()`里生成：手动生成、手动初始化、存放到`registries`中、加载所有的模块到`self.models`字典中（`odoo.modules.load_modules`函数）。在加载模块的过程中，还需要模块中导出模型（load函数）、完善模型（setup_models），根据模型建表、建约束（init_models）。
* 注意LRU是Least Recently Used 近期最少使用算法。这里是一个python实现的功能模块。内部是一个字典。

    ```
    class Registry(Mapping):
        @lazy_classproperty
        def registries(cls):
            """ A mapping from database names to registries. """
            size = config.get('registry_lru_size', None)
            ...
        return LRU(size)
        
        def __new__(cls, db_name):
            """ Return the registry for the given database name."""
            with cls._lock:
                try:
                    return cls.registries[db_name]
                except KeyError:
                    return cls.new(db_name)
                finally:
                    # set db tracker - cleaned up at the WSGI dispatching phase in
                    # odoo.service.wsgi_server.application
                    threading.current_thread().dbname = db_name
                
        ......
    
        @classmethod
        def new(cls, db_name, force_demo=False, status=None, update_module=False):
            ...
            registry = object.__new__(cls)
            registry.init(db_name)
            cls.delete(db_name)
            cls.registries[db_name] = registry
            ...
            odoo.modules.load_modules(registry._db, force_demo, status, update_module)
            ...
    ```

* 从模块中获取模型`load`  
    从模块中，获取所有的模型类，然后调用`_build_model`来完善这个类，完善后的模型类也保存到了模型类的基类中，又一次完成了自产自销。然后返回模块中所有的模型名字。模型的名字就是模型类中的`_name`属性。
    
    ```
    def load(self, cr, module):
        model_names = []
        for cls in models.MetaModel.module_to_models.get(module.name, []):
            # models register themselves in self.models
            model = cls._build_model(self, cr)
            model_names.append(model._name)

        return self.descendants(model_names, '_inherit')
    ```
    
    注意`_build_model`函数，它是所有模型的基类`odoo.models.BaseModel`中的函数。其中`self`参数对应下边的`pool`参数。所以`model`会在`_build_model`的时候保存到`registry`中。
    
    ```
    @classmethod
    def _build_model(cls, pool, cr):
        ...
        ModelClass.pool = pool
        pool[name] = ModelClass
        ...
    ```
* 模型安装`setup_models`  
    根据配置信息、继承信息等从基类中创建新的模型类。通过对`model`的学习下边几个`model`的函数的功能是：
      * `_build_model` 根据模型的信息，创建新的模型类，新的模型类会放在`registry`中
      * `_prepare_setup` 
      * `_setup_base` 安装基础属性，如主键
      * `_setup_fields` 安装各个字段
      * `_setup_complete` 安装`trigger`之类的东西，完成安装
    
    ```
    cr.execute('SELECT * FROM ir_model WHERE state=%s', ('manual',))
    for model_data in cr.dictfetchall():
        model_class = ir_model._instanciate(model_data)
        model_class._build_model(self, cr)
    
    models = env.values()
    for model in models:
        model._prepare_setup()
        
    for model in models:
        model._setup_base(partial)
        
    for model in models:
        model._setup_fields(partial)
        
    for model in models:
        model._setup_complete()
    ```
    
* 模型初始化`init_models`  
    将完善后的模型信息，保存到数据库`ir_model`、`ir_model_data`、`ir_model_field`等字段里。并根据模型建表、建索引、建关联约束。
    
    ```
    model._auto_init()  // 基类中已实现，用来保存信息、建表的。
    model.init()        // 子类中实现，用来建新表，或者修改表
    model._auto_end()   // _auto_init 执行完后，建立表间的外键约束。
    ```

### 模块的依赖关系 odoo.modules.graph.Graph

`odoo`的模块之间可以互相调用，那么就存在一定的依赖关系。这些依赖关系都保存在`__manifest__.py`中。这个模块就是通过数据库、`__manifest__.py`文件，将模块之间的依赖关系转化为一颗树状图来存储（希望这样表述没有错）。

* 每个模块都是一个节点
* 依赖模块是当前节点的子节点

以下是加载过程中生成的一部分树结构

```
base
`-> auth_crypt
`-> web
   `-> web_calendar
   `-> web_diagram
   `-> web_editor
   `-> web_kanban
      `-> base_import
      `-> wb_kanban_gauge
   `-> web_planner
      `-> web_settings_dashboard
   `-> web_tour
```

### 模块的加载 odoo.modules.loading.load_modules

定义了如何导入模块

* 处理模块的查找路径  
    调用`odoo.modules.module.initialize_sys_path`函数：
      * 根据配置、系统安装路径等获取所有的插件安装路径
      * 调用`sys.meta_path.append`定制插件导入方式（主要目的是让odoo和openerp的插件能互相支持）

* 检查数据库是否创建  
    调用`odoo.modules.db`下的函数，检查初始化数据库。
      * 使用建库脚本`odoo/addons/base/base.sql`创建表
      * 将模块信息写入表`ir_module_module`、`ir_model_data`、`ir_module_category`等表。
      
* 创建一个`registry`对象
    由于`registry`自产自销的特点，所以并不是一定要将`registry`对象返回。之后会使用这个`registry`对象进行模块的安装操作。

* 获取运行环境  
    `env = api.Environment(cr, SUPERUSER_ID, {})`
    
    `Enviroment`模拟了一个字典。和Registry类似，也是一个自产自销的类。第一次创建好对象后，会放在类的属性中。
    
    ```
    class Environment(Mapping):
        _local = Local()

        @classproperty
        def envs(cls):
            return cls._local.environments
    
        ...
        
        def __new__(cls, cr, uid, context):
            ...
            envs.add(self)
    ...
    ```
    
* 导入base模块  
    `base`模块提供了odoo的最基础功能。在该模块的`__manifest__.py`中对自己的描述是`The kernel of Odoo, needed for all installation.`，可见它的重要性。
      * 创建依赖管理器`odoo.modules.graph.Graph`，并将`base`加入其它  
      
          ```
          graph = odoo.modules.graph.Graph()
          graph.add_module(cr, 'base', force)
          ```
          
      * 加载`base`模块。注意这里是通过`graph`进行的加载。

          ```
          loaded_modules, processed_modules = load_module_graph(cr, graph, status, perform_checks=update_module, report=report)
          ```
          
          在`load_module_graph`函数中进行如下操作：

          * 遍历graph中所有的模块，抽取模块中的模型，从模型的基类中，创建新的模型类。
          
              ```
              model_names = registry.load(cr, package)
              registry.setup_models(cr, partial=True)
              registry.init_models(cr, model_names, {'module': package.name})
              ```
          * 根据需要安装视图、demo等

              ```
              if hasattr(package, 'init') or hasattr(package, 'update') or package.state in ('to install', 'to upgrade'):
              ...
                  _load_data(cr, module_name, idref, mode, kind='data')
                  ...
                  env['ir.ui.view']._validate_module_views(module_name)
                  ...
              ...
              ```
      * 然后在`registry`中安装`base`里的模型。
          
          ```
          registry.setup_models(cr, partial=True)
          ```

* 标记其它需要加载或更新的模块  
    如果有需要更新的模块，进行标注。这里的标注貌似只是在模块的按钮上显示安装、更新按钮。
    
* 导入标记过的模块  
    安装除base以外的模块。  
      * 从数据库中查询特定状态的模块，例如：`installed`。
      * 将它们放入graph中，通过graph导入
    
* 结束安装并进行清理  
    从数据库查询所有应该导入的模块，检查它们的状态，执行相应的清理。因为有些模块在安装过程中，可能会导入一些测试数据进行单元测试，安装完成以后这些东西应该被清理。
    
* 卸载标记删除的模块
    从数据库中查询所有标记为要删除的模块。从graph中获取所有的依赖，然后从中查找`uninstall_hook`函数，利用`getattr`来调用这些函数
    
    ```
    pkgs = reversed([p for p in graph if p.name in modules_to_remove])
    for pkg in pkgs:
        uninstall_hook = pkg.info.get('uninstall_hook')
        if uninstall_hook:
            py_module = sys.modules['odoo.addons.%s' % (pkg.name,)]
            getattr(py_module, uninstall_hook)(cr, registry)
    ```
* 检查所有模型的视图

    ```
    View._validate_custom_views(model)
    ```
    研究视图的时候再研究这个函数
* 调用每个模块的`_register_hook`

    ```
    for model in env.values():
        model._register_hook()
    ```
    研究模块、模型的时候再详细研究这个`hook`
* 执行`post-install`
    从数据库中查询所有的模块，然后逐个进行单元测试



### 模块的加载 odoo.modules.loading.load\_module\_graph
按照模块之间的依赖关系导入模块。这个是最终将模块、模型、模型数据导入到registry中的函数。其中内部定义的函数`_load_data`，负责将`__manifest__.py`中的数据文件保存到数据库中。具体负责读取不同格式的数据文件的函数是`odoo.tools.convert.convert_file`。
它会根据不同类型的文件类型使用不同的方法解析数据文件，并将数据保存到自己的模型表中，然后保存到`ir_model_data`中。

```
def load_module_graph(cr, graph, status=None, perform_checks=True, skip_modules=None, report=None):
    ...
    def _load_data(cr, module_name, idref, mode, kind):
        ...
        for filename in _get_files_of_kind(kind):
            ...
            tools.convert_file(cr, module_name, filename, idref, mode, noupdate, kind, report)
            ...
    ...
    processed_modules = []
    loaded_modules = []
    registry = odoo.registry(cr.dbname)
    ...
    for index, package in enumerate(graph, 1):
        load_openerp_module(package.name)
        ...
        model_names = registry.load(cr, package)
        ...
        if hasattr(package, 'init') or hasattr(package, 'update') or package.state in ('to install', 'to upgrade'):
            registry.setup_models(cr, partial=True)
            registry.init_models(cr, model_names, {'module': package.name})
            ...
            _load_data(cr, module_name, idref, mode, kind='data')
            ...
            registry._init_modules.add(package.name)
        ...
    ...

```

`tools.convert_file`中以及`ir.model`中的代码片段

```
class xml_import(object):
    ...
    def _tag_template(self, el, data_node=None, mode=None):
        ...
        record_attrs = {
            'id': tpl_id,
            'model': 'ir.ui.view',
        }
        ...
        record = etree.Element('record', attrib=record_attrs)
        ...
        return self._tag_record(record, data_node)
    
    def _tag_record(self, rec, data_node=None, mode=None):
        ...
        id = self.env(context=rec_context)['ir.model.data']._update(rec_model, self.module, res, rec_id or False, not self.isnoupdate(data_node), noupdate=self.isnoupdate(data_node), mode=self.mode)
        ...
        return rec_model, id
        
...

class IrModelData(models.Model):
    ...
    @api.model
    def _update(self, model, module, values, xml_id=False, store=True, noupdate=False, mode='init', res_id=False):
        record = self.env[model].browse(res_id)
        ...
        record = record.create(values)
        ...
        self.sudo().create(...)
        ...
```
`record.create`将数据写入模型自己的表中，`sudo().create`将数据写入`ir.model.data`中。

### 不算总结的总结

通过学习这部分的代码，可以发现模块的加载、更新、删除等操作，都有数据库的深度参与。模块的物理信息会存放在`ir_module_module`等表中。从模块中抽取出来的逻辑信息则会存放在`ir_model`等表中。对模块的操作需要反复的遍历、修改这些表。__所以在使用过程中，数据库的安全、稳定是头等大事__。