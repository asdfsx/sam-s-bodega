+++
keywords = [
  "python",
  "odoo",
]
tags = [
  "python",
  "odoo",
]
topics = [
  "topic 1",
]
description = "odoo的web server"
author = "asdfsx"
draft = false
type = "post"
date = "2016-12-24T20:47:02+08:00"
title = "odoo的web server"

+++

odoo的web服务器实现都在一个包里odoo.service.server。为了提升服务器的性能，提供了3种不同的web服务器，分别使用了thread、gevent、process。而在web服务器里，odoo使用werkzeug这套wsgi库，实现端口监听、请求处理等。

### 服务器的启动

在`odoo.service.server`包中，入口函数是位于文件末尾的`start`函数，它的行为如下：  

* 定义全局变量server
* 加载所谓的`server wide module`（默认的全局模块只有`web`，`web_kanban`）
* 根据配置创建不同的server
* 创建文件监视（为了实现模块的动态加载）
* 启动server


从中可以看到具体处理请求的程序是`odoo.service.wsgi_server.application`，而默认的`server`是基于线程的。

```
def start(preload=None, stop=False):
    """ Start the odoo http server and cron processor.
    """
    global server
    load_server_wide_modules()
    if odoo.evented:
        server = GeventServer(odoo.service.wsgi_server.application)
    elif config['workers']:
        server = PreforkServer(odoo.service.wsgi_server.application)
    else:
        server = ThreadedServer(odoo.service.wsgi_server.application)

    watcher = None
    if 'reload' in config['dev_mode']:
        if watchdog:
            watcher = FSWatcher()
            watcher.start()
        else:
            _logger.warning("'watchdog' module not installed. Code autoreload feature is disabled")
    if 'werkzeug' in config['dev_mode']:
        server.app = DebuggedApplication(server.app, evalex=True)

    rc = server.run(preload, stop)

    # like the legend of the phoenix, all ends with beginnings
    if getattr(odoo, 'phoenix', False):
        if watcher:
            watcher.stop()
        _reexec()

    return rc if rc else 0
```

### 基类 odoo.service.server.CommonServer
作为鸡肋，啥也没有，只有一个关闭socket的函数

### 基于线程的实现 odoo.service.server.ThreadedServer

实现了父类的`run`函数，作为程序的入口吧。

启动过程中会创建两种线程

* 处理http请求的线程  
    用`odoo.service.server.ThreadedWSGIServerReloadable`来实现线程。这个类继承了`werkzeug.serving.ThreadedWSGIServer`，所以具体如何使用线程来完成请求的处理，都被隐藏在`werkzeug`中了。
    
* 处理定时任务的线程  
    在`cron_spawn`函数中创建的，按照配置中的`max_cron_threads`来创建`daemon`线程。线程中通过遍历`odoo.modules.registry.Registry.registries`来获得数据库信息，然后通过`odoo.addons.base.ir.ir_cron.ir_cron._acquire_job`来获取定时任务。

### 基于gevent的实现 odoo.service.server.GeventServer

gevent是一个基于协程的网络库。底层使用greenlet作为轻量级的异步并法方式，使用libev实现基于事件循环的网络请求处理。

GeventServer实现非常简单，感觉和Thread、Process的方案比，感觉缺了些东西。（比如进程、线程版里都会有cron_xxx，目测应该是执行定时任务的程序。）

### 基于进程的实现 odoo.service.server.PreforkServer

让人又爱又恨的进程。与上边的两个实现相比，进程要靠自己一点点实现

* PreforkServer  
    启动函数为`run`，这个函数中也定义了主进程的行为：

    *  建立管道、处理信号；建立socket
    *  处理信号
    *  处理僵尸进程
    *  处理超时连接
    *  创建子进程  

    其余的函数都是为了上述功能
    
* Worker  
    子进程的父类，定义了子进程的大部分行为。包括信号的处理、进程状态的处理。具体的业务放在`process_work`函数中，当然父类不需要实现这个函数，具体业务逻辑放在子类中实现。
    
* WorkerHTTP  
    处理http请求的子进程。启动子进程的时候，会定义一个处理http请求的`self.server`，类型是`BaseWSGIServerNoBind`。从类的定义上可以看出来它继承了`werkzeug.serving.BaseWSGIServer`。
    
