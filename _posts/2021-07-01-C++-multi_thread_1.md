---
layout:     post
title:      C++11线程库使用（一）
subtitle:   线程创建与结束、join与detach
date:       2021-07-1
author:     Zhgaot
header-img: img/C++/C++_cover.png
catalog: true
tags:
    - C++
    - C++11
    - multi-thread
---

# 线程启动与结束、创建线程的方法：join与detach

# 1 线程的创建及运行

## 1.1 `thread` —— 创建线程

1. 调用C++11里的标准库：thread

    ```cpp
    #include <thread>
    ```

2. 通过可调用对象创建线程

    ```cpp
    thread myObj(myPrint);
    /**
     * 1. 这句代码创建了线程，并且线程执行的起点（入口）是myPrint函数；
     * 2. 从这句开始，myPrint函数开始执行；
     * 3. myPrint是一个可调用对象，可以是函数、类对象、lambda表达式
    **/
    ```

## 1.2 `join()` —— 阻塞主线程

1. 函数作用：阻塞主线程，让主线程等待子线程执行完毕后，主线程再继续向下运行
2. 代码示例：

    ```cpp
    thread myObj(myPrint);
    myObj.join();  // 阻塞主线程main，直到子线程myPrint执行完毕
    ```

3. 如果不使用`join()`函数，程序将会报错（但依然可以运行）；这意味着主线程与子线程分别运行，可能主线程会提前结束，主线程结束代表进程结束，然后则会强行杀死子线程，让子线程内部未运行完的代码不再运行。

## 1.3 `detach()` —— 分离主线程与子线程

1. 函数作用：分离主线程与子线程，使二者不会再回合、使二者失去关联，主线程不必再等待子线程执行完毕；一旦使用`detach()`函数，子线程就会驻留在后台运行（守护线程），当子线程运行完成后，由**运行时库**负责清理该线程相关的资源。
2. 代码示例：

    ```cpp
    thread myObj(myPrint);
    myObj.detach();  // 分离主线程main与子线程myPrint
    ```

3. 【注意】一旦使用detach()函数，就不能够再使用join()函数了，否则系统会报告异常

## 1.4 `joinable()` —— 判断

1. 函数作用：判断是否可以成功使用`join()`函数或`detach()`函数。
2. 返回值：bool类型，true/false
3. 代码示例：

    ```cpp
    thread myObj(myPrint);
    if (myObj.joinable()) {
        myObj.join();
    }
    ```

# 2 其他创建线程的方法

## 2.1 用类对象创建线程

1. 基础操作：

    ```cpp
    class A{
    public:
        void operator()(){  // 简单的、不带参数的
            cout << "线程开始执行" << endl;
            /*......*/
            cout << "线程结束执行" << endl;
        }
    };
    int main() {
        A a;
        thread myobj(a);  // 创建线程，使用可调用对象(类对象)
        myobj.join();  // 阻塞主线程等待子线程结束
        cout << "主线程结束" << endl;
        return 0;
    }
    ```

2. 指针或引用+`detach()`所产生的隐患：

    ```cpp
    class A{
    public:
        int& _i;  // 成员变量是一个引用(必须要在构造函数的初始化列表中进行初始化)
    public:
        A(int& i) : _i(i) {}  // 构造函数：传入参数也是引用
        void operator()(){  // 简单的、不带参数的
            for (int k = 0; k < 10; k++){
                cout << "i的值为：" << i << endl;
            }
        }
    };
    int main() {
        int i = 10;
        A a(i);  // 将i引用传递
        thread myobj(a);  // 创建线程，使用可调用对象(类对象)
        myobj.detach();  // 使用detach()分离子线程与主线程
        cout << "主线程结束" << endl;
        return 0;
    }
    ```

    上述代码中，由于使用了`detach()`函数，则子线程与主线程分离，即主线程不必等待子线程运行结束后才继续运行，所以很可能出现主线程先执行完的情况；当主线程先执行完，`main()`函数中的局部变量`i`就会被释放，而传入给可调用对象`a`的引用(指针常量)就会失效，会造成不可预料的后果。因此**不可向可调用对象中进行地址传递或引用传递并调用detach()函数**。

3. 局部对象释放后子线程依然能运行吗？

    对于2中的代码，会出现疑问：如果使用了detach()函数，主线程可能会先执行完，则局部变量a对象也会被释放，那创建的子线程还能继续运行下去吗？

    答案是肯定的！局部变量a对象在主线程运行结束后确实是会被释放，但在创建子线程myobj的时候其实是复制了一份a对象到子线程中；即执行完主线程后，a被销毁，但复制的a依然存在。下面是证明：

    ```cpp
    class A {
    public:
        int _i;  // 成员变量是一个引用(必须要在构造函数的初始化列表中进行初始化)
    public:
        // 构造函数
        A(int i) : _i(i) {
            cout << "A的构造函数被调用" << endl;
        }
        // 拷贝构造函数
        A(const A& a) {
            this->_i = a._i;
            cout << "A的拷贝构造函数被调用" << endl;
        }
        // 析构函数
        ~A() {
            cout << "A的析构函数被调用" << endl;
        }
        // 简单的、不带参数的成员函数
        void operator()() {
            for (int k = 0; k < 10; k++) {
                cout << "i的值为：" << _i << endl;
            }
        }
    };
    int main() {
        int i = 10;
        A a(i);
        thread myobj(a);  // 创建线程，使用可调用对象(类对象)
        myobj.join();
        cout << "主线程结束" << endl;
        return 0;
    }
    ```

    上述代码运行结果：

    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/1_0.png)

## 2.2 用lambda表达式创建线程

```cpp
int main() {
    auto myLambdaThread = [] {
        cout << "我的线程开始执行" << endl;
        cout << "我的线程结束执行" << endl;
    };
    thread myobj(myLambdaThread);
    myobj.detach();
    cout << "主线程结束" << endl;
    return 0;
}
```