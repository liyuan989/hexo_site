---
title: 谨慎使用多线程中的fork
id: 19
categories:
  - 多线程
date: 2015-02-07 11:23:00
tags:
---

## 1\. 前言

在单核时代，大家所编写的程序都是单进程/单线程程序。随着计算机硬件技术的发展，进入了多核时代后，为了降低响应时间，重复充分利用多核cpu的资源，使用多进程编程的手段逐渐被人们接受和掌握。然而因为创建一个进程代价比较大，多线程编程的手段也就逐渐被人们认可和喜爱了。

记得在我刚刚学习线程进程的时候就想，为什么很少见人把多进程和多线程结合起来使用呢，把二者结合起来不是更好吗？现在想想当初真是too young too simple，后文就主要讨论一下这个问题。

## 2\. 进程与线程模型

进程的经典定义就是一个执行中的程序的实例。系统中的每个程序都是运行在某个进程的context中的。context是由程序正确运行所需的状态组成的，这个状态包括存放在存储器中的程序的代码和数据，它的栈、通用目的寄存器的内容、程序计数器（pc）、环境变量以及打开的文件描述符的集合。

进程主要提供给上层的应用程序两个抽象：

*   一个独立的逻辑控制流，它提供一个假象，好像我们程序独占的使用处理器。
*   一个私有的虚拟地址空间，它提供一个假象，好像我们的程序独占的使用存储器系统。

线程，就是运行在进程context中的逻辑流。线程由内核自动调度。每个线程都有它自己的线程context，包括一个唯一的整数线程id、栈、栈指针、程序计数器（pc）、通用目的寄存器和条件码。每个线程和运行在同一进程内的其他线程一起共享进程context的剩余部分。这包括整个用户虚拟地址空间，它是由只读文本（代码）、读/写数据、堆以及所有的共享库代码和数据区域组成。线程也同样共享打开文件的集合。

即进程是资源管理的最小单位，而线程是程序执行的最小单位。

在linux系统中，posix线程可以「看做」为一种轻量级的进程，pthread_create创建线程和fork创建进程都是在内核中调用__clone函数创建的，只不过创建线程或进程的时候选项不同，比如是否共享虚拟地址空间、文件描述符等。

## 3\. fork与多线程

我们知道通过fork创建的一个子进程几乎但不完全与父进程相同。子进程得到与父进程用户级虚拟地址空间相同的（但是独立的）一份拷贝，包括文本、数据和bss段、堆以及用户栈等。子进程还获得与父进程任何打开文件描述符相同的拷贝，这就意味着子进程可以读写父进程中任何打开的文件，父进程和子进程之间最大的区别在于它们有着不同的pid。

