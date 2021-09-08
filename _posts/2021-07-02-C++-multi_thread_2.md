---
layout:     post
title:      C++11线程库使用（二）
subtitle:   detach与线程传参 & 成员函数做线程函数
date:       2021-07-2
author:     Zhgaot
header-img: img/C++/C++_cover.png
catalog: true
tags:
    - C++
    - C++11
    - multi-thread
---

## 1  传递临时对象作为线程参数

#### 1.1  使用`detach()`所导致的线程参数地址问题

```cpp
void myPrint(const int& i, char* pbuf) {  // 参数分别使用引用和指针传参，这将带来隐患
    cout << i << endl;  // 【断点】
    cout << pbuf << endl;
}
int main() {
    int mvar = 1;  // 【断点】逐语句调试
    int& mvary = mvar;  // mvary用于对比传入线程的mvar的地址
    char buf[] = "This is a test!";
    thread myobj(myPrint, mvar, buf);
    myobj.detach();  // 主线程与子线程分离
    cout << "主线程即将结束..." << endl;
    return 0;
}
```

如上述代码所示：第一个子线程参数`i`是引用，第二个线程参数`pbuf`是指针，同时主线程中使用了`detach()`函数将主线程与子线程分离，这将导致主线程可能先执行完，传入子线程的`mvar`和`buf`均被释放，那么子线程后执行时再去取`i`和`pbuf`的地址时，是不是就会取到了不确定的地址了呢？

1. 首先观察参数`mvar`和`i`：

    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/2_0.png)

    debug调试后可知：`mvar`和`mvary`的地址相同，而传入子线程的参数`mvar`与`i`竟然不同！**这说明对于引用，子线程在创建的时候，是复制了一份实参给了形参！** **因此，对于detach()函数和引用的合并使用，是安全的，但不建议这样做！**

2. 其次观察参数`buf`和`pbuf`：

    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/2_1.png)

    debug调试后可知：传入子线程的参数`buf`与`pbuf`竟然相同！那使用`detach()`函数时，如果主线程先执行完了，`buf`被系统回收，此时`pbuf`再去寻`buf`的地址，将不再安全！

**总结：在使用detach时，不推荐用引用，绝对不可用指针！**

#### 1.2  使用`detach()`时如何安全的传递？

1. 首先直观地改变之前的代码：

    ```cpp
    void myPrint(const int i, const string& pbuf) {
        /* i不再使用引用传递；
         * pbuf使用string接收传入的字符串数组
         * 这里pbuf使用const是因为传进来的buf是一个临时变量，const引用可以接收右值，这里也可以使用string&&
         */
        cout << i << endl;
        cout << pbuf << endl;
    }

    int main() {
        int mvar = 1;
        int& mvary = mvar;  // mvary用于对比传入线程的mvar的地址
        char buf[] = "This is a test!";
        thread myobj(myPrint, mvar, buf);
        myobj.detach();
        cout << "主线程即将结束..." << endl;
        return 0;
    }
    ```

    这样修改之后，`pbuf`的地址和`buf`的地址就不一样了。

2. `buf`是何时被转换成`pbuf`的？

    如果`buf`是可能在主线程运行结束后才被转换成`pbuf`的，那依然存在buf先被释放，而pbuf只不过是复制了一份被系统回收的内存进而继续执行而已的问题。经实验证明，确实会出现主线程运行结束才将`buf`转换为`pbuf`，因此上述代码依然有问题！

3. 向子线程传参是传入临时对象以解决问题

    通过查资料发现，向子线程传入临时对象时，就不会出现直到主线程运行结束后才在子线程内创建pbuf的情况，代码如下所示：

    ```cpp
    void myPrint(const int i, const string& pbuf) {
        cout << i << endl;
        cout << pbuf << endl;
    }

    int main() {
        int mvar = 1;
        int& mvary = mvar;  // mvary用于对比传入线程的mvar的地址
        char buf[] = "This is a test!";
        thread myobj(myPrint, mvar, **string(buf)**);  **// 构造临时对象并传入子线程**
        myobj.detach();
        cout << "主线程即将结束..." << endl;
        return 0;
    }
    ```

#### 1.3  证明上述结果

