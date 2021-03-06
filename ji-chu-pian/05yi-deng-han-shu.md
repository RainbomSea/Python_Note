# 一等函数

在 **Python** 中，函数是一等对象。编程语言理论家把“一等对象”定义为满 足下述条件的程序实体：

* 在运行时创建

* 能赋值给变量或数据结构中的元素

* 能作为参数传给函数

* 能作为函数的返回结果

## 把函数视作对象

示例 5-1 中的控制台会话表明，**Python** 函数是对象。这里我们创建了一 个函数，然后调用它，读取它的 __doc__ 属性，并且确定函数对象本身是 `function` 类的实例。

示例 5-1 创建并测试一个函数，然后读取它的 `__doc__` 属性，再检查它的类型:

```py
>>> def factorial(n):  ➊
...     '''returns n!'''
...     return 1 if n < 2 else n * factorial(n-1)
...
>>> factorial(42)
1405006117752879898543142606244511569936384000000000
>>> factorial.__doc__  ➋
'returns n!'
>>> type(factorial)  ➌
<class 'function'>
```

➊ 这是一个控制台会话，因此我们是在“运行时”创建一个函数。

➋ `__doc__` 是函数对象众多属性中的一个。

➌ `factorial`是 function 类的实例。

示例 5-2 展示了函数对象的“一等”本性。我们可以把 `factorial` 函数赋 值给变量 `fact`，然后通过变量名调用。我们还能把它作为参数传给 `map` 函数。`map` 函数返回一个可迭代对象，里面的元素是把第一个参数 （一个函数）应用到第二个参数（一个可迭代对象，这里是 `range(11)`）中各个元素上得到的结果。

示例 5-2 通过别的名称使用函数，再把函数作为参数传递:

```py
>>> fact = factorial
>>> fact
<function factorial at 0x...>
>>> fact(5)
120
>>> map(factorial, range(11))
 <map object at 0x...>
>>> list(map(fact, range(11)))
[1, 1, 2, 6, 24, 120, 720, 5040, 40320, 362880, 3628800]
```

## 高阶函数

接受函数为参数，或者把函数作为结果返回的函数是高阶函数（`higher-order function`）。`map` 函数就是一例，如示例 5-2 所示。

在函数式编程范式中，最为人熟知的高阶函数有 `map`、`filter`、`reduce` 和 `apply`。`apply` 函数在 **Python 2.3** 中标记为过时，在 **Python3** 中移除了，因为不再需要它了。如果想使用不定量的参数调用函数，可以编写 `fn(*args, **keywords)`，不用再编写 `apply(fn, args, kwargs)`。

`map`、`filter` 和 `reduce` 这三个高阶函数还能见到，不过多数使用场景 下都有更好的替代品

### map、filter和reduce的现代替代品

函数式语言通常会提供 `map`、`filter` 和 `reduce` 三个高阶函数（有时 使用不同的名称）。在 **Python3** 中，`map` 和 `filter` 还是内置函数，但是由于引入了列表推导和生成器表达式，它们变得没那么重要了。列表推导或生成器表达式具有 `map` 和 `filter` 两个函数的功能，而且更易于阅读，如示例 5-5 所示。

示例 5-5 计算阶乘列表：map 和 filter 与列表推导比较:

```py
>>> list(map(fact, range(6)))  ➊
[1, 1, 2, 6, 24, 120]
>>> [fact(n) for n in range(6)]  ➋
[1, 1, 2, 6, 24, 120]
>>> list(map(factorial, filter(lambda n: n % 2, range(6))))  ➌
[1, 6, 120]
>>> [factorial(n) for n in range(6) if n % 2]  ➍
[1, 6, 120]
```

❶ 构建 `0!` 到 `5!` 的一个阶乘列表。

❷ 使用列表推导执行相同的操作。

❸ 使用 `map` 和 `filter` 计算直到 `5!` 的奇数阶乘列表。

❹ 使用列表推导做相同的工作，换掉 `map` 和 `filter`，并避免了使用 `lambda` 表达式。

在 **Python 3** 中，`map` 和 `filter` 返回生成器（一种迭代器），因此现在它们的直接替代品是生成器表达式（在 **Python 2** 中，这两个函数返回列表，因此最接近的替代品是列表推导）。

在 **Python 2** 中，`reduce` 是内置函数，但是在 **Python 3** 中放到 `functools` 模块里了。这个函数最常用于求和，自 `2003` 年发布的 **Python 2.3** 开始，最好使用内置的 `sum` 函数。在可读性和性能方面，这是一项重大改善（见示例 5-6）

示例 5-6 使用 reduce 和 sum 计算 0~99 之和:

