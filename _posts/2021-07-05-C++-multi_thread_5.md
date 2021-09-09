---
layout:     post
title:      C++11线程库使用（五）
subtitle:   unique_lock详解
date:       2021-07-5
author:     Zhgaot
header-img: img/C++/C++_cover.png
catalog: true
tags:
    - C++
    - C++11
    - multi-thread
---

> `unique_lock`是一个类模板，在实际工程中推荐使用`lock_guard`，但接收他人项目或阅读第三方代码时可能需要用到`unique_lock`

## 1  unique_lock取代lock_guard

`unique_lock`也是用于取代`lock()`和`unlock()`的，这点与`lock_guard`类似，但`unique_lock`在效率上会更差一点，在内存上占用更多一点。在缺省使用上，`unique_lock`可直接取代`lock_guard`，即对于上一小节的代码，可直接将`unique_lock替换为lock_guard`，如下所示：

```cpp
class A {
public:
	// 把收到的玩家命令入到队列中
	void inMsg() {
		for (int i = 0; i < 10000; i++) {
     std::unique_lock<mutex> myGuard(myMutex);  // 【上锁+解锁】
     cout << "inMsg()执行，插入一个元素：" << i << endl;
     msg.push(i);  // 假设数字i就是收到的玩家命令
		}
	}

	// 判断共享数据(缓冲区)是否为空，若不为空则修改数据
	bool outMsgLULProc(int& command) {
		if (!msg.empty()) {
			std::unique_lock<mutex> myGuard(myMutex);  // 【上锁+解锁】
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

## 2  unique_lock参数详解（都放在第二个参数的位置）

#### 2.1  std::adopt_lock（第二个参数的位置）

无论是`unique_lock`还是`lock_guard`，`std::adopt_lock`均放在**第二个参数的位置**。**该参数表示mutex对象已经被上锁（lock），则无需再次调用构造函数对mutex对象上锁。**因此在使用该参数时，必须要先把mutex对象提前上锁（lock），否则就会报异常。

#### 2.2  std::try_to_lock（第二个参数的位置）

使用了`std::try_to_lock`参数后，程序会尝试锁住mutex对象，但**如果没有上锁成功，则会立即返回而并不会阻塞在那里**。因此，使用`std::try_to_lock`的前提是：提前不能把mutex对象上锁。因为参数`std::adopt_lock`和`std::try_to_lock`二者的使用前提相悖，所以不能同时使用，在使用两者之一时一般都放在`unique_lock`的**第二个参数的位置**。

**在使用`std::try_to_lock`参数时，可以配合mutex对象的成员函数`owns_lock()`来使用，它的作用是检测该mutex对象是否成功上锁，返回布尔类型。**如下所示：

```cpp
class A {
public:
	// 把收到的玩家命令入到队列中
	void inMsg() {
		for (int i = 0; i < 10000; i++) {
			std::unique_lock<mutex> myGuard(myMutex, try_to_lock);  // 【尝试上锁+解锁】
			if (myGuard.owns_lock()) {  // 【尝试拿锁：拿到了锁】
				cout << "inMsg()执行，插入一个元素：" << i << endl;
				msg.push(i);  // 假设数字i就是收到的玩家命令
			}
			else {  // 【尝试拿锁：未拿到锁】
				cout << "inMsg执行，但未拿到锁......" << endl;
			}
		}
	}