1. 【证明】如果传入的不是临时对象，则可能会出现主线程先运行结束，子线程才构造对象的情况

    ```cpp
    /* 自定义一个类A来代替上面的string进行说明 */
    class A {
    public:
        int _i;
    public:
        A(int i) : _i(i) { cout << "A的【构造函数】执行" << endl; }
        A(const A& a) : _i(a._i) { cout << "A的【拷贝构造函数】执行" << endl; }
        ~A() { cout << "A的【析构函数】执行" << endl; }
    };
    /* 子线程：打印一下子线程所生成的对象a的地址 */
    void printAdds(const A& a) {
        cout << "a对象的地址是：" << &a << endl;
    }
    /* 主线程 */
    int main() {
        int m = 1;
        int n = 1;
        thread myobj(printAdds, n);  // 希望int类型的n转换成类类型A的对象，并在printAdds函数中使用
        myobj.detach();
        cout << "主线程即将结束..." << endl;
        return 0;
    }
    ```

    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/2_2.png)

    执行结果证明，很多时候，主线程先运行完毕，子线程才会构造对象，而且因为主线程运行完毕，所以子线程构造对象所打印的语句不会显示出来。

2. 【证明】**如果传入的是临时对象，则一定会在主线程运行结束前构造对象**

    ```cpp
    class A {
    public:
        int _i;
    public:
        A(int i) : _i(i) { cout << "A的【构造函数】执行" << "，当前对象的地址为：" << this << endl; }
        A(const A& a) : _i(a._i) { cout << "A的【拷贝构造函数】执行" << "，当前对象的地址为：" << this << endl; }
        ~A() { cout << "A的【析构函数】执行" << "，当前对象的地址为：" << this << endl; }
    };

    void printAdds(const A& a) {
        cout << "a对象的地址是：" << &a << endl;
    }

    int main() {
        int m = 1;
        int n = 1;
        thread myobj(printAdds, **A(n)**);  // 构造临时对象传入子线程
        myobj.detach();
        cout << "主线程即将结束..." << endl;
        return 0;
    }
    ```

    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/2_3.png)

    执行结果证明，**如果传入子线程的是一个临时对象，则一定会先构造完毕再传入子线程**，这样安全很多。但同时也发现，这里执行了【构造函数】和【拷贝构造函数】，表明在构造了临时对象后，子线程还拷贝了一份对象在线程中使用，而不是直接使用传入的临时对象，这和前述的`int& i`类似。

#### 1.4  结论与建议

1. **若传递类似int这类简单类型的参数，则建议使用值传递，不要使用引用**，以访节外生枝。
2. **如果传递类对象，避免隐式类型转换**(例如`char* => string`或者`int => class A`)。**应全部都在创建线程这一行就构建出临时对象传入子线程，然后在可调用对象形参里使用引用类接**(如果不用引用，则系统还会拷贝一个对象，浪费！)
3. 建议：非必要不适用detach()函数，只使用join()函数，这样就不存在局部变量失效导致线程对内存的非法引用问题。

#### 1.5  线程id与临时对象构造时机的捕获 —— 进一步的证明（选看）

1. 线程id

    每个线程（无论时主线程还是子线程）都对应着一个线程id，每个线程对应的id不同，线程id可以使用C++标准库里的函数`std::this_thread::get_id()`获取。

2. 不向子线程传入临时对象

    使用`join()`函数进行测试，并根据【构造函数】、【拷贝构造函数】、【析构函数】、【子线程函数】、【主线程】的线程id，观察类A的对象是在何时构造的、在何时拷贝构造的

    ```cpp
    class A {
    public:
        int _i;
    public:
        A(int i) : _i(i) { cout << "A的【构造函数】执行" << "，当前对象的地址为：" << this << "，线程id为：" << std::this_thread::get_id() << endl; }
        A(const A& a) : _i(a._i) { cout << "A的【拷贝构造函数】执行" << "，当前对象的地址为：" << this << "，线程id为：" << std::this_thread::get_id() << endl; }
        ~A() { cout << "A的【析构函数】执行" << "，当前对象的地址为：" << this << "，线程id为：" << std::this_thread::get_id() << endl; }
    };

    void printAdds(const A& a) {
        cout << "a对象的地址是：" << &a << "，线程id为：" << std::this_thread::get_id() << endl;
    }

    int main() {
        cout << "主线程id为：" << std::this_thread::get_id() << endl;
        int m = 1;
        int n = 1;
        thread myobj(printAdds, n);
        myobj.join();
        return 0;
    }
    ```

    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/2_4.png)

    根据执行结果：如果不向子线程传递临时对象，则类的对象是在**子线程**中构造的(根据线程id即可知晓)。假如此时使用的是`detach()`函数，则会出现主线程退出后才在子线程根据主线程已经释放的资源来构造子线程所需的对象，危险！

