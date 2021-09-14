---
layout:     post
title:      gdb：No symbol table is loaded
subtitle:   gdb使用报错(No symbol table is loaded)原因总结
date:       2021-7-21
author:     Zhgaot
header-img: img/C++/C++_cover.png
catalog: true
tags:
    - C++
---

## 1 原因一

使用gdb前务必加-g选项，否则将出现如下报错：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/gdb/gdb_problem_0.png)

## 2 原因二

#### 2.1 问题出现

在添加了-g选项后，使用gdb依然报错，如下所示：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/gdb/gdb_problem_1.png)

#### 2.2 问题解决

如2.1节图所示，当前gcc版本为11.1.0，而gdb的版本为7.6.1；报错提示表示：当前我添加了-g选项后gcc11.1.0生成的调试信息是dawnfs5，而gdb7.6.1可支持的为dawnfs2、dawnfs3、dawnfs4；因此这其实是编译环境的问题，只需要升级gdb版本即可。

1. 随便找一个版本下载即可：

    ```bash
    wget http://ftp.gnu.org/gnu/gdb/gdb-9.2.tar.gz
    ```

2. 安装Texinfo文档系统：

    ```bash
    yum install texinfo
    yum install ncurses-devel
    ```

3. 解压 + 生成makefile并编译：

    ```bash
    tar -zxvf gdb-9.2.tar.gz
    cd gdb-9.2
    mkdir build && cd build  # 9.2版本需要新建build
    ./configure
    make  # 9.2版本无需单独执行make install
    ```

4. 将新版本gdb添加至环境变量：

    ```bash
    vim /etc/profile
    export PATH=/usr/local/bin:$PATH  # 此句在profile中添加
    source /etc/profile
    ```

5. 虽然source过了环境变量，但是最好再重启云服务器，因为我之后再VSCode中开启的所有终端依然执行的是旧版本的gdb。