# 函数装饰器和闭包 

函数装饰器用于在源码中“标记”函数，以某种方式增强函数的行为。这是一项强大的功能，但是若想掌握，必须理解闭包。

## 装饰器基础知识

装饰器是可调用的对象，其参数是另一个函数（被装饰的函数）。装饰器可能会处理被装饰的函数，然后把它返回，或者将其替换成另一个函数或可调用对象。

假如有个名为 `decorate` 的装饰器:

```py
@decorate
def target():
    print('running target()')
```

上述代码的效果与下述写法一样：

```py
def target():
    print('running target()')

target = decorate(target)
```

两种写法的最终结果一样：上述两个代码片段执行完毕后得到的 `target` 不一定是原来那个 `target` 函数，而是 `decorate(target)` 返回的函数。

为了确认被装饰的函数会被替换，请看示例 7-1 中的控制台会话。

示例 7-1 装饰器通常把函数替换成另一个函数

```py
>>> def deco(func): 
...     def inner(): 
...         print('running inner()') 
...     return inner  ➊ 
... 
>>> @deco 
... def target():  ➋
...     print('running target()') 
... >>> target()  ➌ 
running inner() 
>>> target  ➍ 
<function deco.<locals>.inner at 0x10063b598> 
```

❶ `deco` 返回 `inner` 函数对象。

❷ 使用 `deco` 装饰 `target`。

❸ 调用被装饰的 `target` 其实会运行 `inner`。

❹ 审查对象，发现 `target` 现在是 `inner` 的引用。

严格来说，装饰器只是语法糖。如前所示，装饰器可以像常规的可调用对象那样调用，其参数是另一个函数。有时，这样做更方便，尤其是做元编程（在运行时改变程序的行为）时。

综上，装饰器的一大特性是，能把被装饰的函数替换成其他函数。第二个特性是，装饰器在加载模块时立即执行。

## Python何时执行装饰器

装饰器的一个关键特性是，它们在被装饰的函数定义之后立即运行。这通常是在导入时(即 **Python** 加载模块时)，如示例7-2中的`registration.py` 模块所示。

示例 7-2 `registration.py` 模块

```py
registry = []  ➊ 

def register(func):  ➋
    print('running register(%s)' % func)  ➌
    registry.append(func)  ➍
    return func  ➎ 
    
@register  ➏ 
def f1():
    print('running f1()') 
    
@register 
def f2():
    print('running f2()') 
    
def f3():  ➐
    print('running f3()') 
    
def main():  ➑
    print('running main()')
    print('registry ->', registry)
    f1()
    f2()
    f3() 
    
    
if __name__=='__main__':
    main()  ➒ 
```

❶ `registry` 保存被 `@register` 装饰的函数引用。 

❷ `register` 的参数是一个函数。

❸ 为了演示，显示被装饰的函数。 

❹ 把 `func` 存入 `registry`。 

❺ 返回 `func`：必须返回函数；这里返回的函数与通过参数传入的一 样。 

❻ `f1` 和 `f2` 被 `@register` 装饰。 

❼ `f3` 没有装饰。 

❽ `main` 显示 `registry`，然后调用 `f1()`、`f2()` 和 `f3()`。 

❾ 只有把 `registration.py` 当作脚本运行时才调用 `main()`。 

把 `registration.py` 当作脚本运行得到的输出如下：

```py
$ python3 registration.py 
running register(<function f1 at 0x100631bf8>) 
running register(<function f2 at 0x100631c80>) 
running main() 
registry -> [<function f1 at 0x100631bf8>, <function f2 at 0x100631c80>] 
running f1() 
running f2() 
running f3()  
```

注意，`register` 在模块中其他函数之前运行（两次）。调用 `register` 时，传给它的参数是被装饰的函数，例如 `<function f1 at 0x100631bf8>`。

加载模块后，`registry` 中有两个被装饰函数的引用：`f1` 和 `f2`。这两个函数，以及 `f3`，只在 `main` 明确调用它们时才执行。

