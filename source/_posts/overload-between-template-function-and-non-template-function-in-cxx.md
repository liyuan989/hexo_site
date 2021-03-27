---
title: 聊聊C++模板函数与非模板函数的重载
id: 27
categories:
  - C++
date: 2014-12-02 12:22:00
tags:
---

## 1\. 前言

函数重载在c++中是一个很重要的特性。之所以有了它才有了操作符重载、iostream、函数子、函数适配器、智能指针等非常有用的东西。

平常在实际的应用中多半要么是模板函数与模板函数重载，或者是非模板函数与非模板重载。而让模板函数与非模板函数重载的情况却很少。

前段时间在项目中偶然遇到了一个模板函数与非模板函数重载的诡异问题，大概相当于下面这种情况：
```c++
template <typename T>
int compare(const T& lhs, const T& rhs)
{
    std::cout << "template compare" << std::endl;
    return 0;
}

int compare(const char* lhs, const char* rhs)
{
    std::cout << "ordinary compare" << std::endl;
    return 0;
}

int main(int argc, char *argv[])
{
    char c1[] = "hello";
    char c2[] = "hello";
    compare(c1, c2);
}
```
最终输出打印的是什么呢？嗯哼？<!--more-->

## 2\. 分析

开始的时候我以为理所当然输出的是「ordinary compare」，就没有在意这里。结果在程序的其他地方调试了很久死活找不出问题的所在，然后索性就把那个非模板函数改成了模板函数的偏特化函数，之前出现的问题就消失了。这才发现问题出现在之前的模板函数与非模板函数重载那里了。那时候的情况就跟上面的代码的情况差不多一个意思。

回到上面代码输出的打印结果上来，在几个主流的编译器上的输出结果是这样的：

*   g++ 4.8.1 ： template compare
*   clang 3.4.2 ： template compare
*   vs2010 ：ordinary compare
    先来看看c++中模板函数与非模板函数的重载决议步骤：

1\.  为这个函数名建立候选函数集合，包括：

*   与被调用函数名字相同的任意普通函数。
*   任意函数模板实例化，在其中，模板实参推断发现了与调用中所用函数实参相匹配的模板实参。

2\.  确定哪些普通函数是可行的（如果有可行函数的话）。候选集合中的每个模板实例都可行的，因为模板实参推断保证函数可以被调用。

3\.  如果需要转换来进行调用，根据转换的种类排列可靠函数，记住，调用模板函数实例所允许的转换是有限的。

*   如果只有一个函数可选，就调用这个函数。
*   如果调用有二义性，从可行函数集合中去掉所有函数模板实例。

4\.  重新排列去掉函数模板实例的可行函数。

*   如果只有一个函数可选，就调用这个函数。
*   否则，调用有二义性。

再说说为什么我一开始认为一定是输出「ordinary compare」。数组c1、c2要作为实参传参给函数的形参的话要转换为指向数组首元素的指针，也就是说对于模板函数和非模板函数来说都要经过一次转换才能完全匹配，那么根据上面的重载决议规则，就应该调用非模板函数。但结果却并非如此。

这个问题当时在知乎问过，来看看陈硕的回答：