3. 向子线程传入临时对象

    ```cpp
    class A {
    public:
        int _i;
    public:
        A(int i) : _i(i) { cout << "A的【构造函数】执行" << "，当前对象的地址为：" << this << "，线程id为：" << std::this_thread::get_id() << endl; }
        A(const A& a) : _i(a._i) { cout << "A的【拷贝构造函数】执行" << "，当前对象的地址为：" << this << "，线程id为：" << std::this_thread::get_id() << endl; }
        ~A() { cout << "A的【析构函数】执行" << "，当前对象的地址为：" << this << "，线程id为：" << std::this_thread::get_id() << endl; }
    };

    void printAdds(const A& a) {
        cout << "a对象的地址是：" << &a << "，线程id为：" << std::this_thread::get_id() << endl;
    }

    int main() {
        cout << "主线程id为：" << std::this_thread::get_id() << endl;
        int m = 1;
        int n = 1;
        thread myobj(printAdds, **A(n)**);  **// 传入的是临时对象**
        myobj.join();
        return 0;
    }
    ```

    ![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/2_5.png)

    根据执行结果：如果向子线程传递临时对象，则类的对象是在**主线程**中构造的(根据线程id即可知晓)。假如此时使用的是`detach()`函数，则一定会在主线程退出前构造好对象。

## 2  类对象作为线程参数：`std::ref`

观察上述代码执行结果发现，即使子线程形参中使用了引用传递，依然会调用类的拷贝构造函数拷贝一份给子线程使用，而不是直接使用主线程构造好的临时对象，同时形参前总需要加`const`关键字，否则就会报错。

这样将导致，子线程修改了类的对象后，影响不了主线程中传入的类的对象。**如果希望在子线程中修改类的对象并对主线程产生影响，同时不再调用拷贝构造函数拷贝一份新的对象造成浪费，则需要使用`std::ref()`函数。**如下所示：

```cpp
class A {
public:
    int _i;
public:
    A(int i) : _i(i) { cout << "A的【构造函数】执行" << endl; }
    A(const A& a) : _i(a._i) { cout << "A的【拷贝构造函数】执行" << endl; }
    ~A() { cout << "A的【析构函数】执行" << endl; }
};

void printAdds(A& a) {  // 这里不再需要加const关键字
    a._i = 200;
}

int main() {
    A a(100);
    cout << "传入子线程前对象a中的：_i = " << a._i << endl;
    thread myobj(printAdds, **std::ref(a)**);  // 直接向子线程传入已在主线程构造好的对象a，希望子线程能够修改并影响主线程
    myobj.join();
    cout << "传入子线程后对象a中的：_i = " << a._i << endl;
    /*cout << "主线程即将结束..." << endl;*/
    return 0;
}
```

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/2_6.png)

但如果使用了`std::ref()`，则不应再使用`detach()`函数，因为这可能导致主线程先于子线程执行完毕，释放传入的局部变量，子线程再修改已释放的局部变量就不再安全了。

## 3  将类的成员函数作为线程入口：成员函数指针作为线程参数

之前是在类内定义operator()函数，将类变为仿函数，然后作为线程入口。但若想将类内任意一个成员函数作为线程入口，则需要将类内成员函数的地址作为线程的第一个参数，如下所示：

```cpp
class A {
public:
    void threadEntry(int num) {
        cout << "【类内任意一个成员函数作为线程入口来使用】" << endl;
    }
};

int main() {
    A a;
    thread myobj(**&A::threadEntry**, a, 100);  // 类的成员函数只有一份，因此地址固定
    myobj.join();  // 由于没有使用std::ref，因此用join和detach均可，否则只能使用join
    cout << "主线程即将结束..." << endl;
    return 0;
}
```

## 4  智能指针作为线程参数：`std::move()`

```cpp
void myPrint(unique_ptr<int> pointer) {  // 向子线程传入智能指针
    cout << "传入的智能指针指向的值为：" << *pointer << endl;
}

int main() {
    unique_ptr<int> pointer(new int(100));
    //thread myobj(myPrint, pointer);  // 无法直接将独占式指针传入子线程中去
    thread myobj(myPrint, std::move(pointer));  // 需要使用move函数将原独占式指针指向的地址移动至子线程中的独占式指针中
    myobj.join();
    return 0;
}
```

![](https://raw.githubusercontent.com/Zhgaot/Zhgaot.github.io/master/img/C++/multi-thread/2_7.png)