如果导入 `registration.py` 模块（不作为脚本运行），输出如下：

```py
>>> import registration 
running register(<function f1 at 0x10063b1e0>)
running register(<function f2 at 0x10063b268>)
```

此时查看 `registry` 的值，得到的输出如下：

```py
>>> registration.registry
[<function f1 at 0x10063b1e0>, <function f2 at 0x10063b268>] 
```

示例 7-2 主要想强调，函数装饰器在导入模块时立即执行，而被装饰的函数只在明确调用时运行。这突出了**Python**程序员所说的导入时和运行时之间的区别。

考虑到装饰器在真实代码中的常用方式，示例 7-2 有两个不寻常的地方。

* 装饰器函数与被装饰的函数在同一个模块中定义。实际情况是，装饰器通常在一个模块中定义，然后应用到其他模块中的函数上。

* `register` 装饰器返回的函数与通过参数传入的相同。实际上，大多数装饰器会在内部定义一个函数，然后将其返回。 

## 变量作用域规则

在示例 7-4 中，我们定义并测试了一个函数，它读取两个变量的值：一个是局部变量 `a`，是函数的参数；另一个是变量 `b`，这个函数没有定义它。

示例 7-4 一个函数，读取一个局部变量和一个全局变量

```py
>>> def f1(a): 
...     print(a) 
...     print(b) 
... 
>>> f1(3) 
3 
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "<stdin>", line 3, in f1 
NameError: name 'b' is not defined 
```

出现错误并不奇怪。 在示例 7-4 中，如果先给全局变量 `b` 赋值，然后再调用 `f`，那就不会出错： 

```py
>>> b = 6
>>> f1(3)
3
6 
```

看一下示例 7-5 中的 `f2` 函数。前两行代码与示例 7-4 中的 `f1` 一样，然后为 `b` 赋值，再打印它的值。可是，在赋值之前，第二个 `print` 失败了。

示例 7-5 `b` 是局部变量，因为在函数的定义体中给它赋值了  

```
>>> b = 6 
>>> def f2(a): 
...     print(a) 
...     print(b) 
...     b = 9 
... 
>>> f2(3) 
3 
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "<stdin>", line 3, in f2 
UnboundLocalError: local variable 'b' referenced before assignment 
```

注意，首先输出了`3`，这表明`print(a)`语句执行了。但是第二个语句`print(b)`执行不了。觉得会打印`6`，因为有个全局变量`b`，而且是在`print(b)`之后为局部变量`b`赋值的。

可事实是，**Python** 编译函数的定义体时，它判断`b`是局部变量，因为在函数中给它赋值了。生成的字节码证实了这种判断，**Python**会尝试从本地环境获取 `b`。后面调用`f2(3)`时，`f2`的定义体会获取并打印局部变量`a`的值，但是尝试获取局部变量`b`的值时，发现`b`没有绑定值。

这不是缺陷，而是设计选择：**Python**不要求声明变量，但是假定在函数定义体中赋值的变量是局部变量。

如果在函数中赋值时想让解释器把 `b` 当成全局变量，要使用 `global` 声明：

```py
>>> b = 6 
>>> def f3(a): 
...     global b 
...     print(a) 
...     print(b) 
...     b = 9 
... 
>>> f3(3) 
3 
6 
>>> b
9 
>>> f3(3) 
3 
9 
>>> b = 30 
>>> b 
30
```

## 闭包

在博客圈，人们有时会把闭包和匿名函数弄混。这是有历史原因的：在函数内部定义函数不常见，直到开始使用匿名函数才会这样做。而且，只有涉及嵌套函数时才有闭包问题。因此，很多人是同时知道这两个概念的。

其实，闭包指延伸了作用域的函数，其中包含函数定义体中引用、但是不在定义体中定义的非全局变量。函数是不是匿名的没有关系，关键是它能访问定义体之外定义的非全局变量。

这个概念难以掌握，最好通过示例理解。