> C++ 这套重载决议规则太复杂，g++/clang 都是resolve为模板，具现化后的模板是：
> int compare<char [6]>(char const (&) [6], char const (&) [6])
> 也就是说T = char[6]，数组没有转化为指针。
>
> 如果把其中一个"hello"改成别的长度的字符串，就是匹配普通版本了。
>
> 如果g++/clang是符合标准的话，我倾向于认为这是C++标准的bug。
>
> FYI, clang consider template is better because it's an Identity Conversion, the other is array-to-pointer:
>
>     #1  clang::compareStandardConversionSubsets (Context=..., SCS1=..., SCS2=...)>
>         at llvm-3.4.2.src/tools/clang/lib/Sema/SemaOverload.cpp:3393>
>     #1  0x00007ffff6cfb353 in clang::CompareStandardConversionSequences (S=..., SCS1=..., SCS2=...)>
>         at llvm-3.4.2.src/tools/clang/lib/Sema/SemaOverload.cpp:3469>
>     #2  0x00007ffff6cfaeff in clang::CompareImplicitConversionSequences (S=..., ICS1=..., ICS2=...)>
>         at llvm-3.4.2.src/tools/clang/lib/Sema/SemaOverload.cpp:3336>
>     #3  0x00007ffff6d0ac37 in clang::isBetterOverloadCandidate (S=..., Cand1=..., Cand2=..., Loc=..., >
>         UserDefinedConversion=false)>
>         at llvm-3.4.2.src/tools/clang/lib/Sema/SemaOverload.cpp:8031>
>     #4  0x00007ffff6d0afd1 in clang::OverloadCandidateSet::BestViableFunction (this=0x7fffffff77a0, S=..., >
>         Loc=..., Best=@0x7fffffff7790: 0x7fffffff7860, UserDefinedConversion=false)>
>         at llvm-3.4.2.src/tools/clang/lib/Sema/SemaOverload.cpp:8148>
>     #5  0x00007ffff6d12220 in clang::Sema::BuildOverloadedCallExpr (this=0x7445e0, S=0x781630, Fn=0x782620, >
>         ULE=0x782620, LParenLoc=..., Args=..., RParenLoc=..., ExecConfig=0x0, AllowTypoCorrection=true)>
>         at llvm-3.4.2.src/tools/clang/lib/Sema/SemaOverload.cpp:10394>
>     #6  0x00007ffff6bceb9a in clang::Sema::ActOnCallExpr (this=0x7445e0, S=0x781630, Fn=0x782620, LParenLoc=..., >
>         ArgExprs=..., RParenLoc=..., ExecConfig=0x0, IsExecConfig=false)>
>         at llvm-3.4.2.src/tools/clang/lib/Sema/SemaExpr.cpp:4470>
>     #7  0x00007ffff7255459 in clang::Parser::ParsePostfixExpressionSuffix (this=0x75f6f0, LHS=...)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseExpr.cpp:1455>
>     #8  0x00007ffff7254925 in clang::Parser::ParseCastExpression (this=0x75f6f0, isUnaryExpression=false, >
>         isAddressOfOperand=false, NotCastExpr=@0x7fffffffa59f: false, isTypeCast=clang::Parser::NotTypeCast)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseExpr.cpp:1279>
>     #9  0x00007ffff7251fba in clang::Parser::ParseCastExpression (this=0x75f6f0, isUnaryExpression=false, >
>         isAddressOfOperand=false, isTypeCast=clang::Parser::NotTypeCast)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseExpr.cpp:419>
>     #10 0x00007ffff7251105 in clang::Parser::ParseAssignmentExpression (this=0x75f6f0, >
>         isTypeCast=clang::Parser::NotTypeCast)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseExpr.cpp:168>
>     #11 0x00007ffff7250f2c in clang::Parser::ParseExpression (this=0x75f6f0, isTypeCast=clang::Parser::NotTypeCast)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseExpr.cpp:120>
>     #12 0x00007ffff727ba85 in clang::Parser::ParseExprStatement (this=0x75f6f0)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseStmt.cpp:371>
>     #13 0x00007ffff727b46b in clang::Parser::ParseStatementOrDeclarationAfterAttributes (this=0x75f6f0, Stmts=..., >
>         OnlyStatement=false, TrailingElseLoc=0x0, Attrs=...)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseStmt.cpp:231>
>     #14 0x00007ffff727abae in clang::Parser::ParseStatementOrDeclaration (this=0x75f6f0, Stmts=..., >
>         OnlyStatement=false, TrailingElseLoc=0x0)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseStmt.cpp:118>
>     #15 0x00007ffff727d7c8 in clang::Parser::ParseCompoundStatementBody (this=0x75f6f0, isStmtExpr=false)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseStmt.cpp:907>
>     #16 0x00007ffff7283373 in clang::Parser::ParseFunctionStatementBody (this=0x75f6f0, Decl=0x782340, >
>         BodyScope=...)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseStmt.cpp:2458>
>     #17 0x00007ffff7223ada in clang::Parser::ParseFunctionDefinition (this=0x75f6f0, D=..., TemplateInfo=..., >
>         LateParsedAttrs=0x7fffffffb8f0)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/Parser.cpp:1171>
>     #18 0x00007ffff7230a62 in clang::Parser::ParseDeclGroup (this=0x75f6f0, DS=..., Context=0, >
>         AllowFunctionDefinitions=true, DeclEnd=0x0, FRI=0x0)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseDecl.cpp:1617>
>     #19 0x00007ffff7222b6b in clang::Parser::ParseDeclOrFunctionDefInternal (this=0x75f6f0, attrs=..., DS=..., >
>         AS=clang::AS_none)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/Parser.cpp:932>
>     #20 0x00007ffff7222c33 in clang::Parser::ParseDeclarationOrFunctionDefinition (this=0x75f6f0, attrs=..., >
>         DS=0x0, AS=clang::AS_none)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/Parser.cpp:948>
>     #21 0x00007ffff72223bb in clang::Parser::ParseExternalDeclaration (this=0x75f6f0, attrs=..., DS=0x0)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/Parser.cpp:807>
>     #22 0x00007ffff72218ab in clang::Parser::ParseTopLevelDecl (this=0x75f6f0, Result=...)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/Parser.cpp:612>
>     #23 0x00007ffff721e1e5 in clang::ParseAST (S=..., PrintStats=false, SkipFunctionBodies=false)>
>         at llvm-3.4.2.src/tools/clang/lib/Parse/ParseAST.cpp:144

