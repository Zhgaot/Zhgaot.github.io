---
layout:     post
title:      C++11线程库使用（八）
subtitle:   async & future & packaged_task & promise
date:       2021-08-8
author:     Zhgaot
header-img: img/C++/C++_cover.png
catalog: true
tags:
    - C++
    - C++11
    - multi-thread
---

## 1  std::async与std::future创建后台任务并接收返回值

#### 1.1  作用描述

之前创建一个线程，并执行线程入口函数，有的时候需要返回结果；但是返回的结果没办法直接返回，而一般需要赋值在全局变量上，或者传入引用或者指针直接在线程中改变某变量的值。

1. `std::async`是一个函数模板，它用来启动一个异步任务。异步任务指自动创建一个线程并开始执行对应的线程入口函数
2. 启动一个异步任务之后，它返回一个`std::future`类类型的对象，`std::future`是一个类模板，异步任务所返回的`std::future`对象里面就包含有线程入口函数所返回的结果；最后，我们可以通过调用`std::future`对象的成员函数`get()`来获取线程入口函数所返回的结果

#### 1.2  代码示例

1. 使用普通函数作为线程入口：

    ```cpp
    #include <thread>
    #include <mutex>
    #include <iostream>
    #include <future>  // 使用future需要导入future库
    using namespace std;

    int myThread(const int& num) {
    	cout << "myThread()函数【开始执行】，" << "线程id = " << std::this_thread::get_id() << endl;  // 打印子线程id
    	std::chrono::milliseconds dura(5000);  // 假装子线程内的代码需要执行5秒
    	std::this_thread::sleep_for(dura);
    	cout << "myThread()函数【结束执行】，" << "线程id = " << std::this_thread::get_id() << endl;  // 打印子线程id
    	return num + 5;
    }

    int main() {
    	cout << "主线程开始执行，线程id = " << std::this_thread::get_id() << endl;
    	int num = 5;
    	std::future<int> result = std::async(myThread, num);  // 流程并不会卡在这里！！！
    	cout << "continue......" << endl;  // 程序将继续执行后序代码
    	cout << result.get() << endl;  // 程序将卡在这里等待myThread执行完毕，直到拿到结果，因此一定要保证子线程必须返回值，否则将永远卡死在这里
    	/*cout << result.get() << endl;*/  // 第二次调用.get()成员函数将报错
      result.wait();  // 等待线程返回，但本身并不返回结果
    	cout << "主线程执行结束" << endl;
    	return 0;
    }
    ```

2. 使用类的成员函数作为线程入口：

    ```cpp
    #include <thread>
    #include <mutex>
    #include <iostream>
    #include <future>  // 使用future需要导入future库
    using namespace std;

    class MyThreadClass {
    public:
    	int myThread(const int& num) {
    		cout << "myThread()函数【开始执行】，" << "线程id = " << std::this_thread::get_id() << endl;  // 打印子线程id
    		std::chrono::milliseconds dura(5000);  // 假装子线程内的代码需要执行5秒
    		std::this_thread::sleep_for(dura);
    		cout << "myThread()函数【结束执行】，" << "线程id = " << std::this_thread::get_id() << endl;  // 打印子线程id
    		return num + 5;
    	}
    };

    int main() {
    	cout << "主线程开始执行，线程id = " << std::this_thread::get_id() << endl;
    	int num = 5;
    	MyThreadClass myThreadObj;
    	std::future<int> result = std::async(&MyThreadClass::myThread, ref(myThreadObj), num);  // 流程并不会卡在这里！！！
    	cout << "continue......" << endl;
    	cout << result.get() << endl;  // 程序将卡在这里等待myThread执行完毕，直到拿到结果，因此一定要保证子线程必须返回值，否则将永远卡死在这里
    	cout << "主线程执行结束" << endl;
    	return 0;
    }
    ```

3. 看一下返回结果（两图之间相差大于等于5miao）：

    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/8_0.png)

    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/8_1.png)

#### 1.3  注意事项

1. 使用`std::future`类模版以及`std::async`函数模版时，首先需要导入`<future>`库；
2. `.get()`函数不可调用两次，第二次将报错；
3. `std::future`还有一个成员函数叫`.wait()`，与`.get()`函数不同，它仅仅等待线程返回，本身并不返回结果；
4. 如果不用`.get()`函数，程序依然是等待线程执行完毕才退出，但是还是使用`.get()`函数比较稳妥；

#### 1.4  std::launch::defered 和 std::launch::async