* WorkerCron  
    处理定时任务的子进程。执行完任务后退出。（这里和线程的实现不一样。线程里是死循环；进程则是执行完就退出，然后由主进程再次拉起？？？待确认啊～～）
    

### 预加载模型 preload_registries
在之后的文章中，我们可以看到`registry`的重要性。在这里我们只需要明确，不管是进程、还是线程的实现，都会在启动服务器时调用这个函数创建`registry`。

```
def preload_registries(dbnames):
    """ Preload a registries, possibly run a test file."""
    # TODO: move all config checks to args dont check tools.config here
    config = odoo.tools.config
    test_file = config['test_file']
    dbnames = dbnames or []
    rc = 0
    for dbname in dbnames:
        try:
            update_module = config['init'] or config['update']
            registry = Registry.new(dbname, update_module=update_module)
            # run test_file if provided
            if test_file:
                _logger.info('loading test file %s', test_file)
                with odoo.api.Environment.manage():
                    if test_file.endswith('yml'):
                        load_test_file_yml(registry, test_file)
                    elif test_file.endswith('py'):
                        load_test_file_py(registry, test_file)

            if registry._assertion_report.failures:
                rc += 1
        except Exception:
            _logger.critical('Failed to initialize database `%s`.', dbname, exc_info=True)
            return -1
    return rc
```
    
### 文件系统监视器 FSWatcher
使用`watchdog`监视指定目录中发生的文件创建、删除、修改事件。主要是为了模块的动态更新。

### 处理请求的 wsgi_server
之前说的3种server，都使用了`odoo.service.wsgi_server.application`来处理请求。

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

所有的请求都会尝试用2个handler来处理一下。`wsgi_xmlrpc`用来处理xmlrpc，只有`POST`类型的、url以`/xmlrpc/`开头的会被这个handler处理。其余的都由`odoo.http.root`来处理。看来`odoo.http.root`应该超级复杂。

### 超级长的 odoo.http
这是一个1651行的，历史悠久的python文件。第一行`# -*- coding: utf-8 -*-`是2013年2月1日写下的。里边集中了请求的封装、模块加载、session管理等东西。这里我们重点先看`root`。  

首先，在文件的末尾找到`root = Root()`。`root`是一个`Root`的对象。哪一个对象怎么作为`handler`作为函数调用的呢？

```
class Root(object):
    ...
    def __call__(self, environ, start_response):
        if not self._loaded:
            self._loaded = True
            self.load_addons()
        return self.dispatch(environ, start_response)
    ...
```

原来是特殊的的`__call__`函数。而且可以看到模块是延迟加载的：只有在第一次被调用的时候才会`load_addons`。然后才会调用`dispatch`去处理请求。

##### 加载模块 load_addons  
加载模块的步骤

* 从`odoo.modules.module.ad_paths`下的所有目录里，获取所有的模块目录。
* 从各个目录里，查找模块的`__manifest__.py`文件（模块的描述文件）。
* 读取模块信息，然后`__import__`模块。
* 将模块、模块信息存到全局变量`addons_module, addons_manifest`中，同时将模块的静态文件地址也保存起来。
* 创建用来处理请求的dispatch

特别注意在最后，`dispatch`函数被特殊处理了一下

```
app = werkzeug.wsgi.SharedDataMiddleware(self.dispatch, statics, cache_timeout=STATIC_CACHE)
        self.dispatch = DisableCacheMiddleware(app)
```

这里先用类中的`dispatch`函数，生成`werkzeug.wsgi.SharedDataMiddleware`的对象`app`；然后再用`app`生成一个`DisableCacheMiddleware`对象，替换掉原来的`dispatch`......

##### 抽丝剥茧 dispatch 
在`load_addons`的最后，原先的`dispatch`函数被层层包裹（真的是两层）。
最外边一层是`DisableCacheMiddleware`：

