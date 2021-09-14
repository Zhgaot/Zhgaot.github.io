---
layout:     post
title:      C++11线程库使用（七）
subtitle:   条件变量condition_variable
date:       2021-07-7
author:     Zhgaot
header-img: img/C++/C++_cover.png
catalog: true
tags:
    - C++
    - C++11
    - multi-thread
---

## 1  条件变量std::condition_variable

条件变量`std::condition_variable`是一个类，它需要实例化后配合**互斥量**来工作。这里引入的条件变量与操作系统中管程提供的条件变量类似，`std::condition_variable`类所生成的对象可以使用成员变量`wait()`和`notify_one()`，它们分别对应了管程中条件变量的wait和signal操作，二者的结合使用能够实现线程间的同步。

## 2  成员函数wait()

#### 2.1  使用前提

1. 条件变量`std::condition_variable`是一个类，首先需要实例化出对象，再通过实例化出的对象调用成员函数`wait()`
2. `wait()`的使用前提是首先让该线程拿到unique_lock<mutex>类型的互斥量并将其上锁

#### 2.1  函数作用

`wait()`函数作用是：(可以通过某一判定条件)来决定是否睡眠当前线程；如果判定条件返回true则继续持有之前获取的锁（互斥量），然后继续执行余下代码；如果判定条件返回false则解锁互斥量并睡眠当前线程（可理解为挂起），直到其他线程使用成员函数`notify_one()`将本线程唤醒，才继续执行余下代码。

#### 2.2  函数参数

`wait()`有两个重载版本，即可以选择填入或不填入第二个参数：

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/7_0.png)

1. `wait()`的第一个参数为**unique_lock<mutex>类型的互斥量**，因此`condition_variable`实例化出的对象必须在unique_lock<mutex>上锁后调用`wait()`函数。
2. `wait()`的第二个参数为**_Predicate谓词**，它可以是**函数指针**（放入函数名即可）、**仿函数**、**lambda表达式，最终要返回的是true或false**。
3. 第二参数_Predicate谓词详解：
    - 当第一次调用`wait()`函数时，如果没有第二个参数，则默认返回false，即解锁当前互斥量并睡眠当前线程，直到其他线程使用成员函数`notify_one()`将本线程唤醒；
    - 当其他线程使用成员函数`notify_one()`将本线程唤醒后，如果没有第二个参数，则默认返回true，即继续持有之前获取的锁（互斥量），然后继续执行余下代码；
    - 在任何时候，如果有第二个参数，则使用谓词来判定应为true还是false

## 3  成员函数notify_one()

#### 3.1  函数作用

个人理解：成员函数`wait()`和成员函数`notify_one()`是用来实现线程间的同步的。例如：在执行B线程的特定代码前，需要先执行A线程的特定代码，因此需要在A线程执行完特定代码后采用V操作，即调用成员函数`notify_one()`，然后在执行B线程的特定代码前采用P操作，即成员函数`wait()`，最终即可实现A线程与B线程的同步。因此成员函数`notify_one()`具体是用来唤醒正在因成员函数`wait()`而睡眠的线程的。

#### 3.2  注意事项

延续上述例子，A线程需要在特定代码后执行成员函数`notify_one()`，B线程需要在特定代码前执行成员函数`wait()`。假如B线程执行了成员函数`wait()`后发现返回true，则可以直接执行特定代码而无需睡眠，之后CPU调度切换到A线程，A线程执行完特定代码后执行成员函数`notify_one()`，由于此时没有正在睡眠的线程，因此该唤醒操作无效，它唤醒不了任何线程，同时也不存在计数唤醒的功能。

## 4  代码示例

依照上述所讲，改进了之前的代码，如下所示：

```cpp
class A {
private:
	queue<int> msgQueue;
	mutex myMutex;
	condition_variable myCondition;  // 生成一个条件变量对象
public:
	// 把收到的消息传入队列中
	void inMsg() {
		for (int i = 0; i < 5000; i++) {
			unique_lock<mutex> myUniqueLock(myMutex);
			cout << "inMsg()执行，插入一个元素：" << i << endl;
			msgQueue.push(i);
			myCondition.notify_one();  // 尝试把wait线程唤醒
		}
	}

	// 把数据从消息队列中取出
	void outMsg() {
		int command = INT_MIN;
		while (true) {
			**unique_lock<mutex> myUniqueLock(myMutex);**
			/* 如果第二个参数的返回值是true，则wait()直接返回；
			 * 如果第二个参数的返回值是false，则wait()将解锁互斥量并堵塞到本行↓
			 * 直到其他某个线程调用notify_one()成员函数为止；
			 * 如果wait()没有第二个参数，则默认第二个参数直接返回false； 
			 */
			myCondition.wait(myUniqueLock, [this] {  // lambda表达式就是可调用对象
				if (!msgQueue.empty())
					return true;
				return false;
				}
			);
			command = msgQueue.front();  // 取出第一个元素给到command
			msgQueue.pop();  // 移除第一个元素
			myUniqueLock.unlock();  // 由于使用的是unique_lock，因此可以提前解锁，让其他线程继续执行
			/* 现在就是后序处理拿出的command的代码了... 这里不再给出 */
			cout << "outMsg()执行，取出一个元素：" << command << endl;
		}
	}
};

int main() {
	A myobj;
	thread outthread(&A::outMsg, ref(myobj));  // 第二个参数是引用，这样才能保证两个线程操纵的是同一个对象
	thread inthread(&A::inMsg, ref(myobj));
	outthread.join();
	inthread.join();
	return 0;
}
```

## 5  成员函数notify_all()

成员函数notify_one()每次只能唤醒一个线程，而成员函数notify_all()每次可以唤醒多个线程。