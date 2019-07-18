# 可迭代的对象、迭代器 和生成器

迭代是数据处理的基石。扫描内存中放不下的数据集时，我们要找到一 种惰性获取数据项的方式，即按需一次获取一个数据项。这就是迭代器模式（`Iterator pattern`）

## Sentence类第1版：单词序列

我们要实现一个 `Sentence` 类，以此打开探索可迭代对象的旅程。我们 向这个类的构造方法传入包含一些文本的字符串，然后可以逐个单词迭 代。第1版要实现序列协议，这个类的对象可以迭代.

示例 14-1 sentence.py：把句子划分为单词序列 

```py
import re
import re

RE_WORD = re.compile('\w+')

class Sentence:
    
    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text) ➊
        
    def __getitem__(self, index):
        return self.words[index]   ➋
    
    def __len__(self):   ➌
        return len(self.words)
        
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)   ➍ 
```

❶ `re.findall` 函数返回一个字符串列表，里面的元素是正则表达式的 全部非重叠匹配。 

❷ `self.words` 中保存的是 `.findall` 函数返回的结果，因此直接返回 指定索引位上的单词。 

❸ 为了完善序列协议，我们实现了 `__len__` 方法；不过，为了让对象可以迭代，没必要实现这个方法。

❹ `reprlib.repr` 这个实用函数用于生成大型数据结构的简略字符串表示形式。

示例 14-2 测试 Sentence 实例能否迭代 

```py
>> s = Sentence('"The time has come," the Walrus said,')  # ➊ 
>>> s 
Sentence('"The time ha... Walrus said,')  # ➋ 
>>> for word in s:  # ➌ 
...     print(word) 
The 
time 
has 
come 
the 
Walrus 
said 
>>> list(s)  # ➍ 
['The', 'time', 'has', 'come', 'the', 'Walrus', 'said'] 
```

❶ 传入一个字符串，创建一个 `Sentence` 实例。 

❷ 注意，`__repr__` 方法的输出中包含 `reprlib.repr` 方法生成的 `...`。 

❸ `Sentence` 实例可以迭代，稍后说明原因。 

❹ 因为可以迭代，所以 `Sentence` 对象可以用于构建列表和其他可迭代的类型。 

### 序列可以迭代的原因：iter函数

解释器需要迭代对象 `x` 时，会自动调用 `iter(x)`。 

内置的 `iter` 函数有以下作用。

1. 检查对象是否实现了 `__iter__` 方法，如果实现了就调用它，获取一个迭代器。 

2. 如果没有实现 `__iter__` 方法，但是实现了 `__getitem__` 方法， **Python** 会创建一个迭代器，尝试按顺序（从索引`0`开始）获取元素。 

3. 如果尝试失败，**Python** 抛出 `TypeError` 异常，通常会提示“`C object is not iterable`”（**C** 对象不可迭代），其中 **C** 是目标对象所属的类。

任何 **Python** 序列都可迭代的原因是，它们都实现了 `__getitem__` 方法。其实，标准的序列也都实现了 `__iter__` 方法，因此你也应该这么做。之所以对 `__getitem__` 方法做特殊处理，是为了向后兼容，而未来可能不会再这么做.

这是鸭子类型（`duck typing`）的极端形式：不仅要实现特殊的 `__iter__` 方法，还要实现 `__getitem__` 方法，而且 `__getitem__` 方法的参数是从 `0` 开始的整数（`int`），这样才认为对象是可迭代的。

在白鹅类型（`goose-typing`）理论中，可迭代对象的定义简单一些，不过 没那么灵活：如果实现了 `__iter__` 方法，那么就认为对象是可迭代的。此时，不需要创建子类，也不用注册，因为 `abc.Iterable` 类实现了 `__subclasshook__` 方法， 下面举个例子:

```py
>>> class Foo: 
...     def __iter__(self): 
...         pass 
... 
>>> from collections import abc 
>>> issubclass(Foo, abc.Iterable) 
True 
>>> f = Foo() 
>>> isinstance(f, abc.Iterable) 
True 
```

不过要注意，虽然前面定义的 `Sentence` 类是可以迭代的，但却无法通过 `issubclass` (`Sentence`, `abc.Iterable`) 测试。

