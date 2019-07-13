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








