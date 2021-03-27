---
title: 凯撒密码
id: 231
categories:
  - 数据结构与算法
date: 2015-03-08 00:30:45
tags:
---

在密码学中，「凯撒密码」是一种最简单且最广为人知的加密技术。它是一种替换加密技术，明文中所有字母都在字母表上向后或向前按照一个固定数目进行偏移后被替换成密文。例如当偏移量为3时，所有的A被替换为D，B被替换为E。这个加密方法是以凯撒的名字命名的，因为当年凯撒用此方法与将军们进行联络。

## 方法

凯撒密码的替换方法是通过排列明文和密文字母表，密文字母通过将明文字母向左或向右移动一个固定数目的位置。例如当偏移量是左移3时（解密是密钥就是3）：
```
明文：ABCDEFGHIJK
密文：DEFGHIJKLMN
```
使用时，加密者查找明文字母表中需要加密的消息中每一个字母所在的位置，并写下密文字母表中对应的字母。解密者则根据事先已知的密钥（偏移量和偏移方向）反过来操作，得到传递的实质信息。例如：
```
明文：HELLO WORLD
密文：EBIIL TLOIA
```
凯撒密码的加密解密方法还可以用数学的求余来计算。用数字0-25代表26个英文字母，这样密钥偏移量的加密方法即为：
```
f(x) = (x + n) mod 26
```
解密方法为：
```
g(x) = (x - n) mod 26
```
<!--more-->

## 实例

前段时间在知乎上看到有个女生提了一个关于破解密码的小问题

> 可以帮忙破解一个密码吗？正文为:
>
> VRPHWLPHV L ZDQW WR FKDW ZLWK BRX, EXW L KDYH QR UHDVRQ WR FKDW ZLWK BRX

可以看到这段密文每隔几个字母就有个空格，并且还有一个标点符号，那么可以推断为使用凯撒加密的方法进行偏移加密过（若是其他复杂的加密方法一般不会出现这样规则的字母序列，而是各种乱码符号），并且空格和标点可能都没有加密。

那么如何破解这段密码呢？

