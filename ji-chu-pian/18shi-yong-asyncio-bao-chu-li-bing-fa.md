# 使用 asyncio 包处理并发

## 线程与协程对比

示例 18-1 `spinner_thread.py`：通过线程以动画形式显示文本式旋 转指针

```py
import threading
import itertools
import time
import sys


class Signal:  ➊
    go = True


def spin(msg, signal):  ➋
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\')::  ➌
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))  ➍
        time.sleep(.1)
        if not signal.go:  ➎
            break

    write(' ' * len(status) + '\x08' * len(status))  ➏


def slow_function()::  ➐
    # 假装等待I/O一段时间
    time.sleep(3)  ➑
    return 42


def supervisor():  ➒
    signal = Signal()
    spinner = threading.Thread(target=spin, args=('thinking!', signal))
    print('spinner object:', spinner)  ➓
    spinner.start()  ⓫
    result = slow_function()  ⓬
    signal.go = False  ⓭
    spinner.join()  ⓮

    return result


def main():
    result = supervisor()  ⓯
    print('Answer:', result)


if __name__ == '__main__':
    main()

```

❶ 这个类定义一个简单的可变对象；其中有个 `go` 属性，用于从外部控 制线程。

❷ 这个函数会在单独的线程中运行。`signal` 参数是前面定义的 `Signal` 类的实例。

❸ 这其实是个无限循环，因为 `itertools.cycle` 函数会从指定的序列中反复不断地生成元素。

❹ 这是显示文本式动画的诀窍所在：使用退格符（`\x08`）把光标移回 来。

❺ 如果 `go` 属性的值不是 `True` 了，那就退出循环。

❻ 使用空格清除状态消息，把光标移回开头。

❼ 假设这是耗时的计算。

❽ 调用 `sleep` 函数会阻塞主线程，不过一定要这么做，以便释放 `GIL`，创建从属线程。

❾ 这个函数设置从属线程，显示线程对象，运行耗时的计算，最后杀 死线程。

❿ 显示从属线程对象。输出类似于 `<Thread(Thread-1, initial)>`。

⓫ 启动从属线程。

⓬ 运行 `slow_function` 函数，阻塞主线程。同时，从属线程以动画形式显示旋转指针。

⓭ 改变 `signal` 的状态；这会终止 `spin` 函数中的那个 `for` 循环。

⓮ 等待 `spinner` 线程结束。

⓯ 运行 `supervisor` 函数。

注意，**Python** 没有提供终止线程的 API，这是有意为之的。若想关闭线程，必须给线程发送消息。这里，这里使用的是 `signal.go` 属性：在主线程中把它设为 `False` 后，`spinner` 线程最终会注意到，然后干净地退出。

示例 18-2 `spinner_asyncio.py`：通过协程以动画形式显示文本式旋转指针

```py
import asyncio
import itertools
import sys


@asyncio.coroutine  ➊
def spin(msg):  ➋
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))
        try:
            yield from asyncio.sleep(.1)  ➌
        except asyncio.CancelledError:  ➍
            break

    write(' ' * len(status) + '\x08' * len(status))


@asyncio.coroutine
def slow_function():  ➎
    # 假装等待I/O一段时间
    yield from asyncio.sleep(3)  ➏
    return 42


@asyncio.coroutine
def supervisor():  ➐
    spinner = asyncio.async(spin('thinking!'))  ➑
    print('spinner object:', spinner) ➒
    result = yield from slow_function() ➓
    spinner.cancel()  ⓫

    return result


def main():
    loop = asyncio.get_event_loop() ⓬
    result = loop.run_until_complete(supervisor())  ⓭
    loop.close()
    print('Answer:', result)


if __name__ == '__main__':
    main()

```

❶ 打算交给 `asyncio` 处理的协程要使用 `@asyncio.coroutine` 装饰。 这不是强制要求，但是强烈建议这么做。原因在本列表后面。

❷ 这里不需要示例 18-1 中 `spin` 函数中用来关闭线程的 `signal` 参数。

❸ 使用 `yield from asyncio.sleep(.1)` 代替 `time.sleep(.1)`，这 样的休眠不会阻塞事件循环。

❹ 如果 `spin` 函数苏醒后抛出 `asyncio.CancelledError` 异常，其原 因是发出了取消请求，因此退出循环。

❺ 现在，`slow_function` 函数是协程，在用休眠假装进行 `I/O` 操作时，使用 `yield from` 继续执行事件循环。

❻ `yield from asyncio.sleep(3)` 表达式把控制权交给主循环，在休眠结束后恢复这个协程。

❼ 现在，`supervisor` 函数也是协程，因此可以使用 `yield from` 驱动 `slow_function` 函数。

❽ `asyncio.async(...)` 函数排定 `spin` 协程的运行时间，使用一个 `Task` 对象包装 `spin` 协程，并立即返回。

❾ 显示 `Task` 对象。输出类似于 `<Task pending coro=<spin() running at spinner_ asyncio.py:12>>`。

❿ 驱动 `slow_function()` 函数。结束后，获取返回值。同时，事件循 环继续运行，因为 `slow_function` 函数最后使用 `yield from asyncio.sleep(3)` 表达式把控制权交回给了主循环。

⓫ `Task` 对象可以取消；取消后会在协程当前暂停的 `yield` 处抛出 `asyncio.CancelledError` 异常。协程可以捕获这个异常，也可以延迟取消，甚至拒绝取消。

⓬ 获取事件循环的引用。

⓭ 驱动 `supervisor` 协程，让它运行完毕；这个协程的返回值是这次调用的返回值。

除非想阻塞主线程，从而冻结事件循环或整个应用，否则不要在 `asyncio` 协程中使用 `time.sleep(...)`。如果协程需要在一 段时间内什么也不做，应该使用 `yield from asyncio.sleep(DELAY)`。

使用 `@asyncio.coroutine` 装饰器不是强制要求，但是强烈建议这么做，因为这样能在一众普通的函数中把协程凸显出来，也有助于调试： 如果还没从中产出值，协程就被垃圾回收了（意味着有操作未完成，因 此有可能是个缺陷），那就可以发出警告。这个装饰器不会预激协程。

这两种 `supervisor` 实现之间的主要区别概述如下。

* `asyncio.Task` 对象差不多与 `threading.Thread` 对象等效。

* `Task` 对象用于驱动协程，`Thread` 对象用于调用可调用的对象。

* `Task` 对象不由自己动手实例化，而是通过把协程传给 `asyncio.async(...)` 函数或 `loop.create_task(...)` 方法获取。

* 获取的 `Task` 对象已经排定了运行时间（例如，由 `asyncio.async` 函数排定）；`Thread`实例则必须调用 `start` 方法，明确告知让它运行。

* 在线程版 `supervisor` 函数中，`slow_function` 函数是普通的函数，直接由线程调用。在异步版 `supervisor` 函数中，`slow_function` 函数是协程，由 `yield from` 驱动。

* 没有 `API` 能从外部终止线程，因为线程随时可能被中断，导致系统处于无效状态。如果想终止任务，可以使用 `Task.cancel()` 实例方法，在协程内部抛出 `CancelledError` 异常。协程可以在暂停的 `yield` 处捕获这个异常，处理终止请求。

* `supervisor` 协程必须在 `main` 函数中由 `loop.run_until_complete` 方法执行。

*
