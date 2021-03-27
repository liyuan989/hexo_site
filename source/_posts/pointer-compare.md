---
title: 浅谈指针的比较
id: 21
categories:
  - C++
date: 2015-01-24 15:19:00
tags:
---

有人说指针是c语言的灵魂，也有人说没学好指针就等于不会c语言。

虽然在现代c++中一般都是推荐尽量避免使用原生的raw指针，而是以smart pointer 和reference替代之。但是无论怎样，对于c/c++来说，指针始终是个绕不过去的坎。究其原因，是因为c/c++都是支持面向底层操作的语言，而面向底层操作就得能操纵内存，这个时候就需要指针了。为什么呢？个人觉得指针实际上就是对机器语言/asm中的通过虚拟地址操作内存的这一行为的一种抽象。

例如

```
movl %eax, (%edx)
```

将寄存器eax中的值写入内存地址为寄存器edx的值的内存中。如果把edx看做一个指针的话，也就相当于

```
*p_edx = value_eax
```

## 1\. 指针的比较

关于指针，有着许多技巧和用途，后文主要谈谈关于c++中指针比较操作的一些容易踏入的坑。

先来看看这段代码
```c++
class BaseA
{
public:
    int a;
};

class BaseB
{
public:
    double b;
};

class Derived : public BaseA, public BaseB
{
};

int main(int argc, char const *argv[])
{
    Derived derivd;
    Derived* pd = &derivd;
    BaseB* pb = &derivd;
    printf("pb = %p\n", pb);
    printf("pd = %p\n", pd);
    if (pb == pd)
    {
        printf("pb == pd\n");
    }
    else
    {
        printf("pb != pd\n");
    }
}
```
输出的结果是：
```
pb = 0028FEE0
pd = 0028FED8
pb == pd
```
可以看到指针pb和pd值并不一样，但是编译器却认为他们相等，为什么呢？<!--more-->

1\.  当2个指针的静态类型以及所指对象的类型都属于同一个继承层次结构中，并且其中一个指针类型是所指对象的静态类型的时候，指针的比较，实际上比较的是两个指针是否指向同一个对象。

若2个指针指向同一个对象，被编译器决议为相等。编译器在比较的时候加上适当的offset值，例如上面的情况，相当于在比较的时候编译器做了这样的改动:
```c
if((pb - sizeof(int) == pd)
```
offset是由c++对象的内存模型决定的，具体的就不再本文讨论的范畴了。

若2个指针指向不同的对象，就被决议为不相等，并且比较的是指针保存的地址的值的大小。
```c++
int main(int argc, char const *argv[])
{
    Derived derived1;
    Derived derived2;
    Derived* pd = &derived1;
    BaseB* pb = &derived2;
    printf("%p\n", pd);
    printf("%p\n", pb);
    if (pd < pb)
    {
        printf("pd < pb\n");
    }
    else if (pd == pb)
    {
        printf("pd == pb\n");
    }
    else
    {
        printf("pd > pb\n");
    }
}
```
得到的结果为：
```
0028FED8
0028FED0
pd > pb
```
2\.  当2个指针的静态类型不属于同一个继承层次结构中，但是2个指针都指向同一个对象的时候，该比较是违法行为，编译器会报编译期错误
```c++
int main(int argc, char const *argv[])
{
    Derived derivd;
    Derived* pd = &derivd;
    int* pb = reinterpret_cast<int*>(&derivd);
    printf("pb = %p\n", pb);
    printf("pd = %p\n", pd);
    if (pb == pd)
    {
        printf("pb == pd\n");
    }
    else
    {
        printf("pb != pd\n");
    }
}
```
编译器报错为：

> error: comparison between distinct pointer types 'int*' and 'Derived*' lacks a cast [-fpermissive]>
> if (pb == pd)

3\.  当2个指针的静态类型以及所指对象类型都属于同一个继承层次结构，但是2个指针的静态类型都不是所指对象的类型时，该比较是违法行为，编译器会报编译期错误：

```c++
int main(int argc, char const *argv[])
{
    Derived derivd;
    BaseB* pb = &derivd;
    BaseA* pa = &derivd;
    printf("pb = %p\n", pb);
    printf("pd = %p\n", pa);
    if (pb == pa)
    {
        printf("pb == pa\n");
    }
    else
    {
        printf("pb != pa\n");
    }
}
```
编译器报错为：

> error: comparison between distinct pointer types 'BaseB*' and 'BaseA*' lacks a cast>
> if (pb == pa)

另外一些其他的行为，例如2个指针的类型不同但同属于一个继承层次，然后通过强制类型转换让他们俩都指向一个不属于该继承层次的对象，这样的行为都是为未定义行为，也许编译器不会报编译期错误，但结果是未定义的，可能是任何结果。

可能有人会说，什么时候指针比较的是他们所保存的地址的值呢呢？

答案是当2个指针的静态类型相同的时候：

```c++
int main(int argc, char const *argv[])
{
    Derived derived1;
    Derived derived2;
    Derived* p1 = &derived1;
    Derived* p2 = &derived2;
    if (p1 < p2)
    {
        printf("p1 < p2\n");
    }
    else if (p1 == p2)
    {
        printf("p1 == p2\n");
    }
    else
    {
        printf("p1 > p2\n");
    }
}
```
结果为：p1 > p2

