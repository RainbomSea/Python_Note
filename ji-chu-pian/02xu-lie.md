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

### 生成器表达式

生成器表达式背后遵守了迭代器协议，可以逐个地产出元素，而不是先建立一个完整的列表，然后再把这个列表传递到某个构造函数里

生成器表达式的语法跟列表推导差不多，只不过把方括号换成圆括号而 已。

用生成器表达式初始化元组和数组:

```py
>>> symbols = '$¢£¥€¤' 
>>> tuple(ord(symbol) for symbol in symbols) ➊ 
(36, 162, 163, 165, 8364, 164) 
>>> import array 
>>> array.array('I', (ord(symbol) for symbol in symbols)) ➋
array('I', [36, 162, 163, 165, 8364, 164])
```

➊ 如果生成器表达式是一个函数调用过程中的唯一参数，那么不需要额外再用括号把它围起来。  
➋ `array` 的构造方法需要两个参数，因此括号是必需的。array 构造方法的第一个参数指定了数组中数字的存储方式。

使用生成器表达式计算笛卡儿积:

```py
>>> colors = ['black', 'white'] 
>>> sizes = ['S', 'M', 'L'] 
>>> for tshirt in ('%s %s' % (c, s) for c in colors for s in sizes): ➊
...     print(tshirt) 
... 
black S 
black M 
black L
white S 
white M 
white L
```

➊ 生成器表达式逐个产出元素，从来不会一次性产出一个含有 6 个 T 恤样式的列表。

## 元组不仅仅是不可变的列表

除了用作不可变的列表，它还可以用于没有字段名的记录。

### 元组和记录

元组其实是对数据的记录：元组中的每个元素都存放了记录中一个字段的数据，外加这个字段的位置。正是这个位置信息给数据赋予了意义。

把元组用作记录:

```py
>>> lax_coordinates = (33.9425, -118.408056)  ➊ 
>>> city, year, pop, chg, area = ('Tokyo', 2003, 32450, 0.66, 8014)  ➋ 
>>> traveler_ids = [('USA', '31195855'), ('BRA', 'CE342567'),  ➌ 
...     ('ESP', 'XDA205856')] 
>>> for passport in sorted(traveler_ids):  ➍ 
...     print('%s/%s' % passport) ➎ 
... 
BRA/CE342567 
ESP/XDA205856 
USA/31195855 
>>> for country, _ in traveler_ids:  ➏ 
...     print(country) 
... 
USA 
BRA 
ESP
```

❶ 洛杉矶国际机场的经纬度。   
❷ 东京市的一些信息：市名、年份、人口（单位：百万）、人口变化 （单位：百分比）和面积（单位：平方千米）。   
❸ 一个元组列表，元组的形式为 \(`country_code`, `passport_number`\)。   
❹ 在迭代的过程中，`passport` 变量被绑定到每个元组上。   
❺ `%` 格式运算符能被匹配到对应的元组元素上。   
❻ `for` 循环可以分别提取元组里的元素，也叫作拆包（`unpacking`）。因为元组中第二个元素对我们没有什么用，所以它赋值给“`_`”占位符

### 元组拆包

元组拆包可以应用到任何可迭代对象上，唯一的硬性要求是，被可迭代对象中的元素数量必须要跟接受这些元素的元组的空 档数一致。除非我们用 \* 来表示忽略多余的元素

```py
>>> lax_coordinates = (33.9425, -118.408056) 
>>> latitude, longitude = lax_coordinates # 元组拆包 
>>> latitude 
33.9425 
>>> longitude 
-118.408056
```

另外一个很优雅的写法当属不使用中间变量交换两个变量的值:

```py
>>> b, a = a, b
```

还可以用 \* 运算符把一个可迭代对象拆开作为函数的参数：

```py
>>> divmod(20, 8) 
(2, 4) 
>>> t = (20, 8) 
>>> divmod(*t) 
(2, 4) 
>>> quotient, remainder = divmod(*t) 
>>> quotient, remainder 
(2, 4)
```

下面是另一个例子，这里元组拆包的用法则是让一个函数可以用元组的 形式返回多个值，然后调用函数的代码就能轻松地接受这些返回值。比 如 os.path.split\(\) 函数就会返回以路径和最后一个文件名组成的元 组 \(path, last\_part\):

```py
>>> import os 
>>> _, filename = os.path.split('/home/luciano/.ssh/idrsa.pub') 
>>> filename 
'idrsa.pub'
```

