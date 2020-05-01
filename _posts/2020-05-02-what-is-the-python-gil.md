---
layout: post
title:  "浅析Python GIL"
date:   2020-05-02 00:40:00 +0800
categories: Python
---

> 本文翻译自[Real Python](https://realpython.com)网站Abhinav Ajitsaria的文章“[What is the Python Global Interpreter Lock (GIL)?](https://realpython.com/python-gil)”。

关键词汇及翻译：

- [GIL](https://wiki.python.org/moin/GlobalInterpreterLock)：Global Interpreter Lock，全局解释器锁
- [CPU-bound](https://en.wikipedia.org/wiki/CPU-bound)：计算密集型
- [I/O-bound](https://en.wikipedia.org/wiki/I/O_bound)：I/O密集型

----

Python全局解释器锁（简称[GIL](https://wiki.python.org/moin/GlobalInterpreterLock)），简而言之就是一个互斥量（或锁），它只允许一个线程控制Python解释器。

这意味着在任何时间点上只能有一个线程处于执行状态。对于执行单线程程序的开发人员而言GIL的影响并不明显，但在CPU密集型、或多线程代码中GIL可能会成为性能瓶颈。

即使在多核CPU多线程架构中，同一时间GIL也只允许一个线程执行，所以在Python语言里面GIL可能是个“臭名昭著”的特性。

**在这篇文章中，你将会了解GIL如何影响Python程序的性能，以及如何减轻它对你编写的代码可能产生的影响。**

## GIL为Python解决了什么问题？

Python语言使用引用计数来管理内存。它意味着在Python中创建的对象会有一个相应的引用计数变量，用来记录指向该对象的引用数量。当这个计数达到零时，该对象所占用的内存就会被释放。

让我们来看看一个简单的示例代码，演示一下引用计数的工作原理：

```python
>>> a = []
>>> b = a
>>> sys.getrefcount(a)
3
```

在上面的例子中，创建了一个长度为0的`list`列表对象，该对象的引用计数为3，它同时被变量`a`、`b`和传递给函数`sys.getrefcount()`的参数引用。

回过头来看GIL：

它涉及的问题是，我们需要保护引用计数变量以防止它同时被两个线程修改。如果发生这种情况，可能会导致泄漏的内存从未被释放，又或者更糟糕的是错误的释放了仍在使用的对象。这样你的Python程序就会崩溃或存在一些奇怪的Bug。

试想下如果我们对多线程间共享的所有数据结构都添加锁机制，那么这些共享数据的引用计数在被修改时就不会出现不一致的情况。

但是给每个对象或对象组都额外添加一个锁就会存在多个锁，这可能会引起死锁（只有存在多个锁的情况下才会发生死锁）。另一个副作用是多个锁的重复获取、释放将会降低程序性能。

GIL是解释器本身的单一锁，它要求任何Python字节码的执行都需要获取该解释器锁。因为只有一个锁所以能够简单有效的防止死锁，并且不会带来太多的性能开销。但是同时也使所有CPU密集型（多线程）Python程序成为了单线程程序。

GIL虽然也被Ruby等其他语言的解释器所使用，但它并不是解决引用计数这个问题的唯一方案。有些语言通过使用引用计数以外的方法来规避GIL对线程安全内存管理的要求，比如垃圾回收等。

另一方面，这些语言往往不得不通过添加其他的特性（如[JIT](https://en.wikipedia.org/wiki/Just-in-time_compilation)编译器）来弥补因缺失GIL引起的单线程性能损失。

## 为什么选择GIL作为解决方案？

既然GIL如此碍事而且无法做到真正的多线程，为什么还要选择GIL，Python语言开发者如何看待这样一个“错误”的决定？

在视频[PyCon 2015 - Python's Infamous GIL by Larry Hastings](https://youtu.be/KVKufdTphKs?t=12m11s)当中，Larry Hastings指出Python语言至今能够这样流行正是因为有GIL这样的设计决定。

当操作系统还没有多线程概念的时候Python就出现了，设计Python语言就是为了更容易、更快速的开发，以便更多的开发者使用它。

Python里面很多扩展或特性是使用C语言实现的，为了避免发生不一致的修改，GIL提供C扩展特性所要求的线程安全内存管理机制。

在Python里面能够非常简单、而且方便的实现GIL，而且它只需要一把锁所以能够提升单线程程序性能。

Python能够很容易的集成非线程安全的C库，这些C库扩展也正是Python语言能够被众多不同社区乐意接受的原因之一。

正如你所见，使用GIL来解决内存管理难题是开发者早期实现CPython时选择的非常务实的解决方案。

## GIL对多线程Python程序的影响

对于一个典型的Python程序（或任何计算器程序），CPU密集型与I/O密集型的性能是有区别的。

CPU密集型程序会尽可能的利用CPU资源，包括矩阵乘法、搜索、图像处理等这些数学运算的程序。

I/O密集型程序会花时间等待输入、输出，包括来自用户、文件、数据库、网络等设备的输入、输出。I/O密集型程序有时不得不等待一定时间，因为依赖的I/O设备或程序在准备好输入、或输出之前可能有自己先要处理完的事情，比如在提示输入前用户可能需要思考下后才决定键入什么内容，又或者数据库查询需要花些时间才能返回结果。

先来看一段CPU密集型程序，倒计时`countdown`：

```python
# single_threaded.py
import time
from threading import Thread

COUNT = 50000000

def countdown(n):
    while n>0:
        n -= 1

start = time.time()
countdown(COUNT)
end = time.time()

print('Time taken in seconds -', end - start)
```

在我的4核CPU系统上输出结果如下：

```bash
$ python single_threaded.py
Time taken in seconds - 6.20024037361145
```

现在我们把上面需要倒数同样数字的`countdown`程序改成两个线程执行：

```python
# multi_threaded.py
import time
from threading import Thread

COUNT = 50000000

def countdown(n):
    while n>0:
        n -= 1

t1 = Thread(target=countdown, args=(COUNT//2,))
t2 = Thread(target=countdown, args=(COUNT//2,))

start = time.time()
t1.start()
t2.start()
t1.join()
t2.join()
end = time.time()

print('Time taken in seconds -', end - start)
```

同样硬件和操作系统条件下的执行输出：

```bash
$ python multi_threaded.py
Time taken in seconds - 6.924342632293701
```

从执行结果中看出两个版本的程序在倒数相同数字时花的时间几乎相同，这是因为GIL锁限制多线程版本无法并行。

那么GIL对I/O密集型程序的影响是怎样的呢？GIL对I/O密集型多线程性能影响不大，因为在等待I/O时GIL锁是在多线程间共享的。

但是，对于CPU密集型多线程程序而言（比如使用多线程处理图形图像），由于GIL锁的存在无法并发多线程，程序退化成单线程执行。而且与单线程实现相相比，多线程实现还可能会适当的增加执行时间（比如的`countdown`示例）。

这是因为多线程之间会抢占GIL，重复的获取、释放锁自然会引发多余的性能开销。

## 为什么GIL还没有被移除？

Python语言开发者收到很多关于GIL的抱怨，但像Python这样流行的语言，移除GIL这样重大的决定将会带来更多向后兼容的问题。

不可否认，GIL当然是可以被移除的，而且历史上就曾经被开发者和研究者多次尝试移除过，可是这些尝试都破坏了很多严重依赖GIL实现的C库扩展。

当然有其它的解决方案能处理GIL解决的问题，但这些解决方案都以牺牲单线程、或I/O密集型多线程程序性能为代价，而且其中一些方案实现起来还特别困难。总而言之，没有任何人希望因为更新Python语言版本而引起程序性能下降。

Python语言的创建者和[BDFL](https://en.wikipedia.org/wiki/Benevolent_dictator_for_life) -- Guido van Rossum，在2007年9月发布的文章[“It isn’t Easy to remove the GIL”](https://www.artima.com/weblogs/viewpost.jsp?thread=214235)就曾解释过：

> 我将非常高兴如果Py3k有个补丁能够移除GIL，而且它不会降低单线程程序（和I/O密集型多线程程序）的性能。

可惜的是，目前所有移除GIL的尝试都没能满足这样的条件。

## 为什么Python 3没有移除GIL？

Python 3确实有机会从头开始设计很多新特性，抛弃一些现有的C扩展依赖，毕竟这些C扩展需要做适当的调整才能移植至Python 3。然而这正是Python 3早期版本被开发者社区接受得比较缓慢的原因。

所以，为什么GIL没有被一并移除呢？

如果在Python 3移除GIL，对比单线程在Python 2的性能将明显有所下降，你可以想象这将带来怎样的后果（或许将没有人愿意更新至Python 3，译者添加）。毫无疑问单线程性能优势受益于GIL，这也是Python 3依然保留GIL的原因。

而且Python 3对现有的GIL做了很大的改进。

刚刚我们讨论了GIL对CPU密集型及I/O密集型多线程程序的影响，但如果有程序同时包含CPU密集型和I/O密集型线程呢？

在这样的程序里面，CPU密集型的线程将会一直抢占并持有Python的GIL锁，而I/O密集型的线程可能没有机会获取到GIL锁，自然没办法得到执行从而被饿死。

出现这种情况的原因是Python有这样一个机制，持有GIL锁的线程在执行**固定间隔**后将会释放GIL锁，此时如果没有其它的线程获取到锁，原来的线程将继续占有GIL并继续执行。

```python
>>> import sys
>>> # The interval is set to 100 instructions:
>>> sys.getcheckinterval()
100
```

这个机制存在的问题在于，针对CPU密集型线程在释放GIL锁后，大部分情况下原线程都将继续抢占到GIL锁。David Beazley对此深有研究并将可视化结果分享在他的博客里面，点击查阅“[The Python GIL Visualized](http://dabeaz.blogspot.com/2010/01/python-gil-visualized.html)”。

Antoine Pitrou于2009在Python 3.2通过添加一种新[机制](https://mail.python.org/pipermail/python-dev/2009-October/093321.html)解决了这个问题。这种机制会检查其它线程获取GIL锁失败的请求数量，并在确保其它线程有机会执行之前，适当的禁止当前线程持续抢占GIL锁。

## 如何应对Python的GIL？

如果你的程序因为GIL引发一些问题（比如性能下降、I/O无法被执行等，译者添加），可以尝试参考下面这些解决方案：

**多进程vs多线程：**最常见的方法是改成多进程模式，用多进程替换出现问题的多线程程序。每个Python进程都有自身独立的Python解释器、内存空间，所以多进程程序里面将不会存在GIL锁的问题。Python的[multiprocessing](https://docs.python.org/2/library/multiprocessing.html)模块允许我们创建多进程，用该模块接口把之前的`countdown`倒计时程序改成多进程模式：

```python
from multiprocessing import Pool
import time

COUNT = 50000000
def countdown(n):
    while n>0:
        n -= 1

if __name__ == '__main__':
    pool = Pool(processes=2)
    start = time.time()
    r1 = pool.apply_async(countdown, [COUNT//2])
    r2 = pool.apply_async(countdown, [COUNT//2])
    pool.close()
    pool.join()
    end = time.time()
    print('Time taken in seconds -', end - start)
```

同样硬件和操作系统条件下的执行输出：

```bash
$ python multiprocess.py
Time taken in seconds - 4.060242414474487
```

对比之前的多线程版本，性能明显有所提升。

可是执行时间并没有减少至单线程的一半，这是因为进程管理也会有额外的开销。多进程没有多线程那么轻量，所以也要小心这可能会成为扩展的瓶颈。

**选择其它的Python解释器：**Python语言有多种解释器实现，比如CPython、Jython、IronPython、PyPy，分别用C、Java、C#、Python语言实现，这些都是目前流行的Python解释器。而GIL只存在于最原生的Python实现里面，即CPython。如果其它Python解释器也提供相同的特性或扩展库，完全可以尝试用其它Python解释器执行你的Python程序。

**继续等待：**很多Python用户受益于GIL给单线程带来的性能优势，而多线程程序员也不必烦恼，因为Python社区里面已经有一些很聪明的开发者正在尝试从CPython中移除GIL，其中就包含[Gilectomy](https://github.com/larryhastings/gilectomy)这个项目。

Python的GIL常常被认为是个很神秘、很困难的话题。但请记住，作为一个Pythonista，你通常只有在编写C扩展库或者CPU密集型多线程程序时才会受到GIL的影响。

最后，希望本文能帮助你理解GIL，或者为你在项目中应对GIL问题带来一些建议性的思考或意见。如果你想了解GIL更底层的工作原理，推荐你观看David Beazley的Youtube视频“Understanding the Python GIL”，点击[观看](https://youtu.be/Obt-vMVdM8s)。
