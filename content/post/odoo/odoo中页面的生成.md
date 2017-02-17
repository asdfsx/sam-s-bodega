+++
date = "2017-01-09T17:40:52+08:00"
title = "odoo中页面的生成"
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
description = "odoo中页面的生成"
author = "asdfsx"
type = "post"
draft = false

+++

在请求的处理中，已经知道在请求处理的最后，会调用`Response`的`render`来生成页面。这里来研究下页面是如何形成的。

以入口地址的处理为例：

* 处理`/`请求的`controller`为`addon.web.controllers.main.Home`，定义在`web`模块中，处理方式是直接跳转到`/web`。  
* 处理`/web`请求的`controller`同上，使用`web.webclient_bootstrap`这个模版来生成页面。

    ```
    class Home(http.Controller):
        @http.route('/', type='http', auth="none")
        def index(self, s_action=None, db=None, **kw):
            return http.local_redirect('/web', query=request.params, keep_hash=True)
        
        @http.route('/web', type='http', auth="none")
        def web_client(self, s_action=None, **kw):
            ensure_db()
            if not request.session.uid:
                return werkzeug.utils.redirect('/web/login', 303)
            if kw.get('redirect'):
                return werkzeug.utils.redirect(kw.get('redirect'), 303)

            request.uid = request.session.uid
            context = request.env['ir.http'].webclient_rendering_context()

            return request.render('web.webclient_bootstrap', qcontext=context)
```

在介绍页面生成之前，先熟悉下可能会用到的模型

### 模型 `odoo.addons.base.ir.ir_ui_view.View`

在`response.render`函数中，需要用到`ir.ui.view`模型来生成页面。下边就是这个模型的定义。从中我们可以知道在数据库中一定会有一张表`ir_ui_view`，同时这个模型会在`ir_model`注册，模型的字段会在`ir_model_fields`中记录。其中一个特殊的字段是`type`，从定义猜测`View`的种类就是列表中的那几种。

`View`模型还提供了如下功能：

* 根据模版生成页面 `render_template`（仅仅算是入口函数）
* 查询模版ID `get_view_id`

    ```
    select ir_model_data.id from ir_model_data where module='web' and name='webclient_bootstrap'
    ```
    
* 模版读取 `read_template`、`read_template`、`read_combined`
* 使用模版引擎`render`生成页面，`render`。默认使用`ir_qweb`。参数中的`self.id`，通过之前对模型的研究，它就是刚才的`res_id`。

```
class View(models.Model):
    _name = 'ir.ui.view'
    _order = "priority,name,id"
    ...
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
    arch = fields.Text(compute='_compute_arch', inverse='_inverse_arch', string='View Architecture', nodrop=True)
    ...
    
    @api.model
    def get_view_id(self, template):
        ...
        return self.env['ir.model.data'].xmlid_to_res_id(template, raise_if_not_found=True)
    
    @api.model
    def render_template(self, template, values=None, engine='ir.qweb'):
        return self.browse(self.get_view_id(template)).render(values, engine)
        
   def _read_template(self, view_id):
        arch = self.browse(view_id).read_combined(['arch'])['arch']
        arch_tree = etree.fromstring(arch)
        self.distribute_branding(arch_tree)
        root = E.templates(arch_tree)
        arch = etree.tostring(root, encoding='utf-8', xml_declaration=True)
        return arch

    @api.model
    def read_template(self, xml_id):
        return self._read_template(self.get_view_id(xml_id))
        
    @api.multi
    def read_combined(self, fields=None):
         """
        Utility function to get a view combined with its inherited views.
        * Gets the top of the view tree if a sub-view is requested
        * Applies all inherited archs on the root view
        * Returns the view with all requested fields
          .. note:: ``arch`` is always added to the fields list even if not
                    requested (similar to ``id``)
        """
        ...
    ...
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
    ...
```


当模型表创建好后，就需要往里添加数据。之前研究`Registry`的时候，已经知道在加载模块的时候，模型的数据文件会写入到自己的数据表，以及`ir_model_data`表。以`web`模块来说，它的数据文件`webclient_templates.xml`中有很多模版，都会保存到数据库中。

### 模型 `odoo.addons.base.ir.ir_qweb.ir_qweb.IrQWeb`
这也是一个只提供功能而不提供数据存储的模型，主要负责模版的读取、页面的生成。它的父类`QWeb`提供了绝大部分的模版处理函数，在子类中只需要根据需要覆盖个别函数就可以了。感觉子类存在更重要的意义在于将模版引擎也当作模型加载到`registry`中。

```

class IrQWeb(models.AbstractModel, QWeb):
    _name = 'ir.qweb'

    @api.model
    def render(self, id_or_xml_id, values=None, **options):
        ...
        return super(IrQWeb, self).render(id_or_xml_id, values=values, **context)
    ...    
    
    def compile(self, id_or_xml_id, options):
        return super(IrQWeb, self).compile(id_or_xml_id, options=options)

    def load(self, name, options):
        ...   
        template = env['ir.ui.view'].read_template(name)
        ... 
    
```

### 模版引擎 `QWeb`

模型`ir.qweb`的父类，定义了模版处理的大部分行为。