	// 判断共享数据(缓冲区)是否为空，若不为空则修改数据
	bool outMsgLULProc(int& command) {
		if (!msg.empty()) {
			std::unique_lock<mutex> myGuard(myMutex);  // 【上锁+解锁】
			std::chrono::milliseconds duar(5000);  // 锁定5秒
			std::this_thread::sleep_for(duar);  // 锁定5秒
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

#### 2.3  std::defer_lock（第二个参数的位置）

同`std::try_to_lock`一样，使用`std::defer_lock`的前提是：不能先lock住mutex对象，否则会报异常。参数`std::defer_lock`的含义是：**初始化一个没有上锁的mutex对象，通过没有上锁的mutex对象，我们可以调用`unique_lock`的很多成员函数**。

## 3  unique_lock的成员函数

如上所述，使用`unique_lock`中的`std::defer_lock`参数，即可初始化一个未被上锁的mutex对象，使unique_lock对象能够使用很多成员函数。当然，即使不使用`std::defer_lock`参数依然可以使用`unique_lock`的成员函数。

#### 3.1  lock()与unlock()

注意：这是unique_lock对象所使用的`lock()`和`unlock()`，而非mutex对象所使用的`lock()`和`unlock()`。但其功能依然是上锁与解锁。另外，最后一次使用`lock()`上锁后，是不需要再使用`unlock()`解锁的，因为unique_lock会自动为其解锁。

**为什么使用`unlock()`呢？因为`lock()`锁住的代码段越少，整个程序的的运行效率就会高。**

例如以下代码可能是某个函数内部的代码：

```cpp
{
	std::unique_lock<mutex> myUniqueLock(myMutex, defer_lock);  // 【初始化了一个未上锁的mutex对象，并绑定在unique_lock对象上】
	myUniqueLock.lock();  // unique_lock对象【上锁】
	/* 开始执行一些应放于临界区内的代码...... */
	myUniqueLock.unlock();  // unique_lock对象【解锁】
	/* 开始执行一些剩余区内的代码...... */
	myUniqueLock.lock();  // unique_lock对象【上锁】
	/* 又！开始执行一些应放于临界区内的代码...... */
	return;
}
```

#### 3.2  try_lock()

成员函数`try_lock()`的作用是：**尝试给互斥量上锁，如果拿到了锁，则返回true，如果拿不到锁，则返回false，因此这个函数是不阻塞的。**通过其作用可知，该成员变量与`unique_lock`的参数`try_to_lock`的作用十分相似。示例如下所示（仅展示某个函数的变化）：

```cpp
// 把收到的玩家命令入到队列中
	void inMsg() {
		for (int i = 0; i < 10000; i++) {
			std::unique_lock<mutex> myGuard(myMutex, defer_lock); // 【初始化了一个未上锁的myMutex】
			if (myGuard.try_lock() == true) {  // 【尝试上锁且成功】
				cout << "inMsg()执行，插入一个元素：" << i << endl;
				msg.push(i);  // 假设数字i就是收到的玩家命令
			}
			else {  // 【尝试上锁但未成功】
				cout << "inMsg执行，但没有拿到锁......" << endl;
			}
		}
	}
```

#### 3.3  release()

成员函数`release()`的作用是：**返回它所管理的mutex对象的指针，并释放所有权；也就是说，unique_lock对象与mutex对象不再有关系。**

注意：要区分`unlock()`和`release()`。**如果在`release()`之前，unique_lock对象或者说mutex对象并未使用`unlock()`解锁，则在`release()`之后，一定要记得对mutex对象进行解锁！！！**如下所示：

```cpp
// 把收到的玩家命令入到队列中
	void inMsg() {
		for (int i = 0; i < 10000; i++) {
			std::unique_lock<mutex> myGuard(myMutex);  // myGuard与myMutex二者已绑定
			std::mutex *pMyMutex = myGuard.release();  // myGuard与myMutex的关系已解除
			cout << "inMsg()执行，插入一个元素：" << i << endl;
			msg.push(i);  // 假设数字i就是收到的玩家命令
			pMyMutex->unlock();  // 由于指针pMyMutex已接管了myMutex，因此有责任将其unlock
		}
	}
```

## 4  unique_lock所有权的传递

1. 【所有权概念】当执行`std::unique_lock<mutex> myUniqueLock(myMutex);`语句时，就把mutex对象`myMutex`与unique_lock对象`myUniqueLock`绑定在一起了！也可以说，`myUniqueLock`拥有`myMutex`的所有权。
2. 【所有权转移】`myUniqueLock`可以把自己对于`myMutex`的所有权转移给其他的unique_lock对象，**需要通过`std::move()`语句**。因此，unique_lock对象对于mutex对象的所有权只能转移，不能复制！如下所示：

    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/5_0.png)

    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/5_1.png)