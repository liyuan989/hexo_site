---
title: C++中多线程与Singleton的那些事儿
id: 20
categories:
  - C++
date: 2015-01-31 15:06:00
tags:
---

## 前言

前段时间在网上看到了个的面试题，大概意思是如何在不使用锁和C++11的情况下，用C++实现线程安全的Singleton。

看到这个题目后，第一个想法就是用Scott Meyer在《Effective C++》中提到的，在static成员函数中构造local static变量的方法来实现，但是经过一番查找、思考，才明白这种实现在某些情况下是有问题的。本文主要将从最基本的单线程中的Singleton开始，慢慢讲述多线程与Singleton的那些事。

## 单线程

在单线程下，下面这个是常见的写法：
```c++
template<typename T>
class Singleton
{
public:
    static T& getInstance()
    {
        if (!value_)
        {
            value_ = new T();
        }
        return *value_;
    }

private:
    Singleton();
    ~Singleton();

    static T* value_;
};

template<typename T>
T* Singleton<T>::value_ = NULL;
```
在单线程中，这样的写法是可以正确使用的，但是在多线程中就不行了。

## 多线程加锁

在多线程的环境中，上面单线程的写法就会产生race condition从而产生多次初始化的情况。要想在多线程下工作，最容易想到的就是用锁来保护shared variable了。下面是伪代码：
```c++
template<typename T>
class Singleton
{
public:
    static T& getInstance()
    {
        {
            MutexGuard guard(mutex_)  // RAII
            if (!value_)
            {
                value_ = new T();
            }
        }
        return *value_;
    }

private:
    Singleton();
    ~Singleton();

    static T*     value_;
    static Mutex  mutex_;
};

template<typename T>
T* Singleton<T>::value_ = NULL;

template<typename T>
Mutex Singleton<T>::mutex_;
```
这样在多线程下就能正常工作了。这时候，可能有人会站出来说这种做法每次调用getInstance的时候都会进入临界区，在频繁调用getInstance的时候会比较影响性能。这个时候，为了解决这个问题，DCL写法就被聪明的先驱者发明了。

## DCL

DCL即double-checked locking。在普通加锁的写法中，每次调用getInstance都会进入临界区，这样在heavy contention的情况下该函数就会成为系统性能的瓶颈，这个时候就有先驱者们想到了DCL写法，也就是进行两次check，当第一次check为假时，才加锁进行第二次check：
```c++
template<typename T>
class Singleton
{
public:
    static T& getInstance()
    {
        if(!value_)
        {
            MutexGuard guard(mutex_);
            if (!value_)
            {
                value_ = new T();
            }
        }
        return *value_;
    }

private:
    Singleton();
    ~Singleton();

    static T*     value_;
    static Mutex  mutex_;
};

template<typename T>
T* Singleton<T>::value_ = NULL;

template<typename T>
Mutex Singleton<T>::mutex_;
```
是不是觉得这样就完美啦？其实在一段时间内，大家都以为这是正确的、有效的做法。实际上却不是这样的。幸运的是，后来有大牛们发现了DCL中的问题，避免了这样错误的写法在更多的程序代码中出现。<!--more-->

那么到底错在哪呢？我们先看看第12行value_ = new T这一句发生了什么：

1.  分配了一个T类型对象所需要的内存。
2.  在分配的内存处构造T类型的对象。
3.  把分配的内存的地址赋给指针value_

主观上，我们会觉得计算机在会按照1、2、3的步骤来执行代码，但是问题就出在这。实际上只能确定步骤1最先执行，而步骤2、3的执行顺序却是不一定的。假如某一个线程a在调用getInstance的时候第12行的语句按照1、3、2的步骤执行，那么当刚刚执行完步骤3的时候发生线程切换，计算机开始执行另外一个线程b。因为第一次check没有上锁保护，那么在线程b中调用getInstance的时候，不会在第一次check上等待，而是执行这一句，那么此时value_已经被赋值了，就会直接返回*value_然后执行后面使用t类型对象的语句，但是在a线程中步骤3还没有执行！也就是说在b线程中通过getInstance返回的对象还没有被构造就被拿去使用了！这样就会发生一些难以debug的灾难问题。

