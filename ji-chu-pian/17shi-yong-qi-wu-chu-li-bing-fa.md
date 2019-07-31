# 使用期物处理并发

## 示例：网络下载的三种风格

为了高效处理网络 `I/O`，需要使用并发，因为网络有很高的延迟，所以 为了不浪费 CPU 周期去等待，最好在收到网络响应之前做些其他的事。

三个示例程序，从网上下载 `20` 个国 家的国旗图像。第一个示例程序 `flags.py` 是依序下载的下载完一个图像，并将其保存在硬盘中之后，才请求下一个图像。另外两个脚本是并发下载的：几乎同时请求所有图像，每下载完一个文件就保存一个文件。`flags_threadpool.py` 脚本使用 `concurrent.futures` 模块，而 `flags_asyncio.py` 脚本使用 `asyncio` 包。

示例 17-1 运行 `flags.py`、`flags_threadpool.py` 和 `flags_asyncio.py` 脚本得到的结果

```py
$ python3 flags.py
BD BR CD CN DE EG ET FR ID IN IR JP MX NG PH PK RU TR US VN ➊
20 flags downloaded in 7.26s ➋
$ python3 flags.py
BD BR CD CN DE EG ET FR ID IN IR JP MX NG PH PK RU TR US VN
20 flags downloaded in 7.20s
$ python3 flags.py
BD BR CD CN DE EG ET FR ID IN IR JP MX NG PH PK RU TR US VN
20 flags downloaded in 7.09s
$ python3 flags_threadpool.py
DE BD CN JP ID EG NG BR RU CD IR MX US PH FR PK VN IN ET TR
20 flags downloaded in 1.37s ➌
$ python3 flags_threadpool.py
EG BR FR IN BD JP DE RU PK PH CD MX ID US NG TR CN VN ET IR
20 flags downloaded in 1.60s
$ python3 flags_threadpool.py
BD DE EG CN ID RU IN VN ET MX FR CD NG US JP TR PK BR IR PH
20 flags downloaded in 1.22s
$ python3 flags_asyncio.py ➍
BD BR IN ID TR DE CN US IR PK PH FR RU NG VN ET MX EG JP CD
20 flags downloaded in 1.36s
$ python3 flags_asyncio.py
RU CN BR IN FR BD TR EG VN IR PH CD ET ID NG DE JP PK MX US
20 flags downloaded in 1.27s
$ python3 flags_asyncio.py
RU IN ID DE BR VN PK MX US IR ET EG NG BD FR CN JP PH CD TR ➎
20 flags downloaded in 1.42s
```

❶ 每次运行脚本后，首先显示下载过程中下载完毕的国家代码，最后 显示一个消息，说明耗时。

❷ flags.py 脚本下载 20 个图像平均用时 7.18 秒。

❸ flags_threadpool.py 脚本平均用时 1.40 秒。

❹ flags_asyncio.py 脚本平均用时 1.35 秒。

❺ 注意国家代码的顺序：对并发下载的脚本来说，每次下载的顺序都不同。

### 依序下载的脚本

示例 17-2 flags.py：依序下载的脚本；另外两个脚本会重用其中几个函数

```py
import os
import time
import sys

import requests  ➊

POP20_CC = ('CN IN US ID BR PK NG BD RU JP '
            'MX PH VN ET EG DE IR TR CD FR').split()  ➋

BASE_URL = 'http://flupy.org/data/flags'  ➌

DEST_DIR = 'downloads/'

def save_flag(img, filename):  ➎
    path = os.path.join(DEST_DIR, filename)
    with open(path,'wb') as fp:
        fp.write(img)

def get_flag(cc):  ➏
    url = '{}/{cc}/{cc}'.format(BASE_URL, cc=cc.lower())
    resp = requests.get(url)
    return resp.content

def show(text):  ➐
    print(text, end=' ')
    sys.stdout.flush()

def download_many(cc_list):  ➑
    for cc in sorted(cc_list):  ➒
        img = get_flag(cc)
        show(cc)
        save_flag(img, cc.lower() + '.gif')

    return len(cc_list)

def main(download_many):  ➓
    t0 = time.time()
    count = download_many(POP20_CC)
    elapsed = time.time() - 10
    msg = '\n{} flags downloaded in {:.2f}s'
    print(msg.format(count, elapsed))

if __name__ == '__main__':
    main(download_many)  ⓫
```