假如有个名为 `avg` 的函数，它的作用是计算不断增加的系列值的均值； 例如，整个历史中某个商品的平均收盘价。每天都会增加新价格，因此 平均值要考虑至目前为止所有的价格。 

```py
>>> avg(10) 
10.0 
>>> avg(11) 
10.5 
>>> avg(12) 
11.0 
```

`avg` 从何而来，它又在哪里保存历史值呢？

初学者可能会像示例 7-8 那样使用类实现。

示例 7-8 `average_oo.py`：计算移动平均值的类 

```py
class Averager():
    
    def __init__(self):
        self.series = []
    
    def __call__(self, new_value):
        self.series.append(new_value)
        total = sum(self.series)
        return total/len(self.series)
```

`Averager` 的实例是可调用对象：

```py
>>> avg = Averager() 
>>> avg(10) 
10.0 
>>> avg(11) 
10.5 
>>> avg(12) 
11.0 
```

示例 7-9 是函数式实现，使用高阶函数 `make_averager`。

示例 7-9 `average.py`：计算移动平均值的高阶函数

```py
def make_averager():
    series = []
    def averager(new_value):
        series.append(new_value)
        total = sum(series)
        return total/len(series)
    
    return averager 
```

调用 `make_averager` 时，返回一个 `averager` 函数对象。每次调用 `averager` 时，它会把参数添加到系列值中，然后计算当前平均值，如 示例 7-10 所示。

示例 7-10 测试示例 7-9

```py
>>> avg = make_averager() 
>>> avg(10) 
10.0 
>>> avg(11) 
10.5 
>>> avg(12)
11.0
```

`Averager` 类的实例 `avg` 在哪里存储历史值很明显：`self.series` 实例属性。但是第二个示例中的 `avg` 函数在哪里寻找 `series` 呢？

注意，`series` 是 `make_averager` 函数的局部变量，因为那个函数的定义体中初始化了 `series：series = []`。可是，调用 `avg(10)` 时，`make_averager` 函数已经返回了，而它的本地作用域也一去不复返了。

在 `averager` 函数中，`series` 是自由变量（free variable）。这是一个技术术语，指未在本地作用域中绑定的变量，参见图 7-1。

