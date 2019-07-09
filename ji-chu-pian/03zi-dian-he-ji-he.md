# 字典和集合

## 泛映射类型

`collections.abc` 模块中有 `Mapping` 和 `MutableMapping` 这两个抽象 基类，它们的作用是为 `dict` 和其他类似的类型定义形式接口（在 **Python2.6** 到**Python3.2**的版本中，这些类还不属于 `collections.abc` 模块，而是隶属于 `collections` 模块）。见图 03-1。

![ZyQnZn.png](https://s2.ax1x.com/2019/07/09/ZyQnZn.png)

> 图03-1：`collections.abc`中的`MutableMapping`和它的超类的`UML`类图（箭头从子类指向超类，抽象类和抽象方法的名称以斜体显示）

然而，非抽象映射类型一般不会直接继承这些抽象基类，它们会直接对 `dict` 或是 `collections.User.Dict` 进行扩展。这些抽象基类的主要作用是作为形式化的文档，它们定义了构建一个映射类型所需要的最基本的接口。然后它们还可以跟 `isinstance` 一起被用来判定某个数据是不是广义上的映射类型：

```py
>>> my_dict = {} 
>>> isinstance(my_dict, abc.Mapping) 
True
```

这里用 `isinstance` 而不是 `type` 来检查某个参数是否为 `dict` 类型， 因为这个参数有可能不是 `dict`，而是一个比较另类的映射类型。

标准库里的所有映射类型都是利用 `dict` 来实现的，因此它们有个共同的限制，即只有可散列的数据类型才能用作这些映射里的键（只有键有这个要求，值并不需要是可散列的数据类型）。

**什么是可散列的数据类型**?

* 如果一个对象是可散列的，那么在这个对象的生命周期中，它的散列值是不变的，而且这个对象需要实现 `__hash__()` 方法。另外可散列对象还要有 `__qe__()` 方法，这样才能跟其他键做比较。如果两个可散列对象是相等的，那么它们的散列值一定是一样的

* 原子不可变数据类型（`str`、`bytes` 和数值类型）都是可散列类 型，`frozenset` 也是可散列的，因为根据其定义，`frozenset` 里只能容纳可散列类型。元组的话，只有当一个元组包含的所有元素都是可散列类型的情况下，它才是可散列的。来看下面的元组 `tt`、`tl` 和 `tf`：

  ```py
   >>> tt = (1, 2, (30, 40)) 
   >>> hash(tt) 
   8027212646858338501 
   >>> tl = (1, 2, [30, 40]) 
   >>> hash(tl) 
   Traceback (most recent call last):
      File "<stdin>", line 1, in <module> 
   TypeError: unhashable type: 'list' 
   >>> tf = (1, 2, frozenset([30, 40])) 
   >>> hash(tf) 
   -4118419923444501110
  ```

* 一般来讲用户自定义的类型的对象都是可散列的，散列值就是它们的 `id()` 函数的返回值，所以所有这些对象在比较的时候都是不相等的。如果一个对象实现了 `__eq__` 方法，并且在方法中用到了这个对象的内部状态的话，那么只有当所有这些内部状态都是不可变的情况下，这个对象才是可散列的。

创建字典的不同方式：

```py
>>> a = dict(one=1, two=2, three=3) 
>>> b = {'one': 1, 'two': 2, 'three': 3} 
>>> c = dict(zip(['one', 'two', 'three'], [1, 2, 3])) 
>>> d = dict([('two', 2), ('one', 1), ('three', 3)]) 
>>> e = dict({'three': 3, 'one': 1, 'two': 2}) 
>>> a == b == c == d == e 
True
```

## 字典推导

字典推导的应用：

```py
>>> DIAL_CODES = [                ➊ 
...         (86, 'China'), 
...         (91, 'India'), 
...         (1, 'United States'), 
...         (62, 'Indonesia'), 
...         (55, 'Brazil'), 
...         (92, 'Pakistan'), 
...         (880, 'Bangladesh'), 
...         (234, 'Nigeria'), 
...         (7, 'Russia'), 
...         (81, 'Japan'), 
...     ] 
>>> country_code = {country: code for code, country in DIAL_CODES}  ➋
>>> country_code 
{'China': 86, 'India': 91, 'Bangladesh': 880, 'United States': 1, 
'Pakistan': 92, 'Japan': 81, 'Russia': 7, 'Brazil': 55, 'Nigeria': 
234, 'Indonesia': 62} 
>>> {code: country.upper() for country, code in country_code.items()  ➌ 
...   if code < 66} 
{1: 'UNITED STATES', 55: 'BRAZIL', 62: 'INDONESIA', 7: 'RUSSIA'}
```

❶ 一个承载成对数据的列表，它可以直接用在字典的构造方法中。 

❷ 这里把配好对的数据左右换了下，国家名是键，区域码是值。 

❸ 跟上面相反，用区域码作为键，国家名称转换为大写，并且过滤掉区域码大于或等于`66`的地区。 