❶ 导入 `requests` 库。这个库不在标准库中，因此依照惯例，在导入标 准库中的模块（`os`、`time` 和 `sys`）之后导入，而且使用一个空行分隔开。

❷ 列出人口最多的 `20` 个国家的 `ISO 3166` 国家代码，按照人口数量降序排列。

❸ 获取国旗图像的网站。

❹ 保存图像的本地目录。

❺ 把 `img`（字节序列）保存到 `DEST_DIR` 目录中，命名为 `filename`。

❻ 指定国家代码，构建 URL，然后下载图像，返回响应中的二进制内容。

❼ 显示一个字符串，然后刷新 `sys.stdout`，这样能在一行消息中看到 进度。在 **Python** 中得这么做，因为正常情况下，遇到换行才会刷新 `stdout` 缓冲。

❽ `download_many` 是与并发实现比较的关键函数。

❾ 按字母表顺序迭代国家代码列表，明确表明输出的顺序与输入一 致。返回下载的国旗数量。

❿ `main` 函数记录并报告运行 `download_many` 函数之后的耗时。

⓫ `main` 函数必须调用执行下载的函数；我们把 `download_many` 函数 当作参数传给 `main` 函数，这样 `main` 函数可以用作库函数，在后面的示例中接收 `download_many` 函数的其他实现。

### 使用concurrent.futures模块下载

`concurrent.futures` 模块的主要特色是 `ThreadPoolExecutor` 和 `ProcessPoolExecutor` 类，这两个类实现的接口能分别在不同的线程 或进程中执行可调用的对象。这两个类在内部维护着一个工作线程或进程池，以及要执行的任务队列。不过，这个接口抽象的层级很高，像下 载国旗这种简单的案例，无需关心任何实现细节。

示例 17-3 `flags_threadpool.py`：使用 `futures.ThreadPoolExecutor` 类实现多线程下载的脚本

```py
from concurrent import futures

from flags import save_flag, get_flag, show, main  ➊

MAX_WORKERS = 20  ➋

def download_one(cc):  ➌
    image = get_flag(cc)
    show(cc)
    save_img(img, cc.lower() + '.gif')
    return cc

def download_many(cc_list):
    workers = min(MAX_WORKERS, len(cc_list))  ➍
    with  futures.ThreadPoolExecutor(workers) as executor:  ➎
        res = executor.map(download_one, sorted(cc_list))  ➏

    return len(list(res))  ➐

if __name__ == '__main__':
    main(download_many)  ➑
```

❶ 重用 `flags` 模块（见示例 17-2）中的几个函数。

❷ 设定 `ThreadPoolExecutor` 类最多使用几个线程。

❸ 下载一个图像的函数；这是在各个线程中执行的函数。

❹ 设定工作的线程数量：使用允许的最大值（`MAX_WORKERS`）与要处 理的数量之间较小的那个值，以免创建多余的线程。

❺ 使用工作的线程数实例化 `ThreadPoolExecutor` 类；`executor.__exit__` 方法会调用 `executor.shutdown(wait=True)` 方法，它会在所有线程都执行完毕前阻塞线程。

❻ `map` 方法的作用与内置的 `map` 函数类似，不过 `download_one` 函数会在多个线程中并发调用；`map` 方法返回一个生成器，因此可以迭代，获取各个函数返回的值。

❼ 返回获取的结果数量；如果有线程抛出异常，异常会在这里抛出，这与隐式调用 `next()` 函数从迭代器中获取相应的返回值一样。

 ❽ 调用 `flags` 模块中的 `main` 函数，传入 `download_many` 函数的增强版。

