---
layout:     post
title:      C++11线程库使用（六）
subtitle:   单例设计模式共享数据问题分析及解决 & call_once
date:       2021-08-6
author:     Zhgaot
header-img: img/C++/C++_cover.png
catalog: true
tags:
    - C++
    - C++11
    - multi-thread
---

## 1  单例设计模式

#### 1.1  概念

单例设计模式的使用频率较高，它指的是：在整个项目中，存在某个或者某些特殊的类（被称为单例类），**由单例类所实例化出的对象，只能够被创建1个**。

#### 1.2  示例（记得看注释）

```cpp
mutex myMutex;

/* 单例类 */
class MyCAS {
private:
	/* 手动定义【私有化的构造函数】，则不能在外部创建该类的对象了 */
	MyCAS() {}
private:
	static MyCAS* m_instance;  // 静态成员变量
public:
	static MyCAS* getInstance() {
		if (m_instance == nullptr) {
			m_instance = new MyCAS();
		}
		return m_instance;
	}
	void func() {
		cout << "*****测*****试*****" << endl;
	}
};
/* 静态成员变量类内声明类外初始化 */
MyCAS* MyCAS::m_instance = nullptr;

int main() {
	/* 通过调用MyCAS的静态函数getInstance()创建对象，返回值是指向该类的指针
	 * 即使第二次还想通过调用该函数生成对象，由于静态成员变量m_instance已不为空，则必定无法创建成功
	 * 第二次再调用该函数时，返回的依然是第一次返回的指针
	**/
	MyCAS* p = MyCAS::getInstance();
	p->func();
}
```

## 2  单例设计模式共享数据问题分析及解决

#### 2.1  安全的方法

在创建其他所有子线程之前，就在主线程中把各个单例类实例化出对象，再把它内部该初始化的数据初始化（例如里面有一些接口函数用于装载配置文件）；这样，从此开始，对象中的数据就变为【只读】数据了，这样所有子线程再访问【只读】数据就安全了。

#### 2.2  工程中可能面临的问题

在实际项目中，可能需要在子线程中来创建单例类的对象，而且创建单例类的对象的子线程还可能不止一个，因此上述代码中用于创建单例类的对象的静态成员函数（例如`MyCAS::getInstance()`）就需要各个子线程互斥地访问了。

1. 【问题出现】

    如下所示，两个线程并发地执行入口函数，则可能两个线程因CPU调度而产生异步，最终都创建出单例类的对象：

    ```cpp
    mutex myMutex;

    /* 单例类 */
    class MyCAS {
    private:
    	/* 手动定义【私有化的构造函数】，则不能在外部创建该类的对象了 */
    	MyCAS() {}
    private:
    	static MyCAS* m_instance;  // 静态成员变量
    public:
    	static MyCAS* getInstance() {
        /* 线程入口函数调用此函数，可能导致thread1和thread2都先执行完if判断，
         * 然后都进入if代码段内，都创建出单例类的对象，这样就不符合单例类的初衷了，
         * 因此，需要将下述代码加锁
        **/
    		if (m_instance == nullptr) {
    			m_instance = new MyCAS();
    		}
    		return m_instance;
    	}
    	void func() {
    		cout << "*****测*****试*****" << endl;
    	}
    };
    /* 静态成员变量类内声明类外初始化 */
    MyCAS* MyCAS::m_instance = nullptr;

    /* 线程入口函数 */
    void myThread() {
    	cout << "我的线程开始执行了..." << endl;
    	MyCAS* p_a = MyCAS::getInstance();  // 这里会因为线程异步，而创建出多个单例类的对象
    	cout << "我的线程执行完毕了..." << endl;
    }

    int main() {
    	/* 创建两个子线程，用同一个入口函数
    	 * 虽然下面两个线程用了同一个入口函数，但这是创建了两个线程，不要混淆！
    	 * 因此，这里会有两个线程并发地执行入口函数
    	**/
    	thread thread1(myThread);
    	thread thread2(myThread);
    	thread1.join();
    	thread2.join();
    }
    ```

2. 【初步解决】

    如上所述，需要将静态成员函数`static MyCAS* getInstance()`内有关“在堆区创建单例类的对象”的代码段让各个线程互斥地方位才行，如下所示：

    ```cpp
    public:
    	static MyCAS* getInstance() {
        /* 线程入口函数调用此函数，可能导致thread1和thread2都先执行完if判断，
         * 然后都进入if代码段内，都创建出单例类的对象，这样就不符合单例类的初衷了，
         * 因此，需要将下述代码加锁
        **/
    		unique_lock<mutex> myUniqueLock(myMutex);  // 【上锁+函数结束后解锁】
    		if (m_instance == nullptr) {
    			m_instance = new MyCAS();
    		}
    		return m_instance;
    	}
    ```