```
class DisableCacheMiddleware(object):
    def __init__(self, app):
        self.app = app
    def __call__(self, environ, start_response):
        def start_wrapped(status, headers):
            referer = environ.get('HTTP_REFERER', '')
            parsed = urlparse.urlparse(referer)
            debug = parsed.query.count('debug') >= 1

            new_headers = []
            unwanted_keys = ['Last-Modified']
            if debug:
                new_headers = [('Cache-Control', 'no-cache')]
                unwanted_keys += ['Expires', 'Etag', 'Cache-Control']

            for k, v in headers:
                if k not in unwanted_keys:
                    new_headers.append((k, v))

            start_response(status, new_headers)
        return self.app(environ, start_wrapped)
```
代码不多直接贴。不出意外的使用了`__call__`，然后可以看到它主要是对返回的`http头`做了特殊的处理。

接下来是一层`werkzeug.wsgi.SharedDataMiddleware`，借用官方文档：

```
A WSGI middleware that provides static content for development environments or simple server setups.
```

它是用来安装模块对应的静态文件。

最后就是`dispatch`函数了。先对请求进行处理：设置session、设置数据库、设置语言；然后交给`ir_http._dispatch`来处理请求；`get_response`将前一步的结果进行处理，生成最终返回的`response`。

```
def dispatch(self, environ, start_response):
    ...
    httprequest = werkzeug.wrappers.Request(environ)
    ...
    explicit_session = self.setup_session(httprequest)
    self.setup_db(httprequest)
    self.setup_lang(httprequest)
    request = self.get_request(httprequest)
    ...
    with request:
        ...
        with odoo.tools.mute_logger('odoo.sql_db'):
            ir_http = request.registry['ir.http']
        ...
        result = ir_http._dispatch()
        ...
        response = self.get_response(httprequest, result, explicit_session)
    return repsonse
    ...
```
但是`registry`从何而来，`ir_http`又是什么。继续探索下去吧。

##### 请求的包装
最初的请求时由`werkzeug.wrappers.Request`生成的。然后经过一步步的设置，将它的session信息、可能访问的数据库连接、使用的语言，都配置存放到请求对象中。然后根据请求的类型，生成`JsonRequest`或者`HttpRequest`。这两个类分别对应`Json`请求和一般的`Http`请求，他们的父类都是`WebRequest`。在他们的父类中，我们看到了神秘的`registry`。

```
@property
def registry(self):
    """
    The registry to the database linked to this request. Can be ``None``
    if the current request uses the ``none`` authentication.
    .. deprecated:: 8.0
        use :attr:`.env`
    """
    return odoo.registry(self.db) if self.db else None
```
使用了`@property`使这个函数可以当作类的属性访问。`odoo.registry`这个函数可以在`odoo/__init__.py`中找到，返回一个Registry对象。初步判断这里时做了一个初始化工作：创建Registry对象，成功的话返回数据库名称，失败返回None。

```
def registry(database_name=None):
    """
    Return the model registry for the given database, or the database mentioned
    on the current thread. If the registry does not exist yet, it is created on
    the fly.
    """
    if database_name is None:
        import threading
        database_name = threading.currentThread().dbname
    return modules.registry.Registry(database_name)
```

##### 模型的注册表 Registry

```
The registry is essentially a mapping between model names and model classes.
There is one registry instance per database.
```
官方文档对`Registry`的说明。

在请求处理的过程中，如果`modules.registry.Registry`还没有实例，那么就创建一个。这个是一个继承了`collections.Mapping`，高度定制化的类。在这个类中，创建了一个类的属性`registries`作为缓存来存放之后生成的`registry`。每次想获取`registry`对象的时候，都是先查询这个缓存。如果缓存中有现成的对象，直接返回，否则生成新的对象。

在创建registry的同时，还进行了模块的实例化：从模块中抽取模型、把模型保存到数据库、将模型的实例存到registry中。

##### 请求的重定向
通过对模块、以及`registry`的了解，我们可以继续探究之前的`ir_http`。

```
ir_http = request.registry['ir.http']
result = ir_http._dispatch()
```

从`registrty`中查找名字为`ir.http`的模型的对象。通过搜索我们找到这个名字对应的类是：`odoo.addons.base.ir.ir_http.IrHttp`，在这个类中，我们找到了这个`_dispatch`方法。可以看到它在初次执行时会将安装好的模块都加载到`odoo.http.routing_map`中。之后当请求到达的时候，从`routing_map`中获得`controller`，然后由controller来处理请求。