![Z4irlD.png](https://s2.ax1x.com/2019/07/13/Z4irlD.png)  

> 图 7-1：averager 的闭包延伸到那个函数的作用域之外，包含自由 变量 series 的绑定 

审查返回的 `averager` 对象，我们发现 **Python** 在 `__code__` 属性（表示编译后的函数定义体）中保存局部变量和自由变量的名称，如示例 7-11所示。

示例 7-11 审查 `make_averager`（见示例 7-9）创建的函数

```py
>>> avg.__code__.co_varnames
('new_value', 'total') 
>>> avg.__code__.co_freevars 
('series',) 
```

`series` 的绑定在返回的 `avg` 函数的 `__closure__` 属性中。`avg.__closure__` 中的各个元素对应于 `avg.__code__.co_freevars` 中的一个名称。这些元素是 `cell` 对象， 有个 `cell_contents` 属性，保存着真正的值。这些属性的值如示例7-12所示。

示例 7-12 接续示例 7-11

```py
>>> avg.__code__.co_freevars 
('series',) 
>>> avg.__closure__ 
(<cell at 0x107a44f78: list object at 0x107a91a48>,) 
>>> avg.__closure__[0].cell_contents 
[10, 11, 12] 
```

综上，闭包是一种函数，它会保留定义函数时存在的自由变量的绑定， 这样调用函数时，虽然定义作用域不可用了，但是仍能使用那些绑定。

注意，只有嵌套在其他函数中的函数才可能需要处理不在全局作用域中的外部变量。

## nonlocal声明 

前面实现 `make_averager` 函数的方法效率不高。在示例 7-9 中，我们把所有值存储在历史数列中，然后在每次调用 `averager` 时使用 `sum` 求和。更好的实现方式是，只存储目前的总值和元素个数，然后使用这两个数计算均值

示例 7-13 计算移动平均值的高阶函数，不保存所有历史值，但有缺陷

```py
def make_averager():
    count = 0
    total = 0
    
    def averager(new_value):
        count += 1
        total += new_valuereturn 
        total / count
    
    return averager 
```

尝试使用示例 7-13 中定义的函数，会得到如下结果

```
>>> avg = make_averager() 
>>> avg(10) 
Traceback (most recent call last):
    ... 
UnboundLocalError: local variable 'count' referenced before assignment
```

问题是，当 `count` 是数字或任何不可变类型时，`count += 1` 语句的作用其实与 `count = count + 1` 一样。因此，我们在 `averager` 的定义体中为 count 赋值了，这会把 `count` 变成局部变量。`total` 变量也受这个问题影响

示例 7-9 没遇到这个问题，因为我们没有给 `series` 赋值，我们只是调用 `series.append`，并把它传给 `sum` 和 `len`。也就是说，我们利用了列表是可变的对象这一事实。 

但是对数字、字符串、元组等不可变类型来说，只能读取，不能更新。 如果尝试重新绑定，例如 `count = count + 1`，其实会隐式创建局部变量 `count`。这样，`count` 就不是自由变量了，因此不会保存在闭包中。 

为了解决这个问题，**Python 3** 引入了 `nonlocal` 声明。它的作用是把变量标记为自由变量，即使在函数中为变量赋予新值了，也会变成自由变 量。如果为 `nonlocal` 声明的变量赋予新值，闭包中保存的绑定会更新。最新版 `make_averager` 的正确实现如示例 7-14 所示。 

示例 7-14 计算移动平均值，不保存所有历史（使用 `nonlocal` 修正） 

```py
def make_averager():
    count = 0
    total = 0
    def averager(new_value):
        nonlocal count, total
        count += 1
        total += new_value
        return total / count
    return averager
```

## 实现一个简单的装饰器 

示例 7-15 定义了一个装饰器，它会在每次调用被装饰的函数时计时，然后把经过的时间、传入的参数和调用的结果打印出来。 

示例 7-15 一个简单的装饰器，输出函数的运行时间

```py
import time 

def clock(func):
    def clocked(*args):  # ➊
        t0 = time.perf_counter()
        result = func(*args)  # ➋
        elapsed = time.perf_counter() - t0
        name = func.__name__
        arg_str = ', '.join(repr(arg) for arg in args)
        print('[%0.8fs] %s(%s) -> %r' % (elapsed, name, arg_str, result))
        return result
    return clocked #  ➌ 
```

❶ 定义内部函数 `clocked`，它接受任意个定位参数。 

❷ 这行代码可用，是因为 `clocked` 的闭包中包含自由变量 `func`。 

❸ 返回内部函数，取代被装饰的函数。示例 7-16 演示了 `clock` 装饰器

示例 7-16 使用 `clock` 装饰器

```py
# clockdeco_demo.py

import time
from clockdeco import clock

@clock 
def snooze(seconds):
    time.sleep(seconds) 
    
@clock
def factorial(n):
    return 1 if n < 2 else n*factorial(n-1) 
    
if __name__=='__main__':
    print('*' * 40, 'Calling snooze(.123)')
    snooze(.123)
    print('*' * 40, 'Calling factorial(6)')
    print('6! =', factorial(6))
```

运行示例 7-16 得到的输出如下：

```py
$ python3 clockdeco_demo.py 
**************************************** Calling snooze(123) 
[0.12405610s] snooze(.123) -> None 
**************************************** Calling factorial(6) 
[0.00000191s] factorial(1) -> 1 
[0.00004911s] factorial(2) -> 2 
[0.00008488s] factorial(3) -> 6 
[0.00013208s] factorial(4) -> 24 
[0.00019193s] factorial(5) -> 120 
[0.00026107s] factorial(6) -> 720 
6! = 720 
```

工作原理

记得吗，如下代码：

```py
@clock def factorial(n):
    return 1 if n < 2 else n*factorial(n-1) 
```

其实等价于：

```py
def factorial(n):
    return 1 if n < 2 else n*factorial(n-1) 
    
factorial = clock(factorial)
```

因此，在两个示例中，`factorial` 会作为 `func` 参数传给 `clock`（参见 示例 7-15）。然后， `clock` 函数会返回 `clocked` 函数，**Python** 解释器在背后会把 `clocked` 赋值给 `factorial`。其实，导入 `clockdeco_demo` 模块后查看 `factorial` 的 `__name__` 属性，会得到 如下结果： 

```py
>>> import clockdeco_demo 
>>> clockdeco_demo.factorial.__name__ 
'clocked'
```

所以，现在 `factorial` 保存的是 `clocked` 函数的引用。自此之后，每次调用 `factorial(n)`，执行的都是 `clocked(n)`。`clocked` 大致做了下面几件事

* 记录初始时间 `t0`。

* 调用原来的 `factorial` 函数，保存结果。

* 计算经过的时间。

* 格式化收集的数据，然后打印出来。

* 返回第 `2` 步保存的结果。 

这是装饰器的典型行为：把被装饰的函数替换成新函数，二者接受相同 的参数，而且（通常）返回被装饰的函数本该返回的值，同时还会做些额外操作。

## 标准库中的装饰器

### 使用functools.lru_cache做备忘

`functools.lru_cache` 是非常实用的装饰器，它实现了备忘（`memoization`）功能。这是一项优化技术，它把耗时的函数的结果保存起来，避免传入相同的参数时重复计算。`LRU` 三个字母是“`Least Recently Used`”的缩写，表明缓存不会无限制增长，一段时间不用的缓存条目会被扔掉。 

生成第 `n` 个斐波纳契数这种慢速递归函数适合使用 `lru_cache`，如示例 7-18 所示

示例 7-18 生成第 `n` 个斐波纳契数，递归方式非常耗时 

```
from clockdeco import clock 

@clock 
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-2) + fibonacci(n-1) 
    
if __name__=='__main__':
    print(fibonacci(6)) 
```

运行 `fibo_demo.py` 得到的结果如下。除了最后一行，其余输出都是 `clock` 装饰器生成的。

```py
$ python3 fibo_demo.py
[0.00000095s] fibonacci(0) -> 0
[0.00000095s] fibonacci(1) -> 1
[0.00007892s] fibonacci(2) -> 1
[0.00000095s] fibonacci(1) -> 1
[0.00000095s] fibonacci(0) -> 0
[0.00000095s] fibonacci(1) -> 1
[0.00003815s] fibonacci(2) -> 1
[0.00007391s] fibonacci(3) -> 2
[0.00018883s] fibonacci(4) -> 3
[0.00000000s] fibonacci(1) -> 1
[0.00000095s] fibonacci(0) -> 0
[0.00000119s] fibonacci(1) -> 1
[0.00004911s] fibonacci(2) -> 1
[0.00009704s] fibonacci(3) -> 2
[0.00000000s] fibonacci(0) -> 0
[0.00000000s] fibonacci(1) -> 1
[0.00002694s] fibonacci(2) -> 1
[0.00000095s] fibonacci(1) -> 1
[0.00000095s] fibonacci(0) -> 0
[0.00000095s] fibonacci(1) -> 1
[0.00005102s] fibonacci(2) -> 1
[0.00008917s] fibonacci(3) -> 2
[0.00015593s] fibonacci(4) -> 3
[0.00029993s] fibonacci(5) -> 5
[0.0
```

浪费时间的地方很明显：`fibonacci(1)` 调用了 `8` 次，`fibonacci(2)` 调用了 `5` 次......但是，如果增加两行代码，使用 `lru_cache`，性能会显著改善，如示例 7-19 所示。
`
示例 7-19 使用缓存实现，速度更快

```py
import functools

from clockdeco import clock 

@functools.lru_cache() # ➊ 
@clock    # ➋ 
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-2) + fibonacci(n-1) 
    
