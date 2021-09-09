---
layout:     post
title:      C++11线程库使用（四）
subtitle:   互斥量概念与用法 & 死锁演示及解决
date:       2021-07-4
author:     Zhgaot
header-img: img/C++/C++_cover.png
catalog: true
tags:
    - C++
    - C++11
    - multi-thread
---

## 1  互斥量(mutex)的概念

互斥量是一个类对象，可理解为一把锁，多个线程尝试用`lock()`成员函数来上锁，但只有一个线程可以锁成功，成功的表示是`lock()`函数返回；若没有上锁成功，则对某一线程来说，流程将卡在`lock()`函数这里不断地去尝试上锁。互斥量的使用要小心，保护的数据不能多不能少。

## 2  互斥量的用法

#### 2.1  成员函数`lock()`和`unlock()`

1. 需使用`<mutex>`互斥量库：`#include <mutex>`
2. 需使用成员函数`lock()`和`unlock()`
3. 步骤：先上锁`lock()` => 操作共享数据 => 再解锁`unlock()`
4. 注意：`lock()`和`unlock()`必须**对称使用**，并不是一个`lock()`就对应一个`unlock()`，对某段共享数据`lock()`上锁后，对后面的每一个分支(例如if-else分支)均需要`unlock()`解锁
5. 示例：

    利用互斥量解决上一小节的【生产者-消费者问题】：

    ```cpp
    class A {
    public:
    	// 把收到的玩家命令入到队列中
    	void inMsg() {
    		for (int i = 0; i < 10000; i++) {
    			cout << "inMsg()执行，插入一个元素：" << i << endl;
    			myMutex.lock();  // 【上锁】
    			msg.push(i);  // 假设数字i就是收到的玩家命令
    			myMutex.unlock();  // 【解锁】
    		}
    	}

    	// 判断共享数据(缓冲区)是否为空，若不为空则修改数据
    	bool outMsgLULProc(int& command) {
    		myMutex.lock();  // 【上锁】
    		if (!msg.empty()) {
    			command = msg.front();  // 返回第一个元素
    			msg.pop();  // 移除第一个元素但不返回
    			myMutex.unlock();  // 【解锁】所有分支均要有unlock
    			return true;
    		}
    		myMutex.unlock();  // 【解锁】所有分支均要有unlock
    		return false;
    	}

    	// 把数据从队列中取出
    	void outMsg() {
    		for (int i = 0; i < 10000; i++) {
    			int command = INT_MIN;
    			bool temp = outMsgLULProc(command);
    			if (temp) {
    				cout << "outMsg()执行，取出一个元素：" << command << endl;
    				/* 这里可以对command进行处理... */
    			}
    			else {
    				cout << "outMsg()执行，但目前消息队列为空：" << i << endl;
    			}
    		}
    		cout << "end" << endl;
    	}
    private:
    	queue<int> msg;  // 专门用于代表玩家发送过来的命令
    	mutex myMutex;  // 创建了一个互斥量
    };

    int main() {
    	A a;
    	thread outObj(&A::outMsg, std::ref(a));
    	thread inObj(&A::inMsg, std::ref(a));
    	outObj.join();
    	inObj.join();
    	return 0;
    }
    ```

    其实按照标准的【生产者-消费者问题】模式，对于上述代码中的`bool outMsgLULProc(int& command)`函数，只需在if判断的内部上锁解锁即可，如下所示：

    ```cpp
    bool outMsgLULProc(int& command) {
    		if (!msg.empty()) {
    			myMutex.lock();  // 【上锁】
    			int command = msg.front();  // 返回第一个元素
    			msg.pop();  // 移除第一个元素但不返回
    			myMutex.unlock();  // 【解锁】
    			return true;
    		}
    		return false;
    	}
    ```

#### 2.2  类模板`std::lock_guard<std::mutex>`

