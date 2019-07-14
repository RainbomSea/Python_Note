# 序列的修改、散列和切片

## 协议和鸭子类型

在 **Python** 中创建功能完善的序列类型无需使用继 承，只需实现符合序列协议的方法。

在面向对象编程中，协议是非正式的接口，只在文档中定义，在代码中 不定义。例如，**Python** 的序列协议只需要 `__len__` 和 `__getitem__` 两 个方法。任何类（如 `Spam`），只要使用标准的签名和语义实现了这两个方法，就能用在任何期待序列的地方。`Spam` 是不是哪个类的子类无关紧要，只要提供了所需的方法即可。

示例 10-3

```py
import collections 

Card = collections.namedtuple('Card', ['rank', 'suit']) 

class FrenchDeck:
    ranks = [str(n) for n in range(2, 11)] + list('JQKA')
    suits = 'spades diamonds clubs hearts'.split()
    
    def __init__(self):
        self._cards = [Card(rank, suit) for suit in self.suits
                                        for rank in self.ranks]
                                        
    def __len__(self):
        return len(self._cards)
        
    def __getitem__(self, position):
        return self._cards[position] 
```

示例 10-3 中的 `FrenchDeck` 类能充分利用 `Python` 的很多功能，因为它 实现了序列协议，不过代码中并没有声明这一点。任何有经验的 `Python` 程序员只要看一眼就知道它是序列，即便它是 `object` 的子类也无妨。 我们说它是序列，因为它的行为像序列，这才是重点。

人们称其为鸭子类型

## 切片原理

示例 10-4 了解 ｀__getitem__｀ 和切片的行为 

```py
>>> class MySeq: 
...     def __getitem__(self, index): 
...         return index  # ➊ 
... 
>>> s = MySeq() 
>>> s[1]  # ➋ 
1 
>>> s[1:4]  # ➌ 
slice(1, 4, None) 
>>> s[1:4:2]  # ➍ 
slice(1, 4, 2) 
>>> s[1:4:2, 9]  # ➎ 
(slice(1, 4, 2), 9) 
>>> s[1:4:2, 7:9]  # ➏ 
(slice(1, 4, 2), slice(7, 9, None)) 
```

❶ 在这个示例中，`__getitem__` 直接返回传给它的值。 

❷ 单个索引，没什么新奇的。 

❸ `1:4` 表示法变成了 `slice(1, 4, None)`。 

❹ `slice(1, 4, 2)` 的意思是从 `1` 开始，到 `4` 结束，步幅为 `2`。 

❺ 神奇的事发生了：如果 `[]` 中有逗号，那么 `__getitem__` 收到的是元组。 

❻ 元组中甚至可以有多个切片对象。

现在，我们来仔细看看 `slice` 本身，如示例 10-5 所示。 

示例 10-5 查看 slice 类的属性

```py
>>> slice  # ➊ 
<class 'slice'> 
>>> dir(slice)  # ➋ 
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__',
 '__format__', '__ge__', '__getattribute__', '__gt__',
 '__hash__', '__init__', '__le__', '__lt__', '__ne__',
 '__new__', '__reduce__', '__reduce_ex__', '__repr__',
 '__setattr__', '__sizeof__', '__str__', '__subclasshook__',
 'indices', 'start', 'step', 'stop'] 
```

❶ `slice` 是内置的类型

❷ 通过审查 `slice`，发现它有 `start`、`stop` 和 `step` 数据属性，以及 `indices` 方法。

在示例 10-5 中，调用`dir(slice)`得到的结果中有个`indices`属性，这个方法有很大的作用，但是鲜为人知。`help(slice.indices)` 给出的信息如下。 

```py
S.indices(len) -> (start, stop, stride)
```

    给定长度为 `len` 的序列，计算 `S` 表示的扩展切片的起始（`start`） 和结尾（`stop`）索引，以及步幅（`stride`）。超出边界的索引会被截掉，这与常规切片的处理方式一样。 

换句话说，`indices` 方法开放了内置序列实现的棘手逻辑，用于优雅地 处理缺失索引和负数索引，以及长度超过目标序列的切片。这个方法 会“整顿”元组，把 `start`、`stop` 和 `stride` 都变成非负数，而且都落在指定长度序列的边界内。 

下面举几个例子。假设有个长度为 `5` 的序列，例如 `'ABCDE'`： 

```py
>>> slice(None, 10, 2).indices(5)  # ➊ 
(0, 5, 2) 
>>> slice(-3, None, None).indices(5)  # ➋ 
(2, 5, 1)
```

❶ `'ABCDE'[:10:2]` 等同于 `'ABCDE'[0:5:2]`

❷ `'ABCDE'[-3:]` 等同于 `'ABCDE'[2:5:1]`