也就是说g++和clang选择匹配模板函数，是因为它们并没有将c1和c2转换为指向数组首元素的指针，而是直接匹配，即T = char  [6]。而vs2010是将他们转换为指向数组首元素的指针后再进行匹配的，所以它选择非模板函数。

那么到底哪个比较正确呢？

我们来看看模板实参推断时对实参的转换规则。

一般而论，不会转换实参以匹配已有的实例化，相反，会产生新的实例。除了产生新的实例化之外，编译器只会执行两种转换：

*   const 转换：接受 const 引用或 const 指针的函数可以分别用非 const对象的引用或指针来调用，无须产生新的实例化。如果函数接受非引用类型，形参类型实参都忽略const，即，无论传递 const 或非 const 对象给接受非引用类型的函数，都使用相同的实例化。
*   数组或函数到指针的转换：如果模板形参不是引用类型，则对数组或函数类型的实参应用常规指针转换。数组实参将当作指向其第一个元素的指针，函数实参当作指向函数类型的指针。

按照实参推导时实参转换规则，因为模板函数的实参是引用类型，不会对数组实参进行到指针的转换，所以直接推断T = typename [n]（typename为数组的类型，n为数组的长度）。对于本文的情况，模板函数直接进行实参推断并匹配，即T = char  [6]，而非模板函数先要将数组转换为指针，再匹配函数。所以我认为正确的应该是匹配模板函数。如果令文中的数组c1和c2的长度不一样，那么模板函数两个实参推断结果不一样而导致匹配失败，进而应该匹配非模板函数

最后说一下，在实际应用中的大多数情况都应该用模板函数与模板函数的偏特化来代替模板函数与普通非模板函数的重载，以避免模板函数与非模板函数的重载导致在不同编译器环境下结果不一样的情况发生。

## 3\. 参考文献

1.  Stanley B.Lippman. [C++ Primer, 4th Edition. 人民邮电出版社, 2006](http://book.douban.com/subject/1767741/)
    （完）