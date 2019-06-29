# Python数据模型

## 如何使用特殊方法

* 特殊方法的存在是为了被`Python`解释器调用的,你自己并不需要调用它们.也就是说没有`my_object.__len()__`这种写法,而应该使用`len(my_object)`在执行 `len(my_object)` 的时候，如果 `my_object` 是一个自定义类的对象，那么`Python`会自己去调用其中由 你实现的 `__len__` 方法。 

* 如果是 `Python` 内置的类型，比如列表`（list）`、字符串`（str）`、 字节序列`（bytearray）`等，那么 `CPython` 会抄个近路，`__len__` 实际 上会直接返回 `PyVarObject` 里的 `ob_size` 属性。`PyVarObject` 是表示 内存中长度可变的内置对象的`C`语言结构体。直接读取这个值比调用一个方法要快很多。 

* 很多时候，特殊方法的调用是隐式的，比如 `for i in x:` 这个语句， 背后其实用的是 `iter(x)`，而这个函数的背后则是 `x.__iter__()` 方 法。当然前提是这个方法在 `x` 中被实现了。

* 通常你的代码无需直接使用特殊方法。除非有大量的元编程存在，直接 调用特殊方法的频率应该远远低于你去实现它们的次数。唯一的例外可 能是 `__init__` 方法，你的代码里可能经常会用到它，目的是在你自己 的子类的 `__init__` 方法中调用超类的构造器。

* 通过内置的函数（例如 `len`、`iter`、`str`，等等）来使用特殊方法是最好的选择。这些内置函数不仅会调用特殊方法，通常还提供额外的好处，而且对于内置的类来说，它们的速度更快。

* 不要自己想当然地随意添加特殊方法，比如 `__foo__` 之类的，因为虽 然现在这个名字没有被 `Python` 内部使用，以后就不一定了。 

### 例子

我们来实现一个二维向量（`vector`）类，这里的向量就是欧几里得几何 中常用的概念，常在数学和物理中使用的那个。

![ZlmtPA.png](https://s2.ax1x.com/2019/06/29/ZlmtPA.png)

> 一个二维向量加法的例子，Vector(2,4) + Vextor(2,1) = Vector(4,5)

为了给这个类设计 API，我们先写个模拟的控制台会话来做 doctest。下面这一段代码就是如图所示的向量加法：

![Zlmoa4.png](https://s2.ax1x.com/2019/06/29/Zlmoa4.png)

注意其中的 `+` 运算符所得到的结果也是一个向量，而且结果能被控制台 友好地打印出来。 

`abs` 是一个内置函数，如果输入是整数或者浮点数，它返回的是输入值 的绝对值；如果输入是复数`（complex number）`，那么返回这个复数的模。为了保持一致性，我们的`API`在碰到`abs`函数的时候，也应该返回该向量的模： 

![ZlnpIH.png](https://s2.ax1x.com/2019/06/29/ZlnpIH.png)

Vector 类的实现:


```python
from math import hypot

class Vector:
    def __init(self, x=0, y=0)
        self.x = x
        self.y = y
    
    def __repr__(self):
        return 'Vector(%r, %r)' % (self.x,  slef.y)
    
    def __abs__(self):
        return hypot(self.x, self.y)
    
    def __bool__(self):
        return bool(abs(self))
    
    def __add__(self, other):
        x = self.x + other.x
        y = self.y + other.y
    
    def __mul__(self, scalar):
        return Vector(self.x * scalar, self.y * scalar)
```


