> 从 Python 3.4 开始，检查对象 x 能否迭代，最准确的方法是： 调用 iter(x) 函数，如果不可迭代，再处理 TypeError 异常。这 比使用 isinstance(x, abc.Iterable) 更准确，因为 iter(x) 函数会考虑到遗留的 __getitem__ 方法，而 abc.Iterable 类则 不考虑。 

迭代对象之前显式检查对象是否可迭代或许没必要，毕竟尝试迭代不可 迭代的对象时，**Python** 抛出的异常信息很明确：`TypeError: 'C' object is not iterable`。如果除了抛出 `TypeError` 异常之外还要做进一步的处理，可以使用 `try/except` 块，而无需显式检查。如果要保存对象，等以后再迭代，或许可以显式检查，因为这种情况可能需要 尽早捕获错误。

## 可迭代的对象与迭代器的对比

* 可迭代的对象

    使用 `iter` 内置函数可以获取迭代器的对象。如果对象实现了能返回迭代器的 `__iter__` 方法，那么对象就是可迭代的。序列都可以迭代；实现了 `__getitem__` 方法，而且其参数是从零开始的索引，这种对象也可以迭代。 

我们要明确可迭代的对象和迭代器之间的关系：**Python** 从可迭代的对象中获取迭代器。 

下面是一个简单的 `for` 循环，迭代一个字符串。这里，字符串 `'ABC'` 是可迭代的对象。背后是有迭代器的，只不过我们看不到： 

```
>>> s = 'ABC' 
>>> for char in s: 
...     print(char) 
... A B C 
```

如果没有 `for` 语句，不得不使用 `while` 循环模拟，要像下面这样写： 

```py
>>> s = 'ABC' 
>>> it = iter(s)  # ➊ 
>>> while True: 
...     try: 
...         print(next(it))  # ➋ 
...     except StopIteration:  # ➌ 
...         del it  # ➍ 
...         break  # ➎ 
... 
A 
B 
C
```

❶ 使用可迭代的对象构建迭代器 `it`。 

❷ 不断在迭代器上调用 `next` 函数，获取下一个字符。

❸ 如果没有字符了，迭代器会抛出 `StopIteration` 异常。 

❹ 释放对 `it` 的引用，即废弃迭代器对象。 

❺ 退出循环。 

`StopIteration` 异常表明迭代器到头了。**Python** 语言内部会处理 `for` 循环和其他迭代上下文（如列表推导、元组拆包，等等）中的 `StopIteration` 异常。

标准的迭代器接口有两个方法。 

* __next__

    返回下一个可用的元素，如果没有元素了，抛出 `StopIteration` 异常。 

* __iter__

    返回 `self`，以便在应该使用可迭代对象的地方使用迭代器，例如在 `for` 循环中。 

这个接口在 `collections.abc.Iterator` 抽象基类中制定。这个类定义了 `__next__` 抽象方法，而且继承自 `Iterable` 类；`__iter__` 抽象 方法则在 `Iterable` 类中定义。如图 14-1 所示。

