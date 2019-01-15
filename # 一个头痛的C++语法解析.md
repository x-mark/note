# 一个头痛的C++语法解析

如下代码

```C++
class background_task
{
public:
  void operator()() const
  {
    do_something();
    do_something_else();
  }
};

background_task f;
std::thread my_thread(f);
```

而如果最后两句写成如下形式

```C++
std::thread my_thread(background_task());
```

这里相当与声明了一个名为my_thread的函数，这个函数带有一个参数(函数指针指向没有参数并返回background_task对象的函数)，返回一个std::thread对象的函数，而非启动了一个线程。

使用在前面命名函数对象的方式，或使用多组括号①，或使用新统一的初始化语法②，可以避免这个问题。
如下所示：

```C++
std::thread my_thread((background_task()));  // 1
std::thread my_thread{background_task()};    // 2
```

## operator

Qt的QUuid中看到有这样一个转换重载

```C++
//转换运行符重载
operator GUID(QUuid);
```