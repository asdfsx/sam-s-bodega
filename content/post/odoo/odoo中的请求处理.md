+++
title = "odoo中的请求处理"
tags = [
  "python",
  "odoo",
]
topics = [
  "topic 1",
]
date = "2016-12-29T15:57:59+08:00"
keywords = [
  "python",
  "odoo",
]
description = "odoo中的请求处理"
author = "asdfsx"
draft = false
type = "post"

+++

# Web请求

web请求的包装是在接收请求时，在`odoo.http.Root`中处理的。请求主要分为

* json请求：主要用来处理json请求、rpc请求。
* http请求：主要用来处理页面访问请求。

### 请求的基类：odoo.http.WebRequest
所有请求的基类，定义了请求处理过程中都可能会用到的一些属性：如csrf、db、registry、session等等。

同时WebQuest还使用了`__enter__`、`__exit__`。这样当使用`with request：`这样当表达式时，会将当前`request`放到`werkzeug.local.LocalStack`中。方便从任何地方使用`odoo.http.request`获取当前请求。

具体处理请求的`endpoint`是通过`set_handler`传入的。`_call_function`会调用`endpoint`来获得返回结果。但是调用`_call_function`的`dispath`是由子类来实现的，基类中没有。

```
_request_stack = werkzeug.local.LocalStack()

request = _request_stack()

class WebRequest(object):
    ...
    @property
    def registry(self):
        return odoo.registry(self.db) if self.db else None
    
    @property
    def db(self):
        return self.session.db if not self.disable_db else None
    
    def csrf_token(self, time_limit=3600):
        ...
    
    ...
    def _call_function(self, *args, **kwargs):
        """ Generates and returns a CSRF token for the current session
        ...
        
    def validate_csrf(self, csrf):
         ...
         
     def __enter__(self):
        _request_stack.push(self)
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        _request_stack.pop()
        ...
    
    def set_handler(self, endpoint, arguments, auth):
        # is this needed ?
        arguments = dict((k, v) for k, v in arguments.iteritems()
                         if not k.startswith("_ignored_"))
        self.endpoint_arguments = arguments
        self.endpoint = endpoint
        self.auth_method = auth
     
    def _call_function(self, *args, **kwargs):
        ...
        return self.endpoint(*args, **kwargs)
    ...
```

### Json请求：odoo.http.JsonRequest
默认接收到的数据是json格式，所以在创建这个对象时，会按照`json`的格式来读取请求数据

```
class JsonRequest(WebRequest):
    def __init__(self, *args):
        ...
        self.jsonrequest = json.loads(request)
        ...
        self.params = dict(self.jsonrequest.get("params", {}))
        ...
    ...
```

### Http请求：odoo.http.HttpRequest
在创建http请求时，从`url`参数中、`form`中、甚至上传的文件中获取请求信息。然后定义了`dispatch`函数，这个函数负责创建并返回`Response`对象。它通过调用基类中的`_call_function`，调用从路由器得来的controller来获取`Response`。
请求处理完成后，还要找到对应的模版，生成显示的页面（也可以根据`lazy`参数延迟生成页面）。

```
class HttpRequest(WebRequest):
    ...
    def __init__(self, *args):
        super(HttpRequest, self).__init__(*args)
        params = collections.OrderedDict(self.httprequest.args)
        params.update(self.httprequest.form)
        params.update(self.httprequest.files)
        params.pop('session_id', None)
        self.params = params

    ...

    def dispatch(self):
        ...
        r = self._call_function(**self.params)
        if not r:
            r = Response(status=204)  # no content
        return r
    
    ...
    
    def render(self, template, qcontext=None, lazy=True, **kw):
        response = Response(template=template, qcontext=qcontext, **kw)
        if not lazy:
            return response.render()
        return response
    ...
```

### Http响应：odoo.http.Response
这个只对应`http`请求，主要的功能就是页面的创建

```
class Response(werkzeug.wrappers.Response):
    ...
    def render(self):
        env = request.env(user=self.uid or request.uid or odoo.SUPERUSER_ID)
        self.qcontext['request'] = request
        return env["ir.ui.view"].render_template(self.template, self.qcontext)
    ...
    
    def flatten(self):
        """ Forces the rendering of the response's template, sets the result
        as response body and unsets :attr:`.template`
        """
        if self.template:
            self.response.append(self.render())
            self.template = None
```

# 控制器

对于一个有大量插件的项目来说，大量的路径管理是个麻烦事。不过`odoo`有自己的解决之道，通过自己定义的一套`Controller`机制。

### Controller的元类：odoo.http.ControllerType

`Controller`的元类。如果一个类使用了这个元类的话，会自动存放到全局字典`controllers_per_module`中。同时为了兼容老版本，会检查`Controller`中的函数是否有`original_func`属性。如果有，会给该函数增加路由属性。


```
controllers_per_module = collections.defaultdict(list)

class ControllerType(type):
    def __init__(cls, name, bases, attrs):
        ...
        name_class = ("%s.%s" % (cls.__module__, cls.__name__), cls)
        ...
        controllers_per_module[module].append(name_class)
```

