+++
author = "asdfsx"
date = "2016-12-13T15:12:39+08:00"
title = "Python的数据模型"
keywords = [
  "python",
  "data model",
]
description = "Python的数据模型"
type = "post"
tags = [
  "python",
  "data model",
]
topics = [
  "topic 1",
]
draft = false

+++

本文基于Python 3.5.2官方文档中数据模型部分完成

# 1.对象，值 和 类型  

Python把数据抽象为Objects。Python程序中所有的数据都体现为对象，或者对象之间的关系。(从某种程度上讲，为了与冯.诺伊曼的`存储程序型计算机`模型相一致，代码同样也体现为对象。)

每个对象都有一个标识，一个类型和一个值。对象一旦创建，它的标识就不会变更；你可以理解为它就像对象在内存中的地址。`is`操作用来比较两个对象的标识；`id()`函数返回一个整数来表示它的标识。

CPython实现细节：id(x)返回x在内存的存放地址。

对象的类型决定了对象支持哪些操作（比如，这个对象有长度吗？），同时也定义了对象中可能存在哪些值。`type()`函数返回一个对象的类型（类型也是一个对象）。就像它的标识，一个对象的类型也是不可变的。

一些对象的值是可修改的。可以修改的的对象被称为可变的`to be mutable`；创建后不能修改的对象被称为不可变的`immutable`。（不可变容器对象的值如果是一个可变对象的引用，那么当后者的值发生改变的时候，前者的值也会改变；但是这个容器仍然被认为是不可变的，因为容器内部包含的对象都没有发生变化。所以，不可变性`immutability`并不意味着拥有一个不可修改的值`unchangeable`，这很微妙）。一个对象是否可变取决于它的类型；比如：数字、字符串、元组是不可变的，字典和数组是可变的。

对象不会明确的被销毁；只有当它们不可达的时候，它们才有可能被垃圾回收。只要可达的对象不要被回收掉，如何实现延迟垃圾回收，并统一释放他们，这就是垃圾回收的实现质量问题了。

CPython实现细节：CPython现在使用引用计数机制和（可选的）循环垃圾延迟检测机制，对象一旦不可达就尽量收集，但是无法保证能回收循环引用的垃圾。查看`gc`模块的文档，来了解更多关于如何回收循环引用垃圾的信息。其他的实现可能不一样，而且CPtyhon也可能会改变策略。不要指望对象不可达以后，就会马上会被回收掉（所以你总是应该明确的关闭文件）。

注意跟踪调试工具会让一些本该回收掉的对象始终可达。同时使用`try...except`处理异常也会使对象始终可达。

一些对象包含对"外部"资源的引用。我们知道当这些对象被垃圾回收后这些资源也会被释放，但是因为垃圾回收不能保证这些对像会被回收，所以这些对象会提供一个明确方式来释放资源，通常是`close()`函数。强烈建议在程序中明确的关闭这样的对象。可以用`try...finally`语句和`with`语句来方便的执行它。

一些对象包含对其他对象的应用；这些对象叫做容器`containers`。这样的例子有元组、数组和字段。应用是容器的值的一部分。大部分的情况下，当我们说起容器中的值，指的是值，而不是容器内对象的标识；而当我们说容器的可变性，指的是容器中当前对象的标识。所以，一个不可变容器（比如元组）包含一个可变对象的引用，它的值是可变的，当那个可变对象发生变更的时候。

类型几乎影响了对象的所有行为。在一些场景下甚至影响对象标识的重要性：对于不可变类型，计算新值的操作可能实际返回的是一个已存在的，拥有相同类型和值的对象的引用；而对于可变对象这是不允许的。比如，执行`a=1; b=1`，a 和 b 可能（也可能不是）指向同一个包含`1`的对象，取决于具体实现; 但是执行`c=[];d[]`，c 和 d 保证指向两个完全不同的，独立的，新创建的空数组。（注意，`c = d = []` 将同一个对象分配给了c 和 d 。）


# 2.标准类型层级

下边是Python中内建类型列表。扩展模块（由C，Java，或其他语言实现的）会定义附加的类型。未来版本的 Python 可能会在此类型层次中增加新的类型 (例如：有理数，高效存储的整数数组等)，不过这些类型通常是在标准库中定义的。

以下个别类型描述中可能有介绍`特殊属性`的段落，它们是供实现访问的，不作为一般用途。这些定义在未来有可能发生改变：

* None  
这个类型只具有一个值，并且这种类型也只有一个对象，这个对象可以通过内建名字`None`访问，在许多场合里它表示无值，例如，没有显式返回值的函数会返回`None`。这个对象的真值为假。

* NotImplemented  
这个类型只具有一个值，并且这种类型也只有一个对象。这个对象可以通过内建名字`NotImplemented`访问。如果操作数没有对应实现，数值方法和复杂比较方法`rich comparison method`应该返回这个值 (解释器会尝试反射操作，或者其它操作，根据具体的操作)。它的真值为真。

* Ellipsis  
这个类型只具有一个值，并且这种类型也只有一个对象。这个对象可以通过字面值 ... 或者内建名字 Ellipsis 访问。它的真值为真。


* numbers.Number  
它们由数值型字面值产生，或者是算术运算符和内建数学函数的返回值。数值型对象是不可变 的，即一旦创建，其值就不可改变。Python 数值型和数学上的数字关系当然是非常密切的，但也受到计算机数值表达能力的限制。  

    Python 区分整数，浮点数和复数:

  * numbers.Integral  
  描述了数学上的整数集 (正负数).  
  有两类整数:
  
      * Integers (int)  
整数类型。表示不限范围的数字。移位和掩码操作符可以认为整数是这样组织的: 负数用二进制补码的一种变体表示，符号位会扩展至左边无限多位.
      * Booleans (bool)  
如此设计整数表示方法的一个目的是，使得负数在移位和掩码操作中能够更有意义.

  * numbers.Real (float)  
浮点数。本类型表示了机器级的双精度浮点数。硬件的底层体系结构 (和 C，Java 实现) 对你隐藏了浮点数取值范围和溢出处理的复杂细节。Python不支持单精度浮点数。使用单精度浮点数的原因一般是为了降低CPU负荷和节省内存，但是这个努力会被 Python 的对象处理代价所抵消，因此没有必要同时支持两种浮点数，使 Python 复杂化.

  * numbers.Complex (complex)  
复数。本类型用一对机器级的双精度浮点数表示复数。关于浮点数的介绍也适用于复数类型。复数 z 的实部和虚部可以通过属性 z.real 和 z.imag 获得.