if __name__=='__main__':
    print(fibonacci(6)) 
```

❶ 注意，必须像常规函数那样调用 `lru_cache`。这一行中有一对括 号：`@functools.lru_cache()`。这么做的原因是，`lru_cache` 可以接受配置参数，稍后说明。 

❷ 这里叠放了装饰器：`@lru_cache()` 应用到 @clock 返回的函数上。

这样一来，执行时间减半了，而且 `n` 的每个值只调用一次函数：

```py
$ python3 fibo_demo_lru.py 
[0.00000119s] fibonacci(0) -> 0 
[0.00000119s] fibonacci(1) -> 1 
[0.00010800s] fibonacci(2) -> 1 
[0.00000787s] fibonacci(3) -> 2 
[0.00016093s] fibonacci(4) -> 3 
[0.00001216s] fibonacci(5) -> 5 
[0.00025296s] fibonacci(6) -> 8 
```

在计算 `fibonacci(30)` 的另一个测试中，示例 7-19 中的版本在 `0.0005` 秒内调用了 `31` 次 `fibonacci` 函数，而示例 7-18 中未缓存的版本调用 `fibonacci` 函数 `2 692 537` 次，在使用 **Intel Core i7** 处理器的笔记本电脑 中耗时 `17.7` 秒。

除了优化递归算法之外，`lru_cache` 在从 **Web** 中获取信息的应用中也发挥巨大作用。 

特别要注意，`lru_cache` 可以使用两个可选的参数来配置。它的签名是：

```
functools.lru_cache(maxsize=128, typed=False) 
``