```
class QWeb(object):
    def render(self, template, values=None, **options):
        body = []
        self.compile(template, options)(self, body.append, values or {})
        return u''.join(body).encode('utf8')
    
    def compile(self, template, options):
        ...
        element, document = self.get_template(template, options)
        ...
        def _compiled_fn(self, append, values):
            log = {'last_path_node': None}
            values = dict(self.default_values(), **values)
            try:
                return compiled(self, append, values, options, log)
            except QWebException, e:
                raise e
            except Exception, e:
                path = log['last_path_node']
                element, document = self.get_template(template, options)
                node = element.getroottree().xpath(path)
                raise QWebException("Error to render compiling AST", e, path, node and etree.tostring(node[0]), name)

        return _compiled_fn
        
    ...
    
    def get_template(self, template, options):
        ...
        document = options.get('load', self.load)(template, options)
        ...
        if document is not None:
            if isinstance(document, etree._Element):
                element = document
                document = etree.tostring(document)
            elif document.startswith("<?xml"):
                element = etree.fromstring(document)
            else:
                element = etree.parse(document).getroot()
            for node in element:
                if node.get('t-name') == str(template):
                    return (node, document)

        raise QWebException("Template not found", name=template)
```


### 模版的获取

* `response.render`会用`ir.ui.view`模型来根据模版来创建页面。具体到`/web`请求的处理，就是使用`ir.ui.view`模型的`render_template`函数，利用`web.webclient_bootstrap`模版来创建页面。

    ```
    class Response(werkzeug.wrappers.Response):

        ...

        def render(self):
            ...
            return env["ir.ui.view"].render_template(self.template, self.qcontext)
    
    ```

* 回到模型`ir.ui.view`，`render_template`函数首先使用模型的名字获得模版在`ir.model.data`中保存的`res_id`，也就是模版在`ir_ui_view`中的主键。对应的sql：

    ```
    select ir_model_data.id from ir_model_data 
    where ir_model_data.module='web' and 
          ir_model_data.name='webclient_bootstrap' 
    order by ir_model_data.module, ir_model_data.name
    ```
    然后用`browse`创建一个`ir_ui_view`对象。然后调用`ir.ui.view`模型的`render`函数生成页面。这个函数默认使用`ir.qweb`引擎的`render`函数来生成页面。
    
    ```
    @api.multi
    def render(self, values=None, engine='ir.qweb'):
        ...
        return self.env[engine].render(self.id, qcontext)
    ```
    
### 模版编译

* 模版引擎`ir.qweb`  
    当`ir.ui.view`中获得模版id以后，使用父类`qweb.render`生成页面。
    * 首先编译模版
        * 读取模版 `get_template`  
            * 从`options`中检查是否有`load`函数，如果没有用自己的`load`函数。注意：因为真正在使用的其实是`ir.qweb`模型，所以这里的`load`函数，其实是被子类`ir.qweb`覆盖过的`load`函数。  
            * 在子类`ir.qweb`中，`load`函数会调用`ir.ui.view`模型的`_read_template`来获取模版。先根据`view_id`获取当前`view`的对象，然后调用`read_combined`读取该对象的`arch`字段。
            * 在`read_combined`中，有一句`arch = self.browse(view_id).read_combined(['arch'])['arch']`，它调用`read_combined`来读取模型的`arch`字段。  
            `arch`是个很神奇的字段。它的定义是`arch = fields.Text(compute='_compute_arch', inverse='_inverse_arch', string='View Architecture', nodrop=True)`，其中`compute`说明这个字段是经过计算才能获得值，而这个`compute`的值就是用来计算的函数--从文件中读取内容，然后放入`arch`中。
            
                ```
                @api.depends('arch_db', 'arch_fs')
                def _compute_arch(self):
                    ...
                    for view in self:
                        arch_fs = None
                        if 'xml' in config['dev_mode'] and view.arch_fs and view.xml_id:
                            fullpath = get_resource_path(*view.arch_fs.split('/'))
                            arch_fs = get_view_arch_from_file(fullpath, view.xml_id)
                            arch_fs = arch_fs and resolve_external_ids(arch_fs, view.xml_id)
                        view.arch = arch_fs or view.arch_db
                ```
        * 模版读到以后，使用`ast`生成一个`python`的语法树，将模版等信息加入其中，最终生成一个`python`函数
        
             ```
             element, document = self.get_template(template, options)
             
             ...
             
             astmod = self._base_module()
             
             ...
             
             body = self._compile_node(element, _options)
             ast_calls = _options['ast_calls']
             _options['ast_calls'] = []
             def_name = self._create_def(_options, body, prefix='template_%s' % name.replace('.', '_'))
             _options['ast_calls'] += ast_calls
             
             ...
             
             astmod.body.extend(_options['ast_calls'])
             
             ...
             
             ns = {}
             unsafe_eval(compile(astmod, '<template>', 'exec'), ns)
             compiled = ns[def_name]           
             ```
    * 返回一个函数`_compiled_fn`，这个函数使用了之前`ast`创建的函数
    
        ```
        def _compiled_fn(self, append, values):
           ...
           return compiled(self, append, values, options, log)
           ...
           
        return _compiled_fn
        ```

这个基于`ast`的模版引擎，个人感觉最大的好处就是，可以将系统中注册的各种资源自由的组合到一起。

最终`/web`获得了由`web.webclient_bootstrap`生成的页面。但是通过对模版以及页面的分析发现，这时浏览器只获得了页面的框架、css、js。而模块中定义的视图则是由`odoo`自己定义的一套javascript引擎，从服务器获取后绘制到页面上的。