```
    @classmethod
    def routing_map(cls):
        if not hasattr(cls, '_routing_map'):
            installed = request.registry._init_modules - {'web'}
            mods = [''] + odoo.conf.server_wide_modules + sorted(installed)
            cls._routing_map = http.routing_map(mods, False, converters=cls._get_converters())
        return cls._routing_map
        
    @classmethod
    def _find_handler(cls, return_rule=False):
        return cls.routing_map().bind_to_environ(request.httprequest.environ).match(return_rule=return_rule)

    @classmethod
    def _dispatch(cls):
        # locate the controller method
        try:
            rule, arguments = cls._find_handler(return_rule=True)
            func = rule.endpoint
        except werkzeug.exceptions.NotFound, e:
            return cls._handle_exception(e)
            
        ......
        
        try:
            request.set_handler(func, arguments, auth_method)
            result = request.dispatch()
        except Exception, e:
            return cls._handle_exception(e)
        return result
```

##### 创建请求的路由器 odoo.http.routing_map
请求的路由信息是保存在`werkzeug.routing.Map`中的。而`routing_map`函数负责创建这个对象，然后通过遍历所有已安装的模块中的`controller`，将所有的路径信息都存放到这个对象中去。

```
def routing_map(modules, nodb_only, converters=None):
    routing_map = werkzeug.routing.Map(strict_slashes=False, converters=converters)
    ......
    for module in modules:
        for _, cls in controllers_per_module[module]:
            o = cls()
            members = inspect.getmembers(o, inspect.ismethod)
            for _, mv in members:
                ...
                endpoint = EndPoint(mv, routing)
                ...
                routing_map.add(werkzeug.routing.Rule(url, endpoint=endpoint, methods=routing['methods'], **kw))
    return routing_map
```

附一个`werkzeug.routing.Map`的例子：

```
from werkzeug.routing import Map, Rule, NotFound, RequestRedirect

url_map = Map([
    Rule('/', endpoint='blog/index'),
    Rule('/<int:year>/', endpoint='blog/archive'),
    Rule('/<int:year>/<int:month>/', endpoint='blog/archive'),
    Rule('/<int:year>/<int:month>/<int:day>/', endpoint='blog/archive'),
    Rule('/<int:year>/<int:month>/<int:day>/<slug>',
         endpoint='blog/show_post'),
    Rule('/about', endpoint='blog/about_me'),
    Rule('/feeds/', endpoint='blog/feeds'),
    Rule('/feeds/<feed_name>.rss', endpoint='blog/show_feed')
])

def application(environ, start_response):
    urls = url_map.bind_to_environ(environ)
    try:
        endpoint, args = urls.match()
    except HTTPException, e:
        return e(environ, start_response)
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return ['Rule points to %r with arguments %r' % (endpoint, args)]
```

至此我们终于可以将收到的请求，转发到对应的`controller`中了。

##### Session 管理
在http访问的过程中，session的管理是很重要的。目前在`odoo`中，使用了基于`werkzeug`的session实现。其中`session`的存储使用的是文件存储。不得不说这个对`odoo`的高可用是一个小小的障碍。不过现在已经有现成的模块可以将这个基于文件的`sessionstore`替换成`redisstore`。基本思路就是继承`odoo.http.Root`，用一个支持`redis`的`session_store`函数将原有的函数覆盖掉。

项目地址：https://github.com/keerati/odoo-redis  

```
class OpenERPSession(werkzeug.contrib.sessions.Session):
    def __init__(self, *args, **kwargs):
    ......
    
class Root(object):
    ......
    @lazy_property
    def session_store(self):
        # Setup http sessions
        path = odoo.tools.config.session_dir
        _logger.debug('HTTP sessions stored in: %s', path)
        return werkzeug.contrib.sessions.FilesystemSessionStore(path, session_class=OpenERPSession)
        
```

### 不算总结的总结
经过不断的补充，终于将web server弄完了。现在算是基本弄清了`odoo`的启动流程，模块是如何加载安装的，请求是如何被处理的。关于`registry`、`module`、`model`、`controller`等会单独开篇去详细研究。

注意：  
文中大部分的`模块加载`仅指`import`。  
真正将其实例化，或者说从模块类 -> 模型／模块类实例，是在`registry`中创建的。