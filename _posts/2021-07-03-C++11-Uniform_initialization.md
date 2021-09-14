---
layout:     post
title:      Uniform Initialization
subtitle:   C++11 - 一致性初始化{}
date:       2021-07-3
author:     Zhgaot
header-img: img/C++/C++_cover.png
catalog: true
tags:
    - C++
    - C++11
---

> 文章参考：侯捷-C++11新特性
>
> ------
>
> 下篇文章：[initializer_list](https://zhgaot.github.io/2021/09/05/C++11-initializer_list/)

### 1 出现原因

在C++11之前，程序员或新手可能很困惑于一件事情：如何初始化一个变量或者一个对象，因为这种初始化可能发生在`小括号()`、`大括号{}`、`赋值运算符=`上。

### 2 如何使用

基于这个原因，C++11导入了“Uniform Initialization(一致性初始化)”，即：任何初始化都可以使用大括号来实现，在变量的后面直接写`{}`，在内部放入初值即可。

### 3 实现原理

`大括号{}`赋初值的实现原理：

1. 首先，编译器看到参数列表`{t1, t2, ..., tn}`便会在幕后根据它们的数据类型自动生成一个`initializer_list<T>`（它的幕后其实是一个`array<T,n>`）；
2. 之后，调用函数（例如ctor）时该`array`内的元素可被编译器分解并逐一地传给该函数；
3. 当然，如果该函数的参数本身就是`initializer_list<T>`，调用者就不能给予数个T参数然后以为它们会被自动转换为一个`initializer_list<T>`而传入；

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/C++-C++11/UI_0.png)

对于所有容器而言，它们都有一个ctor是可以直接接受`initializer_list<T>`参数的版本，而自己写的类不一定有这个版本，那么编译器就会自动将`initializer_list<T>`内的参数一个一个传入。

### 4 Uniform Initialization使用小tips

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/C++-C++11/UI_1.png)