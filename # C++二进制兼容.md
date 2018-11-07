# C++ 二进制兼容设计

在用C++做库开发的时候，经常会面临二进制兼容问题。（所谓二进制兼容这里指 库升级之后可以直接替换之前的dll/so，而使用库的程序不需要重新编译即可运行。这样库的升级是对使用者透明的）  

## 以C函数作为库的接口

以C函数的方式导出接口，C函数是以名称匹配的方式进行链接的，因此，后期增加新的函数接口不会产生二进制兼容问题。

## 导出C++类 pimpl方式（编译器防火墙）

在一些C++开源库（如Qt）中经常会看到类似下面的类定义。这样一是隐藏了实现细节，只对外暴露了接口。还有就是实现了二进制兼容（在看陈硕的[《C++工程实践谈》](!http://chenshuo.com/)中了解到这个称作pimpl（pointer to implementation））。pimpl方法遵循：以non-virtual成员函数作为接口，库变动不会新增virtual函数（新增接口也是增加non-virtual成员函数），数据成员隐藏在Data指针之后，接口类sizeof(A)=sizeof(Data*)。
这样做二进制兼容的核心是sizeof(A) = sizeof(Data*),这样在使用库的程序中，升级之后A对象的size也不会发生变化，non-virtual成员本质上也是一个全局C函数（编译器生成的是隐式增加第一个参数为this指针的全局函数）。（其实pimpl的方式保证二进制兼容本质上也是遵循导出C函数的原则）
```C++
//A.h
class A{
    public:
    A();
    ~A();
    void interface_1();
    void interface_2();
    void interface_3();

    private:
    class Data;
    Data* data;
};

//A.cpp
class Data{
    public:
    data_1;
    data_2;
    data_3;
};

A::A(){}
A::~A(){}
void A::interface_1(){
    //处理data_1
    data->data_1;
}
...
```

在C++ 11上可以改进将Data*换成std::unique_ptr<Data> 这样不需要自己在去实现C++11的move语义。

## 为什么不是虚函数
虚函数作为接口，新增虚函数会造成虚表中offset问题，即使投机采用只在尾部新增虚函数的方式，由于可能存在被继承的情况，还是会破坏子类的虚表offset。最终破坏二进制兼容。