1. 基本使用：

    如同智能指针一样，互斥量的上锁和解锁也为了以防程序员忘记解锁而提供了自动解锁的方法，即类模板`std::lock_guard<std::mutex>`。类模板`std::lock_guard`直接取代`lock()`和`unlock()`，也就是说使用了`std::lock_guard`之后，就不能再使用`lock()`和`unlock()`了，如下所示，其实就是将上述代码删减了lock()和unlock()而添加了类模板std::lock_guard而已：

    ```cpp
    class A {
    public:
    	// 把收到的玩家命令入到队列中
    	void inMsg() {
    		for (int i = 0; i < 10000; i++) {
    			cout << "inMsg()执行，插入一个元素：" << i << endl;
    			std::lock_guard<mutex> myGuard(myMutex);  // 【上锁+解锁】
    			msg.push(i);  // 假设数字i就是收到的玩家命令
    		}
    	}

    	// 判断共享数据(缓冲区)是否为空，若不为空则修改数据
    	bool outMsgLULProc(int& command) {
    		if (!msg.empty()) {
    			std::lock_guard<mutex> myGuard(myMutex);  // 【上锁+解锁】
    			command = msg.front();  // 返回第一个元素
    			msg.pop();  // 移除第一个元素但不返回
    			return true;
    		}
    		return false;
    	}

    	// 把数据从队列中取出
    	void outMsg() {
    		for (int i = 0; i < 10000; i++) {
    			int command = INT_MIN;
    			bool temp = outMsgLULProc(command);
    			if (temp) {
    				cout << "outMsg()执行，取出一个元素：" << command << endl;
    				/* 这里可以对command进行处理... */
    			}
    			else {
    				cout << "outMsg()执行，但目前消息队列为空：" << i << endl;
    			}
    		}
    		cout << "end" << endl;
    	}
    private:
    	queue<int> msg;  // 专门用于代表玩家发送过来的命令
    	mutex myMutex;  // 创建了一个互斥量
    };

    int main() {
    	A a;
    	thread outObj(&A::outMsg, std::ref(a));
    	thread inObj(&A::inMsg, std::ref(a));
    	outObj.join();
    	inObj.join();
    	return 0;
    }
    ```

2. 上锁与解锁时机：

    在上述代码中，用类模板`std::lock_guard<std::mutex>`创造对象时，会调用构造函数，而构造函数会调用lock()函数；当对象需被释放时，会调用析构函数，而析构函数会调用unlock()函数，这即是该类模板的原理。即，上锁时机就是对象创建时，解锁实际是对象析构时。

    如上述代码所示，类模板`std::lock_guard<std::mutex>`所创造的对象可能在函数调用结束时析构，也可能在for循环结束时析构，因此可以使用`{}`来控制上锁区域，即把对共享数据、临界资源的操纵代码放入`{}`中，这样在`}`时则可以析构对象；这种方法很适用于后序还有很多代码需要执行，且后序代码必须在函数或循环内部执行时使用。如下所示（简单情况）：

    ```cpp
    void inMsg() {
    		for (int i = 0; i < 10000; i++) {
    			cout << "inMsg()执行，插入一个元素：" << i << endl;
    			{
    				std::lock_guard<mutex> myGuard(myMutex);  // 【上锁+解锁】
    				msg.push(i);  // 假设数字i就是收到的玩家命令
    			}
    		}
    	}
    ```

## 3  死锁

#### 3.1  死锁演示

**死锁产生的前提条件是必须有两把锁，即两个互斥量。**如下所示：

```cpp
class A {
public:
	// 把收到的玩家命令入到队列中
	void inMsg() {
		for (int i = 0; i < 10000; i++) {
			cout << "inMsg()执行，插入一个元素：" << i << endl;
			myMutex1.lock();  // 实际工程中这两句不一定挨着，可能它们需要保护不同的临界区
			myMutex2.lock();
			msg.push(i);  // 假设数字i就是收到的玩家命令
			myMutex2.unlock();
			myMutex1.unlock();
		}
	}

	// 判断共享数据(缓冲区)是否为空，若不为空则修改数据
	bool outMsgLULProc(int& command) {
		if (!msg.empty()) {
			myMutex2.lock();
			myMutex1.lock();
			command = msg.front();  // 返回第一个元素
			msg.pop();  // 移除第一个元素但不返回
			myMutex1.unlock();
			myMutex2.unlock();
			return true;
		}
		return false;
	}

	// 把数据从队列中取出
	void outMsg() {
		for (int i = 0; i < 10000; i++) {
			int command = INT_MIN;
			bool temp = outMsgLULProc(command);
			if (temp) {
				cout << "outMsg()执行，取出一个元素：" << command << endl;
				/* 这里可以对command进行处理... */
			}
			else {
				cout << "outMsg()执行，但目前消息队列为空：" << i << endl;
			}
		}
		cout << "end" << endl;
	}
private:
	queue<int> msg;  // 专门用于代表玩家发送过来的命令
	mutex myMutex1;
	mutex myMutex2;
};

int main() {
	A a;
	thread outObj(&A::outMsg, std::ref(a));
	thread inObj(&A::inMsg, std::ref(a));
	outObj.join();
	inObj.join();
	return 0;
}
```