`maxsize` 参数指定存储多少个调用的结果。缓存满了之后，旧的结果会被扔掉，腾出空间。为了得到最佳性能，`maxsize` 应该设为`2`的幂。`typed`参数如果设为`True`，把不同参数类型得到的结果分开保存，即把通常认为相等的浮点数和整数参数（如 `1` 和 `1.0`）区分开。顺 便说一下，因为 `lru_cache` 使用字典存储结果，而且键根据调用时传入的定位参数和关键字参数创建，所以被 `lru_cache` 装饰的函数，它 的所有参数都必须是可散列的。

### 单分派泛函数

假设我们在开发一个调试 **Web** 应用的工具，我们想生成 **HTML**，显示不 同类型的 **Python** 对象。 

我们可能会编写这样的函数：

```py
import html 

def htmlize(obj):
    content = html.escape(repr(obj))
    return '<pre>{}</pre>'.format(content) 
```

这个函数适用于任何 **Python** 类型，但是现在我们想做个扩展，让它使用特别的方式显示某些类型。 

* **str**：把内部的换行符替换为 `'<br>\n'`；不使用 `<pre>`，而是使用 `<p>`。

* **int**: 以十进制和十六进制显示数字。

* **list**: 输出一个 **HTML** 列表，根据各个元素的类型进行格式化。

我们想要的行为如示例 7-20 所示。

示例 7-20 生成 **HTML** 的 `htmlize` 函数，调整了几种对象的输出 

```py
>>> htmlize({1, 2, 3})  ➊ 
'<pre>{1, 2, 3}</pre>' >>> 
htmlize(abs) 
'<pre><built-in function abs></pre>'
>>> 
htmlize('Heimlich & Co.\n- a game')  ➋ 
'<p>Heimlich & Co.<br>\n- a game</p>' 
>>> htmlize(42)  ➌ 
'<pre>42 (0x2a)</pre>' 
>>> print(htmlize(['alpha', 66, {3, 2, 1}]))  ➍ 
<ul> 
<li><p>alpha</p></li> 
<li><pre>66 (0x42)</pre></li> 
<li><pre>{1, 2, 3}</pre></li> 
</ul> 
```

❶ 默认情况下，在 `<pre></pre>` 中显示 **HTML** 转义后的对象字符串表示形式。 