## 2\. shared_ptr的owner_before

boost::shared_ptr/std::shared_ptr中有一个owner_before成员函数，原型为
```c++
template <class u> bool owner_before (const shared_ptr<u>& x) const;
template <class u> bool owner_before (const weak_ptr<u>& x) const;
```
当该shared_ptr和x的类型同属一个继承层次时，不管他们类型是否相同，他们两都被决议为“相等”。当他们的类型不属于同一继承层次时，比较的为他们所管理指针的地址值的大小。
```c++
int main(int argc, char const *argv[])
{
    boost::shared_ptr<Derived> pd(new Derived);
    boost::shared_ptr<BaseB> pb(pd);
    printf("%p %p\n", pd.get(), pb.get());
    printf("%d %d\n", pd < pb, pb < pd);  // 0 0
    printf("%d %d\n", pd.owner_before(pb), pb.owner_before(pd));  // 0 0
    boost::shared_ptr<void> p0(pd), p1(pb);
    printf("%p %p\n", p0.get(), p1.get());
    printf("%d %d\n", p0.get() < p1.get(), p1.get() < p0.get());  // 1 0
    printf("%d %d\n", p0.owner_before(p1), p1.owner_before(p0));  // 0 0
}
```
为什么shared_ptr会提供这样的成员函数呢？

因为一个智能指针有可能指向了另一个智能指针指向对象中的某一部分，但又要保证这两个智能指针销毁时，只对那个被指的对象完整地析构一次，而不是两个指针分别析构一次。

在这种情况下，指针就可以分为两种，一种是 stored pointer 它是指针本身的类型所表示的对象（可能是一个大对象中的一部分）；另一种是 owned pointer 指向内存中的实际完整对象（这一个对象可能被许多智能指针指向了它里面的不同部分，但最终只析构一次）。owner-based order 就是指后一种情况，如果内存中只有一个对象，然后被许多 shared pointer 指向了其中不同的部分，那么这些指针本身的地址肯定是不同的，也就是operator<()可以比较它们，并且它们都不是对象的 owner，它们销毁时不会析构对象。但它们都指向了一个对象，在owner-based order 意义下它们是相等的。

[cpluscplus](http://www.cplusplus.com/reference/memory/shared_ptr/owner_before/)中是这样解释的：
> returns whether the object is considered to go before _x_ following a strict weak _owner-based_ order.>
>
> unlike the [operator< overload](http://www.cplusplus.com/shared_ptr:operators), this ordering takes into consideration the [shared_ptr](http://www.cplusplus.com/shared_ptr)'s _owned pointer_, and not the [stored pointer](http://www.cplusplus.com/shared_ptr::get) in such a way that two of these objects are considered equivalent (i.e., this function returns <tt>false </tt>no matter the order of the operands) if they both share ownership, or they are both empty, even if their [stored pointer](http://www.cplusplus.com/shared_ptr::get) value are different.>
>
> the _stored pointer_ (i.e., the pointer the [shared_ptr](http://www.cplusplus.com/shared_ptr) object [dereferences](http://www.cplusplus.com/shared_ptr::operator*) to) may not be the _owned pointer_ (i.e., the pointer deleted on object destruction) if the [shared_ptr](http://www.cplusplus.com/shared_ptr) object is an alias ([alias-constructed](http://www.cplusplus.com/shared_ptr::shared_ptr) objects and their copies).>
>
> this function is called by [owner_less](http://www.cplusplus.com/owner_less) to determine its result.

[cppreference](http://en.cppreference.com/w/cpp/memory/shared_ptr/owner_before)中的解释：

> checks whether this shared_ptr precedes other in implementation defined owner-based (as opposed to value-based) order. the order is such that two smart pointers compare equivalent only if they are both empty or if they both own the same object, even if the values of the pointers obtained by get() are different (e.g. because they point at different subobjects within the same object)>
>
> this ordering is used to make shared and weak pointers usable as keys in associative containers, typically through <span class="t-lc">[std::owner_less](http://en.cppreference.com/w/cpp/memory/owner_less "cpp/memory/owner less").</span>

## 3\. 总结

*   指针之间的比较，要么指针的静态类型相同，要么指针的静态类型不同但他们的类型同属于同一继承层次且其中一个指针的静态类型为所指对象的类型。
*   指针的静态类型相同时，比较的是地址的值的大小。
*   指针的静态类型不同，但是他们的类型属于同一继承层次，并且其中一个指针的静态类型为所指对象的类型时，比较的是两指针是否指向同一对象。若是指向同一对象，则两指针“相等”；若不是指向同一对象，则比较指针的地址值的大小。
*   智能指针shared_ptr/weak_ptr的onwer_before成员函数描述的是：当比较的2个智能指针的类型属于同一继承层次时表现为“相等”的含义；当2个智能指针的类型不属于同一继承层次时，比较的是所管理指针的地址值的大小。

## 4\. 参考文献

1.  cplusplus. [std::shared_ptr::owner_before](http://www.cplusplus.com/reference/memory/shared_ptr/owner_before/)
2.  cppreference. [std::shared_ptr::owner_before](http://en.cppreference.com/w/cpp/memory/shared_ptr/owner_before)
    （完）