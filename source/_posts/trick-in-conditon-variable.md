---
title: 条件变量的陷阱与思考
id: 22
categories:
  - 多线程
date: 2015-01-21 14:10:00
tags:
---

## 1\. 前言

在多线程编程中，互斥锁与条件变量恐怕是最常用也是最实用的线程同步原语。

关于条件变量一共也就pthread_cond_init、pthread_cond_destroy、pthread_cond_wait、pthread_cond_timedwait、pthread_cond_signal、pthread_cond_broadcast这么几个函数，但是在实际使用中却是很容易用错，后文将来分析几种常见使用情况的正确性。

## 2\. 分析

下面是一个辅助基类、便于减少篇幅（为了简单起见，后文中的所有函数调用并未检查返回的错误情况）：
```c++
class ConditionBase
{
public:
    ConditionBase()
    {
        pthread_mutex_init(&mutex_, NULL);
        pthread_cond_init(&cond_, NULL);
    }

    ~ConditionBase()
    {
        pthread_mutex_destroy(&mutex_);
        pthread_cond_destroy(&cond_);
    }

private:
    pthread_mutex_t  mutex_;
    pthread_cond_t   cond_;
};
```

### 2.1.1 版本一

```c++
class Condition1 : public ConditionBase
{
public:
    void wait()
    {
        pthread_mutex_lock(&mutex_);
        pthread_cond_wait(&cond_, &mutex_);
        pthread_mutex_unlock(&mutex_);
    }

    void wakeup()
    {
        pthread_cond_signal(&cond_);
    }
};
```
错误，这种情况有可能丢失事件。当signal发生在wait之前，就会丢失这次signal事件。如下图

![](/images/blog-condition_variable_1.png)
<!--more-->

### 2.1.2 版本二

```c++
class Condition2 : public ConditionBase
{
public:
    void wait()
    {
        pthread_mutex_lock(&mutex_);
        pthread_cond_wait(&cond_, &mutex_);
        pthread_mutex_unlock(&mutex_);
    }

    void wakeup()
    {
        pthread_mutex_lock(&mutex_);
        pthread_cond_signal(&cond_);
        pthread_mutex_unlock(&mutex_);
    }
};
```
错误，同情况一一样有可能丢失事件。当signal事件发生在wait之前就会丢失signal事件。如下图

![](/images/blog-condition_variable_2.png)

### 2.1.3 版本**三**

```c++
class Condition3 : public ConditionBase
{
public:
    Condition3()
        : signal_(false)
    {
    }

    void wait()
    {
        pthread_mutex_lock(&mutex_);
        if (!signal_)
        {
            pthread_cond_wait(&cond_, &mutex_);
        }
        signal_ = false;
        pthread_mutex_unlock(&mutex_);
    }

    void wakeup()
    {
        pthread_mutex_lock(&mutex_);
        signal_ = true;
        pthread_cond_signal(&cond_);
        pthread_mutex_unlock(&mutex_);
    }

private:
    bool signal_;
};
```
错误。引入了bool变量来检查状态，但是遇到spurious wakeup仍然会发生错误。

什么是spurious wakeup？wikipedia中是这样说的：

> spurious wakeup describes a complication in the use of condition variables as provided by certain multithreading apis such as posix threads and the windows api. even after a condition variable appears to have been signaled from a waiting thread's point of view, the condition that was awaited may still be false. one of the reasons for this is a spurious wakeup; that is, a thread might be awoken from its waiting state even though no thread signaled the condition variable.

也就是说一次signal调用唤醒了2个或者2个以上的waiting中的线程，这种现象就是spurious wakeup，虚假唤醒。

apue上这样说：

> posix规范为了简化实现，允许pthread_cond_signal在实现的时候可以唤醒不止一个线程。

在发生的spurious wakeup时候，waiting线程被意外的唤醒，然后到真正signal的时候，waiting线程在之前已经spurious wakeup唤醒了，这样就会造成不易debug的错误。如下图

![](/images/blog-condition_variable_3.png)

### 2.1.4 版本四

```c++
class Condition4 : public ConditionBase
{
public:
    Condition4()
        : signal_(false)
    {
    }

    void wait()
    {
        pthread_mutex_lock(&mutex_);
        while (!signal_)
        {
            pthread_cond_wait(&cond_, &mutex_);
        }
        signal_ = false;
        pthread_mutex_unlock(&mutex_);
    }

    void wakeup()
    {
        pthread_mutex_lock(&mutex_);
        signal_ = true;
        pthread_cond_signal(&cond_);
        pthread_mutex_unlock(&mutex_);
    }

private:
    bool signal_;
};
```
正确。这个是推荐用法，apue, unp，man手册中都是这种用法，在wait上用while循环而不是if就可以正确处理spurious wakeup情况了。当发生spurious wakeup时，wait被意料之外的唤醒，但是循环条件并没有改变，于是循环继续执行pthread_cond_wait，然后继续进入wait，等待被正确的唤醒。

### 2.1.5 版本五

