---
title: 八皇后问题之回溯算法
id: 227
categories:
  - 数据结构与算法
date: 2015-03-05 21:48:35
tags:
---

八皇后问题是一个以[国际象棋](http://zh.wikipedia.org/wiki/%E5%9B%BD%E9%99%85%E8%B1%A1%E6%A3%8B)为背景的问题：如何能在8×8的国际象棋棋盘上放置八个皇后，使得任何一个皇后都无法直接吃掉其他的皇后？为了达到此目的，任何两个皇后都不能处于同一条横线、纵线或斜线上。其中一种解法如下图所示。

![](/images/blog-eight_queen_chessborad.png)

## 思路

使用「[回溯算法](http://zh.wikipedia.org/wiki/%E5%9B%9E%E6%BA%AF%E6%B3%95)」求解八皇后问题是一种常见的思路。如上图中，从上到下依次在每一行放置一个皇后并检验是否满足要求，若某一行的每一列均不能符合要求，则回溯至有其他可放置列的行中进行放置。然后继续向下搜索，直至搜索到最后一行，即找到一个可行的解。

具体思路如下：

1.  初始化棋盘，当前行为第一行，当前列为第一列。
2.  在当前行，当前列的位置上检查是否满足条件（经过这一点的行、列和斜线上均没有其他皇后）。若满足条件则进行步骤3，否则跳到步骤4。
3.  在满足条件的当前位置放置一个皇后：

    *   若当前行是最后一行，则输出棋盘，记录一个解。
    *   若当前行不是最后一行，则将当前行设为下一行，当前列设为下一行的第一列，返回步骤2。

4.  此时，当前位置不满足条件：

    *   若当前列不是最后一列，则当前列设为下一列，返回步骤2。
    *   若当前列是最后一列，则进行回溯，即设当前行为上一行，当前列设为上一行中尚未经过的下一列，返回步骤2。
        <!--more-->

## 实现

先来看一种比较直观的写法，以二维数组来模拟棋盘。
```c++
#include <string.h>
#include <stdio.h>

class EightQueen
{
public:
    EightQueen()
    {
        memset(chessboard_, 0, sizeof(chessboard_));
    }

    void result()
    {
        resolve(0);
    }

private:
    void resolve(int row)
    {
        if (row == 8)
        {
            print();
        }
        else
        {
            for (int j = 0; j < 8; ++j)
            {
                chessboard_[row][j] = 1;
                if (check(row, j))
                {
                    resolve(row + 1);
                }
                chessboard_[row][j] = 0;
            }
        }
    }

    bool check(int row, int column)
    {
        if (row == 0)
        {
            return true;
        }
        for (int i = 0; i < row; ++i)
        {
            if (chessboard_[i][column] == 1)
            {
                return false;
            }
        }
        for (int i = row - 1, j = column - 1; i >= 0 &amp;&amp; j >= 0; --i, --j)
        {
            if (chessboard_[i][j] == 1)
            {
                return false;
            }
        }
        for (int i = row - 1, j = column + 1; i >= 0 &amp;&amp; j < 8; --i, ++j)
        {
            if (chessboard_[i][j] == 1)
            {
                return false;
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
                printf("%2d ", chessboard_[i][j]);
            }
            printf("\n");
        }
        printf("\n");
    }

    int chessboard_[8][8];
};

int main(int argc, char const *argv[])
{
    EightQueen obj;
    obj.result();
    return 0;
}
```
其实上面的写法比较冗余，因为从上到下每次放置一个皇后的时候就已经决定了每个皇后必然独占一行，因此可以用一维数组column[8]来模拟皇后在棋盘上的位置，即数组下标代表行，数组的元素值代表列。这样在check的时候也就用不着检查了是否有皇后在同一行了。
```c++
class EightQueen
{
public:
    void result()
    {
        resolve(0);
    }

private:
    void resolve(int row)
    {
        if (row == 8)
        {
            print();
        }
        else
        {
            for (int i = 0; i < 8; ++i)
            {
                column_[row] = i;
                if (check(row, column_[row]))
                {
                    resolve(row + 1);
                }
            }
        }
    }

    bool check(int row, int column)
    {
        for (int i = 0; i < row; ++i)
        {
            if (column_[i] == column_[row]
                || i - row == column_[i] - column_[row]
                || i + column_[i] == row + column_[row])
            {
                return false;
            }
        }
        return true;
    }

    int column_[8];
};
```

## 结语

「回溯算法」是一种较为容易想到的解决八皇后问题的方法，还有其他一些不同的思路和方法。八皇后问题是一个经典的问题，可以推广为更一般的n皇后摆放问题：这时棋盘的大小变为n×n，而皇后个数也变成n。当且仅当n = 1或n ≥ 4时问题有解。

## 参考文献

1.  Wikipedia. [回溯法](http://zh.wikipedia.org/wiki/%E5%9B%9E%E6%BA%AF%E6%B3%95)
2.  Wikipedia. [国际象棋](http://zh.wikipedia.org/wiki/%E5%9B%BD%E9%99%85%E8%B1%A1%E6%A3%8B)