❷ 为 **str** 对象显示的也是 **HTML** 转义后的字符串表示形式，不过放在 `<p></p>` 中，而且使用 `<br>` 表示换行。 

❸ **int** 显示为十进制和十六进制两种形式，放在 `<pre></pre>` 中。 

❹ 各个列表项目根据各自的类型格式化，整个列表则渲染成 **HTML** 列表。 


因为 **Python** 不支持重载方法或函数，所以我们不能使用不同的签名定义 `htmlize` 的变体，也无法使用不同的方式处理不同的数据类型。在 **Python** 中，一种常见的做法是把 `htmlize` 变成一个分派函数，使用一 串 `if/elif/elif`，调用专门的函数，如 `htmlize_str`、`htmlize_int`，等等。这样不便于模块的用户扩展，还显得笨拙：时间一长，分派函数 `htmlize` 会变得很大，而且它与各个专门函数之间的耦合也很紧密。

**Python 3.4** 新增的 `functools.singledispatch` 装饰器可以把整体方案拆分成多个模块，甚至可以为你无法修改的类提供专门的函数。使用 `@singledispatch` 装饰的普通函数会变成泛函数（`generic function`）： 根据第一个参数的类型，以不同方式执行相同操作的一组函数。 具体做法参见示例 7-21。 

示例 7-21 `singledispatch` 创建一个自定义的 `htmlize.register` 装饰器，把多个函数绑在一起组成一个泛函数

```py
from functools import singledispatch
from collections import abc
import numbers
import html

@singledispatch ➊
def htmlize(obj):
    content = html.escape(repr(obj))
    return '<pre>{}</pre>'.format(content)
    
@htmlize.register(str) ➋
def _(text): ➌
    content = html.escape(text).replace('\n','<br>\n')
    return '<p>{0}</p>'.format(content)

@htmlize.register(numbers.Integral) ➍
def _(n):
    return '<pre>{0} (0x{0:x})</pre>'.format(n)

@htmlize.register(tuple) ➎
@htmlize.register(abc.MutableSequence)
def _(seq):
    inner = '</li>\n<li>'.join(htmlize(item)for item in seq)
    return '<ul>\n<li>' + inner + '</li>\n</ul>
```

❶ `@singledispatch` 标记处理 **object** 类型的基函数。 

❷ 各个专门函数使用 `@«base_function».register(«type»)` 装饰。 

❸ 专门函数的名称无关紧要；_ 是个不错的选择，简单明了。 

❹ 为每个需要特殊处理的类型注册一个函数。`numbers.Integral` 是 `int` 的虚拟超类。

❺ 可以叠放多个 `register` 装饰器，让同一个函数支持不同类型。

只要可能，注册的专门函数应该处理抽象基类（如 `numbers.Integral` 和 `abc.MutableSequence`），不要处理具体实现（如 `int` 和 `list`）。这样，代码支持的兼容类型更广泛。例如，**Python** 扩展可以子类化 `numbers.Integral`，使用固定的位数实现 `int` 类型。

`singledispatch` 机制的一个显著特征是，你可以在系统的任何地方和任何模块中注册专门函数。如果后来在新的模块中定义了新的类型，可以轻松地添加一个新的专门函数来处理那个类型。此外，你还可以为不是自己编写的或者不能修改的类添加自定义函数。 

## 叠放装饰器

把 `@d1` 和 `@d2` 两个装饰器按顺序应用到 `f` 函数上，作用相当于 `f = d1(d2(f))`。 

也就是说，下述代码：

```py
@d1
@d2
def f()
    print('f')
```

等同于：

```py
def f():
    print('f')

f = d1(d2(f))
```

## 参数化装饰器

解析源码中的装饰器时，**Python** 把被装饰的函数作为第一个参数传给装 饰器函数。那怎么让装饰器接受其他参数呢？答案是：创建一个装饰器 工厂函数，把参数传给它，返回一个装饰器，然后再把它应用到要装饰的函数上。

示例 7-22 示例 7-2 中 `registration.py` 模块的删减版