「[穷举法](http://en.wikipedia.org/wiki/Method_of_exhaustion)」是常见破解凯撒密码的方法，即穷举所有可能的偏移量（密钥）并翻译输出明文，然后在这输出的所有明文中人工选取出一个或多个有意义的明文集合，并从这些有意义的明文集合中分析、识别出最终的结果。

因为计算机语言都是用ASCII码表示字符，即字符A-Z是用数值65-90来表示的，并且在穷举偏移量进行计算的时候，偏移量选取的是非负整数0-25，所以在计算时若密文字符ASCII码值加上密钥ASCII码值（偏移量）之和大于90，就要减去26，不然该ASCII码值就不能表示一个字符了，加密者在加密的时候基本上也是这样考虑的。

下面是我当时在这个问题下写解密代码：
```c++
#include <string>
#include <iostream>

int main(int argc, char const *argv[])
{
    std::string cipher = "VRPHWLPHV L ZDQW WR FKDW ZLWK BRX, "
                         "EXW L KDYH QR UHDVRQ WR FKDW ZLWK BRX";
    std::string confession(cipher.size(), ' ');
    for (int i = 0; i <= 25; ++i)
    {
        for (size_t j = 0; j < cipher.size(); ++j)
        {
            if (cipher[j] != ' ' && cipher[j] != ',')
            {
                if (cipher[j] + i >= 'A' && cipher[j] + i <= 'Z')
                {
                    confession[j] = static_cast<char>(cipher[j] + i);
                }
                else
                {
                    confession[j] = static_cast<char>(cipher[j] + i - 26);
                }
            }
            else
            {
                confession[j] = cipher[j];
            }
        }
        std::cout << confession << std::endl;
    }
}
```
这段代码的运行结果为：
```
XTRJYNRJX N BFSY YT HMFY BNYM DTZ, GZY N MFAJ ST WJFXTS YT HMFY BNYM DTZ
YUSKZOSKY O CGTZ ZU INGZ COZN EUA, HAZ O NGBK TU XKGYUT ZU INGZ COZN EUA
ZVTLAPTLZ P DHUA AV JOHA DPAO FVB, IBA P OHCL UV YLHZVU AV JOHA DPAO FVB
AWUMBQUMA Q EIVB BW KPIB EQBP GWC, JCB Q PIDM VW ZMIAWV BW KPIB EQBP GWC
BXVNCRVNB R FJWC CX LQJC FRCQ HXD, KDC R QJEN WX ANJBXW CX LQJC FRCQ HXD
CYWODSWOC S GKXD DY MRKD GSDR IYE, LED S RKFO XY BOKCYX DY MRKD GSDR IYE
DZXPETXPD T HLYE EZ NSLE HTES JZF, MFE T SLGP YZ CPLDZY EZ NSLE HTES JZF
EAYQFUYQE U IMZF FA OTMF IUFT KAG, NGF U TMHQ ZA DQMEAZ FA OTMF IUFT KAG
FBZRGVZRF V JNAG GB PUNG JVGU LBH, OHG V UNIR AB ERNFBA GB PUNG JVGU LBH
GCASHWASG W KOBH HC QVOH KWHV MCI, PIH W VOJS BC FSOGCB HC QVOH KWHV MCI
HDBTIXBTH X LPCI ID RWPI LXIW NDJ, QJI X WPKT CD GTPHDC ID RWPI LXIW NDJ
IECUJYCUI Y MQDJ JE SXQJ MYJX OEK, RKJ Y XQLU DE HUQIED JE SXQJ MYJX OEK
JFDVKZDVJ Z NREK KF TYRK NZKY PFL, SLK Z YRMV EF IVRJFE KF TYRK NZKY PFL
KGEWLAEWK A OSFL LG UZSL OALZ QGM, TML A ZSNW FG JWSKGF LG UZSL OALZ QGM
LHFXMBFXL B PTGM MH VATM PBMA RHN, UNM B ATOX GH KXTLHG MH VATM PBMA RHN
MIGYNCGYM C QUHN NI WBUN QCNB SIO, VON C BUPY HI LYUMIH NI WBUN QCNB SIO
NJHZODHZN D RVIO OJ XCVO RDOC TJP, WPO D CVQZ IJ MZVNJI OJ XCVO RDOC TJP
OKIAPEIAO E SWJP PK YDWP SEPD UKQ, XQP E DWRA JK NAWOKJ PK YDWP SEPD UKQ
PLJBQFJBP F TXKQ QL ZEXQ TFQE VLR, YRQ F EXSB KL OBXPLK QL ZEXQ TFQE VLR
QMKCRGKCQ G UYLR RM AFYR UGRF WMS, ZSR G FYTC LM PCYQML RM AFYR UGRF WMS
RNLDSHLDR H VZMS SN BGZS VHSG XNT, ATS H GZUD MN QDZRNM SN BGZS VHSG XNT
SOMETIMES I WANT TO CHAT WITH YOU, BUT I HAVE NO REASON TO CHAT WITH YOU
TPNFUJNFT J XBOU UP DIBU XJUI ZPV, CVU J IBWF OP SFBTPO UP DIBU XJUI ZPV
UQOGVKOGU K YCPV VQ EJCV YKVJ AQW, DWV K JCXG PQ TGCUQP VQ EJCV YKVJ AQW
```
可以很容易的分析出只有倒数第三行是一段有意义的语句，它就是这段明文的密文信息。

## 结语

题外话，不知道你注意到代码中我用了confession来命名密文变量没，相信看到密文结果后就应该明白我的意思了。记得在知乎上看到某个自称混迹密码学主题很久的人说过一句话：「对于男女之间，但凡看不懂的都是表白。」

大约是有一定道理的。

## 参考文献

1.  Wikipedia. [Caesar Cipher](http://en.wikipedia.org/wiki/Caesar_cipher)
2.  Wikipedia. [Method of exhaustion](http://en.wikipedia.org/wiki/Method_of_exhaustion)