![ZXNJEt.png](https://s2.ax1x.com/2019/07/18/ZXNJEt.png)

> 图 14-1：Iterable 和 Iterator 抽象基类。以斜体显示的是抽象方 法。具体的 Iterable.__iter__ 方法应该返回一个 Iterator 实 例。具体的 Iterator 类必须实现 __next__ 方 法。Iterator.__iter__ 方法直接返回实例本身 

`Iterator` 抽象基类实现 `__iter__` 方法的方式是返回实例本身 （`return self`）。这样，在需要可迭代对象的地方可以使用迭代器。

> 在 Python 3 中，Iterator 抽象基类定义的抽象方法是 it.__next__()，而在 Python 2 中是 it.next()。一如既往，我 们应该避免直接调用特殊方法，使用 next(it) 即可，这个内置的 函数在 Python 2 和 Python 3 中都能使用。 

* 迭代器

    迭代器是这样的对象：实现了无参数的 `__next__` 方法，返回序列中的下一个元素；如果没有元素了，那么抛出 `StopIteration` 异常。 **Python** 中的迭代器还实现了 `__iter__` 方法，因此迭代器也可以迭代。

## Sentence类第2版：典型的迭代器

第 `2` 版 `Sentence` 类根据《设计模式：可复用面向对象软件的基础》一 书给出的模型，实现典型的迭代器设计模式。注意，这不符合 **Python** 的 习惯做法，后面重构时会说明原因。不过，通过这一版能明确可迭代的集合和迭代器对象之间的关系。 

示例 14-4 中定义的 `Sentence` 类可以迭代，因为它实现了特殊的 `__iter__` 方法，构建并返回一个 `SentenceIterator` 实例。《设计模 式：可复用面向对象软件的基础》一书就是这样描述迭代器设计模式的。 

这里之所以这么做，是为了清楚地说明可迭代的对象和迭代器之间的重要区别，以及二者之间的联系。

示例 14-4 `sentence_iter.py`：使用迭代器模式实现 `Sentence` 类 

```py
import re 
import reprlib 

RE_WORD = re.compile('\w+') 

class Sentence:

    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)
        
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)
        
    def __iter__(self):  ➊
        return SentenceIterator(self.words)  ➋ 
        
        
class SentenceIterator:
    def __init__(self, words):
        self.words = words  ➌
        self.index = 0  ➍
        
    def __next__(self):
        try:
            word = self.words[self.index]  ➎
        except IndexError:
            raise StopIteration()  ➏
        self.index += 1  ➐
        return word  ➑
        
    def __iter__(self):  ➒
        return self 
```

❶ 与前一版相比，这里只多了一个 `__iter__` 方法。这一版没有 `__getitem__` 方法，为的是明确表明这个类可以迭代，因为实现了 `__iter__` 方法。 

❷ 根据可迭代协议，`__iter__` 方法实例化并返回一个迭代器。 

❸ `SentenceIterator` 实例引用单词列表。 

❹ `self.index` 用于确定下一个要获取的单词。 

❺ 获取 `self.index` 索引位上的单词。 

❻ 如果 `self.index` 索引位上没有单词，那么抛出 `StopIteration` 异 常。 

❼ 递增 `self.index` 的值。 

❽ 返回单词。 

❾ 实现 `self.__iter__` 方法。 

示例 14-4 中的代码能通过示例 14-2 中的测试。

注意，对这个示例来说，其实没必要在 `SentenceIterator` 类中实现 `__iter__` 方法，不过这么做是对的，因为迭代器应该实现 `__next__` 和 `__iter__` 两个方法，而且这么做能让迭代器通过`issubclass(SentenceInterator, abc.Iterator)` 测试。如果让 `SentenceIterator` 类继承 `abc.Iterator` 类，那么它会继承 `abc.Iterator.__iter__` 这个具体方法。 

### 把Sentence变成迭代器：坏主意 

构建可迭代的对象和迭代器时经常会出现错误，原因是混淆了二者。要 知道，可迭代的对象有个 `__iter__` 方法，每次都实例化一个新的迭代 器；而迭代器要实现 `__next__` 方法，返回单个元素，此外还要实现 `__iter__` 方法，返回迭代器本身。 

因此，迭代器可以迭代，但是可迭代的对象不是迭代器。

除了 `__iter__` 方法之外，你可能还想在 `Sentence` 类中实现 `__next__` 方法，让 `Sentence` 实例既是可迭代的对象，也是自身的迭 代器。可是，这种想法非常糟糕。根据有大量 **Python** 代码审查经验的 `Alex Martelli` 所说，这也是常见的反模式。 

《设计模式：可复用面向对象软件的基础》一书讲解迭代器设计模式 时，在“适用性”一节中说

迭代器模式可用来： 

* 访问一个聚合对象的内容而无需暴露它的内部表示 

* 支持对聚合对象的多种遍历 

* 为遍历不同的聚合结构提供一个统一的接口（即支持多态迭代） 

为了“支持多种遍历”，必须能从同一个可迭代的实例中获取多个独立的 迭代器，而且各个迭代器要能维护自身的内部状态，因此这一模式正确 的实现方式是，每次调用 `iter(my_iterable)` 都新建一个独立的迭代器。这就是为什么这个示例需要定义 `SentenceIterator` 类。 

可迭代的对象一定不能是自身的迭代器。也就是说，可迭代的对象 必须实现 `__iter__` 方法，但不能实现 `__next__` 方法。 

## Sentence类第3版：生成器函数 

实现相同功能，但却符合 **Python** 习惯的方式是，用生成器函数代替 `SentenceIterator` 类。先看示例 14-5，然后详细说明生成器函数

示例 14-5 `sentence_gen.py`：使用生成器函数实现 `Sentence` 类 

```py
import re
import reprlib

RE_WORD = re.compile(`\w+`)

class Sentence:
    
    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)
        
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self)
        
    def __iter__(self):
        for word in self.words:  ➊
            yield word ➋
        return  ➌
```

❶ 迭代 `self.words`。 

❷ 产出当前的 `word`。 

❸ 这个 `return` 语句不是必要的；这个函数可以直接“落空”，自动返回。不管有没有 `return` 语句，生成器函数都不会抛出 `StopIteration` 异常，而是在生成完全部值之后会直接退出。

我们又使用一种不同的方式实现了 `Sentence` 类，而且也能通过示例 14-2 中的测试。 

在示例 14-4 定义的 `Sentence` 类中，`__iter__` 方法调用 `SentenceIterator` 类的构造方法创建一个迭代器并将其返回。而在示 例 14-5 中，迭代器其实是生成器对象，每次调用 `__iter__` 方法都会自动创建，因为这里的 `__iter__` 方法是生成器函数。 

### 生成器函数的工作原理 

只要 **Python** 函数的定义体中有 `yield` 关键字，该函数就是生成器函数。调用生成器函数时，会返回一个生成器对象。也就是说，生成器函数是生成器工厂。

下面以一个特别简单的函数说明生成器的行为： 

```py
>>> def gen_123():  # ➊ 
...     yield 1  # ➋ 
...     yield 2 
...     yield 3 
...
>>> gen_123  # doctest: +ELLIPSIS 
<function gen_123 at 0x...>  # ➌ 
>>> gen_123()   # doctest: +ELLIPSIS 
<generator object gen_123 at 0x...>  # ➍ 
>>> for i in gen_123():  # ➎ 
...     print(i) 
1 
2 
3 
>>> g = gen_123()  # ➏ 
>>> next(g)  # ➐ 
1 
>>> next(g) 
2 
>>> next(g) 
3 
>>> next(g)  # ➑ 
Traceback (most recent call last):
    ... 
StopIteration 
```

❶ 只要 **Python** 函数中包含关键字 `yield`，该函数就是生成器函数。 

❷ 生成器函数的定义体中通常都有循环，不过这不是必要条件；这里我重复使用 `3` 次 `yield`。

❸ 仔细看，`gen_123` 是函数对象。 

❹ 但是调用时，`gen_123()` 返回一个生成器对象。 

❺ 生成器是迭代器，会生成传给 `yield` 关键字的表达式的值。 

❻ 为了仔细检查，我们把生成器对象赋值给 `g`。 

❼ 因为 `g` 是迭代器，所以调用 `next(g)` 会获取 `yield` 生成的下一个元 素。 

❽ 生成器函数的定义体执行完毕后，生成器对象会抛出 `StopIteration` 异常。

生成器函数会创建一个生成器对象，包装生成器函数的定义体。把生成 器传给 next(...) 函数时，生成器函数会向前，执行函数定义体中的 下一个 `yield` 语句，返回产出的值，并在函数定义体的当前位置暂 停。最终，函数的定义体返回时，外层的生成器对象会抛出 `StopIteration` 异常——这一点与迭代器协议一致。

示例 14-6 使用 `for` 循环更清楚地说明了生成器函数定义体的执行过 程。 

示例 14-6 运行时打印消息的生成器函数

```py
>>> def gen_AB():  # ➊ 
...     print('start') 
...     yield 'A'       # ➋ 
...     print('continue') 
...     yield 'B'       # ➌ 
...     print('end.')   # ➍ 
... 
>>> for c in gen_AB():  # ➎ 
...     print('-->', c)  # ➏ 
... 
start  ➐ 
--> A  ➑ 
continue ➒ 
--> B  ➓ 
end.   ⓫ 
>>> ⓬ 
```

❶ 定义生成器函数的方式与普通的函数无异，只不过要使用 `yield` 关键字。

❷ 在 `for` 循环中第一次隐式调用 `next()` 函数时（序号➎），会打印 '`start`'，然后停在第一个 `yield` 语句，生成值 '`A`'。 

❸ 在 `for` 循环中第二次隐式调用 `next()` 函数时，会打印 '`continue`'，然后停在第二个 `yield` 语句，生成值 '`B`'。 

❹ 第三次调用 `next()` 函数时，会打印 '`end.`'，然后到达函数定义体 的末尾，导致生成器对象抛出 `StopIteration` 异常。 

❺ 迭代时，`for` 机制的作用与 `g = iter(gen_AB())` 一样，用于获取 生成器对象，然后每次迭代时调用 `next(g)`。 

❻ 循环块打印 `-->` 和 `next(g)` 返回的值。但是，生成器函数中的 `print` 函数输出结果之后才会看到这个输出。 

❼ '`start`' 是生成器函数定义体中 `print('start')` 输出的结果。 

❽ 生成器函数定义体中的 `yield 'A'` 语句会生成值 `A`，提供给 `for` 循 环使用，而 `A` 会赋值给变量 `c`，最终输出 `--> A`。 

❾ 第二次调用 `next(g)`，继续迭代，生成器函数定义体中的代码由 `yield 'A'` 前进到 `yield 'B'`。文本 `continue` 是由生成器函数定义 体中的第二个 `print` 函数输出的。 

❿ `yield 'B'` 语句生成值 `B`，提供给 `for` 循环使用，而 `B` 会赋值给变量 `c`，所以循环打印出 `--> B`。 

⓫ 第三次调用 `next(it)`，继续迭代，前进到生成器函数的末尾。文本 `end`. 是由生成器函数定义体中的第三个 `print` 函数输出的。到达生成 器函数定义体的末尾时，生成器对象抛出 `StopIteration` 异常。`for` 机制会捕获异常，因此循环终止时没有报错。 

⓬ 现在，希望你已经知道示例 14-5 中 `Sentence.__iter__` 方法的作 用了：`__iter__` 方法是生成器函数，调用时会构建一个实现了迭代器 接口的生成器对象，因此不用再定义 `SentenceIterator` 类了。

## Sentence类第4版：惰性实现 

设计 `Iterator` 接口时考虑到了惰性：`next(my_iterator)` 一次生成一个元素。懒惰的反义词是急迫，其实，惰性求值（`lazy evaluation`）和 及早求值（`eager evaluation`）是编程语言理论方面的技术术语。 

目前实现的几版 `Sentence` 类都不具有惰性，因为 `__init__` 方法急迫地构建好了文本中的单词列表，然后将其绑定到 `self.words` 属性上。 这样就得处理整个文本，列表使用的内存量可能与文本本身一样多（或许更多，这取决于文本中有多少非单词字符）。如果只需迭代前几个单词，大多数工作都是白费力气。 

只要使用的是 **Python 3**，思索着做某件事有没有懒惰的方式，答案通常 都是肯定的。 

`re.finditer` 函数是 `re.findall` 函数的惰性版本，返回的不是列 表，而是一个生成器，按需生成 `re.MatchObject` 实例。如果有很多匹配，`re.finditer` 函数能节省大量内存。我们要使用这个函数让第 4 版 `Sentence` 类变得懒惰，即只在需要时才生成下一个单词。代码如示 例 14-7 所示。 

示例 14-7 `sentence_gen2.py`： 在生成器函数中调用 `re.finditer` 生成器函数，实现 `Sentence` 类 

```py
import re
import reprlib

RE_WORD =  re.cpmpile(`\w+`)

class Sentence:
    
    def __init__(self, text):
        self.text = text ➊
        
    def __repr__(slef):
        return 'Sentence(%s)' % reprlib.repr(self.text)
        
    def __iter__(slef):
        for match in RE_WORD.finditer(self.text):  ➋
            yield match.group()  ➌ 
```

❶ 不再需要 `words` 列表。 

❷ `finditer` 函数构建一个迭代器，包含 `self.text` 中匹配 `RE_WORD` 的单词，产出 `MatchObject` 实例。 

❸ `match.group()` 方法从 MatchObject 实例中提取匹配正则表达式的具体文本。 

## Sentence类第5版：生成器表达式 




