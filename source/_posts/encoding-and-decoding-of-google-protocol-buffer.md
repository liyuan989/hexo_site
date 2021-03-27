---
title: Google Protocol Buffer的编解码原理
id: 239
categories:
  - 网络编程
date: 2015-03-18 22:27:40
tags:
---

Google Protocol Buffer（简称protbuf）是一种紧凑的可扩展的二进制消息格式，并且如上一篇文章[《解析Google Protocol Buffer消息类型的自动反射原理》](http://originlee.com/2015/03/14/analysis-google-protobuf-reflection/)中介绍的那样，它还自带消息类型反射的功能。从Google公布的[benchmark](https://code.google.com/p/thrift-protobuf-compare/wiki/Benchmarking)来看，protobuf的效率还是很高的，这主要依赖于它的高效编解码方式。

先来看一个简单的例子：
```
message Test1
{
    required int32 id = 1;
}
```
在一个应用中，我们将一个Test1的Message中的id设为150，将这个Message进行编码序列化以后以16进制输出到stdout中，会看到这样的3个字节：
```
08 96 01
```
也就是说只用了3个字节就表示了数据类型和数据的值。

## Base 128 Varints

Varints是protobuf的序列化整型数据的一种编码方式，它的优点是整型数据的值越小，编码后所用的字节数越小。

经过Varints编码后的数据，它的每一个字节8bit的高位代表一个标记位。如果标记位是1，则代表下一个字节仍然是当前整型数据的组成；如果标记位是0，则代表下一个字节就是下一个整型数据了。

先看最简单的数值1，一个字节足以表示它，所以它的标记位置0：
```
0000 0001
```
再看数值300，经过Varints编码后的序列：
```
1010 1100 0000 0010
```
从左到右，依次将每个字节的高位（标志位）去掉，若标志位为1，则下一个字节仍是当前数据的组成，若标志位为0，则下一个字节是下一个数据的组成。
```
1010 1100 0000 0010
→ 010 1100  000 0010
```
因为protobuf是以little endian来编码字节序的，所以将两个字节交换、拼接，即可得到原始数据的二进制表示：
```
000 0010  010 1100
→  000 0010 ++ 010 1100
→  100101100
→  256 + 32 + 8 + 4 = 300
```
<!--more-->

## 消息结构

我们知道，protobuf的消息是经过编码序列化的一系列key-value对，一个类型的数据对应一个key-value。value就是原始数据经过编码后的数据，而key由field number和wire type组成。其中field number是指.proto文件中定义的数据变量的field值。
```
message Test
{
    required int32 id = 1; // (1为field number)
}
```
Test类型中的数据id的field number就为1。

wire type是指的是.proto文件中定义的数据变量的类型的序号，protobuf中可以定义的所有数据类型的序号如下表：

| Type | Meaning          | Used For                                 |
| ---- | ---------------- | ---------------------------------------- |
| 0    | Varint           | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1    | 64               | fixed64, sfixed64, double                |
| 2    | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| 3    | Start group      | groups (deprecated)                      |
| 4    | End group        | groups (deprecated)                      |
| 5    | 32-bit           | fixed32, sfixed32, float                 |

所以，数据id的wire type就为0，代表它是用varint进行编码的。

对于key-value中的key的值，是通过移位再相或的方法将field number和wire type用一个字节来存储的：
```
(field_number << 3) | wire_type
```
即低三位表示wire type，其他的位表示field number。

再回到本文开头的例子，先看第一个字节08
```
0000 1000
```
wire type为0，field name为1。符合id的类型是int32，编码方式是Varints，field是1的这一情况。

而后两个字节应该是150经过Varints编码后的序列：
```
96 01 = 1001 0110  0000 0001
       → 000 0001  ++  001 0110 (drop the msb and reverse the groups of 7 bits)
       → 10010110
       → 2 + 4 + 16 + 128 = 150
```

## 有符号整型

上文中提到例子都是wire type为0，也就是Varints编码。然而，对于protobuf来说，编码「标准」的整型（int32，int64）和编码「有符号」的整型（sint32，sint64）的时候，对于负数的处理方式是不同的。如果用int32或者int64来编码一个负数的话，通常需要耗费10个字节来表示，因为负数在计算机中是以补码表示的，相当于一个数值很大的无符号数。

所以这个时候就需要用「有符号」的整型（sint32，sint64）来表示负数了，它们先使用[ZigZag](http://en.wikipedia.org/wiki/Zigzag)编码方式将负数转换为绝对值较小正数，再进行Varints编码。

ZigZag将会这样编码，-1编码为1，1编码为2，-2编码为3......

| Signed Original | Encoded As |
| --------------- | ---------- |
| 0               | 0          |
| 1               | 1          |
| 1               | 2          |
| 2               | 3          |
| 2147483647      | 4294967294 |
| 2147483648      | 4294967295 |

sint32的ZigZag编码公式为
```
(n << 1) ^ (n >> 31)
```
相应的，sint64的ZigZag编码公式为
```
(n << 1) ^ (n >> 63)
```
有一点需要注意，(n >> 31)和(n >> 64)使用的是算术移位（也就是用符号位扩展移位）。

## Non-varint Numbers

wire type为1和5的整型数据都是非Varints编码的。fixed64, sfixed64, double固定用64bit也就是8个字节来表示，fixed32, sfixed32, float固定的用32bit也就是4个字节来表示。

需要注意的是他们都使用little endian来编码字节序的。

## 字符串

wire type为2代表着该类型的数据长度是指定的。也就是用在key-value的中间加上一个经过Varints编码的表示value长度字段，即key-len-value。
```
message Test2
{
    required string str = 2;
}
```
将str设置为「testing」，然后把经过protobuf编码序列化后的数据以16进制的方式输出到stdout
```
12 07 74 65 73 74 69 6e 67
```
第一个字节0x12表示field name为2，wire type为2。第二个字节0x07表示数据长度为7，所以后面7个字节就是使用UTF8编码的字符数据，即「testing」。

## 嵌套消息

下面这个例子中，嵌套使用了上文的Test1类型
```
message Test3
{
    required Test1 c = 3;
}
```
同样的，Test1中c的值设为150，编码后得到的16进制序列如下
```
1a 03 08 96 01
```
第一个字节0x1a表示field number为3，wire type为2。我们发现后三个字节（08 96 01）和第一个例子中的序列一样，所以第二个字节0x03的意义就和string中的一样，代表数据的长度。

## Optional And Repeated Elements

如果数据类型定义为repeated（没有选项packed=true）的话，将存在一个或多个具有相同field number值的key-value对。由于历史遗留的原因，如果repeated数据没有加上[packed=true]选项的话，repeated的系列数据并不一定是连续的。由于可能有多个key-value对的存在，导致效率会略有降低。

如果数据类型定义为optional的话，将存在0个或1个包含field number值的key-value对。

正常情况下，required和optional类型的最多只能出现一次，但是protobuf在做parse的时候，得考虑到所有的情况。所以对数值类型和字符串，如果出现多次，那么将只接受最后出现那个数据。对于嵌套类型，将有相同field值数据像Message::MergeFrom一样进行合并，即

> all singular scalar fields in the latter instance replace those in the former, singular embedded messages are merged, and repeated fields are concatenated

这个规则带来的好处就是，parse两个连续的已编码好的Message的结果与分别单独parse两个Message后再合并得到的结果是一致的。所以，下面两种是情况是相等的。
```c++
// 情况一
MyMessage message;
message.ParseFromString(str1 + str2);

// 情况二
MyMessage message, message2;
message.ParseFromString(str1);
message2.ParseFromString(str2);
message.MergeFrom(message2);
```

## Packed Repeated Fields

protobuf从2.1版本开始，对于repeated类型数据可以加上[packed=true]选项。加上这个选项后，每个repeated类型的数据连续排列，并且所有的repeated类型数据和field number、wrie type、len组成一个大的key-value（相当于key-value, value, value...）。

例如，下面一个例子：
```c++
message Test4
{
    repeated int32 d = 4 [packed=true];
}
```
建立一个Test4类型的Message对象，添加3个d类型变量，值分别为3, 270, 86492。得到编码后的16进制数据为：
```c++
22        // tag (field number 4, wire type 2)
06        // len (6 bytes)
03        // first element (varint 3)
8E 02     // second element (varint 270)
9E A7 05  // third element (varint 86942)
```
需要注意的是只有repeated类型的数据才能设置[packed=true]选项。

## Field Order

尽管我们在编写.proto文件的Message时，可以按任意顺序设置变量类型的field number，但是protobuf在进行编码序列化的时候，会按field number的大小顺序存储数据类型，这样有利于parse时候的一些优化。但由于不是所有的消息都是由一个经过编码序列化的Message对象组成的，所以protobuf也可以parse按任意field number顺序存放的数据。在某些场合下，单纯的将2个Message对象合并（连接）起来也是有用的。

在C++实现中，如果一个Message对象中的数据含有无效的field number值，那么protobuf会把这个对象的数据按任意顺序存放。

## 参考文献

1.  Protocol Buffers. [Encoding](https://developers.google.com/protocol-buffers/docs/encoding)
2.  Wikipedia. [Zigzag](http://en.wikipedia.org/wiki/Zigzag "Zigzag")
3.  Google Project Hosting. [Benchmarking](https://code.google.com/p/thrift-protobuf-compare/wiki/Benchmarking)