### 期物在哪里

期物是 `concurrent.futures` 模块和 `asyncio` 包的重要组件，可是， 作为这两个库的用户，我们有时却见不到期物。示例 17-3 在背后用到了期物, 但是编写的代码没有直接使用。

从 **Python 3.4** 起，标准库中有两个名为 `Future` 的类：`concurrent.futures.Future` 和 `asyncio.Future`。这两个类 作用相同：两个`Future` 类的实例都表示可能已经完成或者尚未完成的延迟计算。这与 `Twisted` 引擎中的 `Deferred` 类、`Tornado` 框架中的 `Future` 类，以及多个 `JavaScript` 库中的 `Promise` 对象类似。

期物封装待完成的操作，可以放入队列，完成的状态可以查询，得到结果（或抛出异常）后可以获取结果（或异常）。

我们要记住一件事：通常情况下自己不应该创建期物，而只能由并发框架（`concurrent.futures`或 `asyncio`）实例化。原因很简单：期物表示终将发生的事情，而确定某件事会发生的唯一方式是执行的时间已 经排定。因此，只有排定把某件事交给 `concurrent.futures.Executor` 子类处理时，才会创建 `concurrent.futures.Future` 实例。例如，`Executor.submit()` 方法的参数是一个可调用的对象，调用这个方法后会为传入的可调用对象排期，并返回一个期物。

客户端代码不应该改变期物的状态，并发框架在期物表示的延迟计算结束后会改变期物的状态，而我们无法控制计算何时结束。

这两种期物都有 `.done()` 方法，这个方法不阻塞，返回值是布尔值， 指明期物链接的可调用对象是否已经执行。客户端代码通常不会询问期物是否运行结束，而是会等待通知。因此，两个 `Future` 类都有 `.add_done_callback()` 方法：这个方法只有一个参数，类型是可调用的对象，期物运行结束后会调用指定的可调用对象。

此外，还有 `.result()` 方法。在期物运行结束后调用的话，这个方法在两个 `Future` 类中的作用相同：返回可调用对象的结果，或者重新抛出执行可调用的对象时抛出的异常。可是，如果期物没有运行结束，`result` 方法在两个 `Future` 类中的行为相差很大。对 `concurrency.futures.Future` 实例来说，调用 `f.result()` 方法会阻塞调用方所在的线程，直到有结果可返回。此时，`result` 方法可以 接收可选的 `timeout` 参数，如果在指定的时间内期物没有运行完毕， 会抛出 `TimeoutError` 异常。`asyncio.Future.result` 方法不支持设定超时时间，在那个库中获取期物的结果最好使用 `yield from` 结构。不过，对 `concurrency.futures.Future` 实例不能这么做。

这两个库中有几个函数会返回期物，其他函数则使用期物，以用户易于理解的方式实现自身。使用 17-3 中的 `Executor.map` 方法属于后者： 返回值是一个迭代器，迭代器的 `__next__` 方法调用各个期物的 `result` 方法，因此我们得到的是各个期物的结果，而非期物本身。

为了从实用的角度理解期物，我们可以使用 `concurrent.futures.as_completed` 函数重写示例 17-3。这个函数的参数是一个期物列表，返回值是一个迭代器，在期物运行结束后产出期物。

示例 17-4 `flags_threadpool_ac.py`：把`download_many` 函数中的 `executor.map` 方法换成 `executor.submit` 方法和 `futures.as_completed` 函数

```py
def download_many(cc_list):
    cc_list = cc_list[:5]  ➊
    with futures.ThreadPoolExecutor(max_workers=3) as executor:  ➋
        to_do = []
        for cc in sorted(cc_list):  ➌
            future = executor.submit(dowload_one, cc)  ➍
            to_do.append(future)  ➎
            msg = 'Scheduled for {}: {}'
            print(msg.format(cc, future))  ➏

        results = []
        for future in futures.as_completed(to_do):  ➐
            res = future.result()  ➑
            msg = '{}  result：{!r}'
            print(msg.format(future, res)) ➒
            results.append(res)

    return len(results)
```