1. `std::launch::defered`与`std::launch::async`是两个用于放在`std::async`中的参数。
2. 如果使用`std::launch::defered`则线程入口函数会直到`.get()`函数处才开始执行，实际上如果使用此参数，主线程根本没有创建一个新的线程，而是直接将线程入口函数放在主线程中执行！
3. 如果使用`std::launch::async`则与不加参数效果相同。

## 2  std::packaged_task

#### 2.1  作用描述

`std::packaged_task`也是一个类模板，它的模板参数是各类可调用对象，通过`std::packaged_task`就可以将各类可调用对象包装起来，方便将来作为线程入口函数来调用。

另外，`packaged_task`包装起来的可调用对象还可以直接调用，所以从这个角度讲，`pakcaged_task`对象也是一个可调用对象。

如果要获取`std::packaged_task`的对象的结果，需要通过`.get_future()`成员函数以及`future`类的对象来获得，也就是说，**该类模板常常需要与future联合使用！！！**

#### 2.2  代码示例

1. 向packaged_task传入函数：

    ```cpp
    int myThread(const int& num) {
    	cout << "myThread()函数【开始执行】，" << "线程id = " << std::this_thread::get_id() << endl;  // 打印子线程id
    	std::chrono::milliseconds dura(5000);  // 假装子线程内的代码需要执行5秒
    	std::this_thread::sleep_for(dura);
    	cout << "myThread()函数【结束执行】，" << "线程id = " << std::this_thread::get_id() << endl;  // 打印子线程id
    	return num + 5;
    }

    int main() {
    	cout << "主线程开始执行，线程id = " << std::this_thread::get_id() << endl;
    	std::packaged_task<int(int)> mypt(myThread);  // 我们把函数myThread通过packaged_task包装起来
    	std::thread mythreadObj(ref(mypt), 10);  // 创建线程并开始执行
    	mythreadObj.join();
    	std::future<int> result = mypt.get_future();  // 使用packaged_task包装后就有了get_future()接口，即可以获得future对象
    	cout << result.get() << endl;
    	cout << "主线程执行结束" << endl;
    	return 0;
    }
    ```

2. 向packaged_task传入lambda表达式：

    ```cpp
    int main() {
    	cout << "主线程开始执行，线程id = " << std::this_thread::get_id() << endl;
    	std::packaged_task<int(int)> mypt([](int num) {
    		cout << "myThread()函数【开始执行】，" << "线程id = " << std::this_thread::get_id() << endl;  // 打印子线程id
    		std::chrono::milliseconds dura(5000);  // 假装子线程内的代码需要执行5秒
    		std::this_thread::sleep_for(dura);
    		cout << "myThread()函数【结束执行】，" << "线程id = " << std::this_thread::get_id() << endl;  // 打印子线程id
    		return num + 5;
    		});  // 我们把lambda表达式通过packaged_task包装起来
    	std::thread mythreadObj(ref(mypt), 10);  // 创建线程并开始执行
    	mythreadObj.join();
    	std::future<int> result = mypt.get_future();  // 使用packaged_task包装后就有了get_future()接口，即可以获得future对象
    	cout << result.get() << endl;
    	cout << "主线程执行结束" << endl;
    	return 0;
    }
    ```

## 3  std::promise

#### 3.1  作用描述

`std::promise`也是一个类模板，其作用为我们能够在某个线程中给它赋值，然后我们可以在其他线程中把这个值取出来用。即，**能够实现两个线程之间的数据传递。**

如果要获取`std::promise`的对象内保存的值，需要调用`.get_future()`成员函数以及`future`对象来获取，**也就是说该类模板常常与future联合使用！！！**

#### 3.2  代码示例

```cpp
void myThread1(std::promise<int>& temp, int calc) {
	/* 假装这里在进行复杂运算 */
	calc = ((calc++) * 10 - 200) / 5;
	std::chrono::milliseconds dura(5000);  // 假装子线程内的代码需要执行5秒
	std::this_thread::sleep_for(dura);
	int result = calc;  // 保存结果
	temp.set_value(result);  // 直接将结果保存至promise的对象中
}

void myThread2(std::future<int>& myFuture) {
	**int result = myFuture.get();**
	cout << "在子线程myThread1中计算得到的值，在子线程myThread2中打印输出：" << result << endl;
}

int main() {
	/* promise能够在某个线程中被赋值，然后可以在其他线程中把这个值取出来用 */
	std::promise<int> myProm;  // 声明一个promise对象，保存的值的类型为int型
	thread myThreadObj1(myThread1, ref(myProm), 100);
	myThreadObj1.join();
	/* 获取结果值 */
	future<int> myFuture = myProm.get_future();  // promise对象与future对象绑定，可以获取线程的返回值
	thread myThreadObj2(myThread2, ref(myFuture));
	myThreadObj2.join();
	return 0;
}
```