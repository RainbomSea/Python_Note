# 继承的优缺点

## 子类化内置类型很麻烦

在 **Python 2.2** 之前，内置类型（如 `list` 或 `dict`）不能子类化。在 **Python 2.2** 之后，内置类型可以子类化了，但是有个重要的注意事项： 内置类型（使用 **C** 语言编写）不会调用用户定义的类覆盖的特殊方法。

至于内置类型的子类覆盖的方法会不会隐式调用，**CPython** 没有制定官方规则。基本上，内置类型的方法不会调用子类覆盖的方法。 例如，`dict` 的子类覆盖的 `__getitem__()` 方法不会被内置类型的 `get()` 方法调用。

示例 12-1: 内置类型 `dict` 的 `__init__` 和 `__update__` 方法会忽略我们覆盖的 `__setitem__` 方法 

```py
>>> class DoppelDict(dict): 
...     def __setitem__(self, key, value): 
...         super().__setitem__(key, [value] * 2)  # ➊ 
... 
>>> dd = DoppelDict(one=1)  # ➋ 
>>> dd 
{'one': 1} 
>>> dd['two'] = 2  # ➌ 
>>> dd 
{'one': 1, 'two': [2, 2]} 
>>> dd.update(three=3)  # ➍ 
>>> dd 
{'three': 3, 'one': 1, 'two': [2, 2]} 
```

❶ `DoppelDict.__setitem__` 方法会重复存入的值（只是为了提供易 于观察的效果）。它把职责委托给超类。

❷ 继承自 `dict` 的 `__init__` 方法显然忽略了我们覆盖的 `__setitem__` 方法：'one' 的值没有重复。 

❸ `[]` 运算符会调用我们覆盖的 `__setitem__` 方法，按预期那样工 作：`'two'` 对应的是两个重复的值，即 `[2, 2]`。 

❹ 继承自 `dict` 的 `update` 方法也不使用我们覆盖的 `__setitem__` 方法：`'three'` 的值没有重复。

原生类型的这种行为违背了面向对象编程的一个基本原则：始终应该从 实例（`self`）所属的类开始搜索方法，即使在超类实现的类中调用也是 如此。在这种糟糕的局面中，`__missing__ `方法却能按预期方式工作，不过这只是特例。 

不只实例内部的调用有这个问题（`self.get()` 不调用 `self.__getitem__()`），内置类型的方法调用的其他类的方法，如果 被覆盖了，也不会被调用。


示例 12-2 `dict.update` 方法会忽略 `AnswerDict.__getitem__` 方法 

```py
>>> class AnswerDict(dict): 
...     def __getitem__(self, key):  # ➊ 
...         return 42 
... 
>>> ad = AnswerDict(a='foo')  # ➋ 
>>> ad['a']  # ➌ 
42 
>>> d = {} 
>>> d.update(ad)  # ➍ 
>>> d['a']  # ➎ 
'foo' 
>>> d 
{'a': 'foo'} 
```

❶ 不管传入什么键，`AnswerDict.__getitem__` 方法始终返回 `42`。

❷ `ad` 是 `AnswerDict` 的实例，以 `('a', 'foo')` 键值对初始化。 

❸ `ad['a']` 返回 `42`，这与预期相符。 

❹ `d` 是 `dict` 的实例，使用 `ad` 中的值更新 `d`。 

❺ `dict.update` 方法忽略了 `AnswerDict.__getitem__` 方法。

直接子类化内置类型（如 `dict`、`list` 或 `str`）容易出错， 因为内置类型的方法通常会忽略用户覆盖的方法。不要子类化内置 类型，用户自己定义的类应该继承 `collections` 模块中的类，例如 `UserDict`、`UserList` 和 `UserString`，这些类做了特殊设计，因此易于扩展。 

示例 12-3 `DoppelDict2` 和 `AnswerDict2` 能像预期那样使用，因为它们扩展的是 `UserDict`，而不是 `dict`

 ```py
 >>> import collections 
 >>> 
 >>> class DoppelDict2(collections.UserDict): 
 ...     def __setitem__(self, key, value): 
 ...         super().__setitem__(key, [value] * 2) 
 ... 
 >>> dd = DoppelDict2(one=1) 
 >>> dd 
 {'one': [1, 1]} 
 >>> dd['two'] = 2 
 >>> dd 
 {'two': [2, 2], 'one': [1, 1]} 
 >>> dd.update(three=3) 
 >>> dd 
 {'two': [2, 2], 'three': [3, 3], 'one': [1, 1]} 
 >>> 
 >>> class AnswerDict2(collections.UserDict): 
 ...     def __getitem__(self, key): 
 ...         return 42 
 ...
 >>> ad = AnswerDict2(a='foo') 
 >>> ad['a'] 
 42 
 >>> d = {} 
 >>> d.update(ad) 
 >>> d['a'] 
 42 
 >>> d 
 {'a': 42} 
 ``` 

## 多重继承和方法解析顺序

任何实现多重继承的语言都要处理潜在的命名冲突，这种冲突由不相关 的祖先类实现同名方法引起。这种冲突称为“菱形问题”，如图 12-1 和 示例 12-4 所示。

