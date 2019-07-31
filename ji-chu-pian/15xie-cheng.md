# 携程

字典为动词`“to yield”`给出了两个释义：产出和让步。对于 **Python** 生成器中的 `yield` 来说，这两个含义都成立。`yield item` 这行代码会产出一个值，提供给 `next(...)` 的调用方；此外，还会作出让步，暂停执行生成器，让调用方继续工作，直到需要使用另一个值时再调用 `next()`。调用方会从生成器中拉取值。

从句法上看，协程与生成器类似，都是定义体中包含 `yield` 关键字的函数。可是，在协程中，`yield` 通常出现在表达式的右边（例如，`datum = yield`），可以产出值，也可以不产出——如果 `yield` 关键字后面没有表达式，那么生成器产出 `None`。协程可能会从调用方接收数据，不过调用方把数据提供给协程使用的是 `.send(datum)` 方法，而不是 `next(...)` 函数。通常，调用方会把值推送给协程。

`yield` 关键字甚至还可以不接收或传出数据。不管数据如何流动，`yield` 都是一种流程控制工具，使用它可以实现协作式多任务：协程可以把控制器让步给中心调度程序，从而激活其他的协程。

从根本上把 `yield` 视作控制流程的方式，这样就好理解协程了。

## 生成器如何进化成协程

