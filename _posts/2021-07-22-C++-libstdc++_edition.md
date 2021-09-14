---
layout:     post
title:      Linux下libstdc++的版本问题
subtitle:   Linux下无法链接到libstdc++最新版本的问题
date:       2021-7-22
author:     Zhgaot
header-img: img/C++/C++_cover.png
catalog: true
tags:
    - C++
    - Tencent-Internship
---

## 1 报错出现
#### 1.1 错误日志信息
腾讯引力计划mini项目中计划使用组内的Bon协议来作为信息的载体。从git上拉下来Bon了C++源码及其提供的demo，对demo进行编译后，出现如下报错：
```bash
[命令]./bon_demo
[报错]./bon_demo: /lib64/libstdc++.so.6: version 'CXXABI_1.3.9' not found (required by ./bon_demo)
[命令]./server_demo_bon
[报错]./server_demo_bon: /lib64/libstdc++.so.6: version 'GLIBCXX_3.4.21' not found (required by ./server_demo_bon) 
[报错]./server_demo_bon: /lib64/libstdc++.so.6: version 'CXXABI_1.3.9' not found (required by ./server_demo_bon)
```

#### 1.2 报错原因
- 没有链接到CXXABI库的最新的版本
- 没有链接到GLIBCXX库的最新的版本

## 2 解决问题
#### 2.1 查看当前版本
查看CXXABI库，果然没有CXXABI_1.3.9版本：
![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/libstdc++/edition_0.png)

查看GLIBCXX库，果然没有GLIBCXX_3.4.21版本：
![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/libstdc++/edition_1.png)

#### 2.2 查找初始文件
一般来说，libstdc++.so.6仅为软连接，查看其当前指向的是谁：
查找所有libstdc++.so.6*的文件，这里发现我的机子上有很多版本，最高版本已经到6.0.29，肯定已经覆盖了最新版本，那么报错的原因就是libstdc++.so.6链接到的libstdc++版本过低：
![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/libstdc++/edition_2.png)

进入文件路径验证一下想法，确实链接到的是同目录下的6.0.19(废话...)：
![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/libstdc++/edition_3.png)

#### 2.3 创建新的链接
1. 进入上图中搜索出的存在libstdc++.so.6.0.29文件的路径，将上图中出现的libstdc++.so.6.0.29拷贝进/usr/lib64目录
2. 删除旧链接
3. 并建立新的链接
4. 问题解决~~~
![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/libstdc++/edition_4.png)

#### 2.4 另外
我之前在自己机子上直接将GCC升级到了11.1.0，因此能直接找到libstdc++.so.6.0.29，但是如果没有升级GCC可能就不仅仅是需要修改软链接，而应该首先升级GCC并安装：[下载链接](http://ftp.tsukuba.wide.ad.jp/software/gcc/releases/)