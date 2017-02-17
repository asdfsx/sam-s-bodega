+++
date = "2017-01-11T12:01:19+08:00"
topics = [
  "topic 1",
]
description = "python的导入系统"
author = "asdfsx"
draft = false
title = "python的导入系统"
tags = [
  "python",
  "import",
]
keywords = [
  "python",
  "import",
]
type = "post"

+++

一个模块中的Python代码通过导入的过程获得对另一个模块中的代码的访问。`import`语句是调用导入机制的最常用方法，但它不是唯一的方法。诸如`importlib.import_module()`和内置`__import__()`之类的函数也可以用于调用导入机制。

import语句组合两个操作；它搜索指定的模块，然后将该搜索的结果绑定到本地作用域中的名称。import语句的搜索操作被定义为使用适当的参数调用`__import__()`函数。`__import__()`的返回值用于执行import语句的名称绑定操作。有关该名称绑定操作的确切详细信息，请参见import语句。

直接调用`__import__()`只执行模块搜索，如果找到，则执行模块创建操作。虽然可能会发生某些副作用，例如导入父包以及更新各种缓存（包括`sys.modules`），但只有import语句会执行名称绑定操作。

当调用`__import__()`作为import语句的一部分时，将调用标准的内置`__import__()`。用于调用导入系统的其他机制（例如`importlib.import_module()`）可以选择颠覆`__import__()`并使用其自己的解决方案来实现导入语义。

首次导入模块时，Python会搜索模块，如果找到，它会创建一个模块对象[1]，并初始化它。如果找不到指定的模块，则会引发ImportError。当执行导入机制时，Python实现各种策略来搜索命名的模块。这些策略可以通过使用下面部分中描述的各种钩子来修改和扩展。

在版本3.3中更改：导入系统更新成完全实现 PEP 302的第二阶段。不再有任何隐式导入机制 - 完整导入系统通过sys.meta_path暴露。此外，已实现原生命名空间包支持（参见 PEP 420）。

### 1. importlib  
importlib模块提供了一个丰富的API，用于与导入系统进行交互。例如`importlib.import_module()`提供了一个比内置的`__import__()`更简单的API来调用导入机制。有关其他详细信息，请参阅importlib库文档。

### 2. 包`Packages`  
Python只有一种模块对象，所有的模块都是这种类型，不管这个模块是否是用Python，C，或者其他语言实现。为了帮助组织模块并提供命名层次结构，Python有一个概念：包。

你可以认为包是文件系统中的一个目录并且模块作为文件存放于目录中，但是不要做这种太字面化的类比因为包和模块不需要源于文件系统。从这篇文档的目的是我们用目录和文件这个方便的类比来解释包和模块。和文件系统一样，包有有层次的组织着，并且包本身也会包含子包，规则的模块也一样。

重要的是请注意所有的包都是模块，但不是所有的模块都是包。换句话说，包只是一种特殊形式的模块。具体来说，包含`__path__`属性的任何模块都被视为包。

所有的模块都有名字。子模块的名字是通过点号从父模块中分离出来的，和Python标准的属性访问语法相似。因此，您可能有一个名为sys的模块和一个名为email的软件包，其中包含一个名为`email.mime`的子包，名为`email.mime.text`的子包。

##### 2.1 普通包
Python defines two types of packages, regular packages and namespace packages. Regular packages are traditional packages as they existed in Python 3.2 and earlier. A regular package is typically implemented as a directory containing an __init__.py file. When a regular package is imported, this __init__.py file is  executed, and the objects it defines are bound to names in the package’s namespace. The __init__.py file can contain the same Python code that any other module can contain, and Python will add some additional attributes to the module when it is imported.

Python定义了两种包，普通包和名字空间包。普通包就是传统的包，它们在Python3.2和更早的版本中就存在。一个普通包就是一个含有一个`__init__py`文件的目录。当一个包被导入时，这个`__init__.py`文件会在暗中执行，它定义的对象会绑定到包的名字空间中的名字上。`__init__.py`文件中可以包含其它模块可以包含的Python代码，而且Python导入这个模块时可以添加些额外的属性到模块中。

比如：下面的文件系统层次定义了一个个包含3个子包的父包。

```
parent/
    __init__.py
    one/
        __init__.py
    two/
        __init__.py
    three/
        __init__.py
```

导入`parent.one`会暗中执行`parent/__init__.py`和`parent/one/__init__.py`。之后再导入`parent.two`或者`parent.three`会执行`parent/two/__init__.py`或者`parent/three/__init__.py`

##### 2.2 名字空间包

### 3. 搜索
##### 3.1 模块缓存`The module cache`
##### 3.2 查询器与装载器`Finders and loaders`
##### 3.3 导入钩子`Import hooks`
##### 3.4 元路径`meta path`
### 4. 装载
当找到模块描述后，导入系统将使用它导入模块。这里模拟一下导入过程中发生了什么：