```py
>>> from functools import reduce  ➊
>>> from operator import add ➋
>>> reduce(add, range(100))  ➌
4950
>>> sum(range(100))  ➍
4950
```

❶ 从 **Python 3.0** 起，`reduce` 不再是内置函数了。

❷ 导入 `add`，以免创建一个专求两数之和的函数。

❸ 计算 `0~99` 之和。

❹ 使用 `sum` 做相同的求和；无需导入或创建求和函数。

`sum` 和 `reduce` 的通用思想是把某个操作连续应用到序列的元素上，累计之前的结果，把一系列值归约成一个值。

## 匿名函数

`lambda` 关键字在 **Python** 表达式内创建匿名函数。

然而，**Python** 简单的句法限制了 `lambda` 函数的定义体只能使用纯表达式。换句话说，`lambda` 函数的定义体中不能赋值，也不能使用 `while` 和 `try` 等 **Python** 语句。

示例 5-7 使用 `lambda` 表达式反转拼写，然后依此给单词列表排序:

```py
>>> fruits = ['strawberry', 'fig', 'apple', 'cherry', 'raspberry', 'banana']
>>> sorted(fruits, key=lambda word: word[::-1])
['banana', 'apple', 'fig', 'raspberry', 'strawberry', 'cherry']
```

除了作为参数传给高阶函数之外，**Python** 很少使用匿名函数。由于句法上的限制，非平凡的 `lambda` 表达式要么难以阅读，要么无法写出。

## 可调用对象

除了用户定义的函数，调用运算符（即 `()`）还可以应用到其他对象上。如果想判断对象能否调用，可以使用内置的`callable()`函数。**Python** 数据模型文档列出了 `7` 种可调用对象:

* 用户定义的函数

    使用 def 语句或 lambda 表达式创建。

* 内置函数

    使用 **C** 语言（**CPython**）实现的函数，如 `len` 或 `time.strftime`。

* 内置方法

    使用 **C** 语言实现的方法，如 `dict.get`。

* 方法

    在类的定义体中定义的函数。

* 类

    调用类时会运行类的 `__new__` 方法创建一个实例，然后运行 `__init__` 方法，初始化实例，最后把实例返回给调用方。因为 **Python** 没有 `new` 运算符，所以调用类相当于调用函数。

* 类的实例

    如果类定义了 `__call__` 方法，那么它的实例可以作为函数调用。

* 生成器函数

    使用 `yield` 关键字的函数或方法。调用生成器函数返回的是生成器对象。

**Python** 中有各种各样可调用的类型，因此判断对象能否调用，最安全的方法是使用内置的 `callable()` 函数：

```py
>>> abs, str, 13
(<built-in function abs>, <class 'str'>, 13)
>>> [callable(obj) for obj in (abs, str, 13)]
[True, True, False]
```

## 用户定义的可调用类型

不仅 **Python** 函数是真正的对象，任何 **Python** 对象都可以表现得像函数。为此，只需实现实例方法 `__call__`。

示例 5-8 实现了 `BingoCage` 类。这个类的实例使用任何可迭代对象构建，而且会在内部存储一个随机顺序排列的列表。调用实例会取出一个元素。

示例 5-8 `bingocall.py`：调用 `BingoCage` 实例，从打乱的列表中取出一个元素

```py
import random

class BingoCage:

    def __init__(self, items):
        self._items = list(items)  ➊
        random.shuffle(self._items)  ➋

    def pick(self):  ➌
        try:
            return self._items.pop()
        except IndexError:
            raise LookupError('pick from empty BingoCage')  ➍

    def __call__(self):  ➎
        return self.pick()
```

❶ `__init__` 接受任何可迭代对象；在本地构建一个副本，防止列表参 数的意外副作用。

❷ `shuffle` 定能完成工作，因为 `self._items` 是列表。

❸ 起主要作用的方法。

❹ 如果 `self._items` 为空，抛出异常，并设定错误消息。

❺ `bingo.pick()` 的快捷方式是 `bingo()`。

下面是示例 5-8 中定义的类的简单演示。注意，`bingo` 实例可以作为函数调用，而且内置的 `callable(...)` 函数判定它是可调用的对象：

```py
>>> bingo = BingoCage(range(3))
>>> bingo.pick()
1
>>> bingo()
0
>>> callable(bingo)
True
```

实现`__call__`方法的类是创建函数类对象的简便方式，此时必须在内部维护一个状态，让它在调用之间可用，例如`BingoCage`中的剩余元素。装饰器就是这样。装饰器必须是函数，而且有时要在多次调用之 间“记住”某些事 [ 例如备忘（`memoization`），即缓存消耗大的计算结果，供后面使用 ]。

