# 对象引用、可变性和垃圾回收

## 变量不是盒子

`Lynn Andrea Stein` 教授 （一位获奖的计算机科学教育工作者，目前在欧林工程学院教书）指出，人们经常使用“变量是盒子”这样的比喻，但是这有碍于理解面向对象语言中的引用式变量。**Python** 变量类似于 **Java** 中的引用式变量，因此 最好把它们理解为附加在对象上的标注。 

在示例 8-1 所示的交互式控制台中，无法使用“变量是盒子”做解释。图 8-1 说明了在 **Python** 中为什么不能使用盒子比喻，而便利贴则指出了变量的正确工作方式。

示例 8-1 变量 `a` 和 `b` 引用同一个列表，而不是那个列表的副本 

```py
>>> a = [1, 2, 3] 
>>> b = a 
>>> a.append(4) 
>>> b 
[1, 2, 3, 4] 
```

![Z41Elj.png](https://s2.ax1x.com/2019/07/13/Z41Elj.png)

> 图 8-1：如果把变量想象为盒子，那么无法解释 Python 中的赋值； 应该把变量视作便利贴，这样示例 8-1 中的行为就好解释了 

Stein 教授还反复讲解了赋值方式。例如讲到 `seesaw` 对象时，她会说“把 变量 `s` 分配给 `seesaw`”，绝不会说“把 `seesaw` 分配给变量 `s`”。对引用式 变量来说，说把变量分配给对象更合理，反过来说就有问题。毕竟，对象在赋值之前就创建了。

## 标识、相等性和别名

### 在==和is之间选择

== 运算符比较两个对象的值（对象中保存的数据），而 is 比较对象的标识。 

### 默认做浅复制

复制列表（或多数内置的可变集合）最简单的方式是使用内置的类型构造方法。例如： 

```py
>>> l1 = [3, [55, 44], (7, 8, 9)] 
>>> l2 = list(l1)  ➊ 
>>> l2 
[3, [55, 44], (7, 8, 9)] 
>>> l2 == l1  ➋ 
True 
>>> l2 is l1  ➌ 
False 
```

❶ `list(l1)` 创建 `l1` 的副本。 

❷ 副本与源列表相等。 

❸ 但是二者指代不同的对象。对列表和其他可变序列来说，还能使用简洁的 `l2 = l1[:]` 语句创建副本。 

然而，构造方法或 `[:]` 做的是浅复制（即复制了最外层容器，副本中的元素是源容器中元素的引用）。如果所有元素都是不可变的，那么这样没有问题，还能节省内存。但是，如果有可变的元素，可能就会导致意想不到的问题。 

在示例 8-6 中，我们为一个包含另一个列表和一个元组的列表做了浅复制，然后做了些修改，看看对引用的对象有什么影响。

```py
l1 = [3, [66, 55, 44], (7, 8, 9)] 
l2 = list(l1)      # ➊ 
l1.append(100)     # ➋ 
l1[1].remove(55)   # ➌ 
print('l1:', l1) 
print('l2:', l2) 
l2[1] += [33, 22]  # ➍ 
l2[2] += (10, 11)  # ➎ 
print('l1:', l1) 
print('l2:', l2) 
```

❶ `l2` 是 `l1` 的浅复制副本。此时的状态如图 8-3 所示。

![Z43OsA.png](https://s2.ax1x.com/2019/07/13/Z43OsA.png)

> 图 8-3：示例 8-6 执行 l2 = list(l1) 赋值后的程序状态。l1 和 l2 指代不同的列表，但是二者引用同一个列表 [66, 55, 44] 和元组 (7, 8, 9)（图表由 Python Tutor 网站生成）

❷ 把 `100` 追加到 `l1` 中，对 `l2` 没有影响。 

❸ 把内部列表 `l1[1]` 中的 `55` 删除。这对 `l2` 有影响，因为 `l2[1]` 绑定的列表与 `l1[1]` 是同一个。

❹ 对可变的对象来说，如 `l2[1]` 引用的列表，`+=` 运算符就地修改列 表。这次修改在 `l1[1]` 中也有体现，因为它是 `l2[1]` 的别名。 

❺ 对元组来说，`+=` 运算符创建一个新元组，然后重新绑定给变量 `l2[2]`。这等同于 `l2[2] = l2[2] + (10, 11)`。现在，`l1` 和 `l2` 中最后位置上的元组不是同一个对象。如图 8-4 所示。

示例 8-6 的输出在示例 8-7 中，对象的最终状态如图 8-4 所示。 

示例 8-7 示例 8-6 的输出 

```py
l1: [3, [66, 44], (7, 8, 9), 100] 
l2: [3, [66, 44], (7, 8, 9)] 
l1: [3, [66, 44, 33, 22], (7, 8, 9), 100] 
l2: [3, [66, 44, 33, 22], (7, 8, 9, 10, 11)] 
```

![Z481y9.png](https://s2.ax1x.com/2019/07/13/Z481y9.png)

> 图 8-4：l1 和 l2 的最终状态：二者依然引用同一个列表对象，现在列表的值是 [66, 44, 33, 22]，不过 l2[2] += (10, 11) 创建一 个新元组，内容是 (7, 8, 9, 10, 11)，它与 l1[2] 引用的元组 (7, 8, 9) 无关（图表由 Python Tutor 网站生成）

## 函数的参数作为引用时 

**Python** 唯一支持的参数传递模式是共享传参（`call by sharing`）。

共享传参指函数的各个形式参数获得实参中各个引用的副本。也就是说，函数内部的形参是实参的别名。

这种方案的结果是，函数可能会修改作为参数传入的可变对象，但是无 法修改那些对象的标识（即不能把一个对象替换成另一个对象）。示例 8-11 中有个简单的函数，它在参数上调用 `+=` 运算符。分别把数字、列 表和元组传给那个函数，实际传入的实参会以不同的方式受到影响。 

示例 8-11 函数可能会修改接收到的任何可变对象 

```py
>>> def f(a, b): 
...     a += b 
...     return a 
... 
>>> x = 1 
>>> y = 2
>>> f(x, y) 
3 
>>> x, y  ➊ 
(1, 2) 
>>> a = [1, 2] 
>>> b = [3, 4] 
>>> f(a, b) 
[1, 2, 3, 4] 
>>> a, b  ➋ 
([1, 2, 3, 4], [3, 4]) 
>>> t = (10, 20) 
>>> u = (30, 40) 
>>> f(t, u) 
(10, 20, 30, 40) 
>>> t, u ➌ 
((10, 20), (30, 40)) 
```

❶ 数字 `x` 没变。

❷ 列表 `a` 变了。 

❸ 元组 `t` 没变。 

### 不要使用可变类型作为参数的默认值 

可选参数可以有默认值，这是 **Python** 函数定义的一个很棒的特性，这样 我们的 API 在进化的同时能保证向后兼容。然而，我们应该避免使用可 变的对象作为参数的默认值。

### 防御可变参数

如果定义的函数接收可变参数，应该谨慎考虑调用方是否期望修改传入的参数。

例如，如果函数接收一个字典，而且在处理的过程中要修改它，那么这个副作用要不要体现到函数外部？具体情况具体分析。这其实需要函数的编写者和调用方达成共识。 

示例 8-14 从 `TwilightBus` 下车后，乘客消失了 

```py
>>> basketball_team = ['Sue', 'Tina', 'Maya', 'Diana', 'Pat']  ➊ 
>>> bus = TwilightBus(basketball_team)  ➋ 
>>> bus.drop('Tina')  ➌ 
>>> bus.drop('Pat') 
>>> basketball_team  ➍ 
['Sue', 'Maya', 'Diana'] 
```

❶ `basketball_team` 中有 `5` 个学生的名字。 

❷ 使用这队学生实例化 `TwilightBus`。 

❸ 一个学生从 `bus` 下车了，接着又有一个学生下车了。 

❹ 下车的学生从篮球队中消失了！

`TwilightBus` 违反了设计接口的最佳实践，即“最少惊讶原则”。学生从校车中下车后，她的名字就从篮球队的名单中消失了，这确实让人惊 讶。

示例 8-15 是 `TwilightBus` 的实现，随后解释了出现这个问题的原因。 

示例 8-15 一个简单的类，说明接受可变参数的风险 

```py
class TwilightBus:
    """让乘客销声匿迹的校车"""
    
    def __init__(self, passengers=None):
        if passengers is None:
            self.passengers = []   ➊
        else:
            self.passengers = passenfers  ➋
        
    def pick(self, name):
        self.passengers.append(name)
        
    def drop(self, name):
        self.passengers.remove(name)   ➌ 
```

❶ 这里谨慎处理，当 `passengers` 为 `None` 时，创建一个新的空列表。 

❷ 然而，这个赋值语句把 `self.passengers` 变成 `passengers` 的别名，而后者是传给 `__init__` 方法的实参（即示例 8-14 中的 `basketball_team）`的别名。 

❸ 在 `self.passengers` 上调用 `.remove()` 和 `.append()` 方法其实会修改传给构造方法的那个列表。 

这里的问题是，校车为传给构造方法的列表创建了别名。正确的做法 是，校车自己维护乘客列表。修正的方法很简单：在 `__init__ 中`，传 入 `passengers` 参数时，应该把参数值的副本赋值给 `self.passengers`，像示例 8-8 中那样做（8.3 节）。

```py
def __init__(self, passengers):
    if passengers is None:
        self.passengers = []
    else:
        self.passengers = list(passengers) ➊ 
```

➊ 创建 `passengers` 列表的副本；如果不是列表，就把它转换成列表。 

在内部像这样处理乘客列表，就不会影响初始化校车时传入的参数了。 此外，这种处理方式还更灵活：现在，传给 `passengers` 参数的值可以 是元组或任何其他可迭代对象，例如 set 对象，甚至数据库查询结果， 因为 list 构造方法接受任何可迭代对象。

## del和垃圾回收

`del` 语句删除名称，而不是对象。`del`命令可能会导致对象被当作垃圾回收，但是仅当删除的变量保存的是对象的最后一个引用，或者无法得到对象时。重新绑定也可能会导致对象的引用数量归零，导致对象被销毁。 

> 如果两个对象相互引用，当它们的引用只存在二者之间时，垃圾回收程序会判定它们都无法获取，进而把它们都销毁。

在 **CPython** 中，垃圾回收使用的主要算法是引用计数。实际上，每个对 象都会统计有多少引用指向自己。当引用计数归零时，对象立即就被销 毁：**CPython** 会在对象上调用 `__del__` 方法（如果定义了），然后释放 分配给对象的内存。**CPython 2.0** 增加了分代垃圾回收算法，用于检测 引用循环中涉及的对象组——如果一组对象之间全是相互引用，即使再 出色的引用方式也会导致组中的对象不可获取。**Python** 的其他实现有更 复杂的垃圾回收程序，而且不依赖引用计数，这意味着，对象的引用数 量为零时可能不会立即调用 `__del__` 方法。

为了演示对象生命结束时的情形，示例 8-16 使用 `weakref.finalize` 注册一个回调函数，在销毁对象时调用。 

示例 8-16 没有指向对象的引用时，监视对象生命结束时的情形 

```py
>>> import weakref 
>>> s1 = {1, 2, 3} 
>>> s2 = s1       ➊ 
>>> def bye():    ➋ 
...     print('Gone    with the wind...') 
... 
>>> ender = weakref.finalize(s1, bye)  ➌ 
>>> ender.alive  ➍ 
True 
>>> del s1 
>>> ender.alive  ➎ 
True 
>>> s2 = 'spam'  ➏ 
Gone with the wind... 
>>> ender.alive 
False 
```

❶ `s1` 和 `s2` 是别名，指向同一个集合，`{1, 2, 3}`。 

❷ 这个函数一定不能是要销毁的对象的绑定方法，否则会有一个指向对象的引用。 

❸ 在 `s1` 引用的对象上注册 `bye` 回调。 

❹ 调用 `finalize` 对象之前，`.alive` 属性的值为 `True`。 

❺ 如前所述，`del` 不删除对象，而是删除对象的引用。 

❻ 重新绑定最后一个引用 `s2`，让 `{1, 2, 3}` 无法获取。对象被销毁了，调用了 `bye` 回调，`ender.alive` 的值变成了 `False`。 

## 弱引用

正是因为有引用，对象才会在内存中存在。当对象的引用数量归零后，垃圾回收程序会把对象销毁。但是，有时需要引用对象，而不让对象存在的时间超过所需时间。这经常用在缓存中。 

弱引用不会增加对象的引用数量。引用的目标对象称为所指对象 （`referent`）。因此我们说，弱引用不会妨碍所指对象被当作垃圾回收。 

弱引用在缓存应用中很有用，因为我们不想仅因为被缓存引用着而始终保存缓存对象。 

示例 8-17 展示了如何使用 `weakref.ref` 实例获取所指对象。如果对象存在，调用弱引用可以获取对象；否则返回 `None`。

示例 8-17 弱引用是可调用的对象，返回的是被引用的对象；如果 所指对象不存在了，返回 None 

```py
>>> import weakref
>>> a_set = {0, 1}
>>> wref = weakref.ref(a_set)  ➊
>>> wref 
<weakref at 0x100637598; to 'set' at 0x100636748>
>>> wref()  ➋
{0, 1}
>>> a_set = {2, 3, 4}  ➌
>>> wref()  ➍
{0, 1}
>>> wref() is None  ➎
False
>>> wref() is None  ➏
True
```

❶ 创建弱引用对象 `wref`，下一行审查它。 

❷ 调用 `wref()` 返回的是被引用的对象，`{0, 1}`。因为这是控制台会 话，所以 `{0, 1}` 会绑定给 `_` 变量。 

❸ `a_set` 不再指代 `{0, 1}` 集合，因此集合的引用数量减少了。但是 `_` 变量仍然指代它。 

❹ 调用 `wref()` 依旧返回 `{0, 1}`。 

❺ 计算这个表达式时，`{0, 1}` 存在，因此 `wref()` 不是 `None`。但是， 随后 `_` 绑定到结果值 `False`。现在 `{0, 1}` 没有强引用了。 

❻ 因为 `{0, 1}` 对象不存在了，所以 `wref()` 返回 `None`。

## Python对不可变类型施加的把戏

对元组 `t` 来说，`t[:]` 不创建副本，而是返回同一个对象的引用。此外，`tuple(t)` 获得的也是同一个元组的引用

示例 8-20 使用另一个元组构建元组，得到的其实是同一个元组 

```py
>>> t1 = (1, 2, 3) 
>>> t2 = tuple(t1) 
>>> t2 is t1  ➊ 
True 
>>> t3 = t1[:] 
>>> t3 is t1  ➋ 
True 
```

❶ `t1` 和 `t2` 绑定到同一个对象。 

❷ `t3` 也是。 

`str`、`bytes` 和 `frozenset` 实例也有这种行为。注意，`frozenset` 实 例不是序列，因此不能使用 `fs[:]`（`fs` 是一个 `frozenset` 实例）。但是，`fs.copy()` 具有相同的效果：它会欺骗你，返回同一个对象的引用，而不是创建一个副本

示例 8-21 字符串字面量可能会创建共享的对象 

```py
>>> t1 = (1, 2, 3) 
>>> t3 = (1, 2, 3)  # ➊ 
>>> t3 is t1  # ➋ 
False 
>>> s1 = 'ABC' 
>>> s2 = 'ABC'  # ➌ 
>>> s2 is s1  # ➍ 
True 
```

❶ 新建一个元组。 

❷ `t1` 和 `t3` 相等，但不是同一个对象。 

❸ 再新建一个字符串。 

❹ 奇怪的事发生了，`a` 和 `b` 指代同一个字符串。

共享字符串字面量是一种优化措施，称为驻留（`interning`）。**CPython** 还 会在小的整数上使用这个优化措施，防止重复创建“热门”数字，如 `0`、`-1` 和 `42`。注意，**CPython** 不会驻留所有字符串和整数，驻留的条件 是实现细节，而且没有文档说明。