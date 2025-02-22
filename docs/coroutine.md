# Attack on Tornado - 协程 🌪







<extoc></extoc>

## 前言

协程其实比线程更为古老 , 那时候只有一个CPU在执行任务 , 为了实现在一个核心上的并发 , 就有了协程 ; 而要支持并发 , 那就必须将任务拆分成多任务 , 再将这些任务按照某种顺序合并成一条新的时间执行线 , 这正是异步程序

异步是实现并发的一种方式 , 实现并发的方式也不仅仅只有异步一种 , 我们在这里只针对单线程以及协程展开

[协程(`Coroutine`)](https://en.wikipedia.org/wiki/Coroutine) , 就是一组可以协同工作 [(协作式)](https://en.wikipedia.org/wiki/Cooperative_multitasking) 的子程序 , 协程实际上是一个很普通的函数 , 或者一个代码块 , 或者子过程 , 再或者说协程就是你可以暂停执行的函数

在 `Python` 中 , 可以暂停执行的函数就是生成器函数 , 也就是使用了 `yield` 关键字的函数 , 但这不代表生成器就是协程 , 只能说生成器可以作为协程使用 , 关于这一点你可以在《流畅的Python》第16章-协程篇找到答案

生成器只能把控制权转交给调用生成器的调用者 , 而协程讲究的是多任务协作 , 协程可以将执行权交给其他协程 ; 生成器用于生产数据 , 协程消费数据 , 且协程与迭代无关 ; 生成器实际上属于协程的子集 , 所以生成器也叫做 "半协程"

## 协程

在上一篇《生成器》中我们了解了 `Python` 中生成器的发展 , 实际上这不仅仅是生成器的发展 , 也是协程的进步

从最初我们使用 `yield` 构建可中断函数 , 再到 `send` 可以调度生成器函数 , 再到 `yield from` 可以构建生成器调用链 (这个调用链至关重要 , 因为 `send` 只是发送数据 , 而 `yield from` 可以直接调度其他生成器)

拥有了这些基础 , 协程的实现就变得简单了起来 

我们先来看两段代码 :


```python
import queue

def task(name, work_queue):
    if work_queue.empty():
        print(f"Task {name} nothing to do")
    else:
        while not work_queue.empty():
            count = work_queue.get()
            total = 0
            print(f"Task {name} running")
            for x in range(count):
                total += 1
            print(f"Task {name} total: {total}")

def main():
    """
    This is the main entry point for the program
    """
    # Create the queue of work
    work_queue = queue.Queue()

    # Put some work in the queue
    for work in [15, 10, 5, 2]:
        work_queue.put(work)

    # Create some synchronous tasks
    tasks = [(task, "One", work_queue), (task, "Two", work_queue)]

    # Run the tasks
    for t, n, q in tasks:
        t(n, q)

if __name__ == "__main__":
    main()
```

运行结果 : 

```python
Task One running
Task One total: 15
Task One running
Task One total: 10
Task One running
Task One total: 5
Task One running
Task One total: 2
Task Two nothing to do
```

接下来我们使用 `yield` 来改造一下 : 

```python
import queue

def task(name, queue):
    while not queue.empty():
        count = queue.get()
        total = 0
        print(f"Task {name} running")
        for x in range(count):
            total += 1
            yield
        print(f"Task {name} total: {total}")

def main():
    """
    This is the main entry point for the program
    """
    # Create the queue of work
    work_queue = queue.Queue()

    # Put some work in the queue
    for work in [15, 10, 5, 2]:
        work_queue.put(work)

    # Create some tasks
    tasks = [task("One", work_queue), task("Two", work_queue)]

    # Run the tasks
    done = False
    while not done:
        for t in tasks:
            try:
                next(t)
            except StopIteration:
                tasks.remove(t)
            if len(tasks) == 0:
                done = True

if __name__ == "__main__":
    main()
```

运行结果 : 

```python
Task One running
Task Two running
Task Two total: 10
Task Two running
Task One total: 15
Task One running
Task Two total: 5
Task One total: 2
```

同样是两个 `task` , 在第二段代码中我们实现了异步执行 , 两个 `task` 交叉协作完成任务 , 这两个 `task` 就是两个协程

不难发现 , 即使我们不用 `yield` , 我们自己通过函数也可以实现这样的并发效果 , 把上面的代码按照 `yield` 拆分成几个函数功能上是一样的 , 我们把拆分的函数叫做子例程 , 实际上 , 子例程可以看做是特定状态的协程 , 任何的子例程都可以转写成不使用 `yield` 的协程

**子例程与协程**

相对于子例程而言 , 协程更加灵活 , 协程更加适合用来实现彼此比较熟悉的程序组件 , 或者说耦合度高一点的组件 

协程的切换概念是 "让步" , 而子例程的切换概念是 "出产" , 一个主动 , 一个被动 , 以下摘自 [Wiki](https://en.wikipedia.org/wiki/Coroutine) : 

- 子例程可以调用其他子例程 , 调用者等待被调用者结束后继续执行 , 故而子例程的生命期遵循后进先出 , 即最后一个被调用的子例程最先结束返回 , 协程的生命期完全由对它们的使用需要来决定
- 子例程的起始处是惟一的入口点 , 每当子例程被调用时，执行都从被调用子例程的起始处开始 , 协程可以有多个入口点 , 协程的起始处是第一个入口点 , 每个 `yield` 返回出口点都是再次被调用执行时的入口点
- 子例程只在结束时一次性的返回全部结果值 , 协程可以在 `yield` 时不调用其他协程 , 而是每次返回一部分的结果值 , 这种协程常称为生成器或迭代器

我们再回到协程 , 在本篇文章开头就已经说过 , 协程实际上早在线程之前就已经出现了 , 但是为什么协程却没有被更广泛的使用 , 而是线程占据主导地位呢 ?

## 线程与协程

线程与协程在某些情况下其实是没有可比性的 , 因为在现代 , 线程是 `CPU` 调度的基本单位 , 而协程根本不会参与 `CPU` 调度

从调度方式来讲 , 线程的调度是由操作系统控制的 , 且是抢占式的 ; 而协程的调度方式是由用户自己控制的 , 这种调度需要我们的程序去控制 , 且也无法像线程一样强制中断 , 需要程序自己主动让出执行权 , 所以协程通常是协作式或者说非抢占式的

从开销成本来讲 , 线程的上下文切换自然要比协程的上下文切换的开销要大 , 且线程属于系统对象 , 对系统资源消耗也比协程要大 , 这也是为什么有的人把协程叫做 "微线程" , 但是要注意 , 这并不意味着协程能和线程媲美

从利用核心来讲 , 线程是可以利用多核的 , 虽然在 `Python` 中由于 `GIL` 锁无法利用多核优势 , 但是实际上很多语言中线程是可以利用多核的 , 而协程只能跑在线程上 , 不管有没有 `GIL` 都无法利用多核

从 `IO` 来讲 ,  当程序中遇到 `IO` 时 , 线程会被阻塞 , 所以协程自然也会被阻塞 , 但是在多线程的情况下每个线程之间的阻塞互不影响 , 而 `CPU` 时间片是由系统分配的 , 如果线程处于等待状态 , 系统自然会将时间片分配给其他线程继续执行 , 这也是我们并发模型中使用最多的多线程模型 , 此时协程针对 `IO` 就已经捉襟见肘了

综上 , 不管是在 `CPU` 密集型的业务下 , 还是在 `IO` 密集型的业务下 , 协程都不如线程 ; `CPU` 密集型场景下 , 我们要提高 `CPU` 的利用率 , 有几个核心那就用几个核心 , 如果一个核心配备一个线程 , 那就没有线程之间的切换了 , 开销自然不算是什么问题 ; `IO` 密集型场景下 , 协程整个阻塞 , 根本无法使用

这样一比较好像协程毫无用处 , 但是不要忘记了 , 遇到 `IO` 当前线程也没法继续执行 , 多线程只不过是切换到了其他就绪的线程罢了 , 在 `Python 2` 中默认线程每执行100个字节码就会切换(可通过 `sys.getcheckinterval()` 获取) , 而在 `Python 3` 中则改成了固定时间切换(可通过 `sys.getswitchinterval` 获取) , 而这种僵硬的调度方式会导致很多无效的切换 , 当线程数越大 , 带来的性能损耗就越大

我们之所以切换实际上更多的原因是由于有 `IO` 阻塞(当然也有公平性) , 所以为了降低这种性能损耗 , 我们可以改变 `IO` 的处理方式 , 让线程可以跳过 `IO` 且不丢失 `CPU` 的利用率 , 在前面的文章中已经介绍了6种IO模型 , 这里要说的正是IO多路复用模型 , 而这个模型带给我们的挑战就是 , 我们需要通过回调的方式来实现我们的业务 , 而回调嵌套就形成了回调链也就是回调地狱

## 回调与协程

我们先来回顾一下《AIO实现》中的例子 : 

回调版本

```python
import socket
from selectors import DefaultSelector, EVENT_READ, EVENT_WRITE

def server(host, port):
    selector = DefaultSelector()
    sock = socket.socket()
    sock.bind((host, port))
    sock.listen(5)
    sock.setblocking(False)

    def accept(sock, mask):
        print('Accept ...')
        conn, addr = sock.accept()

        def connect(conn, mask):
            print('Connect ...')
            data = conn.recv(1024).decode('utf-8')

            def write(conn, mask):
                print('Write ...')

                def success(conn, mask):
                    print('Success ...')
                    selector.unregister(conn)
                    conn.close()

                selector.modify(conn, EVENT_READ, success)

            selector.modify(conn, EVENT_WRITE, write)

            conn.send(b'ok')

        selector.register(conn, EVENT_READ, connect)

    selector.register(sock, EVENT_READ, accept)
    print('启动服务端...')
    while True:
        ready = selector.select()
        for event, mask in ready:
            callable = event.data
            callable(event.fileobj, mask)

if __name__ == '__main__':
    server('localhost', 8888)
```

在回调版本中 , 我们抽出回调部分 : 

```python
def accept(sock, mask):
    print('Accept ...')
    def connect(conn, mask):
        print('Connect ...')
        def write(conn, mask):
            print('Write ...')
            def success(conn, mask):
                print('Success ...')
            selector.modify(conn, EVENT_READ, success)
        selector.modify(conn, EVENT_WRITE, write)
    selector.register(conn, EVENT_READ, connect)
selector.register(sock, EVENT_READ, accept)

# 同步调用(由上而下): accept -> connect -> write -> success
# 异步调用(由内而外): accept(connect(write(success())))
```

回调就意味着程序有依赖关系 , 而依赖关系就会形成回调链 , 回调链一旦过长就会就成了回调地狱 , 因为我们不知道事件什么时候可以完成 , 所以我们必须预先定义好事件完成后将要回调的函数 , 如果我们能够在事件开始监听时中断程序 , 在事件完成回调时恢复程序 , 是不是就不需要预先定义回调函数了呢 ? 

不管回调链有多长 , 我们只需要让程序不断的自我中断 , 再恢复 , 直到程序执行完毕 , 一切似乎就变得简单起来了 , 那我们要怎么去实现呢 ? 答案是 : 协程


那么让我们来改造一下上面这段代码 :

```python
import socket
from inspect import isfunction
from selectors import DefaultSelector, EVENT_READ, EVENT_WRITE


def server(host, port):
    selector = DefaultSelector()
    sock = socket.socket()
    sock.bind((host, port))
    sock.listen(5)
    sock.setblocking(False)

    def handler():
        gen = accept()
        gen.send(None)
        gen.send(gen)

    def accept():
        gen = yield

        print('Accept ...')
        conn, addr = sock.accept()
        selector.register(conn, EVENT_READ, gen)
        yield

        print('Connect ...')
        data = conn.recv(1024).decode('utf-8')
        selector.modify(conn, EVENT_WRITE, gen)
        conn.send(b'ok')
        yield

        print('Write ...')
        selector.modify(conn, EVENT_READ, gen)
        yield

        print('Success ...')
        selector.unregister(conn)
        conn.close()
        yield

    selector.register(sock, EVENT_READ, handler)
    print('启动服务端...')
    while True:
        ready = selector.select()
        for event, mask in ready:
            if isfunction(event.data):
                event.data()
            else:
                try:
                    event.data.send(event.fileobj)
                except StopIteration as e:
                    continue


if __name__ == '__main__':
    server('localhost', 8888)
```

每一次 `yield` 都将执行权限让出 , 等待事件回调发生 , 这就是协程的魅力 , 回调的异步代码变成了一种同步的方式 

不过 , 如果我们这样去编写异步程序 , 那也得费不少劲 , 虽然代码看着同步了 , 但是回调注册却并友好 , 有没有法子可以让我们不关心回调注册 ? 先思考一下 , 在下一篇《AIO协程实现》中 , 我们再来优化

**协程给我们带来的最大好处实际上并不是性能 , 而是我们可以通过协程来将异步代码用同步的方式编写**

现在我们其实还有一个问题那就是协程如何调度 , 这就是我们接下来要说的事件循环

## 事件循环

[事件循环](https://en.wikipedia.org/wiki/Event_loop) 是一种程序结构或设计模式 , 用于在程序中等待和分发事件或者消息 , 简单来说就是当某件事情发生时 , 接下来该做什么 , 通常它是一个死循环 , 因为它需要不断的收集事件并处理事件

在上面的代码中其实我们已经实现了一个最简单的事件循环 : 

```python
# 永不停歇的收集事件并处理事件
while True:
    # 收集就绪的事件列表
    ready = selector.select()
    # 循环处理事件
    for event, mask in ready:
        if isfunction(event.data):
            event.data()
        else:
            try:
                event.data.send(event.fileobj)
            except StopIteration as e:
                continue
```

事件循环上就是一个调度器 , 是我们用户程序之间的调度器 , 就是操作系统调度线程一样 , 事件循环可以用来调度我们的协程 , 所以通常你会发现协程总是和事件循环同时出现 , 所以我们对事件循环的要求一般都比较高 , 因为协程调度的性能直接由事件循环的调度方案决定

在早期的 `Python` 中 , 由 `gevent` 提供了事件循环能力 , 而 `Python 3.4` 时引入 `asyncio` 标准库来提供事件循环能力 

关于事件循环的具体实现 , 请阅读后续文章

## async&await

最后我们来说说 `async` 和 `await` 

在 `Python 3.4` 中

```python
# This also works in Python 3.5.
import asyncio.coroutine

@asyncio.coroutine
def py34_coro():
    yield from stuff()
```

对应的字节码

```python
>>> dis.dis(py34_coro)
  2           0 LOAD_GLOBAL              0 (stuff)
              3 CALL_FUNCTION            0 (0 positional, 0 keyword pair)
              6 GET_YIELD_FROM_ITER
              7 LOAD_CONST               0 (None)
             10 YIELD_FROM
             11 POP_TOP
             12 LOAD_CONST               0 (None)
             15 RETURN_VALUE
```

在 `Python 3.5` 中

```python
async def py35_coro():
    await stuff()
```

对应的字节码

```python
>>> dis.dis(py35_coro)
  1           0 LOAD_GLOBAL              0 (stuff)
              3 CALL_FUNCTION            0 (0 positional, 0 keyword pair)
              6 GET_AWAITABLE
              7 LOAD_CONST               0 (None)
             10 YIELD_FROM
             11 POP_TOP
             12 LOAD_CONST               0 (None)
             15 RETURN_VALUE
```

它们之间的差异仅仅是 `GET_YIELD_FROM_ITER` 和 `GET_AWAITABLE` 的差异 , 而这两个函数实际上都是用来标记协程的 , 所以其实 `yield from` 和 `async/await` 并无两样

`GET_YIELD_FROM_ITER` 可以接收生成器或者协程 , 而 `GET_AWAITABLE` 只接受协程

所以 `async/await` 并没有做什么特殊的提升 , 这两个关键字也主要是为了将协程规范化 , 明确了协程的意义 , 而不是将生成器和协程混在一起

这些也都是有迹可循的 : 

- 3.4：`asyncio` 在 `Python` 标准库中引入 , 但是只是临时的
- 3.5：`async/await` 成为 `Python` 语法的一部分 , 用于表示和等待协程 , 但它们还不是保留关键字
- 3.6：引入了异步生成器和异步循环 , `asyncio` 不再只是临时的 , 而是稳定的
- 3.7：`async/await` 成为保留关键字 , 它们旨在替换 `asyncio.coroutine()` 装饰器

到这里 , 协程的前世今生我们已经理清了 , 不过还有一点 , `gevent` 是有栈协程的代表 , 而 `asyncio` 是无栈协程的代表 , 关于有栈协程和无栈协程的具体细节 , 请关注我后续的文章

最后我们来简单总结一下

## 总结

协程难的原因其实不是它本身 , 而是我们需要从历史出发 , 理清它的前世今生

网络上关于协程相关的文章更多的是用法 , 而从用法出发 , 很多东西都已经被隐藏了 , 导致中间出现了断层

在本篇中 , 我们并没有介绍协程的用法 , 而是从源头出发 : 

- 协程实际上是为了解决单核多任务并发问题
- 协程可以降低异步回调的复杂性 , 将异步代码同步化(这里只是说编写风格)
- 协程需要有事件循环来调度
- `async/awiat` 可以看做是 `yield from` 的语法糖 , 将生成器与协程进行隔离

**参考资料**

- 《流畅的Python》
- [python-async-features](https://realpython.com/python-async-features/)
- [how-the-heck-docs-async-await-work-in-python-3.5](https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/)
- [i-dont-understand-asyncio](https://lucumr.pocoo.org/2016/10/30/i-dont-understand-asyncio/)
- [深入理解 Python 异步编程（上）](https://mp.weixin.qq.com/s/GgamzHPyZuSg45LoJKsofA)

