# 正确重载运算符

## 运算符重载基础

在某些圈子中，运算符重载的名声并不好。这个语言特性可能（已经） 被滥用，让程序员困惑，导致缺陷和意料之外的性能瓶颈。但是，如果 使用得当，`API` 会变得好用，代码会变得易于阅读。**Python** 施加了一些限制，做好了灵活性、可用性和安全性方面的平衡：

* 不能重载内置类型的运算符 

* 不能新建运算符，只能重载现有的

* 某些运算符不能重载——is、and、or 和 not（不过位运算符 &、| 和 ~ 可以） 

## 一元运算符

* `-（__neg__)`

    一元取负算术运算符。如果 x 是 -2，那么 -x == 2。 

* `+（__pos__）`

    一元取正算术运算符。通常，x == +x，但也有一些例外。
    
* `~（__invert__）`

    对整数按位取反，定义为 ~x == -(x+1)。如果 x 是 2，那么 ~x == -3。 
    
支持一元运算符很简单，只需实现相应的特殊方法。这些特殊方法只有 一个参数，`self`。然后，使用符合所在类的逻辑实现。不过，要遵守运 算符的一个基本规则：始终返回一个新对象。也就是说，不能修改 `self`，要创建并返回合适类型的新实例。  
