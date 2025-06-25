---
title: C++ 多继承情况下的指针转换
description: 记录多继承情况下的指针转换
slug: cpp-mutli-inherit-pointer-translation
date: 2019-05-14 00:00:00+0800
image:
categories:
    - learn
tags:
    - c++
    - 多继承
toc: true
---

# 问题 #  

多继承情况下，子类的指针和父类指针的实际数值可能不同，编译器进行了隐式的指针转换。

<!--more-->

# 代码 #
```cpp
#include <iostream>

using namespace std;

class A{
public:
    virtual void print_a() = 0;
};

class B{
public:
    virtual void print_b() = 0;
};

class AB : public A, public B{
public:
    virtual void print_a() override {
        cout << "print_a(): " << reinterpret_cast<uint64_t>(this) << endl;
    }
    virtual void print_b() override {
        cout << "print_b(): " << reinterpret_cast<uint64_t>(this) << endl;
    }
};


int main(){
    AB *ab = new AB();
    cout << "AB: " << reinterpret_cast<uint64_t>(ab) << endl;
    ab->print_a();
    ab->print_b();

    A *a = (A *)ab;
    cout << "A: " << reinterpret_cast<uint64_t>(a) << endl;
    a->print_a();

    B *b = (B *)ab;
    cout << "B: " << reinterpret_cast<uint64_t>(b) << endl;
    b->print_b();

    b = reinterpret_cast<B *>(reinterpret_cast<uint64_t>(ab));
    cout << "after transfer B: " << reinterpret_cast<uint64_t>(b) << endl;
    b->print_b();

    return 0;
}
```

# 输出结果 #  

```
AB: 7110392
print_a(): 7110392
print_b(): 7110392
A: 7110392
print_a(): 7110392
B: 7110396
print_b(): 7110392
after transfer B: 7110392
print_a(): 7110392
```

可以看到，b的指针在经过`(B * )ab`的强制转型后，指针值出现了变换。
但当执行`b->print_b()`时，由于是虚函数，遂执行了`AB::print_b()`，在此函数中`this`会自动还原为ab的指针值。

对于`after transfer B`的输出，将ab的指针转为`uint64_t`再转为`B *`时，得到的b的指针其实已经发生了错位。此时`b->print_b()`会错误的执行`AB::print_a()`。