3. 【提升效率】

    对于2中所写的代码，其实已经完成了让多个线程互斥地创建单例类的对象了；当某一个线程创建了单例类的对象后，静态成员变量`m_instance`将不再是空指针(`nullptr`)，则当其他线程再通过`void myThread()`函数调用静态成员函数`static MyCAS* getInstance()`来创建单例类的对象时，就会因为不满足if条件而直接return之前创建好的对象地址。

    但是！即使此时单例类已经被创建，其他线程依然会尝试拿锁，并进行if条件判断，因为它们并不知道单例类已经创建完成了；每有一个线程上锁，其他线程就无法进入if判断，最终就会出现所有线程都拿过一次锁，再进行里面的if判断，但发现单例类已经创建完毕，这些后序的上锁操作全是徒劳，因此需要一个提升效率的方法。

    这里给出一个小技巧——【双重检查】，如下所示：

    ```cpp
    public:
    	static MyCAS* getInstance() {
    		/* 1. 如果：if (m_instance != nullptr) 条件成立，则说明m_instance一定被new过了，单例类的对象已被创建
    		 * 2. 如果：if (m_instance == nullptr) 条件成立，不代表m_instance一定没被new过，单例类的对象不一定没被创建，
    		 *    比如说：可能已经被thread1创建了，只不过在thread1创建之前，thread2也通过了if (m_instance == nullptr)的判断
    		*/
    		if (m_instance == nullptr) {  // 【双重检查】
    			unique_lock<mutex> myUniqueLock(myMutex);
    			if (m_instance == nullptr) {
    				m_instance = new MyCAS();
    			}
    		}
    		return m_instance;
    	}
    ```

    这种方法可以使得只有少数几个因CPU轮转而都执行过外层`if (m_instance == nullptr)`语句的线程争着去拿锁，而在这些少数线程之一的某个线程创建完单例类的对象后，其余大部分线程都会因为不满足外层`if (m_instance == nullptr)`语句而根本不去上锁，直接返回结果。

## 3  std::call_once()

#### 3.1  作用

`std::call_once()`是c++11中引入的函数。该函数的功能是：**能够保证某个函数只被互斥地调用一次**，只需要将这个函数写在`std::call_once()`的第二个参数处即可。就如同上面的例子一样，即使是多个线程都去调用某个函数，那`std::call_once()`也会使该函数只被其中一个线程调用一次！同时，`std::call_once()`具备互斥量的能力，而且比使用互斥量消耗的资源更少。

#### 3.2  std::once_flag标记

`std::call_once()`需要结合`std::once_flag`标记来使用，`std::call_once()`就是通过该标记来决定某个函数`a()`是否执行；当调用`std::call_once()`成功后，`std::call_once()`就把标记`std::once_flag`设置为【已调用状态】，则后序再调用某个函数`a()`时，函数`a()`就不会再被执行了。

#### 3.3  示例

这里的调用关系是：线程入口函数`void myThread()` => `static MyCAS* getInstance()` => (互斥地且唯一次地访问)`static void createInstance()`

为何不直接将函数`static MyCAS* getInstance()`放入`std::call_once()`中，是因为`std::call_once()`只允许放入返回值为void类型的函数

```cpp
mutex myMutex;
std::once_flag flag;  // 在全局定义一个结构体flag，因为类内必须定义静态变量，但又无法初始化

/* 单例类 */
class MyCAS {
private:
	/* 手动定义【私有化的构造函数】，则不能在外部创建该类的对象了 */
	MyCAS() {}
	**static void createInstance()** {
		/*if (m_instance == nullptr) {
			m_instance = new MyCAS();
		}*/  // 由于当前函数被【互斥且仅一次】地访问，因此无需if判断
		m_instance = new MyCAS();
		cout << "---------------createInstance()被执行---------------" << endl;  // 观察输出到底执行了几次
	}
private:
	static MyCAS* m_instance;  // 静态成员变量
	//static std::once_flag flag;  // 标记
public:
	static MyCAS* getInstance() {
		std::call_once(flag, createInstance);
		return m_instance;
	}
	void func() {
		cout << "*****测*****试*****" << endl;
	}
};
/* 静态成员变量类内声明类外初始化 */
MyCAS* MyCAS::m_instance = nullptr;

/* 线程入口函数 */
void myThread() {
	cout << "我的线程开始执行了..." << endl;
	MyCAS* p = MyCAS::getInstance();
	p->func();
	cout << "我的线程执行完毕了..." << endl;
}

int main() {
	/* 创建两个子线程，用同一个入口函数
	 * 虽然下面两个线程用了同一个入口函数，但这是创建了两个线程，不要混淆！
	 * 因此，这里会有两个线程并发地执行入口函数
	**/
	thread thread1(myThread);
	thread thread2(myThread);
	thread1.join();
	thread2.join();
}
```
    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/6_0.png)