* Sequences  
它们代表了有限有序的集合，并由非负数字作为索引。内建函数`len()`返回序列中的元素个数。当序列的长度为`n`，索引集合中的数字有`0, 1, ... , n-1`。通过`a[i]`来访问序列`a`中的`i`元素。  

    序列也支持切片：`a[i: j]`表示满足`i <= k < j`的所有项 `a[k]`。在作为表达式使用时，这个切片与原始的序列类型相同，这隐含着会重新编号索引，即从零开始。

    个别序列还支持有第三个 “步长” 参数的 扩展切片：`a[i:j:k]`选择了所有索引`x: x = i + n*k, n >= 0`并且`i <= x < j`。

    序列按照可变性可以分为：  
    
    * Immutable sequences  
    一个不可变序列对象一旦创建就不能修改。（如果该对象包含其他对象的引用，其他对象是可以被修改的。但是被一个不可变序列直接引用的对象不能改变。）  
    下边都是不可变序列
    
        * Strings  
        字符串是一组`Unicode code points`组成的序列。所有取值范围在`U+0000 - U+10FFFF`的`code points`都可以用来组成字符串。Python没有字符类型；取而代之的是字符串中的每个`code point`都表示为一个长度为`1`的字符串对象。内建函数`ord()`可以把一个`code point`从它的字符串格式转化为取值范围是`0 - 10FFFF`的整型。`chr()`把一个取值范围是`0 - 10FFFF`的整型数字转化为长度为1的字符串。`str.encode()`可以用指定的字符集把一个字符串转变成一个bytes，`bytes.decode()`用来做相反的操作。
        
        * Tuples  
        元组的元素是任意的python对象。包含2个或者更多元素的元组的格式是一列被`,`分隔的表达式。只包含一个元素的元组的格式是一个以`,`号结尾的表达式（一个表达式自己不能创建一个元组，since parentheses must be usable for grouping of expressions）。一个空元组，用`()`表示。
        
        * Bytes   
        bytes对象是一个不可修改的数组。元素都是`8-bit bytes`, 可以表示为0到256之间的一个整数。  
        bytes字面量(比如，`b'abc'`）和内建函数`bytes()`可以用来创建`bytes`对象。另外`bytes`对象可以通过函数`decode()`解码变成字符串。
        
    * Mutable sequences  
    可变序列在创建后可以被修改。其下标表示和切片表示可以作为赋值语句和`del`语句的目标。  
    目前有两种自带的可变序列。
        * Lists  
        一个list中的元素可以是任意的python对象。`Lists`的格式是在`[]`中放置一些由`,`隔开的表达式。
        * Byte Arrays  
        一个`bytearray`对象是一个可变的数组。它是由内建函数`bytearray()`创建的。除了可变（也因此不可哈希），`byte array`还提供了与不可修改的`bytes`对象相同的接口和功能。  
        扩展模块`array`提供了一个额外的可变序列的例子，`collections`模块也是如此。

* Set types  
这个类型描述的是由有限数量的不可变对象构成的无序集合。因此它们不能用任何索引作为下标，但它们可以被迭代，内建函数`len()`可以返回集合中的元素总数。集合的常用场合是快速测试某元素是否在集合中, 或者是从一个序列中删除重复元素，或者是做一些数学运算，比如求集合的交集，并集，差和对称差。  
集合的元素与字典键一样，都是不可变性对象。注意，数值类型遵守数值比较的正常规则：如果两个数字相同（例如，1 和 1.0），只有一个能存在于集合中。  
当前有两种内建的集合类型：
    * Sets
    表示一个可变的集合。由内建函数`set()`创建，可以被多个函数修改，例如`add()`。
    * Frozen sets
    表示一个不可变的集合。由内建函数`frozenset()`创建。因为`frozenset`不可变而且可以哈希，它可以作为另外一个集合的元素，也可以作为字典的key
    
* Mappings  
表示由任意类型作索引的有限对象集合。下标记法`a[k]`表示在映射类型对象 a 中选择以 k 为索引的项，这该项可以用于表达式，作为赋值语句和`del`语句的目标，内建函数`len()`返回映射对象的元素数量。  
目前只有一种内建映射类型:  

    * Dictionaries  
    表示一个有限对象集合，几乎可以用任意值索引其中的对象。包括列表和字典的值可以是值，但不能是键，或者其它通过值比较而不是以对象标识比较的可变对象也不能作为键，其原因是字典的实现效率要求键的散列值保持不变。数值类型遵守数值比较的正常规则：如果两个数字相同（例如，1 和 1.0）那么它们可以作为字典中同一个实体的索引。  
    字典是可变的, 可以用 {...} 语法创建它们。  
    扩展模块`dbm.ndbm, dbm.gnu, collections`提供了其他映射类型的例子。
    
* Callable types  
下边是函数调用操作可以接受的类型：

  * User-defined functions  
通过函数定义可以创建用户自定义的函数对象（查看函数定义）。在调用它的时候要按照函数定义的参数列表传递相同个数的参数。
        特殊属性：
        
        | Attribute | Meaning | |
        |-----------|---------|---|
        |`__doc__`|函数的文档字符串, 如果没有为`None`; 不会被子类继承|Writable|
        |`__name__`|函数的名字|Writable|
        |`__qualname__`|函数的`qualified name`，在3.3版中新增的|Writable|
        |`__module__`|函数所在的模块，如果没有就是`None`|Writable|
        |`__defaults__`|一个用来存放默认参数的元组，如果没有默认参值为`None`|Writable|
        |`__code__`|函数体经过编译后生成的代码对象|Writable|
        |`__globals__`|指向函数可调用的全局变量的引用-函数所在模块的全局名字空间|Read-only|
        |`__dict__`|名字空间，支持任意的函数属性|Writable|
        |`__closure__`|元组，含有函数自由变量绑定，如果没有自由变量，就为`None`|Read-only|
        |`__annotations__`|存储参数注解（annotations）的字典。键为参数名。如果有返回值，返回值的键为`return`|Writable|
        |`__kwdefaults__`|存储关键字参数默认值的字典|Writable|

  * Instance methods  
    实例方法对象由类、类的实例、可调用对象（通常是用户自定义函数）组合而成。  
    
        特殊的只读属性：`__self__`是类的实例对象；`__func__`是函数对象；`__doc__`是方法的文档（就像`__func__.__doc__`）；`__name__`是方法的名字（就像`__func__.__name__`）；`__module__`是方法所在的模块名字，如果没有就是`None`。
    
        方法也支持对底层函数对象任意属性的访问，但不支持设置。
      
        用户定义方法对象可以通过获取类属性（也可能是通过该类的一个实例）创建，但前提是这个属性是用户定义函数对象，或者类方法对象。
      
        通过检索一个类实例的用户自定义方法，可以创建实例方法对象`instance method object`。实例方法对象的属性`__self__`指向该类实例，这个方法称为是`被绑定的`。这个方法的属性`__func__`指向原来的方法对象。
      
        通过检索另外一个类或着实例的方法对象，可以创建用户自定义方法对象`user-defined method object`。这个操作类似创建方法对象，只不过新的实例等`__func__`属性不在指向原来的方法对象，而是自己的`__func__`。

        在类或实例中检索方法对象过程中生成的实例方法对象是，它的`__self__`属性就是类本身，它的`__func__`属性存放的函数对象就是类中的方法。  

        当一个实例的方法对象被调用的时候，它对应的底层函数（`__func__`）被调用，类实例（`__self__`）被插到参数列表的最前边。例如，类`C`由一个函数`f()`，`x`是`C`的一个实例，`x.f(1)`的行为其实就是`C.f(x, 1)`。  

        如果一个实例方法对象是由类方法对象中推导出来的，保存`类实例`的`__self__`其实就是类本身，所以调用`x.f(1)`或者`C.f(1)`都等于调用`f(C,1)`，f是底层函数。  

  * Generator functions  
生成器函数就是使用了yield语句（请查看yield语句）的函数或者方法。这样的一个函数被调用的时候会返回一个用来执行函数体的迭代器对象：调用这个迭代器的函数`iterator.__next__()`就会让这个函数执行，直到遇到yield语句返回一个值。
当函数遇到return语句或者执行到末尾，会抛出一个`StopIteration`，而且迭代器将会到达所有返回值的末尾。

  * Coroutine functions  
一个用`async def`来定义的函数或方法，叫做协同程序函数。这样的函数在调用时会返回一个协程对象。它可能包含`await`表达式，也可能是`async with`和`async for`语句。详情查看`协程对象`部分

  * Built-in functions  
内建函数是对C函数的一个包装。例如`len()`, `math.sin()`(`math`是一个标准的内建模块)。参数的类型和个数是在C函数中定义的。特殊的只读属性：`__doc__`是函数的文档字符串，如果没有就是`None`；`__name__`是函数的名字；`__self__`被设置为`None`(但是留意下部分)；`__module__`是函数所在模块的名字，如果没有那就是`None`。

  * Built-in methods  
这其实是一个内建函数的伪装，这次通过一个隐藏的参数将一个对象传给了C函数。例如内建函数`alist.append()`，其中`alist`是一个数组对象。在这个例子中，一个特殊的只读参数`__self__`被传递，它表示`alist`。

  * Classes  
类可以被调用。这样的对象就像是自己实例等工厂一样，但是通过覆盖`__new__()`可以改变这个行为。调用参数会被传递到`__new__()`, 然后在大部分情况下传递给`__init__()`来初始化一个实例。

  * Class Instances  
一个纯Python类的实例可以变成可被调用的，只要在类中实现了方法`__call__()`。

* Modules  
模块是Python代码的基本组织单位，它由导入系统`import system`生成，方法有使用import语句，或者调用类似`importlib.import_module()`的方法，或者内建函数`__import__()`。一个模块对象有一个由字典对象实现的名字空间（模块中定义的函数可以通过`__globals__`属性获得这个字典）。模块属性的访问被转换成查找这个字典，例如`m.x`等价于`m.__dict__[" x" ]`。模块对象不包含初始化该模块的代码对象 (因为初始化完成后就不再需要它了)。
对模块属性的赋值会更新模块的名字空间，例如`m.x = 1`等价于`m.__dict__["x"] = 1`。

    只读特殊属性`__dict__`就是模块名字空间的字典对象。

    预定义的可写属性：`__name__`是模块名；`__doc__`是模块的文档字符串或`None`。如果模块是由文件加载的，`__file__`是对应文件的路径名，用C语言编写的静态链接进解释器的模块没有这个属性，而对于从共享库加载的模块，这个属性的值就是共享库的路径。
    
* Custom classes  
自定义类类型一般是由类定义创建的。类的名字空间是由字典对象实现的。类的属性都是从这个字典对象中查找的，例如，`C.x`其实就是`C.__dict__["x"]`(另外还有很多种机制去定位属性)。当属性名没有在这里找到，会继续到基类中去查找属性。基类中的搜索方法使用C3方法解析顺序, 这种方法即便是多重继承里出现了公共祖先类的 “菱形” 结构也能保持正确行为. 关于Python使用的 C3 MRO 额外细节可以在 2.3 版本的附带文档中找到:  

    ```http://www.python.org/download/releases/2.3/mro/.```  
    
    当一个类 (假如是类 C) 的属性引用会产生类方法对象时, 它就会被转换成实例方法对象, 并将这个对象的`__self__`属性指向`C`. 当要产生静态方法对象时, 它会被转换成用静态方法对象包装的对象. 另一种获取与`__dict__`实际内容不同的属性的方法可以参考 实现描述符.

    类属性的赋值会更新类的字典, 而不是基类的字典.

    一个类对象可以被调用 (如上所述), 以产生一个类实例 (下述)。

    特殊属性：`__name__`是类名，`__module__`是类定义所在的模块名；`__dict__`是类的名字空间字典。`__bases__`是基类元组 (可能为空或独元)，基类的顺序以定义时基类列表中的排列次序为准。`__doc__`是类的文档字符串或者 None。
    

* Class instances  
类实例是用类对象调用创建的。类实例有一个用字典实现的名字空间，它是进行属性搜索的第一个地方。如果属性没在那找到，但实例的类中有那个名字的属性，就继续在类属性中查找。如果找到的是一个用户定义函数对象，它被转换成实例方法对象，这个对象的`__self__`属性指向实例本身。静态方法和类方法对象也会按上面`Classes`中的介绍那样进行转换。另一种获取与`__dict__`实际内容不同的属性的方法可以参考 实现描述符。如果没有找到匹配的类属性，但对象的类提供了`__getattr__()`方法，那么最后就会调用它完成属性搜索。  

    属性的赋值和删除会更新实例字典，而不是类的字典。如果类具有方法`__setattr__()`或者`__delattr__()`就会调用它们，而不是直接更新字典。  

    如果提供了相应特别方法的定义，类实例可以伪装成数值，序列或者映射类型，参见 特殊方法名。  

    特殊属性：`__dict__`是属性字典；`__class__`是实例的类。

* I/O objects (also known as file objects)  
一个文件对象表示一个打开的文件。创建一个文件对像有很多方法：内建函数`open()`，还有`os.popen()`, `os.fdopen()`，还有`socket`对象中的`makefile()`方法(还有一些扩展模块kennel也会提供一些函数和方法)。  

    对象`sys.stdin`，`sys.stdout`和`sys.stderr`被初始化为解释器相应的标准输入流，标准输出流和标准错误输出流。它们都以文本模式打开，因此都遵循抽象类`io.TextIOBase`定义的接口。

* Internal types  
有少量解释器内部使用的类型是用户可见的，它们的定义可能会在未来版本中改变，出于完整性的考虑这里也会提一下它们。

    * Code objects  
      代码对象表示`字节编译`过的可执行Python代码，或者称为`bytecode`。代码对象与函数对象的不同在于函数对象包含了函数全局变量的引用（所在模块定义的），而代码对象不包括上下文。默认参数值也保存在函数对象里，而不在代码对象中（因为它们表示的是运行时计算出来的值）。不像函数对象，代码对象是不可变的，并且不包括对可变对象的（直接或间接的）引用。  

        特殊的只读属性：`co_name`给出了函数名；`co_argcount`是位置参数的数目（包括有默认值的参数）；`co_nlocals`是函数使用的局部变量的数目（包括参数）。`co_varnames`是一个包括局部变量名的元组（从参数的名字开始）。`co_cellvars`是一个元组，包括由嵌套函数引用的局部变量名；`co_freevals`元组包括了自由变量的名字；`co_code`是字节编译后的指令序列的字符串表示；`co_consts`元组包括字节码中使用的字面值；`co_names`元组包括字节码中使用的名字；`co_ﬁlename`记录了字节码来自于什么文件；`co_ﬁrstlineno`是函数首行号；`co_lnotab`是一个字符串, 它表示从字节码偏移到行号的映射（细节可以在解释器代码中找到）；`co_stacksize`是需要的堆栈尺寸（包括局部变量）；`co_ﬂags`是一个表示解释器各种标志的整数。  

        `co_ﬂags`定义了如下标志位：如果函数使用了`*arguments`语法接收任意数目的位置参数就会把`0x04`置位；如果函数使用了`**keywords`语法接收任意数量的关键字参数，就会把`0x08`置位。如果函数是一个产生器（generator），就会置为`0x20`。

        Future功能声明`from __future__ import division`也使用了`co_flags`的标志位指出代码对象在编译时是否打开某些特定功能：如果函数是打开了`future division`编译的，就会把`0x2000`置位；之前版本的Python使用过位`0x10`和`0x1000`。  

        `co_flags`中其它位由解释器内部保留。  

        如果代码对象表示的是函数，那么`co_consts`的第一个项是函数的文档字符串，或者为`None`。
    
    * Frame objects  
      栈对象表示执行时的栈桢，它们会在回溯对象中出现（下述）。

        特殊的只读属性：属性`f_back`指向前一个栈（朝着调用者的方向），如果位于堆栈底部它就是`None`；属性`f_code`指向在这个栈结构上执行的代码对象。属性`f_locals`是用于查找局部变量的字典；属性`f_globals`字典用于查找全局变量；属性`f_builtins`字典用于查找内建名字；属性`lasti`以代码对象里指令字符串的索引的形式给出了精确的指令。  
      
        特殊的可写属性：属性`f_trace`如果不是`None`，就是这个栈所在函数的名称（用于调试器）。 属性`f_lineno`是此栈当前行的行号，在跟踪函数里如果写入这个属性，可以使程序跳转到新行上（只能用于最底部的栈桢），调试器可以这样实现跳转命令（即 “指定下一步” 语句）。
        
        栈对象支持一个方法：
        
        `frame.clear()`  
        
        这个方法将清除当前栈中所持有的所有本地变量的引用。另外，如果这个栈属于一个生成器，这个生成器也将结束。这有助与打破涉及栈对象的循环依赖（例如获取异常后对它的异常回溯进行排序以便之后使用）。  
        
        如果这个栈当前正在执行中，异常`RuntimeError`将被触发。

        3.4版新增。
    
    * Traceback objects  
      回溯对象是对一个异常的栈回溯。当异常发生时回创建这个对象。当我们在栈桢内搜索异常处理器时，每当要搜索一个栈桢就会把一个回溯对象会插入到当前回溯对象的前面。在进入异常处理器时，程序就可以使用回溯对象了。（参见`try`语句）。这些回溯对象可以通过 `sys.exc_info()`返回元组的第三项访问。当程序中没有适当的异常处理器，回溯对象就被打印到标准错误输出上。如果工作在交互模式上，也可以通过`sys.last_traceback`访问。  
      特殊的只读属性：`tb_text`是堆栈回溯的下一级（向着发生异常的那个栈桢），或者如果没有下一级就为`None`。属性`tb_frame`指向当前的栈桢对象；属性`tb_lineno`给出发生异常的行号；属性`tb_lasti`精确地指出对应的指令。如果异常发生在没有匹配`except`或`finally`子句的`try`语句中，回溯对象中的行号和指令可能与栈桢对象中的行号和指令不同。

    * Slice objects  
      切片对象用于在`__getitem__()`方法中表示切片信息，也可以用内建函数`slice()`创建。  
    
        特殊的只读属性：`start`是下界；`stop`是上界；`step`是步长，如果忽略任何一个，就取`None`值。这些属性可以是任意类型。  
        
        切片对象只支持一个方法：  
        
        `slice.indices(self, length)`  
        
        这个方法根据整数参数`length`判断切片对象是否能够描述`length`长的元素序列。它返回一个包含三个整数的元组，分别是索引`start`，`stop`和步长`step`。对于索引不足或者说越界的情况，返回值提供的是切片对象中能够提供的最大 (最小) 边界索引。
    
    * Static method objects  
    静态方法对象提供了一种避免上边说过情形的方法，即避免将函数对象转变成方法对象。静态方法对象其实是对任意对象的一个包装，通常是用户自定义函数。当从类或着类的实例中检索一个静态方法对象时，返回的其实就是一个包装过的对象，而且并没有经过之前任何转换。静态方法对象自己时不可调用的，尽管它们包装的都是可调用的。静态方法对象是由内建构造器`staticmethod()`创建的。

    * Class method objects  
    类方法对象，就像静态方法对象，是另外一个对象的包装，通过这个包装可以改变从类和泪实例中检索对象的行为。检索累方法对象的行为已经在上边描述过，在`User-defined methods`部分。类方法对象是由内建构造器`classmethod()`创建的。

# 3.特殊名字的方法

通过定义名字特殊的方法，一个类能够实现特殊语法所调用的操作（例如算术运算，下标及切片操作）。这是Python式的运算符重载`operator overloading`，让类可以定义自己的行为并充分利用语言中的操作符。例如，一个类如果定义了方法`__getitem__()`，然后`x`是这个类的实例，那么`x[i]`基本上就等于`type(x).__getitem__(x, i)`。除非特别标示，在没有适当定义方法的类上执行操作会导致抛出异常，一般是`AttributeError`或者`TypeError`。
 
在实现要模拟任意内建类型的类时，需要特别指出的是 “模拟” 只是达到了满足使用的程度，这点需要特别指出。例如，获取某些序列的单个元素是正常的，但使用切片却是没有意义的（一个例子是在W3C文档对象模型中的`NodeList`接口。）

#### 3.1 基础定制  
* object.\_\_new\_\_(cls[, ...])  
    调用时创建类`cls`的实例。`__new()`是一个静态方法（特殊函数所以你不需要定义），第一个参数是需要创建实例等类。其余的参数会创递给实例构造器（调用类）。`__new__()`的返回值是类的新对象实例（通常是`cls`的实例）。  

    常见的情况下是通过`super(currentclass, cls).__new__(cls[, ...])`的方式，传递对应的参数调用父类的`__new__()`的来实现创建一个类的新实例，然后再返回实例之前根据需求对实例进行相应的修改。  
    
    如果`__new__()`返回类`cls`的新实例，那么新实例的`__init__()`方法将会被调用`__init__(self[, ...])`，`self`就是新实例自己，其余参数就是传递给`__new__()`的参数。  
    
    如果`__new__()`返回的不是类`cls`的实例，那么新实例的`__init__`方法不会被调用。  
    
    `__new__`方法主要用于让不可变类型的子类（比如：int，str，tuple）可以定制化实例的创建。在自定义`metaclass`的时候，这个函数也经常被覆盖，以定制类的创建。
    
* object.\_\_init\_\_(self[, ...])  
    实例被`__new__()`创建以后，并在返回调用者之前被调用。参数就是那些传递给类构造表达式的值。如果基类有`__init__()`方法，推断类的`__init__()`方法，如果有，必须显示的调用它来确保新实例中基类的部分被正确的初始化；比如：`BaseClass.__init__(self, [args...])`。

    因为`__new__()`和`__init__()`在构建对象的时候共同工作（`__new__()`创建对象，`__init__()定制对象`），`__init__()`不会返回任何值；这样做会在运行时导致`TypeError`异常。
    
* object.\_\_del\_\_(self)  
    当实例要被销毁的时候被调用。这个方法也可以被叫做销毁器。如果基类有`__del__()`方法，推断类里是否有`__del__()`方法，如果有，必须显式调用这个函数确保实例中基类部分被正确的销毁。注意：在`__del__()`里可以创建本对象的新引用来达到推迟删除的目的，但这并不是推荐做法。`__del__()`方法在删除最后一个引用后不久调用。但不能保证，在解释器退出时所有存活对象的`__del__()`方法都能被调用。  
    
    ```
    Note：`del x`并不直接调用`x.__del__()`——— 前者将引用计数减一，而后者只有在引用计数减到零时才被调用。引用计数无法达到零的一些常见情况有：对象之间的循环引用 （例如，一个双链表或一个具有父子指针的树状数据结构)；对出现异常的函式的栈桢上对象的引用`(sys.ext_info()[2]`中的回溯对象保证了栈桢不会被删除）；或者交互模式下出现未拦截异常的栈桢上的对象的引用（`sys.last_traceback`中的回溯对象保证了栈桢不会被删除）。第一种情况只有能通过地打破循环才能解决。后两种情况，可以通过将 `sys.last_traceback`赋予`None`解决。只有在打开循环检查器选项时（这是默认的），循环引用才能被垃圾回收机制发现，但前提是Python脚本中的`__del__()`方法不要参与进来。关于`__del__()`与循环检查器是如何相互影响的详细信息，可以参见`gc`模块的介绍，尤其是其中的 garbage 值的描述。
    ```  
      
    ```
    Warning：因为调用`__del__()`方法时环境的不确定性，它执行时产生的异常会被忽略掉，只是在`sys.stderr`打印警告信息。另外，当因为删除模块而调用`__del__()`方法时（例如，程序退出时），有些`__del__()`所引用的全局名字可能已经删除了，或者正在删除（例如，正在清理`import`关系）。 由于这些原因，`__del__()`方法对外部不变式的要求应该保持最小。从Python1.5开始，Python可以保证以单下划线开始的全局名字一定在其它全局名字之前从该模块中删除，如果没有其它对这种全局名字的引用，这个功能有助于保证导入的模块在调用`__del__()`时还是有效的。
    ```
    
* object.\_\_repr\_\_(self)  
    有内建函数`repr()`调用，来获得一个对象的“正式的”表达字符串。如果可能的话，这个字符串是一个可用的python表达式，可以用来重新创建一个具有相同值的对象。如果不可能，会返回一个格式为`<...有用的描述...>`的字符串。返回值必须是一个字符串对象。如果一个类只定义了`__repr__()`而没有`__str__()`，`__repr__()`会同样用来获得一个这个类的实例的“非正式”表达字符串。

    这个通常用于调试，所以描述字符串的信息丰富性和无歧义性是很重要的.
    
* object.\_\_str\_\_(self)  
    由`str(object)`调用，也可以由内建函数`format()`和`print()`调用，来得到一个`非正规的`、容易打印的、表示这个对象的字符串。返回值必须是一个字符串对象。  

    这个方法与`object.__repr__()`的不同之处在于，`__str__()`不一定会返回一个可用的python表达式，返回的是更简单易用的表示。  
        
    默认的实现是由内建类型`object`定义的：调用`object.__repr__()`。  
    
* object.\_\_bytes\_\_(self)  
    由函数`bytes()`调用，来得到一个用来表示对像的一个字节字符串。这会返回一个字节对像。
      
* object.\_\_format\_\_(self, format_spec)  
    由内建函数`format()`（和`str`类的方法`format()`）调用，用来构造对象的“格式化”字符串描述。`format_spec`参数是描述格式选项的字符串。`format_spec`的解释依赖于实现`__format__()`的类型，但一般来说，大多数类要么把格式化任务委托（转交）给某个内建类型，或者使用与内建类型类似的格式化选项。

    关于标准格式语法的描述，可以参考`Format Specification Mini-Language`。

    返回值必须是字符串对象。
    ```
    在Python 3.4版发生的变化：
    如果传递任意的非空字符给`__format__`方法，会产生`TypeError`异常。
    ```
        
* object.\_\_lt\_\_(self, other)  
  object.\_\_le\_\_(self, other)  
  object.\_\_eq\_\_(self, other)  
  object.\_\_ne\_\_(self, other)  
  object.\_\_gt\_\_(self, other)  
  object.\_\_ge\_\_(self, other)  
    它们称为“富比较”方法。运算符与方法名的对应关系如下：`x<y`调用`x.__lt__(y)`，`x<=y`调用`x.__le__(y)`，`x==y`调用`x.__eq__(y)`，`x!=y`调用`x.__ne__(y)`，`x>y`调用`x.__gt__(y)`，`x>=y`调用`x.__ge__(y)`。  
        
    不是所有厚比较方法都要同时实现的，如果个别厚比较方法没有实现，可以直接返回 NotImplemented。从习惯上讲，一次成功的比较应该返回`False`或`True`。但是这些方法也可以返回任何值，所以如果比较运算发生在布尔上下文中（例如`if`语句中的条件测试），Python会在返回值上调用函式`bool()`确定返回值的真值。

    在比较运算符之间并没有潜在的相互关系。`x==y`为真并不意味着`x!=y`为假。因此，如果定义了方法`__eq__()`，那么也应该定义`__ne__()`，这样才可以得到期望的效果。关于如何创建可以作为字典键使用的`hashable`对象，还需要参考`__hash__()`的介绍。  

    没有参数交换版本的方法定义（这可以用于当左边参数不支持操作，但右边参数支持的情况）。`__lt__()`和`__gt__()`相互反射（即互为参数交换版本）；`__le__()`和`__ge__()`相互反射；`__eq__()`和`__ne__()`相互反射。  

    传递给厚比较方法的参数不能是被自动强制类型转换的（coerced）。  

    关于如何从一个根操作自动生成顺序判定操作，可以参考 `functools.total_ordering()`。  

* object.\_\_hash\_\_(self)  
    由内建函式`hash()`，或者是在可散列集合（`hashed collections`，包括 set，frozenset，dict）成员上的操作调用。这个方法应该返回一个整数。只有一个要求，具有相同值的对象应该有相同的散列值。应该考虑以某种方式（例如排斥或）把在对象比较中起作用的部分与散列值关联起来。  

    如果类没有定义`__eq__()`方法，那么它也不应该定义`__hash__()`方法；如果一个类只定义了`__eq__()`方法，那么它是不适合作散列键的。如果可变对象实现了`__eq__()`方法，它也不应该实现`__hash__()`方法，因为可散列集合要求键值是不可变的(如果对象的散列值发生了改变，它会被放在错误的桶（bucket）中）。  

    所有用户定义类默认都定义了方法`__eq__()`和`__hash__()`，这样，所有对象都可以进行相等比较（除了与自身比较）。`x.__hash__()`返回`id(x)`。

    如果子类从父类继承了方法`__hash__()`，但修改了`__eq__()`，这时子类继承的散列值就不再正确了（例如，可能从默认的标识相等的比较切换成了值相等的比较），这时在在类定义时显式地将`__hash__()`设置成`None`就行了。这样，在使用这个子类对象作为散列键时就会抛出`TypeError`异常，或者也可以使用常规的可散列检查`isinstance(obj, collections.Hashable)`确定它的不可散列性（使用这种检查方式时，这种达到不可散列的方法与在`__hash__()`中显式抛出异常的方法是的计算结果就不一样了）。

    如果子类修改了`__eq__()`方法，但需要保留父类的`__hash__()`，它必须显式地告诉解释器`__hash__ = <ParentClass>.__hash__`，否则`__hash__()`的继承会被阻止，就像设置`__hash__`为`None`。

* object.\_\_bool\_\_(self)  
    在实现真值测试和内建操作`bool()`中调用，应该返回`False`和`True`。如果没有这个定义方法，转而使用`__len__()`。非零返回值，当作“真”。如果这两个方法都没有定义，就认为该实例为“真”。
        
#### 3.2 定制访问属性  
以下方法可以用于定制访问类实例属性的含义（例如，赋值，或删除`x.name`）

* object.\_\_getattr\_\_(self, name)  
    在正常方式访问属性无法成功时（就是说，self属性既不是实例的，在类树结构中找不到）使用。`name`是属性名。应该返回一个计算好的属性值，或抛出一个`AttributeError`异常。

    注意，如果属性可以通过正常方法访问，`__getattr__()`是不会被调用的（是有意将`__getattr__()`和`__setattr__()`设计成不对称的）。这样做的原因是基于效率的考虑，并且这样也不会让`__getattr__()`干涉正常属性。注意，至少对于类实例而言，不必非要更新实例字典伪装属性（但可以将它们插入到其它对象中）。需要全面控制属性访问，可以参考以下`__getattribute__()`的介绍。
        
* object.\_\_getattribute\_\_(self, name)  
    在访问类实例的属性时无条件调用这个方法。如果类也定义了方法 `__getattr__()`，那么除非`__getattribute__()`显式地调用了它，或者抛出了`AttributeError`异常，否则它就不会被调用。这个方法应该返回一个计算好的属性值，或者抛出异常`AttributeError`。为了避免无穷递归，对于任何它需要访问的属性，这个方法应该调用基类的同名方法，例如，`object.__getattribute__(self, name)`。  
        
    ```
    Note：但是，通过特定语法或者内建函式，做隐式调用搜索特殊方法时，这个方法可能会被跳过。参见 搜索特殊方法。
    ```
        
* object.\_\_setattr\_\_(self, name, value)  
    在属性要被赋值时调用。这会替代正常机制（即把值保存在实例字典中）。`name`是属性名，`vaule`是要赋的值。

    如果在`__setattr__()`里要对一个实例属性赋值，它应该调用父类的同名方法，例如，`object.__setattr__(self, name, value)`。
    
* object.\_\_delattr\_\_(self, name)  
    与`__setattr__()`类似，但它的功能是删除属性。当`del obj.name`对对象有意义时，才需要实现它。
    
* object.\_\_dir\_\_(self)  
    在对象上调用`dir()`时调用，它需要返回一个序列。`dir()`会将返回的序列转成一个列表，并进行排序。
        
##### 3.2.1 实现描述符  

含有所有者类`owner class`中的方法的类被叫做描述符类，下边的方法只会出现在描述符类的实例中。描述符要么在所有者的类字典中，或者在他们父类的类字典中。在下面的例子里，“属性”指名字出现在所有者类`__dict__`中的属性。

* object.\_\_get\_\_(self, instance, owner)
    调用这个方法可以获得所有者类中的属性（获取类属性）也可以获取类的实例的属性（获取实例属性）。`owner`是所有者类，当`instance`是`None`时从`owner`中获取属性，当`instance`是一个实例时从`instance`中获取属性。这个方法应该返回属性值，或者产生`AttributeError`异常。
        
* object.\_\_set\_\_(self, instance, value)  
    调用这个方法可以为所有者类的实例`instance`的属性设置新值`value`。
        
* object.\_\_delete\_\_(self, instance)  
    调用这个方法来删除所有者类实例`instance`中的属性。
        
##### 3.2.2 使用描述
##### 3.2.3 \_\_slots\_\_
默认情况下，类实例使用字典管理属性。在对象只有少量实例变量时这就会占用不少空间。当有大量实例时，空间消耗会变得更为严重。

这个默认行为可以通过在类定义中定义`__slots__`修改。`__slots__`声明只为该类的所有实例预留刚刚够用的空间。因为不会为每个实例创建`__dict__`，因此空间节省下来了。

* object.\_\_slots\_\_  
    这个类变量可以赋值为一个字符串，一个可迭代对象。或者一个字符串序列（每个字符串表示实例所用的变量名）。如果定义了`__slots__`，Python就会为实例预留出存储声明变量的空间，并且不会为每个实例自动创建`__dict__`和`__weakref__`。  


###### 3.2.3.1 使用\_\_slots\_\_的注意事项  

* 如果从一个没有定义`__slots__`的基类继承，子类一定存在`__dict__`属性。所以，在这种子类中定义`__slots__`是没有意义的。
* 没有定义`__dict__`的实例不支持对不在`__slot__`中的属性赋值。如果需要支持这个功能, 可以把`__dict__`放到`__slots__`声明中。
* 没有定义`__weakref__`的，使用`__slot__`的实例不支持对它的 “弱引用”。如果需要支持弱引用，可以把`__weakref__`放到`__slots__`声明中。
* `__slots__`是在类这一级实现的，通过为每个实例创建描述符 (实现描述符)。因此，不能使用类属性为实例的`__slots__`中定义的属性设置默认值。否则，类属性会覆盖描述符的赋值操作。
* `__slots__`声明的行为只限于其定义所在的类。因此，如果子类没有定义自己的`__slots__`(它必须只包括那些 额外的`slots`)，子类仍然会使用`__dict__`。
* 如果类定义的`slot`与父类中的相同，那么父类`slot`中的变量将成为不可访问的 (除非直接从基类中获取描述符)。这使得程序行为变得有一点模糊，以后可能会增加一个防止出现这种情况的检查。
* 从“变长”内建类型，例如`int`、`str`和`tuple`继承的子类的非空`__slots__`不会起作用。
* 任何非字符串可迭代对象都可以赋给`__slots__`。映射类型也是允许的，但是，以后版本的Python可能给 “键” 赋予特殊意义。
* 只有在两个类的`__slots__`相同时，`__class__`赋值才会正常工作。

#### 3.3 定制类的生成  
默认情况下，类是由`type()`来构建的。类的实体是在新的名字空间中执行，而且类的名字是与`type(name, bases, namespace)`的结果绑定的。  
类的创建过程可以通过在类声明的时候指定`metaclass`定制，也可以通过继承已有的、包含这个参数的类来实现。在下边的例子中`MyClass`和`MySubclass`都是`Meta`的实例：  

```  
class Meta(type):
    pass

class MyClass(metaclass=Meta):
    pass

class MySubclass(MyClass):
    pass  
```

其他在类定义中使用到的关键字变量，都会被传递到下边描述的所有的`metaclass`操作中。  
当一个类定义执行时，会执行如下操作：  


* 选择合适的metaclass  
* 准备类的名字空间  
* 执行类的内容（body）  
* 创建类对象  

    
##### 3.3.1 选择合适的`Metaclass`  

下面定义了什么样的的`metaclass`对于一个类定义来说是合适的  
       
* 如果没有定义基类，也没有特别指定`metaclass`，那么就使用`type()`  
* 如果给定一个`metaclass`，而且它不是`type()`的实例，那么它就被直接用作`metaclass`  
* 如果给定一个`type()`的实例作为`metaclass`，或者定义了基类, 那就使用最底层的`metaclass`  

底层`metaclass`是从给定的`metaclass`和所有基类的metaclass中选出的。底层`metaclass`是所有候选`metaclass`的子类型。如果候选`metaclass`中没有一个满足条件，那么类定义就会失败，并抛出`TypeError`异常。

##### 3.3.2 准备类的名字空间  

一旦合适的`metaclass`被确认后，类的名字空间就准备好了。如果`metaclass`有`__prepare__`属性，就执行`namespace = metaclass.__prepare__(name, bases, **kwds)`(其中的关键字参数，是来自类定义里的)。
        
如果`metaclass`没有`__prepare__`属性，那么类的名字空间就被初始化为一个空的字典实例。

```
See also
PEP 3115 - Metaclasses in Python 3000
Introduced the __prepare__ namespace hook
```
         
##### 3.3.3 执行类的内容（body）  

类的`body`是以类似`exec(body, globals(), namespace)`的方式执行的。与普通调用`exec()`的主要区别在于，当类定义发生在一个函数内部时，词法作用域允许类的`body`（包括任何的函数）引用任何当前或者外部作用域的名称。

尽管里的定义发生在`exec`函数里，但是类中定义的方法仍然无法看到类中的其它名字。如果要使用类的变量必须通过实例或类方法的第一个参数，而且不能被静态静态方法使用。
    
##### 3.3.4 创建类对象  

当类的内容被执行，并构成了类的名字空间后，类对象就通过调用`metaclass(name, bases, namespace, **kwds)`创建好了(传递到这里的关键字变量与传递到`__prepare__`中的一样)。

一个类对象的引用保存在不含参的`super().__class__`里，它是一个由编译器创建的隐藏的闭包，任何类`body`中的方法都可以被`__class__`或者`super`引用。这样的话，如果当前的调用需要使用传递到方法中的第一个参数来识别类或对象时，执行调用的类或者实例这样无参的`super()`就可以正确的识别那些在词法作用域定义的类。
        
当类对象被创建后，它被传递给类中的的装饰器，作为结果的类对象会被绑定在本地名字空间中作为定义好的类。
        
当一个新的类通过`type.__new__`被创建，新的类对象将名字空间参数复制到一个标准的Python字典中，然后将原始的类对象抛弃。新复制的字典成为类对象的`__dict__`属性。
        
##### 3.3.5 `Metaclass`实例  
`metaclass`的使用方式是五花八门的。有些方法已经被验证过了比如，日志、接口检查、自动授权、自动创建参数、代理、框架、自动资源锁／同步。  

这里给出一个`metaclass`的例子，使用`collections.OrderedDict`来记录类中的变量被创建的顺序。  
    
```
class OrderedClass(type):  
    @classmethod  
    def __prepare__(metacls, name, bases, **kwds):  
        return collections.OrderedDict()  

    def __new__(cls, name, bases, namespace, **kwds):  
        result = type.__new__(cls, name, bases, dict(namespace))  
        result.members = tuple(namespace)  
        return result  

class A(metaclass=OrderedClass):  
    def one(self): pass  
    def two(self): pass  
    def three(self): pass  
    def four(self): pass  

>>> A.members  
('__module__', 'one', 'two', 'three', 'four')  
```  
    
当执行`A`的类定义时，首先执行`metaclass`的`__prepare__()`方法，这个方法返回一个空的`collections.OrderedDict`。这个字典会记录类`A`中定义的方法和属性。当这些定义执行完，这个有序字段就成为类的一部分，然后`__new__()`开始被调用。这个方法会创建新的`type`类型，将有序字典的`key`保存到这个类型的属性`memebers`中。  

#### 3.4 定制实例和子类的检查  
以下函数可以用于定制内建函式`isinstance()`和`issubclass()`的默认行为。  

具体地，元类`abc.ABCMeta`实现了这些方法，以允许把抽象基类（ABC）作为任何类或者类型（包括内建类型），也包括其他ABC的“虚拟基类”。

* class.\_\_instancecheck\_\_(self, instance)  
    如果`instance`应该被看作`class`的一个直接或者间接的实例，就返回真。如果定义了这个方法, 它就会被用于实现`isinstance(instance, class)`。
* class.\_\_subclasscheck\_\_(self, subclass)  
    如果`subclass`应该被看作是`class`的一个直接或者间接的基类，就返回真。如果定义了这个方法，它就会被用于实现`issubclass(subclass, class)`。

注意，这个方法会在一个类的类型（元类）中查找。它们不能作为一个类方法在类中定义。这个机制，与在实例上查找要调用的特殊方法的机制是一致的，因为在这个情况下实例本身就是一个类。
        
```
See also:
PEP 3119 - Introducing Abstract Base Classes　 (抽象基类介绍)
包括定制通过 __instancecheck__() 和 __subclasscheck__()　定制 isinstance() 和 issubclass() 行为的规范，以及在语言上增加抽象基类 (见 abc 模块) 这个背景设置这个功能的动机。
```

#### 3.5 模拟可调用对象
* object.\_\_call\_\_(self[, args...])  
    当实例作为函式使用时调用本方法。如果定义了这个方法，`x(arg1, arg2,...)`就相当于`x.__call__(arg1, arg2,...)`。

#### 3.6 模拟容器类型
* object.\_\_len\_\_(self)  
* object.\_\_length_hint\_\_(self)  
* object.\_\_getitem\_\_(self, key)  
* object.\_\_missing\_\_(self, key)  
* object.\_\_setitem\_\_(self, key, value)  
* object.\_\_delitem\_\_(self, key)  
* object.\_\_iter\_\_(self)  
* object.\_\_reversed\_\_(self)  
* object.\_\_contains\_\_(self, item)

#### 3.7 模拟数字类型
以下方法用于模拟数值类型。不同数值类型所支持的操作符并不完全相同，如果一个类型不支持某些操作符（例如非整数值上的位运算），对应的方法就不应该被实现。

* object.\_\_add\_\_(self, other)  
object.\_\_sub\_\_(self, other)  
object.\_\_mul\_\_(self, other)  
object.\_\_matmul\_\_(self, other)  
object.\_\_truediv\_\_(self, other)  
object.\_\_floordiv\_\_(self, other)  
object.\_\_mod\_\_(self, other)  
object.\_\_divmod\_\_(self, other)  
object.\_\_pow\_\_(self, other[, modulo])  
object.\_\_lshift\_\_(self, other)  
object.\_\_rshift\_\_(self, other)  
object.\_\_and\_\_(self, other)  
object.\_\_xor\_\_(self, other)  
object.\_\_or\_\_(self, other)  
    这些方法用于二元算术操作（+，-，*，/，//，%，divmod()，pow()，**，<<，>>，&，^，|）。 例如，在计算表达式`x + y`时,`x`是一个定义了`__add__()`的类的实例，那么就会调用`x.__add__(y)`。方法`__divmod__()`应该与使用 `__floordiv__()`和`__mod__()`的结果相同，但应该与`__truediv__()`无关。注意如果要支持三参数版本的内建函式`pow()`的话，方法`__pow__()`应该被定义成可以接受第三个可选参数的。

    如果以上任一方法无法处理根据参数完成计算的话，就应该返回`NotImplemented`。

* object.\_\_radd\_\_(self, other)  
object.\_\_rsub\_\_(self, other)  
object.\_\_rmul\_\_(self, other)  
object.\_\_rmatmul\_\_(self, other)  
object.\_\_rtruediv\_\_(self, other)  
object.\_\_rfloordiv\_\_(self, other)  
object.\_\_rmod\_\_(self, other)  
object.\_\_rdivmod\_\_(self, other)  
object.\_\_rpow\_\_(self, other)  
object.\_\_rlshift\_\_(self, other)  
object.\_\_rrshift\_\_(self, other)  
object.\_\_rand\_\_(self, other)  
object.\_\_rxor\_\_(self, other)  
object.\_\_ror\_\_(self, other)  
    这些方法用于实现二元算术操作（+，-，*，/，//，%，divmod()，pow()，**，<<，>>，&，^，|），但用于操作数反射（即参数顺序是相反的）。这些函式只有在左操作数不支持相应操作，并且是参数是不同类型时才会被使用。例如，计算表达式`x - y`，`y`是一个定义了方法`__rsub__()`的类实例，那么在 x.__sub__(y) 返回`NotImplemented`时才会调用`y.__rsub__(x)`。

    注意三参数版本的`pow()`不会试图调用`__rpow__()`。（这会导致类型自动转换规则过于复杂）

* object.\_\_iadd\_\_(self, other)  
object.\_\_isub\_\_(self, other)  
object.\_\_imul\_\_(self, other)  
object.\_\_imatmul\_\_(self, other)  
object.\_\_itruediv\_\_(self, other)  
object.\_\_ifloordiv\_\_(self, other)  
object.\_\_imod\_\_(self, other)  
object.\_\_ipow\_\_(self, other[, modulo])  
object.\_\_ilshift\_\_(self, other)  
object.\_\_irshift\_\_(self, other)  
object.\_\_iand\_\_(self, other)  
object.\_\_ixor\_\_(self, other)  
object.\_\_ior\_\_(self, other)  
    这些方法用于实现参数化算术赋值操作（+=，-=，*=，/=，//=，%=，**=，<<=，>>=，&=，^=，|=）。这些方法应该是就地操作的（即直接修改`self`）并返回结果（一般来讲，这里应该是对`self`直接操作，但并不是一定要求如此）。如果没有实现某个对应方法的话，参数化赋值会蜕化为正常方法。例如，执行语句`x += y`时，`x`是一个实现了 `__iadd__()`方法的实例，`x.__iadd__(y)`就会被调用。如果`x`没有定义 `__iadd__()`，就会选择`x.__add__(y)`或者`y.__radd__(x)`，与`x + y`类似。  

* object.\_\_neg\_\_(self)  
object.\_\_pos\_\_(self)  
object.\_\_abs\_\_(self)  
object.\_\_invert\_\_(self)  
    用于实现一元算术操作（-，+，`abs()`和~）。  

* object.\_\_complex\_\_(self)  
object.\_\_int\_\_(self)  
object.\_\_float\_\_(self)  
object.\_\_round\_\_(self[, n])  
    用于实现内建函式`complex()`，`int()`，`float()`和`round()`。应该返回对应的类型值。  

* object.\_\_index\_\_(self)  
    用于实现函式`operator.index()`，或者在Python需要一个整数对象时调用（例如在分片时（slicing），或者在内建函式函式`bin()`，`hex()`和`oct()`中）。这个方法必须返回一个整数。  

#### 3.8 With语句和上下文管理器  
上下文管理器`context manager`是一个对象，这个对象定义了执行`with`语句时要建立的运行时上下文。上下文管理器负责处理执行某代码块时对应的运行时上下文进入和退出。运行时上下文的使用一般通过`with`语句（参见`with`语句），但也可以直接调用它的方法。  

上下文管理器的典型用途包括保存和恢复各种全局状态，锁定和解锁资源，关闭打开的文件等等。  

关于上下文管理的更多信息，可以参考 Context Manager Types。  
    
* object.\_\_enter\_\_(self)  
    进入与这个对象关联的运行时上下文。`with`语句会把这个方法的返回值与`as`子句指定的目标绑定在一起 (如果指定了的话)。
        
* object.\_\_exit\_\_(self, exc_type, exc_value, traceback)  
    退出与这个对象相关的运行时上下文。参数描述了导致上下文退出的异常，如果是无异常退出，则这三个参数都为`None`。  
        
    如果给出了一个异常，而这个方法决定要压制它（即防止它把传播出去），那么它应该返回真。否则，在退出这个方法时，这个异常会按正常方式处理。

    注意方法`__exit__()`不应该把传入的异常重新抛出，这是调用者的责任。
        
```
See also
PEP 0343 - The “with” statement（“with” 语句）
Python with 语句的规范，背景和例子。
```

#### 3.9 搜索特殊方法  
对于定制类，只有在对象类型的字典里定义好，才能保证成功调用特殊方法。这是以下代码发生异常的原因：
    
```
>>> class C:
...     pass
...
>>> c = C()
>>> c.__len__ = lambda: 5
>>> len(c)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: object of type 'C' has no len()
```
    
这个行为的原因在于，有不少特殊方法在所有对象中都得到了实现，例如`__hash__()`和`__repr__()`。如果按照常规的搜索过程搜索这些方法，在涉及到类型对象时就会出错。  
    
```
>>> 1.__hash__() == hash(1)
True
>>> int.__hash__() == hash(int)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: descriptor '__hash__' of 'int' object needs an argument
```
    
试图以这种错误方式调用一个类的未绑定方法有时叫作`metaclass confusion`，可以通过在搜索特殊方法时跳过实例避免：
    
```
>>> type(1).__hash__(1) == hash(1)
True
>>> type(int).__hash__(int) == hash(int)
True
```
    
除了因为正确性的原因而跳过实例属性之外，特殊方法搜索也会跳过`__getattribute__()`方法，甚至是元类中的：
    
```
>>> class Meta(type):
...    def __getattribute__(*args):
...       print("Metaclass getattribute invoked")
...       return type.__getattribute__(*args)
...
>>> class C(object, metaclass=Meta):
...     def __len__(self):
...         return 10
...     def __getattribute__(*args):
...         print("Class getattribute invoked")
...         return object.__getattribute__(*args)
...
>>> c = C()
>>> c.__len__()                 # Explicit lookup via instance
Class getattribute invoked
10
>>> type(c).__len__(c)          # Explicit lookup via type
Metaclass getattribute invoked
10
>>> len(c)                      # Implicit lookup
10
```
    
这种跳过`__getattribute__()`的机制为解释器的时间性能优化提供了充分的余地，代价是牺牲了处理特殊方法时的部分灵活性（为了保持与解释器的一致，特殊方法必须在类对象中定义）。
    
# 4. Coroutines

###### 4.1. 可等待对象
一个可等待对象一般会实现方法`__await__()`。由`async def`函数返回的`coroutine`对象都是可等待的。  
								
```
注意：被`types.coroutine()`或者`asyncio.coroutine()`修饰的，由生成器创建的迭代器对象也是可等待的。但是它并没有实现`__await__()`
```

* object.__await__(self)  
    必须返回一个迭代器。用来实现可等待对象。对于实例，`asyncio.Future`实现了这个方法，来兼容等待表达式。  
    
在Python 3.5版中中新增。  
    
    ```
    See also PEP 492 for additional information about awaitable objects.
    ```
    
###### 4.2. 协程对象

协程对象都是可等待对象。一个协程的执行是可以通过`__await__()`和遍历结果集来控制的。当协程结束并返回时，迭代器抛出`StopIteration`异常，异常的值属性就是这个返回值。如果协程抛出异常，它会通过迭代器传播。协程不应该直接抛出未经捕获的`StopIteration`异常。  

就像生成器一样，协程还有下边这些方法（详情查看，生成器-迭代器方法）。但是和生成器不一样，协程不能直接支持迭代操作。

在Python 3.5.2中的修改：`RuntimeError`会多次等待协程。

* coroutine.send(value)  
    开始或者恢复协程的运行。如果`value`为`None`。如果`value`不是`None`，这个方法会调用导致这个协程挂起的迭代器的`send()`方法。结果（返回值、`StopIteration`、或其它异常）与迭代时`__await__()`的返回值时一样的。    
    
* coroutine.throw(type[, value[, traceback]])  
    在协程中抛出指定异常。这个方法会调用造成协程挂起的迭代器的`throw`方法，如果这个迭代器有这个方法。否则就在挂起位置抛出异常。结果（返回值、`StopIteration`、或其它异常）与迭代时`__await__()`的返回值时一样的。如果协程中没有捕获异常，它会传递给调用者。
    
* coroutine.close()  
    调用这个方法将导致协程被清理并退出。如果协程已被挂起，要先执行造成协程挂起的迭代器的`close`函数，如果迭代器有这个函数的话。然后它会在挂起位置抛出`GeneratorExit`异常，这会造成协程立刻清理自己。最后，这个协程被标记为已完成，哪怕它从来没有执行过。

###### 4.3. 异步迭代器
一个异步迭代器可以调用在实现它的`__aiter__`时，调用异步代码，一个异步迭代器可以在它的`__anext__`方法里调用异步代码。

异步迭代器可以在`async for`语句中使用。

* object.__aiter__(self)  
    必须返回一个异步迭代器对象。

* object.__anext__(self)  
    迭代器返回的下一个值必须是一个可等待的，当迭代操作结束时会抛出一个`StopAsyncIteration`错误。
    
异步迭代器的例子

```
class Reader:
    async def readline(self):
        ...

    def __aiter__(self):
        return self

    async def __anext__(self):
        val = await self.readline()
        if val == b'':
            raise StopAsyncIteration
        return val
```
在Python 3.5版中中新增。




###### 4.4. 异步上下文管理器  

异步上下文管理器可以在它的`__aenter__`和`__aexit__`方法中延迟执行。

异步上下文管理器可以在`async with`语句中使用。

* object.__aenter__(self)  
    这个方法在语义上与`__enter__()`类似，唯一的区别就是它必须返回一个可等待对象。
    
* object.__aexit__(self, exc_type, exc_value, traceback)  
    这个方法在语义上与`__exit__()`类似，唯一的区别就是它必须返回一个可等待对象。
    
异步上下文管理器的例子

```
class AsyncContextManager:
    async def __aenter__(self):
        await log('entering context')

    async def __aexit__(self, exc_type, exc, tb):
        await log('exiting context')
```

在Python 3.5版中中新增。  

```
注意：
在Python 3.5.2版中的修改: 从CPython 3.5.2开始，__aiter__ 可以返回异步迭代器。返回一个可等待对象会导致 PendingDeprecationWarning 。

在CPython 3.5.x中为了保证向后兼容，推荐继续返回可等待对象在 __aiter__ 中。如果你想避免 PendingDeprecationWarning 同时也想保持向后兼容，可以用下边的装饰器：

import functools
import sys

if sys.version_info < (3, 5, 2):
    def aiter_compat(func):
        @functools.wraps(func)
        async def wrapper(self):
            return func(self)
        return wrapper
else:
    def aiter_compat(func):
        return func

例子：

class AsyncIterator:

    @aiter_compat
    def __aiter__(self):
        return self

    async def __anext__(self):
        ...
	
从CPython 3.6开始，PendingDeprecationWarning 将被 DeprecationWarning 替换。在CPython 3.7中，从 __aiter__ 返回一个可等待对象将会导致 RuntimeError。

```

----
半抄袭，半翻译，一点点完善把。  
其实新版文档已经更新了很多内容，抄袭过来的东西反而有可能有问题。  
而自己翻译的内容～～～反正metaclass那部分，有几段我自己都看晕了。  
不管怎么说，至少是帮助自己深入的了解了一下Python。  
自己凑合着看吧。  
https://docs.python.org/3/reference/datamodel.html  
http://docspy3zh.readthedocs.io/en/latest/reference/datamodel.html