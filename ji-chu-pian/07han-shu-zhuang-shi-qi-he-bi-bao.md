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