## 函数内省

除了 `__doc__`，函数对象还有很多属性。使用 `dir` 函数可以探知 `factorial` 具有下述属性：

```py
>>> dir(factorial)
['__annotations__', '__call__', '__class__', '__closure__', '__code__',
'__defaults__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__',
'__format__', '__ge__', '__get__', '__getattribute__', '__globals__',
'__gt__', '__hash__', '__init__', '__kwdefaults__', '__le__', '__lt__',
'__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__',
'__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__',
'__subclasshook__']
```

下面重点说明函数专有而用户定义的一般对象没有的属性。计算两个属性集合的差集便能得到函数专有属性列表（见示例 5-9）。

示例 5-9 列出常规对象没有而函数有的属性:

```py
>>> class C: pass  # ➊
>>> obj = C()  # ➋
>>> def func(): pass  # ➌
>>> sorted(set(dir(func)) - set(dir(obj))) # ➍
['__annotations__', '__call__', '__closure__', '__code__', '__defaults__',
'__get__', '__globals__', '__kwdefaults__', '__name__', '__qualname__']
```

➊ 创建一个空的用户定义的类。

➋ 创建一个实例。

➌ 创建一个空函数。

➍ 计算差集，然后排序，得到类的实例没有而函数有的属性列表。

## 从定位参数到仅限关键字参数

**Python** 最好的特性之一是提供了极为灵活的参数处理机制，而且 **Python 3** 进一步提供了仅限关键字参数（`keyword-only argument`）。与之密切相关的是，调用函数时使用 `*` 和 `**`“展开”可迭代对象，映射到单个参数。下面通过示例 5-10 中的代码展示这些特性，实际使用的代码在示例 5-11 中。

示例 5-10 `tag` 函数用于生成 `HTML` 标签；使用名为 `cls` 的关键 字参数传入`“class”`属性，这是一种变通方法，因为`“class”`是 **Python** 的关键字

```py
def tag(name, *content, cls=None, **attrs):
    """生成一个或多个HTML标签"""
    if cls is not None:
        attrs['class'] = cls

    if attrs:
        attr_str = ''.join(' %s="%s"' % (attr, value)
                          for attr, value
                          in sorted(attrs.items()))
    else:
        attr_str = ''
    if content:
        return '\n'.join('<%s%s>%s</%s>' %
                         (name, attr_str, c, name) for c in content)
    else:
        return '<%s%s />' % (name, attr_str)
```

示例 5-11 `tag` 函数（见示例 5-10）众多调用方式中的几种

```py
>>> tag('br')  ➊
'<br />'
>>> tag('p', 'hello')  ➋
'<p>hello</p>'
>>> print(tag('p', 'hello', 'world'))
<p>hello</p> <p>world</p>
>>> tag('p', 'hello', id=33)  ➌
'<p id="33">hello</p>'
>>> print(tag('p', 'hello', 'world', cls='sidebar'))  ➍
<p class="sidebar">hello</p>
<p class="sidebar">world</p>
>>> tag(content='testing', name="img")  ➎
'<img content="testing" />'
>>> my_tag = {'name': 'img', 'title': 'Sunset Boulevard',
...           'src': 'sunset.jpg', 'cls': 'framed'}
>>> tag(**my_tag)  ➏
'<img class="framed" src="sunset.jpg" title="Sunset Boulevard" />'
```

❶ 传入单个定位参数，生成一个指定名称的空标签。

❷ 第一个参数后面的任意个参数会被 `*content` 捕获，存入一个元组。

❸ `tag` 函数签名中没有明确指定名称的关键字参数会被 `**attrs` 捕 获，存入一个字典。

❹ `cls` 参数只能作为关键字参数传入。

❺ 调用 `tag` 函数时，即便第一个定位参数也能作为关键字参数传入。

❻ 在 `my_tag` 前面加上 `**`，字典中的所有元素作为单个参数传入，同名键会绑定到对应的具名参数上，余下的则被 `**attrs `捕获。

仅限关键字参数是 `Python 3` 新增的特性。在示例 5-10 中，`cls` 参数只能通过关键字参数指定，它一定不会捕获未命名的定位参数。定义函数时若想指定仅限关键字参数，要把它们放到前面有 `*` 的参数后面。如果不想支持数量不定的定位参数，但是想支持仅限关键字参数，在签名中放 一个 *，如下所示：

```py
>>> def f(a, *, b):
...     return a, b
...
>>> f(1, b=2)
(1, 2)
```

注意，仅限关键字参数不一定要有默认值，可以像上例中 `b` 那样，强制 必须传入实参。