协程的底层架构在[PEP 342—Coroutines via Enhanced Generators](https://www.python.org/dev/peps/pep-0342/)中定义，并在 **Python 2.5**（`2006` 年）实现了。自此之后，`yield` 关键字可以在表达式 中使用，而且生成器 `API` 中增加了 `.send(value)` 方法。生成器的调用方可以使用 `.send(...)` 方法发送数据，发送的数据会成为生成器函数中 `yield` 表达式的值。因此，生成器可以作为协程使用。协程是指一个过程，这个过程与调用方协作，产出由调用方提供的值。

除了 `.send(...)` 方法，`PEP 342` 还添加了 `.throw(...)` 和 `.close()` 方法：前者的作用是让调用方抛出异常，在生成器中处理；后者的作用是终止生成器。

[PEP 380—Syntax for Delegating to a Subgenerator](https://www.python.org/dev/peps/pep-0380/)对生成器函数的句法做了两处改动，以便更好地作为协程使用。

* 现在，生成器可以返回一个值；以前，如果在生成器中给 `return` 语句提供值，会抛出 `SyntaxError` 异常。

* 新引入了 `yield from` 句法，使用它可以把复杂的生成器重构成小型的嵌套生成器，省去了之前把生成器的工作委托给子生成器所需的大量样板代码。

## 用作协程的生成器的基本行为

示例 16-1 可能是协程最简单的使用演示

```py
>>> def simple_coroutine(): #  ➊
...     print('-> coroutine started')
...     x = yield  # ➋
...     print('-> coroutine received:', x)
...
>>> my_coro = simple_coroutine()
>>> my_coro  # ➌
<generator object simple_coroutine at 0x100c2be10>
>>> next(my_coro)  # ➍
-> coroutine started
>>> my_coro.send(42)  # ➎
-> coroutine received: 42
Traceback (most recent call last):  # ➏
    ...
StopIteration
```

❶ 协程使用生成器函数定义：定义体中有 `yield` 关键字。

❷ `yield` 在表达式中使用；如果协程只需从客户那里接收数据，那么产出的值是 `None`——这个值是隐式指定的，因为 `yield` 关键字右边没有表达式。

❸ 与创建生成器的方式一样，调用函数得到生成器对象。

❹ 首先要调用 `next(...)` 函数，因为生成器还没启动，没在 `yield` 语句处暂停，所以一开始无法发送数据。

❺ 调用这个方法后，协程定义体中的 `yield` 表达式会计算出 `42`；现在，协程会恢复，一直运行到下一个 `yield` 表达式，或者终止。

❻ 这里，控制权流动到协程定义体的末尾，导致生成器像往常一样抛 出 `StopIteration` 异常。

协程可以身处四个状态中的一个。当前状态可以使用 `inspect.getgeneratorstate(...)` 函数确定，该函数会返回下述字符串中的一个。

* `GEN_CREATED`

    等待开始执行。

* `GEN_RUNNING`

    解释器正在执行。

* `GEN_SUSPENDED`

    在`yield`表达式处暂停

* `GEN_CLOSED`

    执行结束

因为 `send` 方法的参数会成为暂停的 `yield` 表达式的值，所以，仅当协程处于暂停状态时才能调用 `send` 方法，例如 `my_coro.send(42)`。不过，如果协程还没激活（即，状态是 `'GEN_CREATED'`），情况就不同了。因此，**始终要调用 `next(my_coro)` 激活协程——也可以调用 `my_coro.send(None)`，效果与`next(my_coro)`一样**。

如果创建协程对象后立即把 `None` 之外的值发给它，会出现下述错误：

```py
>>> my_coro = simple_coroutine()
>>> my_coro.send(1729)
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
TypeError: can't send non-None value to a just-started generator
```

最先调用 `next(my_coro)` 函数这一步通常称为“预激”（`prime`）协程 （即，让协程向前执行到第一个 `yield` 表达式，准备好作为活跃的协程使用）。

示例 16-2 产出两个值的协程

```py
>>> def simple_coro2(a):
...     print('-> Started: a =', a)
...     b = yield a
...     print('-> Received: b =', b)
...     c = yield a + b
...     print('-> Received: c =', c)
...
>>> my_coro2 = simple_coro2(14)
>>> from inspect import getgeneratorstate
>>> getgeneratorstate(my_coro2) ➊
'GEN_CREATED'
>>> next(my_coro2) ➋
-> Started: a = 14
14
>>> getgeneratorstate(my_coro2) ➌
'GEN_SUSPENDED'
>>> my_coro2.send(28) ➍
-> Received: b = 28
42
>>> my_coro2.send(99) ➎
-> Received: c = 99
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
StopIteration
>>> getgeneratorstate(my_coro2) ➏
'GEN_CLOSED'
```

❶ `inspect.getgeneratorstate` 函数指明，处于 `GEN_CREATED` 状态 （即协程未启动）。

❷ 向前执行协程到第一个 `yield` 表达式，打印 `-> Started: a = 14` 消息，然后产出 `a` 的值，并且暂停，等待为 `b` 赋值。

❸ `getgeneratorstate` 函数指明，处于 `GEN_SUSPENDED` 状态（即协程在 `yield` 表达式处暂停）。

❹ 把数字 `28` 发给暂停的协程；计算 `yield` 表达式，得到 `28`，然后把那个数绑定给 `b`。打印 `-> Received: b = 28` 消息，产出 `a + b` 的值 （`42`），然后协程暂停，等待为 `c` 赋值。

❺ 把数字 `99` 发给暂停的协程；计算 `yield` 表达式，得到 `99`，然后把那个数绑定给 `c`。打印 `-> Received: c = 99` 消息，然后协程终止，导致生成器对象抛出 `StopIteration` 异常。

❻ `getgeneratorstate` 函数指明，处于 `GEN_CLOSED` 状态（即协程执行结束）。

## 示例：使用协程计算移动平均值

示例 16-3 `coroaverager0.py`：定义一个计算移动平均值的协程

```py
def averager():
    total = 0.0
    count = 0
    average = None
    while True: ➊
        term = yield average  ➋
        total += term
        count += 1
        average = total/count
```

➊ 这个无限循环表明，只要调用方不断把值发给这个协程，它就会一直接收值，然后生成结果。仅当调用方在协程上调用 `.close()` 方法， 或者没有对协程的引用而被垃圾回收程序回收时，这个协程才会终止。

➋ 这里的 `yield` 表达式用于暂停执行协程，把结果发给调用方；还用于接收调用方后面发给协程的值，恢复无限循环。

使用协程的好处是，`total` 和 `count` 声明为局部变量即可，无需使用 实例属性或闭包在多次调用之间保持上下文。

示例 16-4 `coroaverager0.py`：示例 16-3 中定义的移动平均值协程的`doctest`

```py
>>> coro_avg = averager()  ➊
>>> next(coro_avg)  ➋
>>> coro_avg.send(10)  ➌
10.0
>>> coro_avg.send(30)
20.0
>>> coro_avg.send(5)
15.0
```

❶ 创建协程对象。

❷ 调用 next 函数，预激协程。

❸ 计算移动平均值：多次调用 `.send(...)` 方法，产出当前的平均值。

在上述 `doctest` 中（示例 16-4），调用 `next(coro_avg)` 函数后，协程会向前执行到 `yield` 表达式，产出 `average` 变量的初始值——`None`， 因此不会出现在控制台中。此时，协程在 `yield` 表达式处暂停，等到调用方发送值。`coro_avg.send(10)` 那一行发送一个值，激活协程， 把发送的值赋给 `term`，并更新 `total`、`count` 和 `average` 三个变量的值，然后开始 `while` 循环的下一次迭代，产出 `average` 变量的值，等待下一次为 `term` 变量赋值。

## 预激协程的装饰器

如果不预激，那么协程没什么用。调用 `my_coro.send(x)` 之前，记住一定要调用 `next(my_coro)`。为了简化协程的用法，有时会使用一**预激装饰器**。

示例 16-5 `coroutil.py`：预激协程的装饰器

```py
from functools import wraps

def coroutine(func):
    """装饰器：向前执行到第一个`yield`表达式，预激`func`"""
    @wraps(func) # 保留原函数的属性
    def primer(*args, **kwargs):  ➊
        gen = func(*args, **kwargs))  ➋
        next(gen)  ➌
        return gen  ➍

    return primer
```

❶ 把被装饰的生成器函数替换成这里的 `primer` 函数；调用 `primer` 函数时，返回预激后的生成器。

❷ 调用被装饰的函数，获取生成器对象。

❸ 预激生成器。

❹ 返回生成器。

示例 16-6 `coroaverager1.py`：使用示例 16-5 中定义的 `@coroutine` 装饰器定义并测试计算移动平均值的协程

```py
"""
用于计算移动平均值的协程
    >>> coro_avg = averager()  ➊
    >>> from inspect import getgeneratorstate
    >>> getgeneratorstate(coro_avg)  ➋
    'GEN_SUSPENDED'
    >>> coro_avg.send(10)  ➌
    10.0
    >>> coro_avg.send(30)
    20.0
    >>> coro_avg.send(5)
    15.0
"""

from coroutil import coroutine  ➍

@coroutine  ➎
def averager():  ➏
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield average
        total += term
        count += 1
        average = total/count
```

❶ 调用 `averager()` 函数创建一个生成器对象，在 `coroutine` 装饰器 的 `primer` 函数中已经预激了这个生成器。

❷ `getgeneratorstate` 函数指明，处于 `GEN_SUSPENDED` 状态，因此 这个协程已经准备好，可以接收值了。

❸ 可以立即开始把值发给 `coro_avg`——这正是 `coroutine` 装饰器的目的。

❹ 导入 `coroutine` 装饰器。

❺ 把装饰器应用到 `averager` 函数上。

❻ 函数的定义体与示例 16-3 完全一样。

## 16.5 终止协程和异常处理

协程中未处理的异常会向上冒泡，传给 `next` 函数或 `send` 方法的调用方（即触发协程的对象）。示例 16-7 举例说明如何使用示例 16-6 中由 装饰器定义的 averager 协程。

示例 16-7 未处理的异常会导致协程终止

```py
>>> from coroaverager1 import averager
>>> coro_avg = averager()
>>> coro_avg.send(40)  # ➊
40.0
>>> coro_avg.send(50)
45.0
>>> coro_avg.send('spam')  # ➋
Traceback (most recent call last):
...
TypeError: unsupported operand type(s) for +=: 'float' and 'str'
>>> coro_avg.send(60)  # ➌
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
StopIteration
```

❶ 使用 `@coroutine` 装饰器装饰的 `averager` 协程，可以立即开始发送 值。

❷ 发送的值不是数字，导致协程内部有异常抛出。

❸ 由于在协程内没有处理异常，协程会终止。如果试图重新激活协程，会抛出 `StopIteration` 异常。

出错的原因是，发送给协程的 `'spam'` 值不能加到 `total` 变量上。

从 **Python 2.5** 开始，客户代码可以在生成器对象上调用两个方法，显式地把异常发给协程

这两个方法是 `throw` 和 `close`。

* `generator.throw(exc_type[, exc_value[, traceback]])`

    致使生成器在暂停的 `yield` 表达式处抛出指定的异常。如果生成器处理了抛出的异常，代码会向前执行到下一个 `yield` 表达式，而产出的值会成为调用 `generator.throw` 方法得到的返回值。如果生成器没有处理抛出的异常，异常会向上冒泡，传到调用方的上下文中。

* `generator.close()`

    致使生成器在暂停的 `yield` 表达式处抛出 `GeneratorExit` 异常。如果生成器没有处理这个异常，或者抛出了 `StopIteration` 异常（通常是指运行到结尾），调用方不会报错。如果收到 `GeneratorExit` 异常，生成器一定不能产出值，否则解释器会抛出 `RuntimeError` 异常。 生成器抛出的其他异常会向上冒泡，传给调用方。

示例 16-8 `coro_exc_demo.py`：学习在协程中处理异常的测试代码

```py
class DemoException(Exception):
    """为这次演示定义的异常对象"""

    def demo_exc_handling():
        print('-> coroutine started')
        while True:
            try:
                x = yield
            except DemoException:  ➊
                print('*** DemoException handled. Continuing')
            else:  ➋
                print('-> coroutine received:{!r}'.format(x))

        raise RuntimeError('This line should never run.') ➌
```

❶ 特别处理 `DemoException` 异常。

❷ 如果没有异常，那么显示接收到的值。

❸ 这一行永远不会执行。

示例 16-8 中的最后一行代码不会执行，因为只有未处理的异常才会中止那个无限循环，而一旦出现未处理的异常，协程会立即终止。

示例 16-9 激活和关闭 `demo_exc_handling`，没有异常

```py
>>> exc_coro = demo_exc_handling()
>>> next(exc_coro)
-> coroutine started
>>> exc_coro.send(11)
-> coroutine received: 11
>>> exc_coro.send(22)
-> coroutine received: 22
>>> exc_coro.close()
>>> from inspect import getgeneratorstate
>>> getgeneratorstate(exc_coro)
'GEN_CLOSED'
```

示例 16-10 把 `DemoException` 异常传入 `demo_exc_handling` 不会导致协程中止

```py
>>> exc_coro = demo_exc_handling()
>>> next(exc_coro)
-> coroutine started
>>> exc_coro.send(11)
-> coroutine received: 11
>>> exc_coro.throw(DemoException)
*** DemoException handled. Continuing...
>>> getgeneratorstate(exc_coro)
'GEN_SUSPENDED'
```

示例 16-11 如果无法处理传入的异常，协程会终止

```py
>>> exc_coro = demo_exc_handling()
>>> next(exc_coro)
-> coroutine started
>>> exc_coro.send(11)
-> coroutine received: 11
>>> exc_coro.throw(ZeroDivisionError)
Traceback (most recent call last):
    ...
ZeroDivisionError
>>> getgeneratorstate(exc_coro)
'GEN_CLOSED'
```

如果不管协程如何结束都想做些清理工作，要把协程定义体中相关的代 码放入 `try/finally` 块中，如示例 16-12。

示例 16-12 `coro_finally_demo.py`：使用 `try/finally` 块在协程终 止时执行操作

```py
class DemoException(Exception):
    """为这次演示定义的异常对象"""

    def demo_exc_handling():
        print('-> coroutine started')
        try:
            while True:
                try:
                    x = yield
                except DemoException:
                    print('*** DemoException handled. Continuing')
                else:  ➋
                    print('-> coroutine received:{!r}'.format(x))
        finally:
            print('-> coroutine ending')

```

## 让协程返回值

示例 16-13 是 `averager` 协程的不同版本，这一版会返回结果。为了说明如何返回值，每次激活协程时不会产出移动平均值。这么做是为了强调某些协程不会产出值，而是在最后返回一个值（通常是某种累计值）。

示例 16-13 中的 `averager` 协程返回的结果是一个 `namedtuple`，两个字段分别是项数（`count`）和平均值（`average`）。本可以只返回平均值，但是返回一个元组可以获得累积数据的另一个重要信息——项数。

示例 16-13 `coroaverager2.py`：定义一个求平均值的协程，让它返 回一个结果。

```py
from collections import namedtuple

Result = namedtuple('Result', 'count average')

def average():
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield
        if term is None:
            break
        total += term
        count += 1
        avervge = total/count

    return Result(count, average)
```

➊ 为了返回值，协程必须正常终止；因此，这一版 `averager` 中有个条件判断，以便退出累计循环。

➋ 返回一个 `namedtuple`，包含 `count` 和 `average` 两个字段。在 **Python 3.3** 之前，如果生成器返回值，解释器会报句法错误。

示例 16-14 `coroaverager2.py`：说明 `averager` 行为的 `doctest`

```py
>>> coro_avg = averager()
>>> next(coro_avg)
>>> coro_avg.send(10)  ➊
>>> coro_avg.send(30)
>>> coro_avg.send(6.5)
>>> coro_avg.send(None)  ➋
Traceback (most recent call last):
    ...StopIteration: Result(count=3, average=15.5)
```

❶ 这一版不产出值。

❷ 发送 `None` 会终止循环，导致协程结束，返回结果。一如既往，生成器对象会抛出`StopIteration` 异常。异常对象的 `value` 属性保存着返回的值。

注意，`return` 表达式的值会偷偷传给调用方，赋值给 `StopIteration` 异常的一个属性。这样做有点不合常理，但是能保留生成器对象的常规 行为——耗尽时抛出 `StopIteration` 异常。

示例 16-15 展示如何获取协程返回的值。

示例 16-15 捕获 `StopIteration` 异常，获取 `averager` 返回的值

```py
>>> coro_avg = averager()
>>> next(coro_avg)
>>> coro_avg.send(10)
>>> coro_avg.send(30)
>>> coro_avg.send(6.5)
>>> try:
...     coro_avg.send(None)
... except StopIteration as exc:
...     result = exc.value
...
>>> result Result(count=3, average=15.5)
```

获取协程的返回值虽然要绕个圈子，但这是 `PEP 380` 定义的方式，当我们意识到这一点之后就说得通了：`yield from` 结构会在内部自动捕获 `StopIteration` 异常。这种处理方式与 `for` 循环处理 `StopIteration` 异常的方式一样：循环机制使用用户易于理解的方式处理异常。对 `yield from` 结构来说，解释器不仅会捕获 `StopIteration` 异常，还会把 `value` 属性的值变成 `yield from` 表达式的值。可惜，我们无法在控制台中使用交互的方式测试这种行为，因为在函数外部使用 `yield from`（以及 `yield`）会导致句法出错。

## 使用yield from

首先要知道，`yield from` 是全新的语言结构。它的作用比 `yield` 多很多，因此人们认为继续使用那个关键字多少会引起误解。在其他语言中，类似的结构使用 `await` 关键字，这个名称好多了，因为它传达了至关重要的一点：在生成器 `gen` 中使用 `yield from subgen()` 时，`subgen` 会获得控制权，把产出的值传给 `gen` 的调用方，即调用方可以直接控制 `subgen`。与此同时，`gen` 会阻塞，等待 `subgen` 终止。

`yield from` 可用于简化 `for` 循环中的 `yield` 表达式。 例如：

```py
>>> def gen():
...     for c in 'AB':
...         yield c
...     for i in range(1, 3):
...         yield i
...
>>> list(gen())
['A', 'B', 1, 2]
```

可以改写为：

```py
>>> def gen():
...     yield from 'AB'
...     yield from range(1, 3)
...

>>> list(gen()) ['A', 'B', 1, 2]
```

`yield from x` 表达式对 `x` 对象所做的第一件事是，调用 `iter(x)`，从中获取迭代器。因此，`x` 可以是任何可迭代的对象。

可是，如果 `yield from` 结构唯一的作用是替代产出值的嵌套 `for` 循环，这个结构很有可能不会添加到 **Python** 语言中。`yield from` 结构的本质作用无法通过简单的可迭代对象说明，而要发散思维，使用嵌套的生成器。

`yield from` 的主要功能是打开双向通道，把最外层的调用方与最内层的子生成器连接起来，这样二者可以直接发送和产出值，还可以直接传入异常，而不用在位于中间的协程中添加大量处理异常的样板代码。有了这个结构，协程可以通过以前不可能的方式委托职责。

示例 16-17 能更好地说明 `yield from` 结构的用法。图 16-2 把该示例中 各个相关的部分标识出来了。

![Zz5VgK.png](https://s2.ax1x.com/2019/07/20/Zz5VgK.png)

> 图 16-2：委派生成器在 yield from 表达式处暂停时，调用方可以直 接把数据发给子生成器，子生成器再把产出的值发给调用方。子生成器返回之后，解释器会抛出 StopIteration 异常，并把返回值附 加到异常对象上，此时委派生成器会恢复

`coroaverager3.py` 脚本从一个字典中读取虚构的七年级男女学生的体重和身高。例如， `'boys;m'` 键对应于 `9` 个男学生的身高（单位是 米），`'girls;kg'` 键对应于 `10` 个女学生的体重（单位是千克）。这个脚本把各组数据传给前面定义的 `averager` 协程，然后生成一个报告，如下所示：

```py
$ python3 coroaverager3.py
 9 boys  averaging 40.42kg
 9 boys  averaging 1.39m
 10 girls averaging 42.04kg
 10 girls averaging 1.43m
```

示例 16-17 中列出的代码显然不是解决这个问题最简单的方案，但是通 过实例说明了 `yield from` 结构的用法。

示例 16-17 `coroaverager3.py`：使用 `yield from` 计算平均值并输 出统计报告

```py
from collections import namedtuple

Result = namedtuple('Result', 'count average')

# 子生成器
def averager():  ➊
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield ➋
        if term is None:  ➌
            break
        total += term
        count += 1
        average = total/count
    return Result(count, average)  ➍


# 委派生成器
def grouper(results, key)： ➎
    while True: ➏
        results[key] = yield from averager()  ➐


# 客户端代码，即调用方
def main(data):  ➑
    result = {}
    for key, values in data.items():
        group = grouper(results, key)  ➒
        next(group)  ➓
        for value in values:)  ⓫
            group.send(vale)
        group.send(None)# 重要！  ⓬

    report(results)

# 输出报告
def report(results):
    for key, result in sorted(results.items()):
        group, unit = key.split(';')
        print('{:2} {:5} averaging {:.2f}{}'.format(
            result.count, group, result.average, unit))

data = {
    'girls;kg':
        [40.9, 38.5, 44.3, 42.2, 45.2, 41.7, 44.5, 38.0, 40.6, 44.5],
    'girls;m':
        [1.6, 1.51, 1.4, 1.3, 1.41, 1.39, 1.33, 1.46, 1.45, 1.43],
    'boys;kg':
        [39.0, 40.8, 43.2, 40.8, 43.1, 38.6, 41.4, 40.6, 36.3],
    'boys;m':
        [1.38, 1.5, 1.32, 1.25, 1.37, 1.48, 1.25, 1.49, 1.46],
}

if __name__ == '__main__':
    main(data)
```

❶ 与示例 16-13 中的 `averager` 协程一样。这里作为子生成器使用。

❷ main 函数中的客户代码发送的各个值绑定到这里的 `term` 变量上。

❸ 至关重要的终止条件。如果不这么做，使用 `yield from` 调用这个协程的生成器会永远阻塞。

❹ 返回的 `Result` 会成为 `grouper` 函数中 `yield from` 表达式的值。

❺ `grouper` 是委派生成器。

❻ 这个循环每次迭代时会新建一个 `averager` 实例；每个实例都是作为协程使用的生成器对象。

❼ `grouper` 发送的每个值都会经由 `yield from` 处理，通过管道传给 `averager` 实例。`grouper` 会在 `yield from` 表达式处暂停，等待 `averager` 实例处理客户端发来的值。`averager` 实例运行完毕后，返 回的值绑定到 `results[key]` 上。`while` 循环会不断创建 `averager` 实例，处理更多的值。

❽ `main` 函数是客户端代码，用 `PEP 380` 定义的术语来说，是“调用方”。这是驱动一切的函数。

❾`group` 是调用 `grouper` 函数得到的生成器对象，传给 `grouper` 函数 的第一个参数是 `results`，用于收集结果；第二个参数是某个 键。`group` 作为协程使用。

❿ 预激 `group` 协程。

⓫ 把各个 `value` 传给 `grouper`。传入的值最终到达 `averager` 函数中 `term = yield` 那一行；`grouper` 永远不知道传入的值是什么。

⓬ 把 `None` 传入 `grouper`，导致当前的 `averager` 实例终止，也让 `grouper` 继续运行，再创建一个 `averager` 实例，处理下一组值。

下面简要说明示例 16-17 的运作方式，还会说明把 `main` 函数中调用 `group.send(None)` 那一行代码（带有“重要！”注释的那一行）去掉会 发生什么事。

* 外层 `for` 循环每次迭代会新建一个 `grouper` 实例，赋值给 `group` 变量；`group` 是委派生成器。

* 调用 `next(group)`，预激委派生成器 `grouper`，此时进入 `while True` 循环，调用子生成器 `averager` 后，在 `yield from` 表达式处暂停。

* 内层 `for` 循环调用 `group.send(value)`，直接把值传给子生成器 `averager`。同时，当前的 `grouper` 实例（`group`）在 `yield from` 表达式处暂停。

* 内层循环结束后，`group` 实例依旧在 `yield from` 表达式处暂停， 因此，`grouper` 函数定义体中为 `results[key]` 赋值的语句还没有执行。

* 如果外层 `for` 循环的末尾没有 `group.send(None)`，那么 `averager` 子生成器永远不会终止，委派生成器 `group` 永远不会再 次激活，因此永远不会为 `results[key]` 赋值。

* 外层 `for` 循环重新迭代时会新建一个 `grouper` 实例，然后绑定到 `group` 变量上。前一个 `grouper` 实例（以及它创建的尚未终止的 `averager` 子生成器实例）被垃圾回收程序回收。

## yield from的意义

示例 16-17 阐明了下述四点`yield fromx`的行为。

* 子生成器产出的值都直接传给委派生成器的调用方（即客户端代码）。

* 使用 `send()` 方法发给委派生成器的值都直接传给子生成器。如果 发送的值是 `None`，那么会调用子生成器的 `__next__()` 方法。如 果发送的值不是 `None`，那么会调用子生成器的 `send()` 方法。如 果调用的方法抛出 `StopIteration` 异常，那么委派生成器恢复运行。任何其他异常都会向上冒泡，传给委派生成器。

* 生成器退出时，生成器（或子生成器）中的 `return expr` 表达式 会触发 `StopIteration(expr)` 异常抛出。

* `yield from` 表达式的值是子生成器终止时传给 `StopIteration` 异常的第一个参数。

`yield from` 结构的另外两个特性与异常和终止有关。

* 传入委派生成器的异常，除了 `GeneratorExit` 之外都传给子生成器的 `throw()` 方法。如果调用 `throw()` 方法时抛出 `StopIteration` 异常，委派生成器恢复运行。`StopIteration`之外的异常会向上冒泡，传给委派生成器。

* 如果把 `GeneratorExit` 异常传入委派生成器，或者在委派生成器 上调用 `close()` 方法，那么在子生成器上调用 `close()` 方法，如 果它有的话。如果调用 `close()` 方法导致异常抛出，那么异常会向上冒泡，传给委派生成器；否则，委派生成器抛出 `GeneratorExit` 异常。

示例 16-18 简化的伪代码，等效于委派生成器中的 RESULT = yield from EXPR 语句（这里针对的是最简单的情况：不支持 .throw(...) 和 .close() 方法，而且只处理 StopIteration 异 常）

```py
_i = iter(EXPR)  ➊
try:
    _y = next(_i)  ➋
except StopIteration as _e:
    _r = _e.value  ➌
else:
    while 1:  ➍
        _s = yield _y  ➎
        try:
            _y = _i.send(_s)  ➏
        except StopIteration as _e:  ➐
            _r = _e.value
            break

RESULT = _r  ➑
```

❶ `EXPR` 可以是任何可迭代的对象，因为获取迭代器 `_i`（这是子生成器）使用的是 `iter()` 函数。

❷ 预激子生成器；结果保存在 `_y` 中，作为产出的第一个值。

❸ 如果抛出 `StopIteration` 异常，获取异常对象的 `value` 属性，赋值 给 `_r`——这是最简单情况下的返回值（`RESULT`）。

❹ 运行这个循环时，委派生成器会阻塞，只作为调用方和子生成器之间的通道。

❺ 产出子生成器当前产出的元素；等待调用方发送 `_s` 中保存的值。注意，这个代码清单中只有这一个 `yield` 表达式。

❻ 尝试让子生成器向前执行，转发调用方发送的 `_s`。

❼ 如果子生成器抛出 `StopIteration` 异常，获取 `value` 属性的值，赋值给 `_r`，然后退出循环，让委派生成器恢复运行。

❽ 返回的结果（`RESULT`）是 `_r`，即整个 `yield from` 表达式的值。

在这段简化的伪代码中这些变量是：

* `_i`（迭代器）

    子生成器

* `_y`（产出的值）

    子生成器产出的值

* `_r`（结果）

    最终的结果（即子生成器运行结束后 `yield from` 表达式的值）

* `_s`（发送的值）

    调用方发给委派生成器的值，这个值会转发给子生成器

* `_e`（异常）

    异常对象（在这段简化的伪代码中始终是 `StopIteration` 实例）

除了没有处理 `.throw(...)` 和 `.close()` 方法之外，这段简化的伪代 码还在子生成器上调用 `.send(...)` 方法，以此达到客户调用 `next()` 函数或 `.send(...)` 方法的目的。首次阅读时不要担心这些细微的差 别。前面说过，即使 `yield from` 结构只做示例 16-18 中展示的事情， 示例 16-17 也依旧能正常运行。

但是，现实情况要复杂一些，因为要处理客户对 `.throw(...)` 和 `.close()` 方法的调用，而这两个方法执行的操作必须传入子生成器。

此外，子生成器可能只是纯粹的迭代器，不支持 `.throw(...)` 和 `.close()` 方法，因此 `yield from` 结构的逻辑必须处理这种情况。如果子生成器实现了这两个方法，而在子生成器内部，这两个方法都会触发异常抛出，这种情况也必须由 `yield from` 机制处理。调用方可能会无缘无故地让子生成器自己抛出异常，实现 `yield from` 结构时也必须 处理这种情况。最后，为了优化，如果调用方调用 `next(...)` 函数或 `.send(None)` 方法，都要转交职责，在子生成器上调用 `next(...)` 函数；仅当调用方发送的值不是 `None` 时，才使用子生成器的 `.send(...)` 方法。

为了方便对比，下面列出 PEP 380 中扩充 `yield from` 表达式的完整伪代码，而且加上了带标号的注解。

```py
_i = iter(EXPR)  ➊
try:
    _y = next(_i)  ➋
except StopIteration as _e:
    _r = _e.value  ➌
else:
    while 1:  ➍
        try:
            _s = yield _y  ➎
        except GeneratorExit as _e:  ➏
            try:
                _m = _i.close
            except AttributeError:
                pass
            else:
                _m()
            raise _e
        except BaseException as _e:  ➐
            _x = sys.exc_info()
            try:
                _m = _i.throw
            except AttributeError:
                raise _e
            else:  ➑
                try:
                    _y = _m(*_x)
                except StopIteration as _e:
                    _r = _e.value
                    break
        else:  ➒
            try:  ➓
                if _s is None:  ⓫
                    _y = next(_i)
                else:
                    _y = _i.send(_s)
                except StopIteration as _e:  ⓬
                    _r = _e.value
                    break

RESULT = _r  ⓭
```

❶ EXPR 可以是任何可迭代的对象，因为获取迭代器 `_i`（这是子生成器）使用的是 `iter()` 函数。

❷ 预激子生成器；结果保存在 `_y` 中，作为产出的第一个值。

❸ 如果抛出 `StopIteration` 异常，获取异常对象的 `value` 属性，赋值 给 `_r`——这是最简单情况下的返回值（`RESULT`）。

❹ 运行这个循环时，委派生成器会阻塞，只作为调用方和子生成器之间的通道。

❺ 产出子生成器当前产出的元素；等待调用方发送 `_s` 中保存的值。这 个代码清单中只有这一个 `yield` 表达式。

❻ 这一部分用于关闭委派生成器和子生成器。因为子生成器可以是任何可迭代的对象，所以可能没有 `close` 方法。

❼ 这一部分处理调用方通过 `.throw(...)` 方法传入的异常。同样，子生成器可以是迭代器，从而没有 `throw` 方法可调用——这种情况会导 致委派生成器抛出异常。

❽ 如果子生成器有 `throw` 方法，调用它并传入调用方发来的异常。子生成器可能会处理传入的异常（然后继续循环）；可能抛出 `StopIteration` 异常（从中获取结果，赋值给 `_r`，循环结束）；还可 能不处理，而是抛出相同的或不同的异常，向上冒泡，传给委派生成 器。

❾ 如果产出值时没有异常......

❿ 尝试让子生成器向前执行......

⓫ 如果调用方最后发送的值是 `None`，在子生成器上调用 `next` 函数， 否则调用 `send` 方法。

⓬ 如果子生成器抛出 `StopIteration` 异常，获取 `value` 属性的值，赋 值给 `_r`，然后退出循环，让委派生成器恢复运行。

⓭ 返回的结果（`RESULT`）是 `_r`，即整个 `yield from` 表达式的值