❶ 这次演示只使用人口最多的 `5` 个国家。

❷ 把 `max_workers` 硬编码为 `3`，以便在输出中观察待完成的期物。

❸ 按照字母表顺序迭代国家代码，明确表明输出的顺序与输入一致。

❹ `executor.submit` 方法排定可调用对象的执行时间，然后返回一个期物，表示这个待执行的操作。

❺ 存储各个期物，后面传给 `as_completed` 函数。

❻ 显示一个消息，包含国家代码和对应的期物。

❼ `as_completed` 函数在期物运行结束后产出期物。

❽ 获取该期物的结果。

❾ 显示期物及其结果。

注意，在这个示例中调用 `future.result()` 方法绝不会阻塞，因为 `future` 由`as_completed` 函数产出。

示例 17-5 `flags_threadpool_ac.py` 脚本的输出

```py
$ python3 flags_threadpool_ac.py
Scheduled for BR: <Future at 0x100791518 state=running> ➊
Scheduled for CN: <Future at 0x100791710 state=running>
Scheduled for ID: <Future at 0x100791a90 state=running>
Scheduled for IN: <Future at 0x101807080 state=pending> ➋
Scheduled for US: <Future at 0x101807128 state=pending>
CN <Future at 0x100791710 state=finished returned str> result: 'CN' ➌
BR ID <Future at 0x100791518 state=finished returned str> result: 'BR' ➍
<Future at 0x100791a90 state=finished returned str> result: 'ID'
IN <Future at 0x101807080 state=finished returned str> result: 'IN'
US <Future at 0x101807128 state=finished returned str> result: 'US'

5 flags downloaded in 0.70s
```

❶ 排定的期物按字母表排序；期物的 `repr()` 方法会显示期物的状态：前三个期物的状态是 running，因为有三个工作的线程。

❷ 后两个期物的状态是 `pending`，等待有线程可用。

❸ 这一行里的第一个 `CN` 是运行在一个工作线程中的 `download_one` 函数输出的，随后的内容是 `download_many` 函数输出的。

❹ 这里有两个线程输出国家代码，然后主线程中的 `download_many` 函数输出第一个线程的结果。

严格来说，我们目前测试的并发脚本都不能并行下载。使用 `concurrent.futures` 库实现的那两个示例受 `GIL`（`Global Interpreter Lock`，全局解释器锁）的限制，而 `flags_asyncio.py` 脚本在单个线程中运行。

## 阻塞型I/O和GIL

**CPython** 解释器本身就不是线程安全的，因此有全局解释器锁（`GIL`）， 一次只允许使用一个线程执行 **Python** 字节码。因此，一个 `Python` 进程通常不能同时使用多个 `CPU` 核心。

编写 **Python** 代码时无法控制 `GIL`；不过，执行耗时的任务时，可以使用 一个内置的函数或一个使用 `C` 语言编写的扩展释放 `GIL`。其实，有个使 用 **C** 语言编写的 **Python** 库能管理 `GIL`，自行启动操作系统线程，利用全部可用的 `CPU` 核心。这样做会极大地增加库代码的复杂度，因此大多 数库的作者都不这么做。

然而，标准库中所有执行阻塞型 `I/O` 操作的函数，在等待操作系统返回结果时都会释放 `GIL`。这意味着在 **Python** 语言这个层次上可以使用多线 程，而 `I/O` 密集型 **Python** 程序能从中受益：一个 **Python** 线程等待网络响应时，阻塞型 `I/O` 函数会释放 `GIL`，再运行一个线程

> Python 标准库中的所有阻塞型 I/O 函数都会释放 GIL，允许其 他线程运行。time.sleep() 函数也会释放 GIL。因此，尽管有 GIL，Python 线程还是能在 I/O 密集型应用中发挥作用。