在进行拆包的时候，我们不总是对元组里所有的数据都感兴趣，`_`占位符能帮助处理这种情况，上面这段代码也展示了它的用法。

除此之外，在元组拆包中使用 \* 也可以帮助我们把注意力集中在元组的部分元素上。

用`*`来处理剩下的元素

在 **Python** 中，函数用 `*args` 来获取不确定数量的参数算是一种经典写法了。

于是 **Python 3** 里，这个概念被扩展到了平行赋值中：

```py
>>> a, b, *rest = range(5) 
>>> a, b, rest 
(0, 1, [2, 3, 4]) 
>>> a, b, *rest = range(3) 
>>> a, b, rest 
(0, 1, [2]) 
>>> a, b, *rest = range(2) 
>>> a, b, rest 
(0, 1, [])
```

在平行赋值中，`*`前缀只能用在一个变量名前面，但是这个变量可以出现在赋值表达式的任意位置：

```py
>>> a, *body, c, d = range(5) 
>>> a, body, c, d 
(0, [1, 2], 3, 4) 
>>> *head, b, c, d = range(5) 
>>> head, b, c, d 
([0, 1], 2, 3, 4)
```

### 嵌套元组拆包

接受表达式的元组可以是嵌套式的，例如 `(a, b, (c, d))`。只要这个接受元组的嵌套结构符合表达式本身的嵌套结构，**Python**就可以作出正确的对应。

用嵌套元组来获取经度:

```py
metro_areas = [
    ('Tokyo','JP',36.933,(35.689722,139.691667)),  # ➊
    ('Delhi NCR', 'IN', 21.935, (28.613889, 77.208889)),
    ('Mexico City', 'MX', 20.142, (19.433333, -99.133333)),
    ('New York-Newark', 'US', 20.104, (40.808611, -74.020386)),
    ('Sao Paulo', 'BR', 19.649, (-23.547778, -46.635833)), 
] 
print('{:15} | {:^9} | {:^9}'.format('', 'lat.', 'long.')) 
fmt = '{:15} | {:9.4f} | {:9.4f}' 
for name, cc, pop, (latitude, longitude) in metro_areas:  # ➋
    if longitude <= 0:  # ➌
        print(fmt.format(name, latitude, longitude))
```

❶ 每个元组内有 4 个元素，其中最后一个元素是一对坐标。   
❷ 我们把输入元组的最后一个元素拆包到由变量构成的元组里，这样 就获取了坐标。   
❸ `if longitude <= 0`: 这个条件判断把输出限制在西半球的城市。

输出:

```py
                |   lat.    |   long. 
Mexico City     |   19.4333 |  -99.1333 
New York-Newark |   40.8086 |  -74.0204 
Sao Paul        |  -23.5478 |  -46.6358
```