### Controller的基类：odoo.http.Controller

所有Controller的父类，可以看到使用了上边创建的元类。

```
class Controller(object):
    __metaclass__ = ControllerType
    
```

### 例子：WEB 模块的 Controller

我们以`server-wide`模块`web`为例子，看看`Controller`的具体使用。  
当第一次访问`odoo`时，请求由`index`函数处理，被重定向到`/web`。
当访问`/web`时，会用`web.webclient_bootstrap`模版来生成页面。

```
class Home(http.Controller):
    @http.route('/', type='http', auth="none")
    def index(self, s_action=None, db=None, **kw):
        return http.local_redirect('/web', query=request.params, keep_hash=True)
    
    @http.route('/web', type='http', auth="none")
    def web_client(self, s_action=None, **kw):
        ...
        return request.render('web.webclient_bootstrap', qcontext=context)
    
    @http.route('/web/dbredirect', type='http', auth="none")
    def web_db_redirect(self, redirect='/', **kw):
        ensure_db()
        return werkzeug.utils.redirect(redirect, 303)
        
    @http.route('/web/login', type='http', auth="none")
    def web_login(self, redirect=None, **kw):
        ...
        return request.render('web.login', values)
```

* 首先继承`http.Controller`，直接注册到`controllers_per_module`中
* 用来处理请求的都使用装饰器`http.route`
    包装以后，生成一个新函数`response_wrap`，包含两个属性：
      * `routing` 存放路由相关的信息，如：请求类型、路由地址等。  
          之前在`http.routing_map`函数中，依靠这个rouging获取路由的各个信息
      * `original_func` 存放原始的函数。（这个怀疑是为了兼容旧版本）
    
    ```
    def route(route=None, **kw):
        ...
        routing = kw.copy()
        def decorator(f):
            ...
            @functools.wraps(f)
            def response_wrap(*args, **kw):
                response = f(*args, **kw)
                ...
                return response
            response_wrap.routing = routing
            response_wrap.original_func = f
            return response_wrap
        return decorator
    ```
    
* 请求处理完后返回结果，通常是下一个要显示的页面
    * `http.local_redirect`进行重定向  
        其实就是对`werkzeug.utils.redirect`的一个包装
        
    * `request.render`创建`response`（支持lazy render）  
        可以看到如果是`lazy`的话，只返回`Response`对象。  
        如果不是`lazy`的话，就直接生成最终的结果。  
        
        ```
        以HttpRequest为例：
        
        def render(self, template, qcontext=None, lazy=True, **kw):
            response = Response(template=template, qcontext=qcontext, **kw)
            if not lazy:
                return response.render()
            return response
        
        class Response(werkzeug.wrappers.Response):
            ...
            def render(self):
                """ Renders the Response's template, returns the result
                """
                env = request.env(user=self.uid or request.uid or odoo.SUPERUSER_ID)
                self.qcontext['request'] = request
                return env["ir.ui.view"].render_template(self.template, self.qcontext)
            ...
        ```
    * `Response`会根据提供的模版渲染出最终的页面  
        查找名为`ir.ui.view`的模型`odoo.addons.base.ir.ir_ui_view.View`，然后调用里面的`render_template`、`render`生成最终的页面


# 路由器
当定义了大量的`controller`后，如何将请求分发到这些`controller`上就要靠路由器了。
在`odoo`中，路由的功能由`IrHttp`实现。

### `odoo.addons.base.ir.ir_http.IrHttp`
`IrHttp`也被定义为一个模型，不过它的基类是`AbstractModel`。也就是说，这个模型不会在数据库中建表，只是一个纯粹提供功能的模型。主要功能：

* `_dispatch`请求转发  

    `_dispatch`算是入口函数。它接收根据请求路径从`routing_map`中查找`handler`。然后进行用户校验。最后将`routing_map`中查找到的`handler`放入请求对象中。最后请求在`dispatch`的时候处理请求。

* `routing_map`路由表的初始化  
    当类中没有`_routing_map`属性时，使用`odoo.http.routing_map`创建这个属性。`odoo.http.routing_map`会遍历之前说过的`controllers_per_module`，将所有的`controller`、路径信息保存到`werkzeug.routing.Map`中。最后的返回值就是这个`Map`。
    
* `_find_handler`查找`controller`
    从routing_map中获取初期请求的`controller`。获取到`routing_map`后，用标准的`werkzeug`获取陆游的方法匹配路由表，即：
    
    ```
    urls = url_map.bind_to_environ(environ)
    endpoint, args = urls.match()
    ```
    
* `_authenticate`请求的安全验证


总之在`IrHttp`中，它根据需要创建`routing_map`，从`routing_map`中获取`controller`，然后将`controller`放入请求对象中，然后调用下`request.dispatch`
并将结果返回。

