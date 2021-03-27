---
title: C++的SFINAE
id: 258
categories:
  - C++
date: 2015-05-02 23:45:23
tags:
---

> Substitution Failure Is Not An Error

SFINAE，即匹配失败不是错误。它是C++中一种很有用的规则。在决议重载的模板函数或者特化/偏特化类的时候，如果对一个较为「特化」版的候选者的模板形参推断（匹配）失败，将转而对其他次「特化」的候选者进行模板参数推断（匹配），而不是返回一个错误。

先看一个简单的例子：

``` c++
struct Test
{
    typedef int foo;
};

template<typename T>
void bar(typename T::foo) // #1
{
}

template<typename T>
void bar(T) // #2
{
}

int main(int argc, char* argv[])
{
    bar<Test>(0); // call #1
    bar<int>(0);  // call #2, not error.
}
```

第一个函数调用，显示实例化形参为Test类型，而Test有一个类型foo，所以匹配#1版本成功。

第二个函数调用，因为编译器总是最先尝试推断（匹配）「最特化」的候选函数，所以先对#1版本进行匹配，而int是一个内置类型，并没有子类型foo，所以#1版本匹配失败，然后转而对#2版本的候选函数进行匹配，即T实例化为int类型，匹配成功。

利用SFINAE这种特性，就可以在编译期做很多事情了。<!--more-->

比如下面这种方法可以判断某个类型是否含有名字为destroy的public成员函数。

```c++
template<typename T>
struct has_destroy
{
    template<typename U>
    static char check(typeof(&U::destroy));

    template<typename U>
    static int check(...);

    static const bool value = (sizeof(check<T>(0)) == 1);
};

template<typename T>
const bool has_destroy<T>::value;

class Foo
{
public:
    void destroy()
    {
    }
};

int main(int argc, char* argv[])
{
    printf("%d\n", static_cast<int>(has_destroy<Foo>::value));  // 1
    printf("%d\n", static_cast<int>(has_destroy<int>::value));  // 0
}
```
当T有名为destroy的public成员函数时，初始化value的时候会匹配第一个check，而当T为int的时候会匹配第二个check。

我们知道sizeof的值是在编译期的时候就会被计算出来的，但是在编译期又不可能执行函数调用，所以当check匹配成功后，就直接用所成功匹配的check的返回类型作为sizeof的参数进行计算。

还可以利用SFINAE来判断一个类型是否是某个类型，比如可以用它来判断某个类型是否是指针：

```c++
template<typename T>
struct is_pointer
{
    template<typename U>
    static char check(U*);

    template<typename R, typename U>
    static char check(R U::*);

    static int check(...);

    static T obj;
    static const bool value = (sizeof(check(obj)) == 1);
};

template<typename T>
T is_pointer<T>::obj;

template<typename T>
const bool is_pointer<T>::value;

int main(int argc, char* argv[])
{
    printf("%d\n", static_cast<int>(is_pointer<int*>::value)); // 1
    printf("%d\n", static_cast<int>(is_pointer<int>::value));  // 0

    typedef int (*callback)(int, int);
    printf("%d\n", static_cast<int>(is_pointer<callback>::value)); // 1
}
```

*   当T是普通指针或者普通函数指针的时候匹配第一个check。
*   当T是数据成员指针或者成员函数指针的时候匹配第二个check。
    当然了，这种方法有一定的局限性，如果T是一个纯虚基类或者T的构造函数不是public，抑或是没有默认构造函数，那么就无法正常工作了。

[另一个比较好的方法](http://stackoverflow.com/questions/3177686/how-to-implement-is-pointer)是用利用类特化/偏特化时候的SFINAE来实现。
