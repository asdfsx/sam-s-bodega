+++
date = "2016-12-24T20:37:57+08:00"
author = "asdfsx"
keywords = [
  "odoo",
  "python",
]
description = "odoo的命令"
draft = false
type = "post"
title = "odoo的命令"
tags = [
  "odoo",
  "python",
]
topics = [
  "topic 1",
]

+++

Odoo命令都在odoo.cli包中

### 元类：CommandType
```
commands = {}

class CommandType(type):
    def __init__(cls, name, bases, attrs):
        super(CommandType, cls).__init__(name, bases, attrs)
        name = getattr(cls, name, cls.__name__.lower())
        cls.name = name
        if name != 'command':
            commands[name] = cls
```
所有使用了这个元类的类，都会注册到包内的全局字典`commands`中：类名为key，类为value

### 所有命令的基类：Command
```
class Command(object):
    """Subclass this class to define new odoo subcommands """
    __metaclass__ = CommandType

    def run(self, args):
        pass
```
可以看到基类使用`CommandType`作为元类。这意味着所有的子类都会注册到全局字典中。  
同时还定义了个函数`run`。这算是预定义了要实现功能的函数。

### 命令的实现
以下命令全部都是`Command`的子类，所以都使用了原类`CommandType `，并实现了父类的`run`函数。

#### odoo.cli.command.Help
根据其它命令中的内容，生成帮助信息，并输出。

#### odoo.cli.deploy.Deploy
将本地的一个模块部署到指定的服务器上
需要给定本地模块的地址，和远程服务器地址
会将本地模块压缩成zip，然后通过http登录服务器并上传

#### odoo.cli.scaffold.Scaffold
用来生成一个模块的骨架。主要为了方便做二次开发。

默认使用的模版文件就在odoo.cli包内的template目录里

#### odoo.cli.shell.Shell
启动odoo，然后使用ipython、ptpython、bpython等创建一个交互环境。可以查询运行中的odoo的一些信息。

个人认为，这个也是方便开发调试的一个工具

#### odoo.cli.start.Start
也是一个方便开发调试的工具，官方文档的说明是`为你的项目快速启动odoo`。在启动参数中需要指定项目地址；如果不指定会从当前目录查找。


#### odoo.cli.server.Server
最重要的命令，启动odoo服务器。