>在 **Python3** 之前，元组可以作为形参放在函数声明中，例如 `def fn(a, (b, c), d):`。然而 **Python3** 不再支持这种格式，具体原因见于[PEP 3113—Removal of Tuple Parameter Unpacking](http://python.org/dev/peps/pep-3113/)。需要弄清楚的是，这个改变对函数调用者并没有影响，它改变的是某些函数的声明方式。

元组已经设计得很好用了，但作为记录来用的话，还是少了一个功能：我们时常会需要给记录中的字段命名。

### 具名数组

`collections.namedtuple`是一个工厂函数，它可以用来构建一个带字段名的元组和一个有名字的类——这个带名字的类对调试程序有很大帮助。

> 用`namedtuple`构建的类的实例所消耗的内存跟元组是一样的，因为字段名都被存在对应的类里面。这个实例跟普通的对象实例比起来也要小一些，因为 **Python** 不会用 `__dict__` 来存放这些实例的属性。

定义和使用具名元组:

```py
>>> from collections import namedtuple 
>>> City = namedtuple('City', 'name country population coordinates')  ➊ 
>>> tokyo = City('Tokyo', 'JP', 36.933, (35.689722, 139.691667))  ➋ 
>>> tokyo 
City(name='Tokyo', country='JP', population=36.933, coordinates=(35.689722, 139.691667)) 
>>> tokyo.population  ➌ 
36.933 
>>> tokyo.coordinates (35.689722, 139.691667) 
>>> tokyo[1] 
'JP' 
```

❶ 创建一个具名元组需要两个参数，一个是类名，另一个是类的各个字段的名字。后者可以是由数个字符串组成的可迭代对象，或者是由空 格分隔开的字段名组成的字符串。 
❷ 存放在对应字段里的数据要以一串参数的形式传入到构造函数中 （注意，元组的构造函数却只接受单一的可迭代对象）。
❸ 你可以通过字段名或者位置来获取一个字段的信息。 

除了从普通元组那里继承来的属性之外，具名元组还有一些自己专有的属性：`_fields`类属性、类方法`_make(iterable)`和实例方法 `_asdict()`。 

```py
>>> City._fields  ➊ 
('name', 'country', 'population', 'coordinates') 
>>> LatLong = namedtuple('LatLong', 'lat long') 
>>> delhi_data = ('Delhi NCR', 'IN', 21.935, LatLong(28.613889, 77.208889)) 
>>> delhi = City._make(delhi_data)  ➋ 
>>> delhi._asdict()  ➌ 
OrderedDict([('name', 'Delhi NCR'), ('country', 'IN'), ('population',
21.935), ('coordinates', LatLong(lat=28.613889, long=77.208889))]) 
>>> for key, value in delhi._asdict().items():
        print(key + ':', value) 
name: Delhi NCR 
country: IN 
population: 21.935 
coordinates: LatLong(lat=28.613889, long=77.208889) 
```

❶ `_fields`属性是一个包含这个类所有字段名称的元组。 
❷ 用`_make()`通过接受一个可迭代对象来生成这个类的一个实例，它的作用跟`City(*delhi_data)`是一样的。 
❸ `_asdict()` 把具名元组以`collections.OrderedDict`的形式返回，我们可以利用它来把元组里的信息友好地呈现出来。 

## 切片

### 为什么切片和区间会忽略最后一个元素

* 当只有最后一个位置信息时，我们也可以快速看出切片和区间里有几个元素：`range(3)` 和 `my_list[:3]` 都返回 `3` 个元素。 

* 当起止位置信息都可见时，我们可以快速计算出切片和区间的长度，用后一个数减去第一个下标`（stop - start）`即可。 

* 这样做也让我们可以利用任意一个下标来把序列分割成不重叠的两部分，只要写成 `my_list[:x]` 和 `my_list[x:]` 就可以了，如下所示。

  ```py
  >>> l = [10, 20, 30, 40, 50, 60] 
  >>> l[:2] # 在下标2的地方分割 
  [10, 20] 
  >>> l[2:] 
  [30, 40, 50, 60] 
  >>> l[:3] # 在下标3的地方分割 
  [10, 20, 30]
  >>> l[3:] 
  [40, 50, 60] 
  ```

### 对对象进行切片 

一个众所周知的秘密是，我们还可以用 `s[a:b:c]` 的形式对 `s` 在 `a` 和 `b` 之间以 `c` 为间隔取值。`c`的值还可以为负，负值意味着反向取值。

  ```py
  >>> s = 'bicycle' 
  >>> s[::3] 
  'bye' 
  >>> s[::-1] 
  'elcycib' 
  >>> s[::-2] 
  'eccb' 
  ```
  
`a:b:c` 这种用法只能作为索引或者下标用在 `[]` 中来返回一个切片对象：`slice(a, b, c)`。对 `seq[start:stop:step]` 进行求值的时候，**Python** 会调用 `seq.__getitem__(slice(start, stop, step))`

  ```py
  >>> invoice = """
  ... 0.....6................................40........52...55........ 
  ... 1909  Pimoroni PiBrella                    $17.50    3    $52.50
  ... 1489  6mm Tactile Switch x20                $4.95    2     $9.90 
  ... 1510  Panavise Jr. - PV-201                $28.00    1    $28.00 
  ... 1601  PiTFT Mini Kit 320x240               $34.95    1    $34.95 
  ... """ 
  >>> SKU = slice(0, 6) 
  >>> DESCRIPTION = slice(6, 40) 
  >>> UNIT_PRICE = slice(40, 52) 
  >>> QUANTITY = slice(52, 55) 
  >>> ITEM_TOTAL = slice(55, None) 
  >>> line_items = invoice.split('\n')[2:] 
  >>> for item in line_items: 
  ...     print(item[UNIT_PRICE], item[DESCRIPTION]) 
  ...
      $17.50   Pimoroni PiBrella
      $4.95   6mm Tactile Switch x20
      $28.00   Panavise Jr. - PV-201
      $34.95   PiTFT Mini Kit 320x240 
  ```