```py
registry = [] 

def register(func):
    print('running register(%s)' % func)
    registry.append(func)
    return func 
    
@register 
def f1():
    print('running f1()') 
    
print('running main()') 
print('registry ->', registry) 
f1() 
```

### 一个参数化的注册装饰器

为了便于启用或禁用 `register` 执行的函数注册功能，我们为它提供一个可选的 `active` 参数，设为 `False` 时，不注册被装饰的函数。实现方式参见示例 7-23。从概念上看，这个新的 `register` 函数不是装饰器， 而是装饰器工厂函数。调用它会返回真正的装饰器，这才是应用到目标函数上的装饰器。 

示例 7-23 为了接受参数，新的 `register` 装饰器必须作为函数调用

```py
registry = set()  ➊ 

def register(active=True):  ➋
    def decorate(func):  ➌
        print('running register(active=%s)->decorate(%s)'
                % (active, func))
        if active:   ➍
            registry.add(func)
        else:
            registry.discard(func)  ➎
        return func  ➏
        
    return decorate  ➐ 
    
@register(active=False)  ➑ 
def f1():
    print('running f1()') 
    
@register()  ➒ 
def f2():
    print('running f2()') 
    
def f3():
    print('running f3()') 
```

❶ `registry` 现在是一个 `set` 对象，这样添加和删除函数的速度更快。 

❷ `register` 接受一个可选的关键字参数。 

❸ `decorate` 这个内部函数是真正的装饰器；注意，它的参数是一个函数。 

❹ 只有 `active` 参数的值（从闭包中获取）是 `True` 时才注册 `func`。

❺ 如果 `active` 不为真，而且 `func` 在 `registry` 中，那么把它删除。 

❻ `decorate` 是装饰器，必须返回一个函数。 

❼ `register` 是装饰器工厂函数，因此返回 `decorate`。 

❽ `@register` 工厂函数必须作为函数调用，并且传入所需的参数。 

❾ 即使不传入参数，`register` 也必须作为函数调用（`@register()`），即要返回真正的装饰器 `decorate`。 

这里的关键是，`register()` 要返回 `decorate`，然后把它应用到被装 饰的函数上。

示例 7-23 中的代码在 `registration_param.py` 模块中。如果导入，得到的结果如下：

```
>>> import registration_param 
running register(active=False)->decorate(<function f1 at 0x10063c1e0>) 
running register(active=True)->decorate(<function f2 at 0x10063c268>) 
>>> registration_param.registry 
{<function f2 at 0x10063c268>} 
```

如果不使用 `@` 句法，那就要像常规函数那样使用 `register`；若想把 `f` 添加到 `registry` 中，则装饰 `f` 函数的句法是 `register()(f)`；不想添加（或把它删除）的话，句法是 `register(active=False)(f)`。示 例 7-24 演示了如何把函数添加到 `registry`中，以及如何从中删除函 数。 

```py
>>> from registration_param import * 
running register(active=False)->decorate(<function f1 at 0x10073c1e0>
running register(active=True)->decorate(<function f2 at 0x10073c268>)
>>> registry # ➊
{<function f2 at 0x10073c268>}
>>> register()(f3) # ➋
running register(active=True)->decorate(<function f3 at 0x10073c158>)
<function f3 at 0x10073c158>
>>> registry # ➌
{<function f3 at 0x10073c158>, <function f2 at 0x10073c268>}
>>> register(active=False)(f2) # ➍
running register(active=False)->decorate(<function f2 at 0x10073c268>)
<function f2 at 0x10073c268>
>>> registry # ➎
{<function f3 at 0x10073c158>}
```

❶ 导入这个模块时，`f2` 在 `registry` 中。 

❷ `register()` 表达式返回 `decorate`，然后把它应用到 `f3` 上。 

❸ 前一行把 `f3` 添加到 `registry` 中。 

❹ 这次调用从 `registry` 中删除 `f2`。 

❺ 确认 `registry` 中只有 `f3`。 
