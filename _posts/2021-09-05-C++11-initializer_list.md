---
layout:     post
title:      initializer_list
subtitle:   C++11 - initializer_list
date:       2021-09-5
author:     Zhgaot
header-img: img/C++-C++11/C++_cover.png
catalog: true
tags:
	- C++
    - C++11
---

> 文章参考：侯捷-C++11新特性

## 1 initializer_list<T>的使用

#### 1.1 函数参数可以为initializer_list

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++-C++11/initializer_list_0.png)

#### 1.2 函数调用实验

【注意】下例中的赋值例子，依然调用的是构造函数，而不是赋值拷贝函数，因为这是直接创造了一个新的对象而不是旧对象的新赋值！

1. 当类中同时包含“特定个数参数”和“initializer_list”重载时：

    ```cpp
    class P {
    public:
    	P(int a, int b) {  // a specific number of arguments
    		cout << "P(int, int), a=" << a << ", b=" << b << endl;
    	}
    	P(initializer_list<int> initlist) {  // an initializer list
    		cout << "P(initializer_list<int>), values=";
    		for (auto i : initlist)
    			cout << i << " ";
    		cout << endl;
    	}
    };

    P p(77, 5);       // *P(int, int), a=77, b=5*
    P q{77, 5};       // *P(initializer_list<int>), values=77 5*
    P r{77, 5, 42};   // *P(initializer_list<int>), values=77 5 42*
    P s={77, 5};      // *P(initializer_list<int>), values=77 5*
    ```

2. 当类中同时包含“特定个数参数”函数时：

    ```cpp
    class P {
    public:
    	P(int a, int b) {  // a specific number of arguments
    		cout << "P(int, int), a=" << a << ", b=" << b << endl;
    	}
    	/*
    	P(initializer_list<int> initlist) {  // an initializer list
    		cout << "P(initializer_list<int>), values=";
    		for (auto i : initlist)
    			cout << i << " ";
    		cout << endl;
    	}
    	*/
    };

    P p(77, 5);       // *P(int, int), a=77, b=5*
    P q{77, 5};       // *P(int, int), a=77, b=5* (可调用)
    P r{77, 5, 42};   // ERROR
    P s={77, 5};      // *P(int, int), a=77, b=5* (可调用)
    ```

## 2 initializer_list源码

#### 2.1 initializer_list源码

1. `initializer_list`内部是一个`std::array`类型的指针（迭代器），意思是其内部指向了一个`std::array`；**因此initializer_list内部只有一个指针，而并没有真正包含一个array！**
2. 但`initializer_list`**内部并无深拷贝的拷贝函数重载，因此它会进行浅拷贝**，需注意！
3. 编译器在看到{}时会自动创建一个`std::array`类型的对象，然后会将`std::array`类型的对象传入`initializer_list`的private的构造函数里，让`initializer_list`内部的指针`_M_array_`指向此array，并将内部的`_M_len`设定为此array的长度；

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++-C++11/initializer_list_1.png)

#### 2.2 相关的std::array源码示例

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++-C++11/initializer_list_2.png)

## 3 initializer_list在std中的使用

**如今所有容器都接受指定任意数量的值用于构建、赋值或insert()/assign()等；同时，max()和min()也愿意接受任意数量的参数。**

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++-C++11/initializer_list_3.png)

**关于max()和min()，它们之前只接受两个参数比较大小，现在如果使用{}的话可以接受任意数量的参数比较大小了！**

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++-C++11/initializer_list_4.png)