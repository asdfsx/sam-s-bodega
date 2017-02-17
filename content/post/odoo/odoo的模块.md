+++
title = "odoo的模块"
tags = [
  "python",
  "odoo",
]
keywords = [
  "python",
  "odoo",
]
description = "odoo的模块"
draft = false
type = "post"
date = "2016-12-28T16:49:21+08:00"
topics = [
  "topic 1",
]
author = "asdfsx"

+++

上一篇`odoo`的模块管理，主要是odoo的模块加载。这一篇来看看模块到底是什么。  
抄袭官网文档：

```
Both server and client extensions are packaged as modules which are optionally loaded in a database.

Odoo modules can either add brand new business logic to an Odoo system, or alter and extend existing business logic: a module can be created to add your country's accounting rules to Odoo's generic accounting support, while the next module adds support for real-time visualisation of a bus fleet.

Everything in Odoo thus starts and ends with modules.
```
在研究`odoo`的过程中，也能感受到模块的重要性。`odoo`核心的代码只有webserver、模块管理、数据库连接，其余的功能基本上都是由各个`addons`目录中的模块实现，甚至最基础的显示框架都是由模块来实现。

### 模块的内容

* 业务模型  
    由Python类组成，并由Odoo负责进行持久化
* 数据文件  
    XML、CSV格式的文件，包括元数据、配置文件、指标纬度等等
* web controller  
    处理浏览器发来的各种请求
* 静态页面文件  
    包括图片、js、css

### 模块的结构
通过`odoo-bin scaffold`可以创建一个最基础的`odoo`模块，从这个模块中我们可以看到一个模块的大致结构

* `__init__.py`  
    所有的python包都有的导入文件
* `__manifest__.py`  
    模块描述文件，内部就是一个字典，里边记录了模块名、描述、作者、分类、版本、依赖、数据文件等信息。
* `controllers`  
    存放web controller。`controller`类继承了`odoo.http.Controller`，用来定义访问的路径。
* `demo`  
    存放`demo`使用的数据文件
* `security`  
    存放模型的访问权限，通常是一个csv文件`ir.model.access.csv`
* `views`  
    存放定义页面、显示用的配置文件。
* `i18n`
    存放国际化使用的各种语言文件。
* `tests`
    存放测试用的程序
* `models`  
    存放业务模型。模型继承了`odoo.models.Model`。大部分的数据库操作由`odoo`的`ORM`层实现。