volatile关键字也不会影响执行顺序的不确定性。

在多核心机器的环境下，2个核心同时执行上面的a、b两个线程时，由于第一次check没有锁保护，依然会出现使用实际没有被构造的对象的情况。

关于dcl问题的详细讨论分析，可以参考Scott Meyer的paper：[《C++ and the Perils of Double-Checked Locking》](http://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf)

不过在新的C++11中，这个问题得到了解决。因为新的C++11规定了新的内存模型，保证了执行上述3个步骤的时候不会发生线程切换，相当这个初始化过程是「原子性」的的操作，DCL又可以正确使用了，不过在C++11下却有更简洁的多线程singleton写法了，这个留在后面再介绍。

关于新的C++11的内存模型，可以参考：[C++11中文版FAQ：内存模型](http://www.chenlq.net/books/cpp11-faq/cpp11-faq-chinese-version-series-memory-model.html)、[C++11FAQ：Memory Model](http://www.stroustrup.com/C++11FAQ.html#memory-model)、[C++ Data-Dependency Ordering: Atomics and Memory Model](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2556.html)

可能有人要问了，那么有什么办法可以在C++11之前的版本下，使得DCL正确工作呢？要使其正确执行的话，就得在步骤2、3直接加上一道memory barrier。强迫cpu执行的时候按照1、2、3的步骤来运行。(经网友@[shines77](http://www.cnblogs.com/shines77/)提醒，因没有锁的缘故这里需要用RCU技法，即read-copy-update)
```c++
static T& getInstance()
{
    if(!value_)
    {
        MutexGuard guard(mutex_);
        if (!value_)
        {
            T* p = static_cast<T*>(operator new(sizeof(T)));
            new (p) T();
            // insert some memory barier
            value_ = p;  // RCU method
        }
    }
    return *value_;
}
```
也许有人会说，你这已经把先前的value_ = new T()这一句拆成了下面这样的两条语句, 为什么还要在后面插入some memory barrier？
```c++
T* p = static_cast<T*>(operator new(sizeof(T)));
new (p) T();
```
原因是现代处理器都是以<span dir="auto">[Out-of-order execution](http://en.wikipedia.org/wiki/out-of-order_execution)（乱序执行）的方式来执行指令的。现代cpu基本都是多核心的，一个核包含多个执行单元。例如，一个现代的intel CPU 包含6个执行单元，可以做一组数学，条件逻辑和内存操作的组合。每个执行单元可以做这些任务的组合。这些执行单元并行地操作，允许指令并行地执行。如果从其它 CPU来观察，这引入了程序顺序的另一层不确定性。
</span>

<span dir="auto">如果站在单个CPU核心的角度上讲，它（一个CPU核心）看到的程序代码都是单线程的，所以它在内部以自己的「优化方式」乱序、并行的执行代码，然后保证最终的结果和按代码逻辑顺序执行的结果一致。但是如果我们编写的代码是多线程的，当不同线程访问、操作共享内存区域的时候，就会出现CPU实际执行的结果和代码逻辑所期望的结果不一致的情况。这是因为以单个CPU核心的视角来看代码是「单线程」的。</span>

<span dir="auto">所以为了解决这个问题，就需要memory barrier了，利用它来强迫CPU按代码的逻辑顺序执行。例如上面改动版本的getInstance代码中，因为第10行有memory barrier，所以CPU执行第9、10、11按「顺序」执行的。即使在CPU核心内是并行执行指令（比如一个单元执行第9行、一个单元执行第11行）的，但是他们在退役单元（retirement unit）更新执行结果到通用寄存器或者内存中时也是按照9、10、11顺序更新的。例如一个单元a先执行完了第11行，CPU让单元A等待直到执行第9行的单元B执行完成并在退役单元更新完结果以后再在退役单元更新A的结果。</span>

<span dir="auto">memory barreir是一种特殊的处理器指令，他指挥处理器做下面三件事：（参考文章[Mutex And Memory Visibility](http://www.domaigne.com/blog/computing/mutex-and-memory-visibility/)</span><span dir="auto">）</span>

*   <span dir="auto">刷新store buffer。</span>
*   等待直到memory barreir之前的操作已经完成。
*   不将memory barreir之后的操作移到memory barreir之前执行。

通过使用memory barreir，可以确保之前的乱序执行已经全部完成，并且未完成的写操作已全部刷新到主存。因此，数据一致性又重新回到其他线程的身边，从而保证正确内存的可见性。<span dir="auto">实际上，原子操作以及通过原子操作实现的模型（例如一些锁之类的），都是通过在底层加入memory barrier来实现的。</span>

至于如何加入memory barrier，在unix上可以通过内核提供的barrier()宏来实现。或者直接嵌入ASM汇编指令mfence也可以，barrier宏也是通过该指令实现的。

关于memory barreir<span dir="auto">可以参考文章[Memory Barriers/Fences](http://mechanical-sympathy.blogspot.com/2011/07/memory-barriersfences.html)。
</span>

## Meyers Singleton

Scott Meyer在《Effective C++》中提出了一种简洁的Singleton写法
```c++
template<typename T>
class Singleton
{
public:
    static T& getInstance()
    {
        static T value;
        return value;
    }

private:
    Singleton();
    ~Singleton();
};
```
先说结论：

*   单线程下，正确。
*   C++11及以后的版本（如C++14）的多线程下，正确。
*   C++11之前的多线程下，不一定正确。

原因在于在c++11之前的标准中并没有规定local static变量的内存模型，所以很多编译器在实现local static变量的时候仅仅是进行了一次check（参考《深入探索C++对象模型》），于是getInstance函数被编译器改写成这样了：

```c++
bool initialized = false;
char value[sizeof(T)];

T& getInstance()
{
    if (!initialized)
    {
       initialized = true;
       new (value) T();
    }
    return *(reinterpret_cast<T*>(value));
}
```
于是乎它就是不是线程安全的了。

但是在C++11却是线程安全的，这是因为新的C++标准规定了当一个线程正在初始化一个变量的时候，其他线程必须得等到该初始化完成以后才能访问它。

在[C++11 standard](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3337.pdf)中的§6.7 [stmt.dcl] p4：

> if control enters the declaration concurrently while the variable is being initialized, the concurrent execution shall wait for completion of the initialization.

在stackoverflow中的[Is Meyers implementation of Singleton pattern thread safe?](http://stackoverflow.com/questions/1661529/is-meyers-implementation-of-singleton-pattern-thread-safe)这个问题中也有讨论到。

不过有些编译器在c++11之前的版本就支持这种模型，例如g++，从g++4.0开始，Meyers Singleton就是线程安全的，不需要C++11。其他的编译器就需要具体的去查相关的官方手册了。

## Atomic Singleton

在C++11之前的版本下，除了通过锁实现线程安全的singleton外，还可以利用各个编译器内置的atomic operation来实现。（假设类atomic是封装的编译器提供的atomic operation）
```c++
template<typename T>
class Singleton
{
public:
    static T& getInstance()
    {
        while (true)
        {
            if (ready_.get())
            {
                return *value_;
            }
            else
            {
                if (initializing_.getAndSet(true))
                {
                    // another thread is initializing, waiting in circulation
                }
                else
                {
                    value_ = new T();
                    ready_.set(true);
                    return *value_;
                }
            }
        }
    }

private:
    Singleton();
    ~Singleton();

    static Atomic<bool>  ready_;
    static Atomic<bool>  initializing_;
    static T*            value_;
};

template<typename T>
Atomic<int> Singleton<T>::ready_(false);

template<typename T>
Atomic<int> Singleton<T>::initializing_(false);

template<typename T>
T* Singleton<T>::value_ = NULL;
```
肯定还有其他的写法，但是思路都差不多，需要区分三种状态：

*   对象已经构造完成
*   对象还没有构造完成，但是某一线程正在构造中
*   对象还没有构造完成，也没有任何线程正在构造中

## pthread_once

如果是在unix平台的话，除了使用atomic operation外，在不适用C++11的情况下，还可以通过pthread_once来实现singleton。

pthread_once的原型为

```c++
int pthread_once(pthread_once_t *once_control, void (*init_routine)(void))
```

apue中对于pthread_once是这样说的：

> 如果每个线程都调用pthread_once，系统就能保证初始化例程init_routine只被调用一次，即在系统首次调用pthread_once时。

所以，我们就可以这样来实现Singleton了
```c++
template<typename T>
class Singleton : Nocopyable
{
public:
    static T& getInstance()
    {
        threads::pthread_once(&once_control_, init);
        return *value_;
    }

private:
    static void init()
    {
        value_ = new T();
    }

    Singleton();
    ~Singleton();

    static pthread_once_t  once_control_;
    static T*              value_;
};

template<typename T>
pthread_once_t Singleton<T>::once_control_ = PTHREAD_ONCE_INIT;

template<typename T>
T* Singleton<T>::value_ = NULL;
```
如果需要正确的释放资源的话，可以在init函数里面使用glibc提供的atexit函数来注册相关的资源释放函数，从而达到了只在进程退出时才释放资源的这一目的。

## static object

现在再回头看看本文开头说的面试题的要求，不用锁和c++11，那么可以通过atomic operation来实现，但是有人会说atomic不是夸平台的，各个编译器的实现不一样。那么其实通过static object来实现也是可行的。
```c++
template<typename T>
class Singleton
{
public:
    static T& getInstance()
    {
        return *value_;
    }

private:
    Singleton();
    ~Singleton();

    class Helper
    {
    public:
        Helper()
        {
            Singleton<T>::value_ = new T();
        }

        ~Helper()
        {
            delete value_;
            value_ = NULL;
        }
    };

    friend class Helper;

    static T*      value_;
    static Helper  helper_;
};

template<typename T>
T* Singleton<T>::value_ = NULL;

template<typename T>
typename Singleton<T>::Helper Singleton<T>::helper_;
```
在进入main之前就把singleton对象构造出来就可以避免在进入main函数后的多线程环境中构造的各种情况了。这种写法有一个前提就是不能在main函数执行之前调用getInstance，因为c++标准只保证静态变量在main函数之前之前被构造完成。

可能有人会说如果helper的初始化先于value_初始化的话，那么helper_初始化的时候就会使用尚没有被初始化的value_，这个时候使用其返回的对象就会出现问题，或者在后面value_「真正」初始化的时候会覆盖掉helper_初始化时赋给value_的值。

实际上这种情况不会发生，value_的初始化一定先于helper_，因为C++标准保证了这一行为：

> the storage for objects with static storage duration (basic.stc.static) shall be zero-initialized (dcl.init) before any other initialization takes place. zero-initialization and initialization with a constant expression are collectively called static initialization; all other initialization is dynamic initialization. objects of pod types (basic.types) with static storage duration initialized with constant expressions (expr.const) shall be initialized before any dynamic initialization takes place. objects with static storage duration defined in namespace scope in the same translation unit and dynamically initialized shall be initialized in the order in which their definition appears in the translation unit.

stackoverflow中的一个问题也讨论了相关的行为，[When are static C++ class members initialized?](http://stackoverflow.com/questions/1421671/when-are-static-c-class-members-initialized)

## local static

上面一种写法只能在进入main函数后才能调用getInstance，那么有人说，我要在main函数之前调用怎么办？

嗯，办法还是有的。这个时候我们就可以利用local static来实现，C++标准保证函数内的local static变量在函数调用之前被初始化构造完成，利用这一特性就可以达到目的：
```c++
template<typename T>
class Singleton
{
private:
    Singleton();
    ~Singleton();

    class Creater
    {
    public:
        Creater()
            : value_(new T())
        {
        }

        ~Creater()
        {
            delete value_;
            value_ = NULL;
        }

        T& getValue()
        {
            return *value_;
        }

        T* value_;
    };

public:
    static T& getInstance()
    {
        static Creater creater;
        return creater.getValue();
    }

private:
    class Dummy
    {
    public:
        Dummy()
        {
            Singleton<T>::getInstance();
        }
    };

    static Dummy dummy_;
};

template<typename T>
typename Singleton<T>::Dummy Singleton<T>::dummy_;
```
这样就可以了。dummy_的作用是即使在main函数之前没有调用getInstance，它依然会作为最后一道屏障保证在进入main函数之前构造完成Singleton对象。这样就避免了在进入main函数后的多线程环境中初始化的各种问题了。

但是此种方法只能在main函数执行之前的环境是单线程的环境下才能正确工作。

实际上，上文所讲述了各种写法中，有一些不能在main函数之前调用。有一些可以在main函数之前调用，但是必须在进入main之前的环境是单线程的情况下才能正常工作。具体哪种写法是属于这两种情况就不一一分析了。总之，个人建议最好不要在进入main函数之前获取Singleton对象。因为上文中的各种方法都用到了staitc member，而c++标准只保证static member在进入main函数之前初始化，但是不同编译单元之间的static member的初始化顺序却是未定义的， 所以如果在main之前就调用getInstance的话，就有可能出现实现Singleton的static member还没有初始化就被使用的情况。

如果万一要在main之前获取Singleton对象，并且进入main之前的环境是多线程环境，这种情形下，还能保证正常工作的写法只有C++ 11下的Meyers Singleton，或者如g++ 4.0及其后续版本这样的编译器提前支持内存模型情况下的C++ 03也是可以的。

## 参考文献

1.  Scott Meyers. [Effective C++:55 Specific Ways to Improve Your Programs and Designs,3rd Edition. 电子工业出版社, 2011](http://book.douban.com/subject/5387403/)
2.  Stanley B. Lippman. [深度探索C++对象模型. 电子工业出版社, 2012](http://book.douban.com/subject/10427315/)
3.  Scott Meyers. [C++ and the Perils of Double-Checked Locking. 2004](http://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf)
4.  陈良乔(译). [C++11 FAQ中文版](http://www.chenlq.net/cpp11-faq-chs)
5.  Bjarne Stroustrup. [C++11 FAQ](http://www.stroustrup.com/C++11FAQ.html)
6.  Paul E. McKenney, Hans-J. Boehm, Lawrence Crowl. [C++ Data-Dependency Ordering: Atomics and Memory Model. 2008](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2556.html)
7.  Wikipedia. [Out-of-order execution](http://en.wikipedia.org/wiki/Out-of-order_execution)
8.  Loïc. [Mutex And Memory Visibility, 2009](http://www.domaigne.com/blog/computing/mutex-and-memory-visibility/)
9.  Randal E.Bryant, David O'Hallaron. [深入理解计算机系统(第2版). 机械工业出版社, 2010](http://book.douban.com/subject/5333562/)
10.  Martin Thompson. [Memory Barriers/Fences, 2011](http://mechanical-sympathy.blogspot.com/2011/07/memory-barriersfences.html)
11.  [Working Draft, Standard For Programing Language C++. 2012](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3337.pdf)
12.  W.Richard Stevens. [UNIX环境高级编程(第3版), 人民邮电出版社, 2014](http://book.douban.com/subject/25900403/)
13.  stackoverflow. [Is Meyers implementation of Singleton pattern thread safe](http://stackoverflow.com/questions/1661529/is-meyers-implementation-of-singleton-pattern-thread-safe)
14.  stackoverflow. [When are static C++ class members initialized](http://stackoverflow.com/questions/1421671/when-are-static-c-class-members-initialized)
     （完）