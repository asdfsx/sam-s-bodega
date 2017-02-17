+++
title = "odoo的阶段性总结：服务器的启动、模块的加载、请求处理、页面生成"
tags = [
  "python",
  "odoo",
]
author = "asdfsx"
type = "post"
date = "2017-01-05T15:32:00+08:00"
topics = [
  "topic 1",
]
keywords = [
  "python",
  "odoo",
]
description = "odoo的阶段性总结：服务器的启动、模块的加载、请求处理、页面生成"
draft = false

+++


经过漫长的阅读代码，搞清了启动的过程。先简单做个总结。如有遗漏之后再做补充。
### 系统的启动，模块的加载
结合之前研究过的`registry`，系统启动时会发生如下动作：

* 首先加载全局模块`web`，`web_kanban`。（在没有确定数据库地址之前，只能显示数据库选择页面。所以这个时候只需要这两个模块就可以了）
    * 在`import controller`时，由于元类的作用，`controller`类会自动加载到解释器中
* 根据配置创建web服务器（线程的、进程的），所有的服务器都使用`odoo.service.wsgi_server.application`来处理请求。
    * 具体处理请求的是`odoo.http.Root`
        * 根据需要看要不要再次加载插件
        * 只有当首次接收请求的时候，才会执行加载插件
* 启动服务器
    * 在启动web服务器之前，首先创建`registry`。（在进程的实现中，registry会在进程`fork`之前创建，`fork`之后`registry`会被拷贝到各个进程的内存空间中）
    * 当数据库选定之后，`registry`会根据配置去加载模块
        * 先加载`base`模块
            * 先创建`base`模块的依赖关系图`graph`
            * 使用`graph`加载`base`
                * 获取模块中的所有模型
                * 组装配置模型类
                    * 根据模型的属性，创建新的模型类，并将模型类注册到`registry`中
                    * 根据模型的属性，为新的模型类添加字段，关联关系等
                * 初始化模型
                    * 根据模型类，创建模型类对应的表
                * 装载定义在`__manifast__.py`中的模块的数据
                    * 获取文件列表
                    * 调用`odoo.tools.convert.convert_file`装载文件。
                        * 判断文件类型，根据文件类型使用不同的方法解析
                        * 根据情况将数据写入`ir.model.data`
                        * 根据情况将数据写入模型自己的数据表中
        * 根据配置标注其它需要加载的模块
        * 根据标注加载模块
            * 使用`graph`进行模块的记载（下边以`web`模块为例，看一下模块数据的加载）
                * 获取数据文件：`views/webclient_templates.xml`
                * 调用`odoo.tools.convert.convert_file`装载文件
                    * 创建`xml`专用的解析器对象`xml_import`
                    * 解析`xml`文件
                        * 遍历整个`xml`文档树，根据节点的类型调用不同的函数来进行处理。具体到`views/webclient_templates.xml`，这个文件由`template`组成，对应的函数是`_tag_template`。这个函数在结尾调用`_tag_record`，这个函数会将数据文件里的内容写入`ir_model_data`表中。
    * 模块加载完毕后，服务器开始运行。等待处理请求。

### 请求处理

* 首次接受到请求
    * `odoo.http.Root`根据请求创建`JsonRequest`或者`HttpRequest`
    * 从`Registry`中查找`ir.http`模型
    * 使用`ir.http`模型`_dispatch`转发请求
        * `_dispatch`查询路由表
            * 如果路由表不存在，使用`odoo.http.routing_map`创建路由表
            * 使用路由表获取处理请求的具体`controller`
            * 校验用户
            * 将`controller`放入请求对象中
            * 调用`request.dispatch`处理请求
    * 在`request.dispatch`函数中（以HttpRequest为例），对请求参数、请求头、等进行校验，然后通过调用`_call_function`来调用之前放入`request`的`controller`。
    * 如果`controller`中的处理函数设置了延迟生成页面，则在`dispatch`结束的时候生成页面。

### 页面生成
请求处理完成后，要生成显示的页面。以入口地址为例：

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

* 请求的处理，之前已经提到过了。`request.render`函数里，会创建`Response`对象。不管是否使用延迟创建页面，最终是通过`response.render`来生成页面。
* `response.render`会用`ir.ui.view`模型来根据模版来创建页面。具体到`/web`请求的处理，就是使用`ir.ui.view`模型的`render_template`函数，利用`web.webclient_bootstrap`模版来创建页面。
    * 先用`ir.ui.view`模型的`get_view_id`函数中，先在`ir.model.data`模型中查找模版：根据模版的名字，获取`res_id`：
        ```
        select ir_model_data.id from ir_model_data where module='web' and name='webclient_bootstrap'
        ```
    * 使用基类的`browse`函数，创建一个`View`对象，保存`res_id`
    * 调用`View`的`render`函数
        * 调用`ir.qweb`的`render`函数
            * 调用`QWeb`的`render`函数
                * 使用`ir.qweb`的`load`读入模版
                    * 使用`ir.ui.view`中的`read_template`读入模版
                        * 从`ir_ui_view`表中获取`arch_fs`信息，然后读取对应的文件作为`arch`。
                * 调用`ast`对模版进行处理，生成一个`python`函数
                * 用生成的函数生成最终的页面

通过访问`/web`只是获得了页面的框架、样式表、js等静态资源。然后`odoo`会通过自己实现的一套`js`框架，从服务器获取展示所需的其它部分绘制到页面上。

接下来的部分基于对前端的一些猜测：
当浏览器打开由模版生成的页面后，将页面上相关的静态资源下载到本地。其中的`js`加载到浏览器后，会向服务器发起请求，来获取当前页面上要的元素：

* /web/action/load  
    获取要展示模块的行为  
    json请求：
    
    ```
    {jsonrpc:"2.0",method="call",params:{action_id:261}}
    ```
    服务器会从`ir_action`中查询`action_id`对应的记录，并返回`action`相关的信息
    
    ```
    { jsonrpc:"2.0",
    result:{ ...
        xml_id:"qingjia.action_qingjia_qingjd",
        ...
        res_model:"qingjiaj.qingjd",
        search_view:"{'name':'default','arch':....}",
        ..
        }
    }
    ```
    odoo的js框架会根据收到的数据，在页面上添加响应的元素（如按钮）
    
* /web/dataset/call_kw/qingjia.qingjd/load_views  
    加载模块展示需要用到的view
    json请求：
    
    ```
    {jsonrpc:"2.0", method:"call", params:{
    model:"qingjia.qingjd",
    method:"load_views",
    kwargs:{
    views:[[null, "list"],[null,"from"],[false,"search"]],
    ...
    }
    }}
    ```
    服务器会服务器返回结果：
    
    ```
    {jsonrpc:"2.0"
         result:{
             fields:{
                 id:{...},
                 create_date:{...},
                 ...
             },
             fields_views:{
                 form:{},
                 list:{},
                 search:{},
             },
             ...
         }
    }
    ```
    可以看到在一次请求中，获得了3种view。这些结果会缓存在浏览器，odoo的js框架之后就不需要反复的去获取view。同时js框架会根据以上内容在页面上生成相应的内容。  
    
......