![ZLnjmQ.png](https://s2.ax1x.com/2019/07/17/ZLnjmQ.png)

> 图 12-1：（左）说明“菱形问题”的 UML 类图；（右）虚线箭头是示 例 12-4 使用的方法解析顺序 

示例 12-4 `diamond.py`：图 12-1 中的 `A`、`B`、`C` 和 `D` 四个类 

```py
class A:
   def ping(self):
      print('ping:', self) 


class B(A):
   def pong(self):
      print('pong:', self) 


class C(A):
   def pong(self):
      print('PONG:', self) 
   
class D(B, C):
   def ping(self):
      super().ping()
      print('post-ping:', self)
      
   def pingpong(self):
      self.ping()
      super().ping()
      self.pong()
      super().pong()
      C.pong(self) 
```

注意，`B` 和 `C` 都实现了 `pong` 方法，二者之间唯一的区别是，`C.pong` 方法输出的是大写的 `PONG`。

在 `D` 的实例上调用 `d.pong()` 方法的话，运行的是哪个 `pong` 方法呢？ 在 **C++** 中，程序员必须使用类名限定方法调用来避免这种歧义。**Python** 也能这么做.

示例 12-5 在 `D` 实例上调用 `pong` 方法的两种方式

```py
>>> from diamond import * 
>>> d = D() 
>>> d.pong()  # ➊ 
pong: <diamond.D object at 0x10066c278> 
>>> C.pong(d)  # ➋ 
PONG: <diamond.D object at 0x10066c278> 
```

❶ 直接调用 `d.pong()` 运行的是 `B` 类中的版本。 

❷ 超类中的方法都可以直接调用，此时要把实例作为显式参数传入。 

**Python** 能区分 `d.pong()` 调用的是哪个方法，是因为 **Python** 会按照特定 的顺序遍历继承图。这个顺序叫方法解析顺序（`Method Resolution Order`，`MRO`）。类都有一个名为 `__mro__` 的属性，它的值是一个元 组，按照方法解析顺序列出各个超类，从当前类一直向上，直到 `object` 类。`D` 类的 `__mro__` 属性如下

```py
>>> D.__mro__ 
(<class 'diamond.D'>, <class 'diamond.B'>, <class 'diamond.C'>, 
<class 'diamond.A'>, <class 'object'>) 
```

若想把方法调用委托给超类，推荐的方式是使用内置的 `super()` 函数。在 **Python 3** 中，这种方式变得更容易了，如示例 12-4 中 `D` 类的 `pingpong` 方法所示。 然而，有时可能需要绕过方法解析顺序，直接调用某个超类的方法——这样做有时更方便。例如，`D.ping` 方法可以 这样写： 

```py
def ping(self):
   A.ping(self)  # 而不是super().ping()
   print('post-ping:', self) 
```

注意，直接在类上调用实例方法时，必须显式传入 `self` 参数，因为这 样访问的是未绑定方法（unbound method）。

然而，使用 `super()` 最安全，也不易过时。调用框架或不受自己控制 的类层次结构中的方法时，尤其适合使用 `super()`。使用 `super()` 调 用方法时，会遵守方法解析顺序

```py
>>> from diamond import D 
>>> d = D() 
>>> d.ping()  # ➊ 
ping: <diamond.D object at 0x10cc40630>  # ➋ 
post-ping: <diamond.D object at 0x10cc40630>  # ➌ 
```

❶ `D` 类的 `ping` 方法做了两次调用。 

❷ 第一个调用是 `super().ping()`；`super` 函数把 `ping` 调用委托给 `A` 类；这一行由 `A.ping` 输出。 

❸ 第二个调用是 `print('post-ping:', self)`，输出的是这一行。 

示例 12-7 `pingpong` 方法的 5 个调用

```py
>>> from diamond import D 
>>> d = D() 
>>> d.pingpong() 
ping: <diamond.D object at 0x10bf235c0>  # ➊ 
post-ping: <diamond.D object at 0x10bf235c0> 
ping: <diamond.D object at 0x10bf235c0>  # ➋ 
pong: <diamond.D object at 0x10bf235c0>  # ➌ 
pong: <diamond.D object at 0x10bf235c0>  # ➍ 
PONG: <diamond.D object at 0x10bf235c0>  # ➎ 
```

❶ 第一个调用是 `self.ping()`，运行的是 `D` 类的 `ping` 方法，输出这 一行和下一行。 

❷ 第二个调用是 `super().ping()`，跳过 `D` 类的 `ping` 方法，找到 `A` 类 的 `ping` 方法。 

❸ 第三个调用是 `self.pong()`，根据 `__mro__` ，找到的是 `B` 类实现的 `pong` 方法。 

❹ 第四个调用是 `super().pong()`，也根据 `__mro__` ，找到 `B` 类实现 的 `pong` 方法。 

➎ 第五个调用是 `C.pong(self)`，忽略 `mro` ，找到的是 `C` 类实现的 `pong` 方法。 

方法解析顺序不仅考虑继承图，还考虑子类声明中列出超类的顺序。也就是说，如果在 `diamond.py` 文件（见示例 12-4）中把 `D` 类声明为 `class D(C, B):`，那么 `D` 类的 `__mro__` 属性就会不一样：先搜索 `C` 类，再搜索 `B` 类。 