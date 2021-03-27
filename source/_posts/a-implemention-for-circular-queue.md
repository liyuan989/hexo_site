---
title: 循环队列的一种实现模型
id: 24
categories:
  - 数据结构与算法
date: 2014-12-30 16:41:00
tags:
---

前段时间在知乎上看到这样一个小题目：

用基本类型实现一队列，队列要求size是预先定义好的的。而且要求不可以使用语言自带的api，如c++的STL。普通的实现很简单，但是现在要求要尽可能的时间和空间复杂度的优化，要和语言自带的api比较时间和空间。这个队列还要支持如下的操作：

*   **constructor**:初始化队列
*   **enqueue**:入队
*   **dequeue**:出队

## 1\. 实现

队列是一种基本的数据结构，在平常的应用中十分广泛，多数情况队列都是用链表实现的。但是对于本题而言，用链表实现就有这样一个问题：由于每个结点都存在至少一个指向前一个结点或后一个结点的指针，这就带来了空间复杂度的加大，所以并不太适合要求。

这个时候我想到了boost中的boost::circular_buffer，它是通过类似于数组的底层结构实现的一个循环buffer。而数组的优点是空间复杂度够小(除去维持数据结构的索引项，空间复杂度为线性)，再实现成循环结构可以最大化的利用空间。而且在队列这样一种只在前后端插入删除的情况下，其push和pop的时间复杂度也只有O(1)。<!--more-->

基本实现如下：

```c++
#ifndef __CIRCULAR_QUEUE_H__
#define __CIRCULAR_QUEUE_H__

#include <stddef.h>

template<typename T>
class circular_queue
{
public:
    explicit circular_queue(size_t maxsize)
        : maxsize_(maxsize + 1),
          head_(0),
          rear_(0),
          array_(new T[maxsize_])
    {
    }

    circular_queue(size_t maxsize, const T& val)
        : maxsize_(maxsize + 1),
          head_(0),
          rear_(0),
          array_(new T[maxsize_])
    {
        for (size_t i = 0; i != maxsize; ++i)
        {
            array_[i] = val;
        }
        rear_ = maxsize;
    }

    circular_queue(const circular_queue& rhs)
        : maxsize_(rhs.maxsize_),
          head_(rhs.head_),
          rear_(rhs.rear_),
          array_(new T[maxsize_])
    {
        for (int i = 0; i != maxsize_; ++i)
        {
            array_[i] = rhs.array_[i];
        }
    }

    ~circular_queue()
    {
        delete [] array_;
    }

    circular_queue& operator=(const circular_queue& rhs)
    {
        if (this == &rhs)
        {
            return *this;
        }
        delete [] array_;
        maxsize_ = rhs.maxsize_;
        head_ = rhs.head_;
        rear_ = rhs.rear_;
        array_ = new T[maxsize_];
        for (int i = 0; i < maxsize_; ++i)
        {
            array_[i] = rhs.array_[i];
        }
        return *this;
    }

    bool empty() const
    {
        return head_ == rear_;
    }

    size_t size() const
    {
        return (rear_ - head_ + maxsize_) % maxsize_;
    }

    T& front()
    {
        return array_[head_];
    }

    const T& front() const
    {
        return array_[head_];
    }

    void push(const T& val)
    {
        if ((rear_ + 1) % maxsize_ != head_)
        {
            array_[rear_] = val;
            rear_ = (rear_ + 1) % maxsize_;
        }
    }

    void pop()
    {
        if (head_ != rear_)
        {
            head_ = (head_ + 1) % maxsize_;
        }
    }

private:
    size_t  maxsize_;
    int     head_;
    int     rear_;
    T*      array_;
};
```

*   队列长度 = 数组长度 - 1
*   预留了一个单位的数组元素空间作为队尾标记。

这个只是简陋的实现，没有考虑到一些情况，比如线程安全、STL算法，函数对象的兼容等。代码只是简单的测试了一下，如有错误欢迎指正:)

总的来说，这种思路的循环队列有以下优点：

*   使用固定的内存，不需要隐式或意外的内存分配。
*   从前端或后端进行快速的常量时间的插入和删除元素。
*   快速的常量时间的对元素进行随机访问。(如果需要的话可以定义operator[])
*   适用于实时和对性能有严格要求的应用程序。

还可以进一步扩展，当队列满的时候，从一端插入则覆盖冲洗掉另一端的数据，这样的一个模型可以应用于这些场合：

*   保存最近接收到的取样数据，在新的取样数据到达时覆盖最旧的数据。
*   一种用于保存特定数量的最后插入元素的快速缓冲。
*   高效的固定容量FIFO(先进先出)或LIFO(后进先出)队列，当队列满时删除最旧的(即最早插入的)元素。

## 2\. 参考文献

1.  boost中文手册. [circular_buffer<T, Alloc>](http://cpp.ezbty.org//myfiles/boost/libs/circular_buffer/doc/circular_buffer.html)
    （完）
