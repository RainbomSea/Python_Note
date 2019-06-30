# 序列构成的数组

## 内置序列类型

**Python** 标准库用C实现了丰富的序列类型.举例如下.

* 容器序列
    
    `list`、`tuple`、和 `collections.deque` 这些序列能够存放不同类型的数据
    
* 扁平数据

    `str`、'bytes'、`bytearry`、`memoryview` 和 `array.array`,这类序列只能容纳一种类型
    
容器序列存放的是它们所包含的任意类型的对象的引用，而扁平序列里存放的是值而不是引用。换句话说，扁平序列其实是一段连续的内存空间。由此可见扁平序列其实更加紧凑，但是它里面只能存放诸如字符、字节和数值这种基础类型。

序列类型还能按照能否被修改来分类。 

* 可变序列
    
    `list`、`bytearray`、`array.array`、`collections.deque`和`memoryview`。

* 不可变序列

    `tuple`、`str` 和 `bytes`

图02-1 显示了可变序列（`MutableSequence`）和不可变序列（`Sequence`）的差异，同时也能看出前者从后者那里继承了一些方 法。虽然内置的序列类型并不是直接从`Sequence`和 `MutableSequence`这两个抽象基类（`Abstract Base Class`，`ABC`）继承而来的，但是了解这些基类可以帮助我们总结出那些完整的序列类型包含了哪些功能。

![Z1b5pn.png](https://s2.ax1x.com/2019/06/30/Z1b5pn.png)

> 图02-1: 这个`UML`类图列举了`collections.abc`中的几个类（超类在左边，箭头从子类指向超类，斜体名称代表抽象类和抽象方法）

## 列表推导和生成器表达式 

列表推导是构建列表（`list`）的快捷方式，而生成器表达式则可以用来创建其他任何类型的序列

举例: 字符串变成 `Unicode` 码位:

```py
>>> symbols = '$¢£¥€¤' 
>>> codes = [ord(symbol) for symbol in symbols] 
>>> codes
[36, 162, 163, 165, 8364, 164] 
```

列表推导不会再有变量泄漏的问题 

**Python 2.x** 中，在列表推导中 `for` 关键词之后的赋值操作可能会影响列表推导上下文中的同名变量。
像下面这个**Python 2.7**控制台对话：
```py
Python 2.7.6 (default, Mar 22 2014, 22:59:38) 
[GCC 4.8.2] on linux2 
Type "help", "copyright", "credits" or "license" for more information.
>>> x = 'my precious' 
>>> dummy = [x for x in 'ABC'] 
>>> x 
'C' 
```
如你所见，`x` 原本的值被取代了，但是这种情况在 **Python 3** 中是不会出现的。

列表推导、生成器表达式，以及同它们很相似的集合（`set`）推导和字典（`dict`）推导，在 **Python3**中都有了自己的局部作用域，就像函数似的。表达式内部的变量和赋值只在局部起作用，表达式的上下文里的同名变量还可以被正常引用，局部变量并不会影响到它们。 

这是**Python3**代码:

```py
>>> x = 'ABC' 
>>> dummy = [ord(x) for x in x] 
>>> x ➊ 
'ABC' 
>>> dummy ➋ 
[65, 66, 67]
```

➊ `x`的值被保留了。

➋ 列表推导也创建了正确的列表

### 列表推导同filter和map的比较

`filter`和`map`合起来能做的事情，列表推导也可以做，而且还不需要借助难以理解和阅读的`lambda`表达式

用列表推导和 `map/filter` 组合来创建同样的表单

 ```py
 >>> symbols = '$¢£¥€¤' 
 >>> beyond_ascii = [ord(s) for s in symbols if ord(s) > 127] 
 >>> beyond_ascii 
 [162, 163, 165, 8364, 164] 
 >>> beyond_ascii = list(filter(lambda c: c > 127, map(ord, symbols))) 
 >>> beyond_ascii 
 [162, 163, 165, 8364, 164]
 ``` 
 
列表推导可以帮助我们把一个序列或是其他可迭代类型中的元素过滤或 是加工，然后再新建一个列表。**Python** 内置的 `filter` 和 `map` 函数组合起来也能达到这一效果，但是可读性上打了不小的折扣。 

