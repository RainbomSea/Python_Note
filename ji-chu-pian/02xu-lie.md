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

> 在 **Python3** 之前，元组可以作为形参放在函数声明中，例如 `def fn(a, (b, c), d):`。然而 **Python3** 不再支持这种格式，具体原因见于[PEP 3113—Removal of Tuple Parameter Unpacking](http://python.org/dev/peps/pep-3113/)。需要弄清楚的是，这个改变对函数调用者并没有影响，它改变的是某些函数的声明方式。

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

### 多维切片和省略

`[]` 运算符里还可以使用以逗号分开的多个索引或者是切片，外部库 `NumPy` 里就用到了这个特性，二维的 `numpy.ndarray` 就可以用 `a[i, j]` 这种形式来获取，抑或是用 `a[m:n, k:l]` 的方式来得到二维切片。要正确处理这种 `[]` 运算符的话，对象的特殊方法 `__getitem__` 和 `__setitem__` 需要以元组的形式来接收 `a[i, j]` 中的索引。也就是说，如果要得到 `a[i, j]` 的值，Python 会调用 `a.__getitem__((i, j))`。

**Python**内置的序列类型都是一维的，因此它们只支持单一的索引，成对出现的索引是没有用的。

**省略**（`ellipsis`）的正确书写方法是三个英语句号（`...`），而不是 `Unicdoe` 码位 `U+2026` 表示的半个省略号（`...`）。省略在 **Python** 解析器眼里是一个符号，而实际上它是 `Ellipsis` 对象的别名，而 `Ellipsis` 对象又是 `ellipsis` 类的单一实例。 它可以当作切片规范的一部分，也可以用在函数的参数清单中，比如 `f(a, ..., z)`，或 `a[i:...]`。在 `NumPy` 中，`...` 用作多维数组切片的快捷方式。如果 `x` 是四维数组，那 么 `x[i, ...]` 就是 `x[i, :, :, :]` 的缩写。

> 是的，你没看错，`ellipsis` 是类名，全小写，而它的内置实例写作 `Ellipsis`。这其实跟 `bool` 是小写，但是它的两个实例写作 `True` 和 `False` 异曲同工。

### 给切片赋值

如果把切片放在赋值语句的左边，或把它作为 `del` 操作的对象，我们就可以对序列进行嫁接、切除或就地修改操作。

```py
>>> l = list(range(10)) 
>>> l 
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9] 
>>> l[2:5] = [20, 30] 
>>> l 
[0, 1, 20, 30, 5, 6, 7, 8, 9] 
>>> del l[5:7] 
>>> l 
[0, 1, 20, 30, 5, 8, 9] 
>>> l[3::2] = [11, 22]
>>> l 
[0, 1, 20, 11, 5, 22, 9] 
>>> l[2:5] = 100  ➊ 
Traceback (most recent call last):
  File "<stdin>", line 1, in <module> 
TypeError: can only assign an iterable
>>> l[2:5] = [100] 
>>> l 
[0, 1, 100, 22, 9]
```

➊ 如果赋值的对象是一个切片，那么赋值语句的右侧必须是个可迭代对象。即便只有单独一个值，也要把它转换成可迭代的序列。

## 对序列使用+和\*

**Python** 程序员会默认序列是支持 `+` 和 `*` 操作的。通常 `+` 号两侧的序列由 相同类型的数据所构成，在拼接的过程中，两个被操作的序列都不会被修改，**Python** 会新建一个包含同样类型数据的序列来作为拼接的结果。

如果想要把一个序列复制几份然后再拼接起来，更快捷的做法是把这个序列乘以一个整数。同样，这个操作会产生一个新序列：

```py
>>> l = [1, 2, 3] 
>>> l * 5 
[1, 2, 3, 1, 2, 3, 1, 2, 3, 1, 2, 3, 1, 2, 3] 
>>> 5 * 'abcd' 
'abcdabcdabcdabcdabcd'
```

`+` 和 `*` 都遵循这个规律，不修改原有的操作对象，而是构建一个全新的序列。

建立由列表组成的列表

有时我们会需要初始化一个嵌套着几个列表的列表，譬如一个列表可能 需要用来存放不同的学生名单，或者是一个井字游戏板上的一行方块。想要达成这些目的，最好的选择是使用列表推导.

```py
>>> board = [['_'] * 3 for i in range(3)]  ➊ 
>>> board 
[['_', '_', '_'], ['_', '_', '_'], ['_', '_', '_']] 
>>> board[1][2] = 'X' ➋ 
>>> board 
[['_', '_', '_'], ['_', '_', 'X'], ['_', '_', '_']]
```

➊ 建立一个包含 `3` 个列表的列表，被包含的 `3` 个列表各自有 `3` 个元 素。

➋ 把第 `1` 行第 `2` 列的元素标记为 `X`，再打印出这个列表。

另一个方法，这个方法看上去是个诱人的捷径，但实际上它是错的。

```py
>>> weird_board = [['_'] * 3] * 3 ➊ 
>>> weird_board 
[['_', '_', '_'], ['_', '_', '_'], ['_', '_', '_']] 
>>> weird_board[1][2] = 'O' ➋ 
>>> weird_board 
[['_', '_', 'O'], ['_', '_', 'O'], ['_', '_', 'O']]
```

➊ 外面的列表其实包含`3`个指向同一个列表的引用。当我们不做修改的时候，看起来都还好。  
➋ 一旦我们试图标记第`1`行第`2`列的元素，就立马暴露了列表内的`3`个引用指向同一个对象的事实。

上面代码错误本质上跟下面的代码犯的错误一样

```py
row=['_'] * 3 
board = [] 
for i in range(3):
  board.append(row)  ➊
```

➊ 追加同一个行对象（`row`）`3` 次到游戏板（`board`）。

相反, 正确的方法等同于这样做：

```py
>>> board = [] 
>>> for i in range(3): 
...     row=['_'] * 3 # ➊ 
...     board.append(row) 
... 
>>> board 
[['_', '_', '_'], ['_', '_', '_'], ['_', '_', '_']] 
>>> board[2][0] = 'X' 
>>> board  # ➋ 
[['_', '_', '_'], ['_', '_', '_'], ['X', '_', '_']]
```

➊ 每次迭代中都新建了一个列表，作为新的一行（`row`）追加到游戏板\(`board`）。

➋ 正如我们所期待的，只有第 `2` 行的元素被修改。

## 序列的增量赋值

增量赋值运算符 `+=` 和 `*=` 的表现取决于它们的第一个操作对象。简单起见，我们把讨论集中在增量加法（`+=`）上，但是这些概念对 `*=` 和其他增量运算符来说都是一样的。

`+=`背后的特殊方法是 `__iadd__`（用于“就地加法”）。但是如果一个类没有实现这个方法的话，**Python**会退一步调用`__add__` 。 考虑下面这 个简单的表达式：

```py
>>> a += b
```

如果 `a` 实现了 `__iadd__` 方法，就会调用这个方法。同时对可变序列（例如 `list`、`bytearray` 和 `array.array`）来说，`a` 会就地改动，就像调用了 `a.extend(b)` 一样。但是如果 `a` 没有实现 `__iadd__` 的话，`a += b` 这个表达式的效果就变得跟 `a = a + b` 一样了：首先计算 `a + b`，得到一个新的对象，然后赋值给 `a`。也就是说，在这个表达式中， 变量名会不会被关联到新的对象，完全取决于这个类型有没有实现 `__iadd__` 这个方法。

总体来讲，可变序列一般都实现了 `__iadd__` 方法，因此 `+=` 是就地加法。而不可变序列根本就不支持这个操作，对这个方法的实现也就无从谈起。

接下来有个小例子，展示的是 `*=` 在可变和不可变序列上的作用：

```py
>>> l = [1, 2, 3] 
>>> id(l) 
4311953800  ➊ 
>>> l *= 2 
>>> l 
[1, 2, 3, 1, 2, 3] 
>>> id(l) 
4311953800  ➋
>>> t = (1, 2, 3) 
>>> id(t) 
4312681568  ➌ 
>>> t *= 2 
>>> id(t) 
4301348296  ➍
```

❶ 刚开始时列表的 `ID`。

❷ 运用增量乘法后，列表的 `ID` 没变，新元素追加到列表上。

❸ 元组最开始的`ID`。

❹ 运用增量乘法后，新的元组被创建。

对不可变序列进行重复拼接操作的话，效率会很低，因为每次都有一个新对象，而解释器需要把原来对象中的元素先复制到新的对象里，然后 再追加新的元素

> `str` 是一个例外，因为对字符串做 `+=` 实在是太普遍了，所以 **CPython** 对它做了优化。为 `str` 初始化内存的时候，程序会为它留出额外的可扩展空间，因此进行增量操作的时候，并不会涉及复制原有字符串到新位置这类操作。

### 一个关于+=的谜题

读完下面的代码，然后回答这个问题：示例中的两个表达式到底会产生什么结果？

```py
>>> t = (1, 2, [30, 40]) 
>>> t[2] += [50, 60]
```

到底会发生下面 4 种情况中的哪一种？

a. t 变成 (1, 2, [30, 40, 50, 60])。 
b. 因为 tuple 不支持对它的元素赋值，所以会抛出 TypeError 异常。 
c. 以上两个都不是。 
d. a 和 b 都是对的。 

答案是 `d`，也就是说 `a` 和 `b` 都是对的！示例是运行这段代码得到的结果，用的**Python** 版本是 `3.4`，但是在 `2.7` 中结果也一样。

没人料到的结果：t[2] 被改动了，但是也有异常抛出 

```py
>>> t = (1, 2, [30, 40]) 
>>> t[2] += [50, 60] 
Traceback (most recent call last):
  File "<stdin>", line 1, in <module> 
TypeError: 'tuple' object does not support item assignment 
>>> t (1, 2, [30, 40, 50, 60]) 
```

下面来看看示例中 **Python** 为表达式 `s[a] += b` 生成的字节码，可能这个现象背后的原因会变得清晰起来。 

```py
>>> dis.dis('s[a] += b')
1           0 LOAD_NAME                  0(s)
            3 LOAD_NAME                  1(a)
            6 DUP_TOP_TWO7 BINARY_SUBSCR          ➊
            8 LOAD_NAME                  2(b)
            11 INPLACE_ADD                        ➋
            12 ROT_THREE
            13 STORE_SUBSCR                       ➌
            14 LOAD_CONST                 0(None)
            17 RETURN_VALUE 
```

➊ 将 `s[a]` 的值存入 `TOS`（`Top Of Stack`，栈的顶端）。 

➋ 计算 `TOS += b`。这一步能够完成，是因为 `TOS` 指向的是一个可变对象（也就是示例里的列表）。 

➌ `s[a] = TOS` 赋值。这一步失败，是因为 `s` 是不可变的元组（示例中的元组 t）。

* 不要把可变对象放在元组里面。 

* 增量赋值不是一个原子操作。我们刚才也看到了，它虽然抛出了异 常，但还是完成了操作。 
* 查看 Python 的字节码并不难，而且它对我们了解代码背后的运行机 制很有帮助。 

## list.sort方法和内置函数sorted

`list.sort` 方法会就地排序列表，也就是说不会把原列表复制一份。这也是这个方法的返回值是 `None` 的原因，提醒你本方法不会新建一个列表。在这种情况下返回 `None` 其实是 **Python** 的一个惯例：如果一个函数或者方法对对象进行的是就地改动，那它就应该返回 `None`，好让调用者知道传入的参数发生了变动，而且并未产生新的对象。

与 `list.sort` 相反的是内置函数 `sorted`，它会新建一个列表作为返回值。这个方法可以接受任何形式的可迭代对象作为参数，甚至包括不可变序列或生成器。而不管 `sorted` 接受的是怎样的参数，它最后都会返回一个列表。 

不管是 `list.sort` 方法还是 `sorted` 函数，都有两个可选的关键字参数。 

* reverse

  如果被设定为 `True`，被排序的序列里的元素会以降序输出（也就是说把最大值当作最小值来排序）。这个参数的默认值是 `False`。 

* key

  一个只有一个参数的函数，这个函数会被用在序列里的每一个元素 上，所产生的结果将是排序算法依赖的对比关键字。比如说，在对一些字符串排序时，可以用 `key=str.lower` 来实现忽略大小写的排序，或 者是用 `key=len` 进行基于字符串长度的排序。这个参数的默认值是恒等函数（`identity function`），也就是默认用元素自己的值来排序。
  
下面通过几个小例子来看看这两个函数和它们的关键字参数：

>这几个例子还说明了 **Python** 的排序算法——Timsort——是稳定的，意思是就算两个元素比不出大小，在每次排序的结果里它们的相对位置是固定的。T

```py
>>> fruits = ['grape', 'raspberry', 'apple', 'banana'] 
>>> sorted(fruits) 
['apple', 'banana', 'grape', 'raspberry']  ➊ 
>>> fruits 
['grape', 'raspberry', 'apple', 'banana']  ➋ 
>>> sorted(fruits, reverse=True) 
['raspberry', 'grape', 'banana', 'apple']  ➌ 
>>> sorted(fruits, key=len) 
['grape', 'apple', 'banana', 'raspberry']  ➍ 
>>> sorted(fruits, key=len, reverse=True) 
['raspberry', 'banana', 'grape', 'apple']  ➎ 
>>> fruits 
['grape', 'raspberry', 'apple', 'banana']  ➏ 
>>> fruits.sort()                          ➐ 
>>> fruits 
['apple', 'banana', 'grape', 'raspberry']  ➑ 
```

❶ 新建了一个按照字母排序的字符串列表。 

❷ 原列表并没有变化。 

❸ 按照字母降序排序。 

❹ 新建一个按照长度排序的字符串列表。因为这个排序算法是稳定 的，`grape` 和 `apple` 的长度都是 5，它们的相对位置跟在原来的列表里是一样的。 

❺ 按照长度降序排序的结果。结果并不是上面那个结果的完全翻转，因为用到的排序算法是稳定的，也就是说在长度一样时，`grape` 和 `apple` 的相对位置不会改变。 

❻ 直到这一步，原列表 `fruits` 都没有任何变化。 

❼ 对原列表就地排序，返回值 `None` 会被控制台忽略。 

❽ 此时 `fruits` 本身被排序。 

## 用bisect来管理已排序的序列 

`bisect` 模块包含两个主要函数，`bisect` 和 `insort`，两个函数都利用二分查找算法来在有序序列中查找或插入元素。 

### 用bisect来搜索

`bisect(haystack, needle)` 在 `haystack`（干草垛）里搜索 `needle`（针）的位置，该位置满足的条件是，把 `needle` 插入这个位置之后，`haystack` 还能保持升序。也就是在说这个函数返回的位置前面的值，都小于或等于 `needle` 的值。其中 `haystack` 必须是一个有序的序列。你可以先用 `bisect(haystack, needle)` 查找位置 `index`，再用 `haystack.insert(index, needle)` 来插入新值。但你也可用 `insort` 来一步到位，并且后者的速度更快一些。

示例利用几个精心挑选的 `needle`，向我们展示了 `bisect` 返回的不同位置值。

在有序序列中用 bisect 查找某个元素的插入位置:

```py
import bisect 
import sys 
HAYSTACK = [1, 4, 5, 6, 8, 12, 15, 20, 21, 23, 23, 26, 29, 30] 
NEEDLES = [0, 1, 2, 5, 8, 10, 22, 23, 29, 30, 31] 
ROW_FMT = '{0:2d} @ {1:2d}    {2}{0:<2d}' 
def demo(bisect_fn):
    for needle in reversed(NEEDLES):
        position = bisect_fn(HAYSTACK, needle)  ➊
        offset = position * '  |'  ➋
        print(ROW_FMT.format(needle, position, offset))  ➌ 
        
if __name__ == '__main__':
    if sys.argv[-1] == 'left':  ➍
        bisect_fn = bisect.bisect_left
    else:
        bisect_fn = bisect.bisect
        
    print('DEMO:', bisect_fn.__name__)  ➎
    print('haystack ->', ' '.join('%2d' % n for n in HAYSTACK))
    demo(bisect_fn) 
```

❶ 用特定的 `bisect` 函数来计算元素应该出现的位置。 

❷ 利用该位置来算出需要几个分隔符号。 

❸ 把元素和其应该出现的位置打印出来。 

❹ 根据命令上最后一个参数来选用 `bisect` 函数。 

❺ 把选定的函数在抬头打印出来。

![ZrWe4P.png](https://s2.ax1x.com/2019/07/08/ZrWe4P.png)

`bisect` 的表现可以从两个方面来调教。 

* 首先可以用它的两个可选参数——`lo` 和 `hi`——来缩小搜寻的范围。`lo` 的默认值是 `0`，`hi` 的默认值是序列的长度，即 `len()` 作用于该序列的返回值

* 其次，`bisect`函数其实是`bisect_right`函数的别名，后者还有个姊妹函数叫 `bisect_left`。它们的区别在于`bisect_left`返回的插入位置是原序列中跟被插入元素相等的元素的位置，也就是新元素会被放置于它相等的元素的前面，而 `bisect_right`返回的则是跟它相等的元素之后的位置。这个细微的差别可能对于整数序列来讲没什么用，但是 对于那些值相等但是形式不同的数据类型来讲，结果就不一样了。比如说虽然 `1 == 1.0` 的返回值是 `True`，`1` 和 `1.0` 其实是两个不同的元素。图显示的是用 `bisect_left` 来运行上述示例的结果。 

  ![ZrfSVs.png](https://s2.ax1x.com/2019/07/08/ZrfSVs.png)

`bisect`可以用来建立一个用数字作为索引的查询表格，比如说把分数和成绩对应起来

```py
>>> def grade(score, breakpoints=[60, 70, 80, 90], grades='FDCBA'):
...     i = bisect.bisect(breakpoints, score) 
...     return grades[i] 
... 
>>> [grade(score) for score in [33, 99, 77, 70, 89, 90, 100]] 
['F', 'A', 'C', 'C', 'B', 'A', 'A'] 
```

### 用bisect.insort插入新元素 

排序很耗时，因此在得到一个有序序列之后，我们最好能够保持它的有序。`bisect.insort` 就是为了这个而存在的。 

`insort(seq, item)` 把变量 `item` 插入到序列 `seq` 中，并能保持 `seq` 的升序顺序。

```py
import bisect 
import random 
SIZE=7
random.seed(1729) 
my_list = [] 
for i in range(SIZE):
    new_item = random.randrange(SIZE*2)
    bisect.insort(my_list, new_item)
    print('%2d ->' % new_item, my_list) 
```

![ZrfhWV.png](https://s2.ax1x.com/2019/07/08/ZrfhWV.png)

`insort` 跟 `bisect` 一样，有 `lo` 和 `hi` 两个可选参数用来控制查找的范 围。它也有个变体叫 `insort_left`，这个变体在背后用的是 `bisect_left`。 

## 当列表不是首选时 

虽然列表既灵活又简单，但面对各类需求时，我们可能会有更好的选 择。比如，要存放 `1000` 万个浮点数的话，数组（`array`）的效率要高 得多，因为数组在背后存的并不是 `float` 对象，而是数字的机器翻 译，也就是字节表述。这一点就跟 **C** 语言中的数组一样。再比如说，如果需要频繁对序列做先进先出的操作，`deque`（双端队列）的速度应该会更快。

### 数组

如果我们需要一个只包含数字的列表，那么`array.array`比`list`更高效。数组支持所有跟可变序列有关的操作，包括 `.pop`、`.insert` 和 `.extend`。另外，数组还提供从文件读取和存入文件的更快的方法，如 `.frombytes` 和 `.tofile`。 

**Python**数组跟**C**语言数组一样精简。创建数组需要一个类型码，这个类型码用来表示在底层的**C**语言应该存放怎样的数据类型。比如`b`类型码代表的是有符号的字符（`signed char`），因此 `array('b')`创建出的数组就只能存放一个字节大小的整数，范围从 `-128` 到 `127`，这样在序列很大的时候，我们能节省很多空间。而且 **Python**不会允许你在数组里存放除指定类型之外的数据

一个浮点型数组的创建、存入文件和从文件读取的过程 :

```py
>>> from array import array  ➊ 
>>> from random import random 
>>> floats = array('d', (random() for i in range(10**7)))  ➋ 
>>> floats[-1]  ➌ 
0.07802343889111107 
>>> fp = open('floats.bin', 'wb') 
>>> floats.tofile(fp)  ➍ 
>>> fp.close() 
>>> floats2 = array('d')  ➎ 
>>> fp = open('floats.bin', 'rb') 
>>> floats2.fromfile(fp, 10**7)  ➏ 
>>> fp.close() 
>>> floats2[-1]  ➐ 
0.07802343889111107 
>>> floats2 == floats  ➑ 
True 
```

❶ 引入 `array` 类型。 

❷ 利用一个可迭代对象来建立一个双精度浮点数组（类型码是 `'d'`）， 这里我们用的可迭代对象是一个生成器表达式。 

❸ 查看数组的最后一个元素。 

❹ 把数组存入一个二进制文件里。 

❺ 新建一个双精度浮点空数组。 

❻ 把 `1000` 万个浮点数从二进制文件里读取出来。 

❼ 查看新数组的最后一个元素。 

❽ 检查两个数组的内容是不是完全一样。 

从 **Python 3.4** 开始，数组类型不再支持诸如 `list.sort()` 这种就地排序方法。要给数组排序的话，得用 `sorted` 函数新建一个数组： 

```py
a = array.array(a.typecode, sorted(a)) 
```

### 内存视图

`memoryview` 是一个内置类，它能让用户在不复制内容的情况下操作同一个数组的不同切片。

内存视图其实是泛化和去数学化的 `NumPy` 数组。它让你在不需要复制内容的前提下，在数据结构之间共享内存。其中数据结构可以是任何形式，比如 `PIL` 图片、`SQLite` 数据库和 `NumPy` 的数组，等等。这个功能在处理大型数据集合的时候非常重要。 

`memoryview.cast` 的概念跟数组模块类似，能用不同的方式读写同一块内存数据，而且内容字节不会随意移动。这听上去又跟 **C** 语言中类型转换的概念差不多。`memoryview.cast`会把同一块内存里的内容打包成一个全新的 `memoryview`对象给你。 

通过改变数组中的一个字节来更新数组里某个元素的值:

```py
>>> numbers = array.array('h', [-2, -1, 0, 1, 2]) 
>>> memv = memoryview(numbers)  ➊ 
>>> len(memv) 
5 
>>> memv[0]  ➋ 
-2 
>>> memv_oct = memv.cast('B')  ➌ 
>>> memv_oct.tolist()  ➍ 
[254, 255, 255, 255, 0, 0, 1, 0, 2, 0] 
>>> memv_oct[5] = 4  ➎ 
>>> numbers 
array('h', [-2, -1, 1024, 1, 2])  ➏ 
```

❶ 利用含有 `5` 个短整型有符号整数的数组（类型码是 `'h'`）创建一个 `memoryview`。 

❷ `memv` 里的 5 个元素跟数组里的没有区别。 

❸ 创建一个 `memv_oct`，这一次是把 `memv` 里的内容转换成 `'B'` 类型， 也就是无符号字符。 

❹ 以列表的形式查看 `memv_oct` 的内容。 

❺ 把位于位置 `5` 的字节赋值成 `4`。 

❻ 因为我们把占 `2` 个字节的整数的高位字节改成了 `4`，所以这个有符号整数的值就变成了 `1024`。 

### 双向队列和其他形式的队列 

利用 `.append` 和 `.pop` 方法，我们可以把列表当作栈或者队列来用（比 如，把 `.append` 和 `.pop(0)` 合起来用，就能模拟栈的“先进先出”的特 点）。但是删除列表的第一个元素（抑或是在第一个元素之前添加一个元素）之类的操作是很耗时的，因为这些操作会牵扯到移动列表里的所有元素。

`collections.deque` 类（双向队列）是一个线程安全、可以快速从两端添加或者删除元素的数据类型。而且如果想要有一种数据类型来存放“最近用到的几个元素”，`deque` 也是一个很好的选择。这是因为在新建一个双向队列的时候，你可以指定这个队列的大小，如果这个队列满员了，还可以从反向端删除过期的元素，然后在尾端添加新的元素。 

```py
>>> from collections import deque 
>>> dq = deque(range(10), maxlen=10)  ➊ 
>>> dq 
deque([0, 1, 2, 3, 4, 5, 6, 7, 8, 9], maxlen=10) 
>>> dq.rotate(3)  ➋ 
>>> dq 
deque([7, 8, 9, 0, 1, 2, 3, 4, 5, 6], maxlen=10) 
>>> dq.rotate(-4) 
>>> dq 
deque([1, 2, 3, 4, 5, 6, 7, 8, 9, 0], maxlen=10) 
>>> dq.appendleft(-1)  ➌ 
>>> dq 
deque([-1, 1, 2, 3, 4, 5, 6, 7, 8, 9], maxlen=10) 
>>> dq.extend([11, 22, 33])  ➍ 
>>> dq 
deque([3, 4, 5, 6, 7, 8, 9, 11, 22, 33], maxlen=10) 
>>> dq.extendleft([10, 20, 30, 40])  ➎ 
>>> dq 
deque([40, 30, 20, 10, 3, 4, 5, 6, 7, 8], maxlen=10) 
```

❶ `maxlen` 是一个可选参数，代表这个队列可以容纳的元素的数量，而
且一旦设定，这个属性就不能修改了。 

❷ 队列的旋转操作接受一个参数 `n`，当 `n > 0` 时，队列的最右边的 `n` 个元素会被移动到队列的左边。当 `n < 0` 时，最左边的 `n` 个元素会被移动到右边。 

❸ 当试图对一个已满（`len(d) == d.maxlen`）的队列做尾部添加操作 的时候，它头部的元素会被删除掉。注意在下一行里，元素 `0` 被删除 了。 

❹ 在尾部添加 `3` 个元素的操作会挤掉 `-1、1` 和 `2`。 

❺ `extendleft(iter)` 方法会把迭代器里的元素逐个添加到双向队列的左边，因此迭代器里的元素会逆序出现在队列里。

除了 `deque` 之外，还有些其他的 **Python** 标准库也有对队列的实现。 

* queue

  提供了同步（线程安全）类 `Queue`、`LifoQueue` 和 `PriorityQueue`，不同的线程可以利用这些数据类型来交换信息。这三个类的构造方法都有一个可选参数 `maxsize`，它接收正整数作为输入 值，用来限定队列的大小。但是在满员的时候，这些类不会扔掉旧的元 素来腾出位置。相反，如果队列满了，它就会被锁住，直到另外的线程 移除了某个元素而腾出了位置。这一特性让这些类很适合用来控制活跃 线程的数量。 

* multiprocessing

  这个包实现了自己的 `Queue`，它跟 `queue.Queue` 类似，是设计给进程间通信用的。同时还有一个专门的 `multiprocessing.JoinableQueue` 类型，可以让任务管理变得更方便。 

* asyncio

  **Python3.4** 新提供的包，里面有`Queue`、`LifoQueue`、`PriorityQueue` 和 `JoinableQueue`，这些类受到 `queue` 和 `multiprocessing` 模块的影响，但是为异步编程里的任务管理提供了专门的便利。 
  
* heapq

  跟上面三个模块不同的是，`heapq` 没有队列类，而是提供了 `heappush` 和 `heappop` 方法，让用户可以把可变序列当作堆队列或者优先队列来使用。 