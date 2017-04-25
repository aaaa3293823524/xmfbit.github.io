---
title: Effective CPP 阅读 - Chapter 2 构造拷贝和赋值函数
date: 2017-04-24 20:50:10
tags:
     - cpp
---
在第二章中，作者主要关注了在C++的OOP“联邦”中行事的注意事项。主要包括有虚函数的情况下的继承以及copying function（拷贝构造函数和拷贝赋值函数）的处理。
<!-- more-->

## 05 了解C++默默编写并调用了哪些函数

C++编译器会自动为类添加默认构造函数和拷贝构造函数和析构函数，以及拷贝赋值函数。

在默认构造函数中，会调用基类的构造函数以及各个成员变量的构造函数。

在拷贝构造函数和拷贝赋值函数中，将单纯地将源对象中的non-static的成员变量拷贝到目标对象（浅拷贝）。

## 06 若不想使用编译器自动生成的函数，那就明确拒绝

有的时候，我们的类故意设计为不能拷贝构造或赋值。这时候，可以将拷贝构造函数和拷贝赋值函数声明为`private`，并且不提供函数的实现。

## 07 为多态基类声明虚析构函数

多态是OOP的基本概念之一。在C++中，可能遇到这样的情景：使用base class的指针或引用指向derived派生类对象，以此实现运行时的不同逻辑。这种情况下，要为基类声明虚析构函数。这是因为，当派生类对象经由一个base class指针被删除时，如果该base class带有非虚的析构函数，其结果是未定义的。常常会造成派生类自己的成员变量不能被销毁，造成内存泄露。

而那些意图并非是用来当做base class的类来说，随意将其虚构函数声明为`virtual`也是不恰当的。因为这会额外引入虚函数表，造成对象体积的无谓增大，给性能造成影响，而且丧失了对C语言的移植性。

由于那些被用来作为base class等待派生类继承的类通常情况下都有虚函数存在（派生类正是对虚函数重写，实现了多态），所以这一条款可以归纳如下：

> 那些有虚函数的类，几乎确定都应该有一个虚析构函数。

对于STL而言，记住其中的容器都不是为了继承而设计的，不要继承STL中的容器，包括`vector`，`string`等。

有的时候，我们可能会想声明一个抽象基类作为接口。当手上没有纯虚函数时候，可以将析构函数声明为纯虚的。然而这时候问题来了，我们需要为这个纯虚析构函数提供了一个空定义。

```
class ABC {
public:
    virtual ~ABC() = 0;    
};
ABC::~ABC() {}
```

这看上去很违背常理，因为一般情况下纯虚函数不需要实现。这里是因为当派生类被销毁时，其析构函数中会调用`~ABC()`，所以必须为这个函数提供一份定义。

## 08 别让异常逃离析构函数
如果在析构函数中抛出异常，会导致未定义的行为。

析构函数不应该抛出异常。如果析构函数调用的函数可能会抛出异常，要在析构函数中捕获异常，然后结束程序（这不是未定义行为）或者吞掉它们。

## 09 绝不在构造函数和析构函数中调用虚函数
你不应该在构造函数和析构函数中调用虚函数，这样的调用通常不会导致你想要的结果。

为什么呢？

如果我们在派生类的构造函数中使用了从基类中继承而来的虚函数。在派生类的构造函数之前，基类的构造函数被调用，这时候，所调用的虚函数实际是基类中的那个版本！析构函数同理。

直接在构造函数中调用虚函数看上去很容易避免。然而在构造函数中你可能会调用其他的初始化函数，你应该确保这些初始化函数中没有调用虚函数。否则，你可能会陷入到苦涩头疼的调试中去。编译器通常不会发现此类问题，但是在程序运行中，如果基类的虚函数是纯虚函数，程序很可能中止（和后面的相比，也许这还算好的）。如果基类中的虚函数有自己的实现，那么你可能就会头疼于程序的表现为何出乎意料（期望调用派生类的重写版本，实际仍是基类的原始版本）。

## 10 令`operator=`返回一个`*this`的引用
为了实现连锁赋值，赋值操作符必须返回一个引用，指向操作符左侧的赋值实参。这条规范被大多数人遵守。除非有确实好的理由，否则最好按规范办事。

所谓连锁赋值，是指下面这样的情况：
```
a = b = c;
```

这一条款不仅适用于赋值运算符，也适用于`+=`等。
```
class A {
public:
    A& operator=(const A& rhs) {
        // ...
        return *this;
    }
};
```

## 11 在`operator=`中处理自我赋值
所谓自我赋值，是指：
```
Widget w;
// ...
w = w;
```

自我赋值常常发生在同一个对象的不同别名之间。在实现赋值运算符时，应注意处理这一现象。

一种方法是进行“证同测试”，如下：

```
A& operator=(const A& rhs) {
    if(&rhs == this) return *this; //证同测试
    // ...
}
```
不过作者指出，这种方法不具备异常安全性。另一种方法是在函数内部，合理安排语句顺序，防止提前释放该对象本身的资源。

还有一种方法，使用copy-swap方法，首先拷贝构造一个`rhs`的拷贝，然后交换该拷贝和`*this`，

毫无疑问，使用证同测试方法和作者后文的方法都会造成性能的些许下降，这需要根据具体情况具体分析，合理采用。

## 12 复制对象时勿忘每一个成分
这一条款是指存在继承时，实现copying函数（指拷贝构造函数和拷贝赋值函数）不要忘记base class部分成员。看下面的例子：
```
class Derived: public Base{
private:
    int a;
public:
    Derived(const Derived& rhs):a(rhs.a) {}
    Derived& operator=(const Derived& rhs) {
        a = rhs.a;
        return *this;
    }
};
```

上面的代码只是拷贝了派生类新加入的成员`a`，而对基类中已有的成员未作处理。要记住，
>任何时候自己实现copying函数时，要担起重责大任，小心地复制其基类的成员。由于基类成员往往声明为`private`，所以，一般调用基类的成员函数进行拷贝。将上面的代码修改为：

```
Derived(const Derived& rhs):Base(rhs), a(rhs.a) {}
Derived& operator=(const Derived& rhs) {
    Base::operator=(rhs);
    a = rhs.a;
    return *this;
}
```

此外，两种copying函数的实现往往是相似的。然而，不要试图在一个函数中调用另一个函数。把相似代码提取出来，写成一个独立的`init()`函数是一个更好的选择。