下面简单说明如何在 `CPU` 密集型作业中使用 `concurrent.futures` 模块轻松绕开 `GIL`。

### 使用concurrent.futures模块启动进程

`concurrent.futures` 模块的文档副标题 是“`Launching parallel tasks`”（执行并行任务）。这个模块实现的是真正的并行计算，因为它使用 `ProcessPoolExecutor` 类把工作分配给多个 **Python** 进程处理。因此，如果需要做 `CPU` 密集型处理，使用这个模块能绕开 `GIL`，利用所有可用的 `CPU` 核心。

`ProcessPoolExecutor` 和 `ThreadPoolExecutor` 类都实现了通用的 `Executor` 接口，因此使用 `concurrent.futures` 模块能特别轻松地把基于线程的方案转成基于进程的方案。

下载国旗的示例或其他 `I/O` 密集型作业使用 `ProcessPoolExecutor` 类得不到任何好处。这一点易于验证，只需把示例 17-3 中下面这几行：

```py
def download_many(cc_list):
    workers = min(MAX_WORKERS, len(cc_list))
    with futures.ThreadPoolExecutor(workers) as executor:
```

改成：

```py
def download_many(cc_list)
    with futures.ProcessPoolExecutor() as executor:
```

对简单的用途来说，这两个实现 Executor 接口的类唯一值得注意的区 别是，`ThreadPoolExecutor.__init__` 方法需要 max_workers 参 数，指定线程池中线程的数量。在 `ProcessPoolExecutor` 类中，那个参数是可选的，而且大多数情况下不使用——默认值是 `os.cpu_count()` 函数返回的 `CPU` 数量。这样处理说得通，因为对 `CPU` 密集型的处理来说，不可能要求使用超过 `CPU` 数量的职程。而对 `I/O` 密集型处理来说，可以在一个 `ThreadPoolExecutor` 实例中使用 `10`个、`100` 个或 `1000` 个线程；最佳线程数取决于做的是什么事，以及可用内存有多少，因此要仔细测试才能找到最佳的线程数。

经过几次测试，使用 `ProcessPoolExecutor` 实例下载 `20` 面国旗的时间增加到了 `1.8` 秒，而原来使用 `ThreadPoolExecutor` 的版本是 `1.4` 秒。主要原因可能是，电脑用的是四核 `CPU`，因此限制只能有 `4` 个并发下载，而使用线程池的版本有 `20` 个工作的线程。

`ProcessPoolExecutor` 的价值体现在 `CPU` 密集型作业上。

## 实验Executor.map方法

若想并发运行多个可调用的对象，最简单的方式是使用示例 17-3 中见 过的 `Executor.map` 方法。

示例 17-6 `demo_executor_map.py`：简单演示 `ThreadPoolExecutor` 类的 `map` 方法

```py
from time import sleep, strftime
from concurrent import futures

def display(*args):  ➊
    print(strftime('[%H:%M:%S]'), end=' ')
    print(*args)

def loiter(n):  ➋
    msg = '{}loiter({}): doing nothing for {}s...'
    display(msg.format('\t'*n, n, n))
    sleep(n)
    msg = '{}loiter({}):done.'
    display(msg.format('\t'*n, n))
    return n * 10 ➌

def main():
    display()
    executor = futures.ThreadPoolExecutor(max_workers=3))  ➍
    results = executor.map(loiter, range(5))  ➎
    display('results:', results)  ➏
    display('Waiting for individual results:)
    for i, result in enumerate(results): ➐
        display('result {}:{}'.format(i, result))

main()
```

❶ 这个函数的作用很简单，把传入的参数打印出来，并在前面加上 `[HH:MM:SS]` 格式的时间戳。

❷ `loiter` 函数什么也没做，只是在开始时显示一个消息，然后休眠 `n` 秒，最后在结束时再显示一个消息；消息使用制表符缩进，缩进的量由 `n` 的值确定。