但是有一点需要注意的是，在linux中，fork的时候只复制当前线程到子进程，在[fork(2)-Linux Man Page](http://linux.die.net/man/2/fork)中有着这样一段相关的描述：

> the child process is **created with a single thread**--the one that called fork. the entire virtual address space of the parent is replicated in the child, including the states of mutexes, condition variables, and other pthreads objects; the use of pthread_atfork(3) may be helpful for dealing with problems that this can cause.

也就是说除了调用fork的线程外，其他线程在子进程中“蒸发”了。

这就是多线程中fork所带来的一切问题的根源所在了。<!--more-->

## 4\. 互斥锁

互斥锁，就是多线程fork大部分问题的关键部分。

在大多数操作系统上，为了性能的因素，锁基本上都是实现在用户态的而非内核态（因为在用户态实现最方便，基本上就是通过原子操作或者之前文章中提到的memory barrier实现的），所以调用fork的时候，会复制父进程的所有锁到子进程中。

问题就出在这了。从操作系统的角度上看，对于每一个锁都有它的持有者，即对它进行lock操作的线程。假设在fork之前，一个线程对某个锁进行的lock操作，即持有了该锁，然后另外一个线程调用了fork创建子进程。可是在子进程中持有那个锁的线程却"消失"了，从子进程的角度来看，这个锁被“永久”的上锁了，因为它的持有者“蒸发”了。

那么如果子进程中的任何一个线程对这个已经被持有的锁进行lock操作话，就会发生死锁。

当然了有人会说可以在fork之前，让准备调用fork的线程获取所有的锁，然后再在fork出的子进程的中释放每一个锁。先不说现实中的业务逻辑以及其他因素允不允许这样做，这种做法会带来一个问题，那就是隐含了一种上锁的先后顺序，如果次序和平时不同，就会发生死锁。

如果你说自己一定可以按正确的顺序上锁而不出错的话，还有一个隐含的问题是你所不能控制的，那就是库函数。

因为你不能确定你所用到的所有库函数都不会使用共享数据，即他们都是完全线程安全的。有相当一部分线程安全的库函数都是在内部通过持有互斥锁的方式来实现的，比如几乎所有程序都会用到的c/c++标准库函数malloc、printf等等。

比如一个多线程程序在fork之前难免会分配动态内存，这就必然会用到malloc函数；而在fork之后的子进程中也难免要分配动态内存，这也同样要用到malloc，可这却是不安全的，因为有可能malloc内部的锁已经在fork之前被某一个线程所持有了，而那个线程却在子进程中消失了。

## 5\. exec与文件描述符

按照上文的分析，似乎多线程中在fork出的子进程中立刻调用exec函数是唯一明智的选择了，其实即使这样做还是有一点不足。因为子进程会继承父进程中所有已打开的文件描述符，所以在执行exec之前子进程仍然可以读写父进程中的文件，但如果你不希望子进程能读写父进程里的某个已打开的文件该怎么办？

或许fcntl设置文件属性是一种办法：
```c
int fd = open("file", O_RDWR | O_CREAT);
if (fd < 0)
{
    perror("open");
}
fcntl(fd, F_SETFD, FD_CLOEXEC);
```
但是如果在open打开file文件之后，调用fcntl设置CLOEXEC属性之前有其他线程fork出了子进程了的话，这个子进程仍然是可以读写file文件。如果用锁的话，就又回到了上文所讨论的情况了。

从linux 2.6.23版本的内核开始，我们可以在open中设置O_CLOEXEC标志了，相当于“打开文件再设置CLOEXEC”成为了一个原子操作。这样在fork出的子进程执行exec之前就不能读写父进程中已打开的文件了。

## 6\. pthread_atfork

如果你不幸真的碰到了一个要解决多线程中fork的问题的时候，可以尝试使用pthread_atfork：
```c
int pthread_atfork(void (*prepare)(void), void (*parent)void(), void (*child)(void));
```

*   prepare处理函数由父进程在fork创建子进程前调用，这个函数的任务是获取父进程定义的所有锁。
*   parent处理函数是在fork创建了子进程以后，但在fork返回之前在父进程环境中调用的。它的任务是对prepare获取的所有锁解锁。
*   child处理函数在fork返回之前在子进程环境中调用，与parent处理函数一样，它也必须解锁所有prepare中所获取的锁。

因为子进程继承的是父进程的锁的拷贝，所有上述并不是解锁了两次，而是各自独自解锁。可以多次调用pthread_atfork函数从而设置多套fork处理程序，但是使用多个处理程序的时候。处理程序的调用顺序并不相同。parent和child是以它们注册时的顺序调用的，而prepare的调用顺序与注册顺序相反。这样可以允许多个模块注册它们自己的处理程序并且保持锁的层次（类似于多个RAII对象的构造析构层次）。

需要注意的是pthread_atfork只能清理锁，但不能清理条件变量。在有些系统的实现中条件变量不需要清理。但是在有的系统中，条件变量的实现中包含了锁，这种情况就需要清理。但是目前并没有清理条件变量的接口和方法。

## 7\. 结语

*   在多线程程序中最好只用fork来执行exec函数，不要对fork出的子进程进行其他任何操作。
*   如果确定要在多线程中通过fork出的子进程执行exec函数，那么在fork之前打开文件描述符时需要加上CLOEXEC标志。

## 8\. 参考文献

1.  Randal E.Bryant, David O'Hallaron. [深入理解计算机系统(第2版). 机械工业出版社, 2010](http://book.douban.com/subject/5333562/)
2.  W.Richard Stevens. [UNIX环境高级编程(第3版), 人民邮电出版社, 2014](http://book.douban.com/subject/25900403/)
3.  Linux Man Page. [fork(2)](http://linux.die.net/man/2/fork)
4.  Damian Pietras. [Threads and fork(): think twice before mixing them, 2009](http://www.linuxprogrammingblog.com/threads-and-fork-think-twice-before-using-them)
5.  云风. [极不和谐的 fork 多线程程序, 2011](http://blog.codingnow.com/2011/01/fork_multi_thread.html)
    （完）
