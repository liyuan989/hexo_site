---
title: 八皇后问题之全排列算法
id: 229
categories:
  - 数据结构与算法
date: 2015-03-06 22:52:35
tags:
---

在上一篇文章[《八皇后问题之回溯算法》](http://liyuanlife.com/blog/2015/03/05/backtracking-algorithm-for-eight-queen/)中介绍了如何使用「回溯算法」解决八皇后问题。关于八皇后问题，还有一种巧妙的解法，那就是本文将要介绍的「全排列」算法了。

## 全排列

学过高中数学的都知道，n个数字的全排列一共有n的阶乘个排列方法，即n!。例如数列{1, 2, 3, 4, 5}，一共有5×4×3×2×1=120个排列方法，那么对于计算机程序语言来说，如何实现这种全排列呢？

对于数列12345来说，可以看成是1和2345这两个数字的全排列，即12345和23451。而对于2345来说又可以看成是2和345这两个数字的全排列，即2345和3452，依次类推。当1和2345的排列组合算出后，再依次算2和1345，3和1245......

在计算机算法层面上来讲，这是一种递归的思想。我们把一组数分解成2部分，其中一部分只有1个数，另一部分有多个数。然后把这个有多个数的部分再分解，这样依次分解下去，当最终分解的2部分都是1个数的情况时就得到了全排列的一个解。然后原路返回至之前状态的下一种情况，再按这样的思想进行分解，最终得到所有全排列的解。整个过程，其实就是所谓的「递归」。

下面是全排列基本思想的实现，在实际算法中，每次分解2部分时，总是把当前数组的第一个数当做两部分中的其中一部分。这样当需要不同的数字充当只有一个数的那部分时，只需将它和数组的第一个数交换即可。当本次分解计算完成后，再交换回来，恢复原状态。
```c++
void print(int* data, int size)
{
    static int count = 0;
    printf("The %d solution:\n", ++count);
    for (int i = 0; i < size; ++i)
    {
        printf("%2d ", data[i]);
    }
    printf("\n");
}

void permutation(int* data, int size, int position)
{
    if (size == position)
    {
        print(data, size);
    }
    else if (size > position)
    {
        for (int i = position; i < size; ++i)
        {
            std::swap(data[position], data[i]);
            permutation(data, size, position + 1);
            std::swap(data[position], data[i]);
        }
    }
}
```
<!--more-->

## 思路

再回到八皇后问题上来。

首先，我们把棋盘看成一个8×8的坐标系，横坐标和纵坐标的范围均为0-7。由于八个皇后中的任意两个都不能处于同一行，那么可以得出每个皇后占据一行。于是我们定义一个column[8]数组来表示8个皇后在棋盘上的位置，其中数组的下标代表行数，每个元素的值代表列数，即column[0]=1代表一个皇后的位置是第0行、第1列。

然后，把column数组的每个元素用0-7中不同的值进行初始化，这代表着每个皇后不仅独占一行，而且还独占一列。这个时候我们对column数组进行全排列。

可是有人会说这样的话8!=40320，而八皇后只有92个解呀。其实我们忽略了每个皇后还独占两个方向上的斜线这一条件，我们只需检查这8!个排列中的每一种情况的8个皇后是否独占2条斜线即可。

## 实现

思路理清之后，编写代码就不是很困难了。
```c++
class EightQueen
{
public:
    EightQueen()
    {
        memset(column_, 0, sizeof(column_));
        for (int i = 0; i < 8; ++i)
        {
            column_[i] = i;
        }
    }

    void result()
    {
        permutation(column_, 8, 0);
    }

private:
    void permutation(int* data, int size, int position)
    {
        if (size == position)
        {
            if (check())
            {
                print();
            }
        }
        else if (size > position)
        {
            for (int i = position; i < size; ++i)
            {
                std::swap(data[position], data[i]);
                permutation(data, size, position + 1);
                std::swap(data[position], data[i]);
            }
        }
    }

    bool check()
    {
        for (int i = 0; i < 8; ++i)
        {
            for (int j = i + 1; j < 8; ++j)
            {
                if ((i -j == column_[i] - column_[j])
                    || (i + column_[i] == j + column_[j]))
                {
                    return false;
                }
            }
        }
        return true;
    }

    void print()
    {
        static int count = 0;
        printf("The %d solution:\n", ++count);
        for (int i = 0; i < 8; ++i)
        {
            for (int j = 0; j < 8; ++j)
            {
                if (column_[i] == j)
                {
                    printf("%2d ", 1);
                }
                else
                {
                    printf("%2d ", 0);
                }
            }
            printf("\n");
        }
        printf("\n");
    }

    int column_[8];
};

int main(int argc, char const *argv[])
{
    EightQueen eight_queen;
    eight_queen.result();
    return 0;
}
```

## 结语

其实我们可以仔细想想，无论是之前介绍的「回溯算法」还是本文讲到的「全排列」，本质上来讲都是[深度优先搜索](http://en.wikipedia.org/wiki/Depth-first_search)（DFS），因为2种方法都可以看做是在一个有向无环图上做DFS。

对于回溯算法，把棋盘看成一个从上到下的有向无环图，整个算法过程从第0行到第7行进行DFS，而对于每一行搜索确定列的时候又相当于把棋盘看成从左到右的有向无环图，即从第0列到第7列进行DFS。

对于全排列，比如12345，想象一个5×5的棋盘，其中每一列的5个数字都是12345中不同的一个数字，整个算法过程就是在棋盘上从左到右进行DFS。

## 参考文献

1.  Wikipedia. [Depth-first search](http://en.wikipedia.org/wiki/Depth-first_search)
    &nbsp;