❸ `loiter` 函数返回 `n * 10`，以便让我们了解收集结果的方式。

❹ 创建 `ThreadPoolExecutor` 实例，有 `3` 个线程。

❺ 把五个任务提交给 `executor`（因为只有 `3` 个线程，所以只有 `3` 个任务会立即开始：`loiter(0)`、`loiter(1)` 和 `loiter(2)`）；这是非阻塞调用。

❻ 立即显示调用 `executor.map` 方法的结果：一个生成器，如示例 17-7 中的输出所示。

❼ `for` 循环中的 `enumerate` 函数会隐式调用 `next(results)`，这个函数又会在（内部）表示第一个任务（`loiter(0)`）的 `_f` 期物上调用 `_f.result()` 方法。`result` 方法会阻塞，直到期物运行结束，因此这个循环每次迭代时都要等待下一个结果做好准备。

示例 17-7 示例 17-6 中 demo_executor_map.py 脚本的运行示例

```py
$ python3 demo_executor_map.py
[15:56:50] Script starting. ➊
[15:56:50] loiter(0): doing nothing for 0s... ➋
[15:56:50] loiter(0): done.
[15:56:50]      loiter(1): doing nothing for 1s... ➌
[15:56:50]              loiter(2): doing nothing for 2s...
[15:56:50] results: <generator object result_iterator at 0x106517168> ➍
[15:56:50]                      loiter(3): doing nothing for 3s... ➎
[15:56:50] Waiting for individual results: [15:56:50] result 0: 0 ➏
[15:56:51]      loiter(1): done. ➐
[15:56:51]                              loiter(4): doing nothing for 4s...
[15:56:51] result 1: 10 ➑
[15:56:52]              loiter(2): done. ➒
[15:56:52] result 2: 20
[15:56:53]                      loiter(3): done.
[15:56:53] result 3: 30
[15:56:55]                              loiter(4): done. ➓
[15:56:55] result 4: 40
```

❶ 这次运行从 `15:56:50` 开始。

❷ 第一个线程执行 `loiter(0)`，因此休眠 `0` 秒，甚至会在第二个线程 开始之前就结束，不过具体情况因人而异。

❸ `loiter(1)` 和 `loiter(2)` 立即开始（因为线程池中有三个职程，可 以并发运行三个函数）。

❹ 这一行表明，`executor.map` 方法返回的结果（`results`）是生成器；不管有多少任务，也不管 `max_workers` 的值是多少，目前不会阻塞。

❺ `loiter(0)` 运行结束了，第一个职程可以启动第四个线程，运行 `loiter(3)`。

❻ 此时执行过程可能阻塞，具体情况取决于传给 `loiter` 函数的参 数：`results` 生成器的 `__next__` 方法必须等到第一个期物运行结束。 此时不会阻塞，因为 `loiter(0)` 在循环开始前结束。注意，这一点之 前的所有事件都在同一刻发生——`15:56:50`。

❼ 一秒钟后，即 `15:56:51`，`loiter(1)` 运行完毕。这个线程闲置，可以开始运行 `loiter(4)`。

❽ 显示 `loiter(1)` 的结果`：10`。现在，`for` 循环会阻塞，等待 `loiter(2)` 的结果。

❾ 同上：`loiter(2)` 运行结束，显示结果；`loiter(3)` 也一样。

❿ `2` 秒钟后 `loiter(4)` 运行结束，因为 `loiter(4)` 在 `15:56:51` 时开始，休眠了 `4` 秒。

`executor.submit` 和 `futures.as_completed` 这个组合比 `executor.map` 更灵活，因为 `submit` 方法能处理不同的可调用对象和参数，而 `executor.map` 只能处理参数不同的同一个可调用对象。此外，传给 `futures.as_completed` 函数的期物集合可以来 自多个 `Executor` 实例，例如一些由 `ThreadPoolExecutor` 实例 创建，另一些由 `ProcessPoolExecutor` 实例创建。