```c++
class Condition5 : public ConditionBase
{
public:
    Condition5()
        : signal_(false)
    {
    }

    void wait()
    {
        pthread_mutex_lock(&mutex_);
        while (!signal_)
        {
            pthread_cond_wait(&cond_, &mutex_);
        }
        signal_ = false;
        pthread_mutex_unlock(&mutex_);
    }

    void wakeup()
    {
        pthread_mutex_lock(&mutex_);
        signal_ = true;
        pthread_mutex_unlock(&mutex_);
        pthread_cond_signal(&cond_);
    }

private:
    bool signal_;
};
```
正确。版本五与版本四的唯一区别就是在唤醒的时候先解锁再调用signal发起唤醒。这么做不会有错误，但是可能有较大几率会使线程调度不在最理想状态，例如在wakeup调用中的解锁以后，调用signal以前，系统调度发生线程切换，使得signal没有在第一时间被发出。

这是在stackoverflow的一个帖子中的说法：[http://stackoverflow.com/questions/4544234/calling-pthread-cond-signal-without-locking-mutex](http://stackoverflow.com/questions/4544234/calling-pthread-cond-signal-without-locking-mutex)
> Note that you can actually move the pthread_cond_signal() itself after the pthread_mutex_unlock(), but this can result in less optimal scheduling of threads, and you've necessarily locked the mutex already in this code path due to changing the condition itself.

### 2.1.6 版本六

```c++
class Condition6 : public ConditionBase
{
public:
    Condition6()
        : signal_(false)
    {
    }

    void wait()
    {
        pthread_mutex_lock(&mutex_);
        while (!signal_)
        {
            pthread_cond_wait(&cond_, &mutex_);
        }
        signal_ = false;
        pthread_mutex_unlock(&mutex_);
    }

    void wakeup()
    {
        pthread_mutex_lock(&mutex_);
        pthread_cond_signal(&cond_);
        signal_ = true;
        pthread_mutex_unlock(&mutex_);
    }

private:
    bool signal_;
};
```
正确，版本六与版本四的区别状态的改变和发起signal唤醒信号的顺序互换了，由于整个wakeup过程都在mutex的包含之下，所以并没有影响。但是个人更推荐版本四，因为更符合逻辑，不然apue和unp也不会都用版本四的写法顺序了:)

### 2.1.7 版本七

```c++
class Condition7 : public ConditionBase
{
public:
    Condition7()
        : signal_(false)
    {
    }

    void wait()
    {
        pthread_mutex_lock(&mutex_);
        while (!signal_)
        {
            pthread_cond_wait(&cond_, &mutex_);
        }
        signal_ = false;
        pthread_mutex_unlock(&mutex_);
    }

    void wakeup()
    {
        pthread_mutex_lock(&mutex_);
        signal_ = true;
        pthread_cond_broadcast(&cond_);
        pthread_mutex_unlock(&mutex_);
    }

private:
    bool signal_;
};
```
正确。与版本四的区别是这里用了broadcast，而不是signal。对于这种只有一个wait线程的时候，是没有问题的。但是当有多个wait线程的时候，使用broadcast就把所有wait线程都唤醒了。

另外，当我们使用条件变量cond实现事件等待器的时候，就要用broadcast而不是signal了，因为当有多个事件挂起在wait调用上等待时，signal只能唤醒其中的一个等待线程，并且我们不能期待它唤醒具体的某一个线程，因为这个是不可控的。

### 2.1.8 版本八

```c++
class Condition8 : public ConditionBase
{
public:
    Condition8()
        : signal_(false)
    {
    }

    void wait()
    {
        pthread_mutex_lock(&mutex_);
        while (!signal_)
        {
            pthread_cond_wait(&cond_, &mutex_);
        }
        signal_ = false;
        pthread_mutex_unlock(&mutex_);
    }

    void wakeup()
    {
        signal_ = true;
        pthread_cond_signal(&cond_);
    }

private:
    bool signal_;
};
```
错误。存在data race，从而导致有可能丢失事件。当wakeup调用发生在wait调用中的进入while循环之后，调用pthread_cond_wait之前，就会丢失signal事件。如下图

![](/images/blog-condition_variable_8.png)

另外，在wait调用中，必须用一个mutex同时保护条件状态和cond的pthread_cond_wait的调用，而不能用2个mutex，一个保护条件状态，一个保护pthread_cond_wait，pthread_cond_signal的调用。

这样仍然会出现race condition。比如先发生wait调用，保护条件的mutex加锁、检查条件、解锁，然后切换线程调用wakeup发送signal信号，再切回wait线程，保护cond的mutex加锁、进入pthread_cond_wait的waiting状态中，从而丢失了之前的signal事件。

## 3\. 总结

使用条件变量，调用signal/broadcast的时候，无法知道是否已经有线程等在wait上了。因此，一般要先改变条件状态，然后再发送signal/broadcast信号。然后在wait调用线程上先检查条件状态，只有当条件状态为假的时候才进入pthread_cond_wait进行等待，从而防止丢失signal/broadcast事件。并且检查条件、pthread_cond_wait，修改条件、signal/broadcast都要在同一个mutex的保护下进行。

## 4\. 参考文献

1.  W.Richard Stevens. [UNIX环境高级编程(第3版), 人民邮电出版社, 2014](http://book.douban.com/subject/25900403/)
2.  Wikipedia. [Spurious_wakeup](http://en.wikipedia.org/wiki/spurious_wakeup)
3.  stackoverflow. [Calling pthread_cond_signal without locking mutex](http://stackoverflow.com/questions/4544234/calling-pthread-cond-signal-without-locking-mutex)
    （完）