```
class IrHttp(models.AbstractModel):

    ...
    
    @classmethod
    def routing_map(cls):
        if not hasattr(cls, '_routing_map'):
            ...
            cls._routing_map = http.routing_map(mods, False, converters=cls._get_converters())
        return cls._routing_map
        
    ...
    
    @classmethod
    def _find_handler(cls, return_rule=False):
        return cls.routing_map().bind_to_environ(request.httprequest.environ).match(return_rule=return_rule)
        
    ...
    
    @classmethod
    def _authenticate(cls, auth_method='user'):
        ...
        if request.session.uid:
            request.session.check_security()
        ...
        
    ...
    
    @classmethod
    def _dispatch(cls):
        # locate the controller method
        try:
            rule, arguments = cls._find_handler(return_rule=True)
            func = rule.endpoint
        except werkzeug.exceptions.NotFound, e:
            return cls._handle_exception(e)

        # check authentication level
        try:
            auth_method = cls._authenticate(func.routing["auth"])
        except Exception as e:
            return cls._handle_exception(e)

        processing = cls._postprocess_args(arguments, rule)
        if processing:
            return processing

        # set and execute handler
        try:
            request.set_handler(func, arguments, auth_method)
            result = request.dispatch()
            if isinstance(result, Exception):
                raise result
        except Exception, e:
            return cls._handle_exception(e)

        return result
    ...
```
        
# 请求处理器 
### `odoo.http.Root`
所有种类的`server`都使用`odoo.service.wsgi_server.application`处理请求。而在其中，最重要的一个请求处理器是：`odoo.http.root`，它主要负责页面请求的处理；另外一个`wsgi_xmlrpc`，主要负责处理`xmlrpc`一类的数据接口请求。

对于`xmlrpc`请求来说，处理起来相对容易：根据路径、参数，获取对应的函数；执行函数后将返回的结果转化成`json`返回就可以了。

对于另外一类来说就复杂很多：要将请求包装成`JsonRequest`或者`HttpRequest`；要根据路径找到对应的`controller`；根据需要查找模版生成页面......


```
def wsgi_xmlrpc(environ, start_response):
    ...
    if environ['REQUEST_METHOD'] == 'POST' and environ['PATH_INFO'].startswith('/xmlrpc/'):
    ...
    
    
def application_unproxied(environ, start_response):
    ...
    with odoo.api.Environment.manage():
        # Try all handlers until one returns some result (i.e. not None).
        for handler in [wsgi_xmlrpc, odoo.http.root]:
            result = handler(environ, start_response)
            if result is None:
                continue
            return result
    ...
```

从上边的代码请求被`wsgi_xmlrpc，odoo.http.root`两个函数处理。`odoo.http.root`可以被当作函数使用，全都因为其中的`__call__`函数。从下边代码中可以看到当`__call__`首次运行时：首先加载模块；然后后`root`会在`dispatch`函数中判断请求类型生成`JsonRequest`或`HttpRequest`；从`request`中查找`ir.http`模型（其实就是从全局的`registry`中）；通过`ir.http`模型的`_dispatch`来获得返回的结果；最后将结果生成最终的`Response`。

注意：之前在`HttpRequest`类的`render`函数中，可以设置请求的惰性处理（或者叫延迟处理）。在`dispatch`函数的最后，在`get_response`函数中，可以看到它调用了一个`result.flatten`函数。这个函数定义在`Response`对象中，它的作用是强制`render`。
也就是说，如果设置了`HttpRequest`延迟处理，那么直到dispatch进行完才会创建出页面，在此之前`response`只是一个对象。

```
class Root(object):
    def __init__(self):
        self._loaded = False
        
    def __call__(self, environ, start_response):
        """ Handle a WSGI request
        """
        if not self._loaded:
            self._loaded = True
            self.load_addons()
        return self.dispatch(environ, start_response)
    
    def dispatch(self, environ, start_response):
        ...
        request = self.get_request(httprequest)
        ...
        with request:
            ...
            ir_http = request.registry['ir.http']
            ...
            result = ir_http._dispatch()
            ...
        ...
            response = self.get_response(httprequest, result, explicit_session)
        return response(environ, start_response)
        
    def get_request(self, httprequest):
        # deduce type of request
        if httprequest.args.get('jsonp'):
            return JsonRequest(httprequest)
        if httprequest.mimetype in ("application/json", "application/json-rpc"):
            return JsonRequest(httprequest)
        else:
            return HttpRequest(httprequest)
        
    def get_response(self, httprequest, result, explicit_session):
        if isinstance(result, Response) and result.is_qweb:
            ...
            result.flatten()
            ...
        ...
```

### 不算总结的总结
经过若干次的补充，请求的处理貌似完成了。  
终于搞清了`Request`是如何一步步变成最终的`Resopnse`。但是页面是如何生成的，依然是个问题。`odoo`中使用了一套自己的模版体系来定义页面。通过`ir.ui.view`这个模型将`Response`变成最终的页面。  
之后，就来研究一下`View`。
        