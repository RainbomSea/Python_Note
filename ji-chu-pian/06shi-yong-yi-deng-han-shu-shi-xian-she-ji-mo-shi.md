# 使用一等函数实现设计模式

## 案例分析: 重构“策略”模式

如果合理利用作为一等对象的函数，某些设计模式可以简化，“策略”模 式就是其中一个很好的例子。

### 经典的“策略”模式

图 6-1 中的 `UML` 类图指出了“策略”模式对类的编排。

![ZWIYAx.png](https://s2.ax1x.com/2019/07/12/ZWIYAx.png)

> 图 6-1：使用“策略”设计模式处理订单折扣的 `UML` 类图

《设计模式：可复用面向对象软件的基础》一书是这样概述“策略”模式 的： 

    定义一系列算法，把它们一一封装起来，并且使它们可以相互替 换。本模式使得算法可以独立于使用它的客户而变化。 

电商领域有个功能明显可以使用“策略”模式，即根据客户的属性或订单 中的商品计算折扣。 

假如一个网店制定了下述折扣规则。 

* 有 1000 或以上积分的顾客，每个订单享 `5%` 折扣。 

* 同一订单中，单个商品的数量达到 `20` 个或以上，享 `10%` 折扣。 

* 订单中的不同商品达到 `10` 个或以上，享 `7%` 折扣。 

简单起见，我们假定一个订单一次只能享用一个折扣。

* 上下文

    把一些计算委托给实现不同算法的可互换组件，它提供服务。在这 个电商示例中，上下文是 Order，它会根据不同的算法计算促销折扣。 策略

* 策略

    实现不同算法的组件共同的接口。在这个示例中，名为 Promotion 的抽象类扮演这个角色。

* 具体策略

    “策略”的具体子类。`fidelityPromo`、`BulkPromo` 和 `LargeOrderPromo` 是这里实现的三个具体策略。
    
示例 6-1 实现 Order 类，支持插入式折扣策略 

```py
from abc import ABC, abstractmethod
from collections import namedtuple

Customer = namedutuple('Customer', 'name fidelity')


class LineItem:

    def __init__(self, product, quantity, price):
        self.product = product
        self.quantity = quantity
        self.price = price
        
    def total(self):
        return self.price * self.quantity


class Order:  # 上下文
    
    def __init__(self, customer, cart, promotion=None):
        self.customer = customer
        self.cart = list(cart)
        self.promotion = promotion
    
    def total(self):
        if not hasattr(self, '__total'):
            self.__total = sum(item.total() for item in self.cart)
        return self.__total
    
    def due(self):
        if self.promotion is None:
            discount = 0
        else:
            discount = self.promotion.discount(self)
        return self.total() - discount
    
    def __repr__(self):
        fmt = '<Order total: {:.2f} due: {:.2f}>'
        return fmt.format(self.total(), self.due())

    
class Promotion(ABC)： # 策略：抽象基类
    
    @abstractmethod
    def discount(self, order):
        """返回折扣金额（正值）"""

      
class FidelityPromo(Promotion):  # 第一个具体策略
    """为积分为1000或以上的顾客提供5%折扣"""
    def discount(self, order):
        return order.total() * .05 if order.customer.fidelity >= 1000 else 0 

               
class BulkItemPromo(Promotion):  # 第二个具体策略
    """单个商品为20个或以上时提供10%折扣"""
    def discount(self, order):
        discount = 0
        for item in order.cart:
            if item.quantity >= 20:
                discount += item.total() * .1
        return discount


class LargeOrderPromo(Promotion):  # 第三个具体策略
    """订单中的不同商品达到10个或以上时提供7%折扣"""
    def discount(self, order):
        distinct_items = {item.product for item in order.cart}
        if len(distinct_items) >= 10:
            return order.total() * .07
        return 0 
```

> 注意，在示例 6-1 中，我把 Promotion 定义为抽象基类（Abstract Base Class，ABC），这么做是为了使用 @abstractmethod 装饰器，从而明 确表明所用的模式。

示例 6-2 是一些 `doctest`，在某个实现了上述规则的模块中演示和验证相关操作。

示例 6-2 使用不同促销折扣的 Order 类示例

```py
>>> joe = Customer('John Doe', 0) ➊
>>> ann = Customer('Ann Smith', 1100)
>>> cart = [LineItem('banana', 4, .5), ➋
...         LineItem('apple', 10, 1.5),
...         LineItem('watermellon', 5, 5.0)]
>>> Order(joe, cart, FidelityPromo()) ➌
<Order total: 42.00 due: 42.00>
>>> Order(ann, cart, FidelityPromo()) ➍
<Order total: 42.00 due: 39.90>
>>> banana_cart = [LineItem('banana', 30, .5), ➎
...                LineItem('apple', 10, 1.5)]
>>> Order(joe, banana_cart, BulkItemPromo()) ➏
<Order total: 30.00 due: 28.50>
>>> long_order = [LineItem(str(item_code), 1, 1.0) ➐
...               for item_code in range(10)]
>>> Order(joe, long_order, LargeOrderPromo()) ➑
<Order total: 10.00 due: 9.30>
>>> Order(joe, cart, LargeOrderPromo())
<Order total: 42.00 due: 42.00> 
```

❶ 两个顾客：`joe` 的积分是 `0`，`ann` 的积分是 `1100`。

❷ 有三个商品的购物车。

❸ `fidelityPromo` 没给 `joe` 提供折扣。

❹ `ann` 得到了 `5%` 折扣，因为她的积分超过 `1000`。

❺ `banana_cart` 中有 `30` 把香蕉和 `10` 个苹果。

❻ `BulkItemPromo` 为 `joe` 购买的香蕉优惠了 `1.50` 美元。

❼ `long_order` 中有 `10` 个不同的商品，每个商品的价格为 `1.00` 美元。

❽ `LargerOrderProm`o 为 `joe` 的整个订单提供了 `7%` 折扣。


### 使用函数实现“策略”模式 

在示例 6-1 中，每个具体策略都是一个类，而且都只定义了一个方法， 即 `discount`。此外，策略实例没有状态（没有实例属性）。你可能会说，它们看起来像是普通的函数——的确如此。示例 6-3 是对示例 6-1 的重构，把具体策略换成了简单的函数，而且去掉了 `Promo` 抽象类。

示例 6-3 Order 类和使用函数实现的折扣策略 

```py
from collections import 
Customer = namedtuple('Customer', 'name fidelity')


class LineItem:
    
    def __init__(self, product, quantity, price):
        self.product = product
        self.quantity = quantity
        self.price = price
        
    def total(self):
        return self.price * self.quantity


class Order:  # 上下文

    def __init__(self, customer, cart, promotion=None):
        self.customer = customer
        self.cart = list(cart)
        self.promotion = promotion
    
    def total(self):
        if not hasattr(self, '__total'):
            self.__total = sum(item.total() for item in self.cart)
        return self.__total
        
    def due(self):
        if self.promotion is None:
            discount = 0
        else:
            discount = self.promotion(self)  ➊
        return self.total() - discount
        
    def __repr__(self):
        fmt = '<Order total: {:.2f} due: {:.2f}>'
        return fmt.format(self.total(), self.due())


➋ 

def fidelity_promo(order):  ➌
    """为积分为1000或以上的顾客提供5%折扣"""
    return order.total() * .05 if order.customer.fidelity >= 1000 else 0 
    
def bulk_item_promo(order):
    """单个商品为20个或以上时提供10%折扣"""
    discount = 0
    for item in order.cart:
        if item.quantity >= 20:
            discount += item.total() * .1
    return discount 
    
def large_order_promo(order):
    """订单中的不同商品达到10个或以上时提供7%折扣"""
    distinct_items = {item.product for item in order.cart}
    if len(distinct_items) >= 10:
        return order.total() * .07
    return 0 
```

❶ 计算折扣只需调用 self.promotion() 函数。

❷ 没有抽象类。

❸ 各个策略都是函数。

示例 6-3 中的代码比示例 6-1 少 `12` 行。不仅如此，新的 `Order` 类使用起来更简单，如示例 6-4 中的 `doctest` 所示。

示例 6-4 使用函数实现的促销折扣的 `Order` 类示例

```py
>>> joe = Customer('John Doe', 0)  ➊
>>> ann = Customer('Ann Smith', 1100)
>>> cart = [LineItem('banana', 4, .5),
...         LineItem('apple', 10, 1.5),
...         LineItem('watermellon', 5, 5.0)]
>>> Order(joe, cart, fidelity_promo)  ➋
<Order total: 42.00 due: 42.00>
>>> Order(ann, cart, fidelity_promo)
<Order total: 42.00 due: 39.90>
>>> banana_cart = [LineItem('banana', 30, .5),
...                LineItem('apple', 10, 1.5)]
>>> Order(joe, banana_cart, bulk_item_promo)  ➌
<Order total: 30.00 due: 28.50>
>>> long_order = [LineItem(str(item_code), 1, 1.0)
...               for item_code in range(10)]
>>> Order(joe, long_order, large_order_promo)
<Order total: 10.00 due: 9.30>
>>> Order(joe, cart, large_order_promo)
<Order total: 42.00 due: 42.00> 
``` 

❶ 与示例 6-1 一样的测试固件。 

❷ 为了把折扣策略应用到 `Order` 实例上，只需把促销函数作为参数传入。

❸ 这个测试和下一个测试使用不同的促销函数。