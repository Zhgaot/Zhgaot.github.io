---
layout:     post
title:      undefined reference to `XXX'
subtitle:   undefined reference to `XXX'问题记录与解决
date:       2021-8-15
author:     Zhgaot
header-img: img/C++/C++_cover.png
catalog: true
tags:
    - C++
    - Tencent-Internship
---

> 本篇博客记录了我所遇到的关于undefined reference to `XXX'报错的工程原因以及解决思路，同时参考了一篇[参考文章](https://github.com/Captain1986/CaptainBlackboard)，在此文章中记录的原因在本博客中将不再进行实验与讲解

## 1 原因概述

我们在Linux下用C/C++工作的时候，经常会遇到`undefined reference to 'XXX'`的问题，直白地说就是在**链接**(从.cpp源代码到可执行的ELF文件，要经过预处理->编译->链接三个阶段，此时预处理和编译已经通过了)的时候，链接器找不到XXX这个函数的定义了。

我参考了这篇文章[👉"undefined reference to XXX"问题总结发现](https://github.com/Captain1986/CaptainBlackboard)，其中共总结了6点原因：

1. 链接时缺少定义了XXX的源文件或者目标文件或者库文件；
2. 产生依赖时，链接顺序不对；
3. 函数定义和声明不一致；
4. C和C++混合编程；
5. 把模板函数写进了cpp文件中；
6. api hinden；

更多详情请直接参见该文章，本篇博客将追加一些其他原因，或者涉及上述原因的例子，以及相应的解决方法。

## 2 inline function导致的报错

### 2.1 问题描述

为了了解std::string的原理，自己实现了一个String类，将类的声明写于头文件String.h中、将类的实现写于String.cpp中，然后在main.cpp中使用String类来实例化对象，希望测试一系列的构造、拷贝构造、拷贝复制、析构等类内成员函数。结果出现了如下报错：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/undefined_reference/undefined_reference_0.png)

### 2.2 问题分析与解决

经过问题查找发现，报错的原因是因为**在String.cpp文件中的类内成员函数的实现前都加上了inline**。

经过对inline使用方法的查询([用法说明](https://www.runoob.com/w3cnote/cpp-inline-usage.html))([报错参考](https://blog.csdn.net/PGZXB/article/details/107965531))发现：**inline要求每个调用内联函数的文件都出现该函数的定义**，所以之前将inline-function写在String.cpp文件中的做法导致了编译器不能看到定义，只能看到String.h文件中的声明，只有在链接时才能得知定义，但这违反了inline的规则。因此，**如果非要使类内成员函数成为inline-function而又想将其在类外定义，最好将其实现也写在头文件中**，这样当其他.cpp文件(比如main.cpp)include此头文件时，就能满足调用内联函数的文件出现该函数的定义，也就能通过编译了。

知晓上述原因后就很好解决了：

1. 可以将想定义为inline-function的函数写在.h文件中，而非inline-function的函数写在.cpp文件中；
2. 如果想将所有函数都定义为inline-function，则将它们都写与.h文件中，但此时.h文件应改为.hpp文件；

### 2.3 扩展延申

1. 如果将函数的**实现**放在**头文件**中，那么每一个包含该头文件的cpp文件都将得到一份关于该函数的**定义**，那么链接器会报**函数重定义**错误。
2. 如果将函数的**实现**放在**头文件**，并且标记为**inline** 那么每一个包含该头文件的cpp文件都将得到一份关于该函数的**定义**，并且链接器**不会报错**。
3. 如果将函数的**实现**放在**cpp文件**中，并且**没有标记为inline**,那么该函数可以被连接到其他编译单元中。
4. 如果将函数的**实现**放在**cpp文**件中，并且**标记为inline**, 那么该函数对其他编译单元不可见（类似static的效果），也就是其他cpp文件不能链接该函数库，也就是`undefined reference to 'XXX'`报错。

## 3 库的树形链接问题

### 3.1 问题描述

#### 3.1.1 问题背景

在腾讯实习时接手了一个图像编解码项目，最开始肯定要编译一下前辈之前所写的demo康康运行效果，此demo调用了前辈所写的图片解码接口（img_decode），但是出现了巨多的`undefined reference to 'XXX'`报错。

#### 3.1.2 问题过程

1. 目录结构

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/undefined_reference/undefined_reference_1.png)

1. 头文件包含

本项目使用cmake辅助编译，在CMakeLists.txt中包含了demo所需的所有头文件：

```bash
INCLUDE_DIRECTORIES(
    ${CMAKE_SOURCE_DIR}/img_decode
    ${CMAKE_SOURCE_DIR}/opencv/include
    ${CMAKE_SOURCE_DIR}/hevc
    ${CMAKE_SOURCE_DIR}/magick/include/ImageMagick-6
)
```

1. 库文件链接

一开始，我只链接了接口img_decode所生成的静态链接库libImageDecode.a，之后就出现了巨多的`undefined reference to 'XXX'`报错；于是我找到demo原始makefile文件，**发现此demo仅将自己的.cpp文件链接为静态链接库，但其依赖的其他库文件它并未一同打包进静态库，因此我将此接口依赖的静态链接库也写入了CMakeLists.txt中（注：也链接了pthread）**：

```bash
LINK_DIRECTORIES(
    ${CMAKE_SOURCE_DIR}/img_decode  # 接口生成的静态库的路径
    ${CMAKE_SOURCE_DIR}/magick/lib  # 解码库1的路径
    ${CMAKE_SOURCE_DIR}/hevc        # 解码库2的路径
    ${CMAKE_SOURCE_DIR}/opencv/lib  # 解码库3的路径
)
LINK_LIBRARIES([接口生成的静态库] [解码库1涉及的库] [解码库2涉及的库] [解码库3涉及的库] pthread)
```

1. 链接库文件后依然报错

在链接了接口所依赖的库之后，运行build.sh脚本后执行cmake，依然报错，如下图所示：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/undefined_reference/undefined_reference_2.png)

···

### 3.2 问题解决

#### 3.2.1 问题分析

如上所述，`undefined reference to 'XXX'`即表明在链接时找不到某个函数的实现文件。在本例中，可能出现两种错误导致报错：

1. **libImageDecode.a所依赖的库又链接了其他库，形成了树形依赖**，因此虽然我已参考demo的makefile链接了第二层库，但是原始demo依然编译不通过，因此说明可能依然是**链接的库不完全**导致报错；
2. 当库产生了树形依赖时，存在库文件的链接顺序问题，即：**依赖其他库的库一定要放到被依赖库的前面**；

#### 3.2.2 解决方案一：直接寻找缺失库并在链接时注意顺序

如果**树形依赖**的库很多，则此种方法还是较为辛苦，需要先找到所有.a的静态链接库，然后使用`nm命令`寻找报错中所有缺失的函数；如下示例，通过`nm命令`找到了包含`png_get_io_ptr`的静态链接库liblibpng.a：

```bash
pwd  # 输出：/data/code/img_decode/qbar_image_decode/opencv/share/OpenCV/3rdparty/lib
nm --defined-only -CA ./*.a | grep png_get_io_ptr  # 输出：./liblibpng.a:png.c.o:0000000000000000 T png_get_io_ptr
```

#### 3.2.3 解决方案二：参考编译通过的例子即可方便找到需要链接的库

可以通过使用了此接口，且已经编译通过了的项目，找到其编译时所连接的所有静态链接库，然后在自己的项目中尝试。

## 4 拉取的环境缺少库

### 4.1 问题描述

#### 4.1.1 问题背景

在**“3 库的树形链接问题”**中，我在尝试解决一个demo的编译问题，但其实在我链接了所有依赖的库后，依然没有解决：依然报错`undefined reference to 'XXX'`，如下图所示：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/undefined_reference/undefined_reference_3.png)

#### 4.1.2 编译流程

该项目采用的编译流程为：

1. 通过build.sh脚本文件去git上拉取组内规定的通用编译环境gcc-4.9.4（存放在root/multimedia_env路径下），并设置环境变量；
2. 通过build.sh脚本文件执行CMakeLists.txt；

### 4.2 原因分析

#### 4.2.1 原因排查一

由报错提示可知，与上一篇一样，出现`undefined reference to 'XXX'`，这应该依然是库的链接问题；于是上网搜索`operator delete`，发现其一般定义在**libstdc++.a静态链接库**中，那就直接全局搜索此链接库，由于我们使用的是拉取的环境，因此可能链接到的库文件如下红框内所示：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/undefined_reference/undefined_reference_4.png)

首先，进入上图中红框内的路径(`~/multimedia_env/compile_enviroment/gcc-4.9.4/lib64`)，干脆使用nm命令(`nm --defined-only -CA * | grep "operator delete(void*"`)，直接搜索全部文件查看是否有谁定义了`operator delete(void*, unsigned long)`，结果发现：libstdc++.a、libstdc++.so*以及其他所有库文件均未定义`operator delete(void*, unsigned long)`：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/undefined_reference/undefined_reference_5.png)

同理，在路径`~/multimedia_env/compile_enviroment/gcc-4.9.4/lib`下搜索也获得了相同的结果...

#### 4.2.2 原因排查二

但是之前使用此接口的项目却能够编译通过，而且之前的项目也都是从git上拉取的gcc-4.9.4编译环境。因此，我将当前图像编解码项目的demo拷贝到之前的项目进行编译，发现可以通过！

这只能说明demo生成的可执行文件一定定义了`operator delete(void*, unsigned long)`，因此采用`ldd命令`追溯编译后的demo的可执行文件查看其链接的动态链接库：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/undefined_reference/undefined_reference_6.png)

从上图中能够看到，demo的**可执行文件img_demo链接了/lib64/libstdc++.so.6，通过nm查询到此动态链接库中果然定义了`operator delete(void*, unsigned long)`，可这个库恰好通过软链接指向了我本地的gcc-11.1.0环境的库(libstdc++.so.6.0.29)**。

之后发现，之前使用此接口的项目好像根本没有拉取编译环境，因为在当前路径下找不到拉取的gcc-4.9.4环境，并且从日志信息发现使用的是最初的gcc-4.8.5版本...

### 4.3 问题解决

这里提出两个折中办法，但殊途同归，但只能作通过编译的办法，而上述问题没有真正解决。

#### 4.3.1 直接采用本地gcc编译

写了一个脚本文件，直接使用g++命令编译，链接所有需要的库，直接使用本地的gcc-11.1.0编译环境：

```bash
#!/bin/bash

if [ ! -e demo ];
    then
        g++ -o demo -pthread -std=c++14 -D_GLIBCXX_USE_CXX11_ABI=0 -fpermissive -g -fno-strict-aliasing -O3 -Wall -Wno-error -D_GNU_SOURCE -D_REENTRANT -fPIC -m64 image_test2.cpp -Iimg_decode -Iopencv/include -Ihevc -Imagick/include/ImageMagick-6 -Limg_decode -lImgDecode -Lmagick/lib -lMagick++-6.Q16 -lMagickWand-6.Q16 -lMagickCore-6.Q16 -Lhevc -lWxHevcDecoder -Lopencv/lib -lopencv_shape -lopencv_stitching -lopencv_objdetect -lopencv_superres -lopencv_videostab -lopencv_calib3d -lopencv_features2d -lopencv_highgui -lopencv_videoio -lopencv_imgcodecs -lopencv_video -lopencv_photo -lopencv_ml -lopencv_imgproc -lopencv_flann -lopencv_core -Lopencv/share/OpenCV/3rdparty/lib -llibjpeg -llibwebp -llibpng -llibtiff -llibjasper -lIlmImf -lstdc++ -lm -lrt -lz -ldl
fi
SHELL_FOLDER=$(cd "$(dirname "$0")";pwd)

export LD_LIBRARY_PATH=${SHELL_FOLDER}/hevc:$LD_LIBRARY_PATH
```

#### 4.3.2 指定本地gcc-11.1.0对应的库

在指定链接库的路径时添加一行，使其能够链接到本地gcc-11.1.0的库：

```bash
LINK_DIRECTORIES(
    ......
     /usr/local/gcc-11.1.0-install/lib64
    ......
)
```