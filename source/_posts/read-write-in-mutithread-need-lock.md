---
title: 多线程读写shared_ptr需要加锁
id: 266
categories:
  - C++
date: 2015-05-22 21:52:15
tags:
---

在现代C++中，通过使用shared_ptr这样的智能指针能够很好的降低内存泄漏的可能性，但是在多线程中无保护的读写shared_ptr则有可能带来race condition的情况。

对此，[boost官方文档](http://cpp.ezbty.org//myfiles/boost/libs/smart_ptr/shared_ptr.htm#ThreadSafety)是这样描述的：
> **shared_ptr** 对象提供与内建类型一样的线程安全级别。一个 **shared_ptr** 实例可以同时被多个线程“读”（仅使用不变操作进行访问）。不同的 **shared_ptr** 实例可以同时被多个线程“写入”（使用类似**operator=** 或 **reset** 这样的可变操作进行访问）（即使这些实例是拷贝，而且共享下层的引用计数）。
>
> 从 Boost 版本 1.33.0 开始，**shared_ptr** 在以下平台上使用了 lock-free 实现：
>
> *   GNU GCC on x86 or x86-64;
> *   GNU GCC on IA64;
> *   Metrowerks CodeWarrior on PowerPC;
> *   GNU GCC on PowerPC;
> *   Windows.
>     可以看到现在版本的shared_ptr本身的引用计数是安全并且无锁的，但是shard_ptr对象本身则不是，为什么呢？

简单来说，因为shared_ptr有不止引用计数一个数据成员，它有指向引用计数对象的指针和指向实际数据的指针这两个成员，虽然引用计数本身是原子化的，但是两个成员一起作为一个对象在多线程环境下读写时，就不能原子化了。

## 数据结构模型

![](/images/blog-shared_ptr_1.png)

shared_ptr是引用计数类型的智能指针，并且绝大多数实现都是存放在堆上的空间里。

具体来说，一个shared_ptr<Foo>包含两个成员，一个是指向实际数据Foo的指针，一个是指向堆上的引用计数对象的指针ref_count。引用计数对象有多个数据成员，具体如上图所示，其中allocator和deleter是可选的。

<!--more-->

## 多线程无保护读写shared_ptr的问题

为了简单起见，后文图例中的引用计数对象均表示为一个引用计数数据。

下面是shared_ptr<Foo> x(new Foo)的内存数据结构：

![](/images/blog-shared_ptr_2.png)

当我们执行语句shared_ptr<Foo> y = x，期望的结果对应于下面的内存数据结构.

![](/images/blog-shared_ptr_3.png)

但是在多线程情况下读写，ptr和cnt两个成员的拷贝不能原子化，会出现race condition。拷贝一个shared_ptr对象分下面两个步骤进行，且不能原子化。

1）拷贝指向实际数据的指针ptr。

![](/images/blog-shared_ptr_4.png)

2）拷贝指向引用计数对象的指针cnt。

![](/images/blog-shared_ptr_5.png)

至于两个步骤的先后顺序与具体的实现有关，但大部分都是先1后2。因为这两个步骤不能原子化，若线程切换发生在两个步骤的中间时刻，就会出现race condition。

## 实例

假设有3个shared_ptr<Foo>对象x、g、n：
```c++
shared_ptr<Foo> g(new Foo);
shared_ptr<Foo> x;
shared_ptr<Foo> n(new Foo);

// in thread A
x = g;

// in trhead B
g = n;
```
1）初始状态。

![](/images/blog-shared_ptr_6.png)

2）线程A执行x = g，但是只执行了一半（拷贝指针ptr），然后发生线程切换（由线程A切换到线程B）。

![](/images/blog-shared_ptr_7.png)

3）线程B执行g = n，完成了拷贝的前一部分（拷贝指针ptr）。

![](/images/blog-shared_ptr_8.png)

4）线程B继续执行g = n，完成了拷贝的后一部分（拷贝引用计数指针cnt），然后发生线程切换（由线程B切换回线程A），但此时Foo1对象已被销毁（因为Foo1的引用计数减为0），x中的ptr成为了空悬指针。

![](/images/blog-shared_ptr_9.png)

5）返回线程B，继续执行x = g，完成后半部分的拷贝（拷贝引用计数指针cnt），原本x的cnt指针应该指向Foo1对象所属的引用计数对象，但由于Foo1对象已经被销毁的缘故，不仅x的cnt指针错误的指向了Foo2对象所属的引用计数对象，还使得Foo2对象的引用计数也错误的增加了1个计数。

![](/images/blog-shared_ptr_10.png)

## 结语

由于shared_ptr的两个数据成员不能读写原子化，所以在多线程环境读写时，一定要在mutex的保护下进行，否则就有可能发生race condition，从而出现意想不到的的结果。

## 参考文献

1.  boost中文手册. [shared_ptr](http://cpp.ezbty.org//myfiles/boost/libs/smart_ptr/shared_ptr.htm#ThreadSafety)
    &nbsp;