#### 3.2  死锁的一般解决方案

只要保证两个互斥量上锁的顺序一致，就不会造成死锁。

#### 3.3  函数模板`std::lock(..., ..., ...)`

函数模板`std::lock()`在多个互斥量时才使用，它能够一次锁住两个或者两个以上的互斥量（至少两个），它的存在是用于解决因为上锁的顺序问题而导致的死锁问题。如果互斥量中有一个没有锁住，它就先令线程释放掉之前锁住的互斥量，再令线程等待，等所有互斥量均锁住，线程才能向下运行。

修改部分上述代码，如下所示：

```cpp
void inMsg() {
		for (int i = 0; i < 10000; i++) {
			cout << "inMsg()执行，插入一个元素：" << i << endl;
			lock(myMutex1, myMutex2);
			msg.push(i);  // 假设数字i就是收到的玩家命令
			myMutex2.unlock();
			myMutex1.unlock();
		}
	}

	// 判断共享数据(缓冲区)是否为空，若不为空则修改数据
	bool outMsgLULProc(int& command) {
		if (!msg.empty()) {
			lock(myMutex1, myMutex2);
			command = msg.front();  // 返回第一个元素
			msg.pop();  // 移除第一个元素但不返回
			myMutex1.unlock();
			myMutex2.unlock();
			return true;
		}
		return false;
	}

	// 把数据从队列中取出
	void outMsg() {
		for (int i = 0; i < 10000; i++) {
			int command = INT_MIN;
			bool temp = outMsgLULProc(command);
			if (temp) {
				cout << "outMsg()执行，取出一个元素：" << command << endl;
				/* 这里可以对command进行处理... */
			}
			else {
				cout << "outMsg()执行，但目前消息队列为空：" << i << endl;
			}
		}
		cout << "end" << endl;
	}
```

通过这种方式避免了死锁的发生，但依然需要手动unlock。

函数模板`std::lock()`是一次锁住多个互斥量，但一般工程上不会两个`lock()`连在一起，一次使用情况不多，谨慎使用。

#### 3.4  `std::lock_guard`的`std::adopt_lock`参数

`std::adopt_lock`是一个结构体对象，起一个标记作用，一般写入`std::lock_guard`的参数中，作用就是表示该互斥量已经lock过了，不需要在`std::lock_guard`的构造函数中再调用`lock()`了，主要就可以解决3.3小节遗留下的问题。一般将`std::lock_guard`语句放在函数模板`std::lock()`语句后并在参数中加入`adopt_lock`即可，如下所示：

```cpp
class A {
public:
	// 把收到的玩家命令入到队列中
	void inMsg() {
		for (int i = 0; i < 10000; i++) {
			cout << "inMsg()执行，插入一个元素：" << i << endl;
			lock(myMutex1, myMutex2);
			lock_guard<mutex> myGuard1(myMutex1, adopt_lock);
			lock_guard<mutex> myGuard2(myMutex2, adopt_lock);
			msg.push(i);  // 假设数字i就是收到的玩家命令
			//myMutex2.unlock();
			//myMutex1.unlock();
		}
	}

	// 判断共享数据(缓冲区)是否为空，若不为空则修改数据
	bool outMsgLULProc(int& command) {
		if (!msg.empty()) {
			lock(myMutex1, myMutex2);
			lock_guard<mutex> myGuard1(myMutex1, adopt_lock);
			lock_guard<mutex> myGuard2(myMutex2, adopt_lock);
			command = msg.front();  // 返回第一个元素
			msg.pop();  // 移除第一个元素但不返回
			//myMutex1.unlock();
			//myMutex2.unlock();
			return true;
		}
		return false;
	}

	// 把数据从队列中取出
	void outMsg() {
		for (int i = 0; i < 10000; i++) {
			int command = INT_MIN;
			bool temp = outMsgLULProc(command);
			if (temp) {
				cout << "outMsg()执行，取出一个元素：" << command << endl;
				/* 这里可以对command进行处理... */
			}
			else {
				cout << "outMsg()执行，但目前消息队列为空：" << i << endl;
			}
		}
		cout << "end" << endl;
	}
private:
	queue<int> msg;  // 专门用于代表玩家发送过来的命令
	mutex myMutex1;
	mutex myMutex2;
};

int main() {
	A a;
	thread outObj(&A::outMsg, std::ref(a));
	thread inObj(&A::inMsg, std::ref(a));
	outObj.join();
	inObj.join();
	return 0;
}
```