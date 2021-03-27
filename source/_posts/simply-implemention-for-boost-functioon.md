---
title: 'boost::function的简单实现'
id: 26
categories:
  - C++
date: 2014-12-17 12:35:00
tags:
---

boost::function和boost:bind是一对强大的利器。相信用过的童鞋多少有些体会。

虽然平时在用boost::function，但是用的时候心中总会一些不安，因为不知道它是怎么实现的。于是，就自己琢磨着简单的实现一下，搞明白基本的原理。

对于这个简单实现，有以下几个目标：

*   选取比较常见的接收2个参数的情况。
*   支持普通函数/函数指针、成员函数指针。
*   兼容函数对象、函数适配器/boost::bind。

## 1\. 实现

首先，定义一个基类：
```c++
template<typename R, typename T1, typename T2>
class base
{
public:
    virtual ~base()
    {
    }

    virtual R operator()(T1, T2) = 0;
};
```
然后再实现一个普通函数/函数指针的版本：
```c++
template<typename R, typename T1, typename T2>
class func : public base<R, T1, T2>
{
public:
    func(R (*ptr)(T1, T2))
        : ptr_(ptr)
    {
    }

    virtual R operator()(T1 a, T2 b)
    {
        return ptr_(a, b);
    }

private:
    R (*ptr_)(T1, T2);
};
```
<!--more-->

接着，实现支持成员函数指针的版本：
```c++
template<typename R, typename Class, typename T>
class member : public base<R, Class, T>
{
};

template<typename R, typename Class, typename T>
class member<R, Class*, T> : public base<R, Class*, T>
{
public:
    member(R (Class::*ptr)(T))
        : ptr_(ptr)
    {
    }

    virtual R operator()(Class* obj, T a)
    {
        return (obj->*ptr_)(a);
    }

private:
    R (Class::*ptr_)(T);
};
```
可能有的童鞋要问，为什么这里要有一个空的member类呢？这个问题放到下面解释。

自然的轮到最后一个种情况，函数对象/boost::bind类型的了：
```c++
template<typename T, typename R, typename T1, typename T2>
class functor : public base<R, T1, T2>
{
public:
    functor(const T& obj)
        : obj_(obj)
    {
    }

    virtual R operator()(T1 a, T2 b)
    {
        return obj_(a, b);
    }

private:
    T obj_;
};
```
最后，就是可用的function类了，实现如下：
```c++
template<typename T>
class function
{
};

template<typename R, typename T1, typename T2>
class function<R (T1, T2)>
{
public:
    template<typename Class, typename _R, typename _T2>
    function(_R (Class::*ptr)(_T2))
        : ptr_(new member<R, T1, T2>(ptr))
    {
    }

    template<typename _R, typename _T1, typename _T2>
    function(_R (*ptr)(_T1, _T2))
        : ptr_(new func<R, T1, T2>(ptr))
    {
    }

    template<typename T>
    function(const T& obj)
        : ptr_(new functor<T, R, T1, T2>(obj))
    {
    }

    ~function()
    {
        delete ptr_;
    }

    virtual R operator()(T1 a, T2 b)
    {
        return ptr_->operator()(a, b);
    }

private:
    base<R, T1, T2>* ptr_;
};
```
大家可能注意到了，和前面的member类一样，function也有一个空的类，那么这些有什么用呢？

这么做的原因，主要是利用模板偏特化来进行类型萃取，正常的function声明的时候，比如function<int (int, int)>而不是func<int, int, int>。所以用模板的偏特化的版本
```c++
template<typename R, typename T1, typename T2>
class function<R (T1, T2)>
```
就可以把int (int, int)萃取为R = int，T1 = int，T2 = int了。

同理，对于member类，由于一般我们将成员函数指针绑定到function的时候，比如int function(Type*, int)，其中Type是成员函数所属类。也就是说在function中的成员ptr_的类型是base<int, Type*, int>，那么在function的构造函数中构造的member类的类型就是member<int, Type*, int>，也就是class = Type*，但是我们需要的却是class = Type。所以这里得用偏特化萃取一下：
```c++
template<typename R, typename Class, typename T>
class member<R, Class*, T> : public base<R, Class*, T>
```
这样得到的class模板形参就会被编译器决议为Type，而不是Type*了。

另外提一下，在function的3种情况的构造函数是模板成员函数，而不是普通成员函数：
```c++
template<typename Class, typename _R, typename _T2>
function(_R (Class::*ptr)(_T2))
    : ptr_(new member<R, T1, T2>(ptr))
{
}

template<typename _R, typename _T1, typename _T2>
function(_R (*ptr)(_T1, _T2))
    : ptr_(new func<R, T1, T2>(ptr))
{
}

template<typename T>
function(const T& obj)
    : ptr_(new functor<T, R, T1, T2>(obj))
{
}
```
前2种情况，普通函数/函数指针对应的构造函数和成员函数指针对应的构造函数实现为成员模板，主要是为了兼容参数的隐式转换，例如声明一个function的类型为function<int (int, int)> foo，调用的时候却传入两个double类型，foo(1.1, 2.2)， double类型隐式转换成了int类型。这样也符合boost:function本来的兼容可转换的调用物这一特性。

而第3种情况的成员模板，是为了获取传入的函数对象/boost::bind的类型，以便在存储在functor的数据成员中，这也是为什么functor类的模板参数比其他版本多了一个的原因。

然后，我们来测试一下:
```c++
int get(int a, int b)
{
    std::cout << a+b << std::endl;
    return 0;
}

class Point
{
public:
    int get(int a)
    {
        std::cout << "Point::get called: a = "<< a << std::endl;
        return a;
    }
    int doit(int a, int b)
    {
        std::cout << "Point::doit called: a = "<< a+b << std::endl;
        return a+b;
    }
};

int main(int argc, char const *argv[])
{
    function<int (int, int)> foo(get);
    foo(10.1, 10.3);

    function<int (Point*, int)> bar(&Point::get);
    Point point;
    bar(&point, 30);

    function<int (int, int)> obj(boost::bind(&Point::doit, &point, _1, _2));
    obj(90, 100);
}
```
结果为：
```
20
Point::get called: a = 30
Point::doit called: a = 190
```
可以看到，输出的内容正是所期望的结果。

## 2\. 参考文献

1.  boost中文手册. [Improved Function Object Adapters 改良的函数对象适配器](http://cpp.ezbty.org//myfiles/boost/libs/functional/index.html)
    （完）