```
module = None
if spec.loader is not None and hasattr(spec.loader, 'create_module'):
    # It is assumed 'exec_module' will also be defined on the loader.
    module = spec.loader.create_module(spec)
if module is None:
    module = ModuleType(spec.name)
# The import-related module attributes get set here:
_init_module_attrs(spec, module)

if spec.loader is None:
    if spec.submodule_search_locations is not None:
        # namespace package
        sys.modules[spec.name] = module
    else:
        # unsupported
        raise ImportError
elif not hasattr(spec.loader, 'exec_module'):
    module = spec.loader.load_module(spec.name)
    # Set __loader__ and __package__ if missing.
else:
    sys.modules[spec.name] = module
    try:
        spec.loader.exec_module(module)
    except BaseException:
        try:
            del sys.modules[spec.name]
        except KeyError:
            pass
        raise
return sys.modules[spec.name]
```
注意以下内容：  

* 如果在`sys.modules`中已经存在给定名称的模块对象，导入系统将会返回这个对象。
* 在装载器执行模块代码之前，模块将保存在`sys.modules`中。这点非常重要，因为模块可能会导入自己（直接或间接）；把它预先添加到`sys.modules`会避免循环绑定和反复装载。
* 如果装载失败，失败的模块 - 而且只有失败的模块 - 会从`sys.modules`中移除。而`sys.modules`中已有的模块，和导入过程中成功导入的那些模块仍然会保留在`sys.modules`中。与之不同的是重新装载`reloading`，即使失败也会保留在`sys.modules`中。
* 当模块创建后但是还没有执行，导入机构会设置依赖导入`import-related`的模块属性（`__init_module_attrs`上边的伪代码例子中），在之后的部分会详细介绍。
* 执行模块时装在过程中很重要的一步，在这一步中会创建模块的名字空间。执行由装载器全权负责，它会决定生成什么和如何生成。
* 在装载过程中创建的模块有可能不是最终返回的模块，如果它被传递给`exec_module()`。

3.4版发生的变化：导入系统接管loader的引用功能，之前是由`importlib.abc.Loader.load_module()`来执行。

##### 4.1 装载器
##### 4.2 子模块
##### 4.3 模块描述
##### 4.4 导入时的模块属性

* \_\_name\_\_
* \_\_loader\_\_
* \_\_package\_\_
* \_\_spec\_\_
* \_\_path\_\_
* \_\_file\_\_
* \_\_cached\_\_

##### 4.5 module.\_\_path\_\_
##### 4.6 Module reprs

### 5. 基于路径的查找器
##### 5.1 Path entry finders
##### 5.2 Path entry finder protocol
### 6. 替换标准的导入系统  
最可靠的替换整个导入系统的方法是删除`sys.meta_path`的默认内容，用自定义的`meta path hook`来彻底替换。

替换内置`__import__()`函数就只能对导入系统的行为造成影响，而不能影响使用导入系统的其它API。这个技术可以在包内部使用，以修改包内部的导入行为。

为了有选择的避免通过`hook`导入包早于扫描`meta path`（甚至早于关闭整个标准导入系统），可以通过从`find_spec`直接抛出`ModuleNoFoundError`异常而不是返回一个`None`。后者会让后续的`meta path`搜索继续执行，而抛出异常会立即中断搜索。

### 7. `__main__`的特殊考虑
`__main__`模块是Python导入系统产生的特殊模块。就像别处提到的，`__main__`模块是在解释器启动时被直接创建的，就像`sys`和`builtings`。但是与它们不同，`__main__`不是严格意义上的内建模块。因为`__main__`的创建方式由标志位、参数、以及解释器的实现来决定。

##### 7.1 `__main__.__spec__`
根据`__main__`的创建方式，`__main__.__spec__`会被设置为合适的值或者`None`

当Python启动时使用`-m`参数，`__spec__`的值就是对应模块或者包的模块名。当执行一个目录、`zip`文件或其他`sys.path`时，`__main__`模块也会被装载，这时`__spec__`也
会产生。

在以下情况下`__main__.__spec__`被设置为`None`，当产生`__main__`的代码不在一个可导入模块下：

* 解释器命令行
* 通过`-c`转换
* 从`stdin`读入后执行
* 直接执行源码或者字节码文件

注意最后一种情况里`__main__.__spec__`通常时`None`，即便这个文件可以当作模块字节导入。如果`__main__`需要模块的元数据，执行时可以使用`-m`参数。

注意即便`__main__`对应一个可导入模块，而且`__main__.__spec__`也照样设置了，它们仍然被看作不同的模块。这是因为代码块被`if __name__ == "__main__"`保护着：只有当模块是用于创建`__main__`名字空间，而不是是普通的导入才会被执行。

### 8. Open issues
### 9. References
导入机制从Python诞生至今不断的进化。原始的包说明书现在仍然可以使用，尽管很多细节已经发生了变化。


----
翻的真烂！给自己刨的坑略多、略大！愚公移山慢慢弄吧！