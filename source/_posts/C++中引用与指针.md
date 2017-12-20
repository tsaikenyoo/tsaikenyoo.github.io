---
title: C++中引用与指针
date: 2017-12-17 22:12:20
tags: 
- CPP
categories:
- 代码随录
---

C++ 中引用 `&` 与指针 `*` 是两个十分相似的概念，我之前在写 C++ 的时候，有时也会疑惑，到底什么时候该用引用，什么时候该用指针呢？

指针来源于C语言，而 C++ 又是起源于C语言，所以 C++ 中自然保持了对指针的支持。这是 C++ 中指针存在的原因。

那么既然有了指针，为什么 C++ 还要创造”引用“这个概念呢？引用的发明，是为了解决 **运算符重载** 的问题。举个例子，使用 C++ 定义了一个类 `ArrayLike`，我们希望这个类能像数组一样使用下标取元素，即 `arr[0]`。那么此时就需要对运算符 `[]` 进行重载。

那么问题来了，重载的运算符的返回值是什么呢？假设 `ArrayLike` 储存的数据类型为 `int`，该运算符可以返回 `int` 类型，但有时我们希望通过下标赋值，如 `a[1]=2;`，此时 `int`作为返回值显然不合适，此时显然只能返回指针 `int*`。但若是返回 `int*`，则按下标取到的内容只是地址，所以还需要增加取值符来获取内容，如 `*(a[5])`。

因此，引用被发明了出来。在运算符重载中，引用具有指针的可赋值功能，也无需对结果增加取值符。例子如下：

```cpp
class ArrayLike{
private:
    int* arr;
public:
    ArrayLike(){...}
    ~ArrayLike(){...}
    
    // return int
    int operator[](int idx){
        return *arr[idx];
    }

    // return int*
    int* operator[](int idx){
        return arr[idx];
    }

    // return int&
    int& operator[](int idx){
        return *arr[idx];
    }
};
```
当然，在实际应用中，引用和指针更多地被用于 `避免值拷贝`，在这一点上两者的功能十分相似。

一、在声明变量时避免值拷贝。

```cpp
int a = 1;

// use pointer
int* a_ptr = &a;

// use reference
int& a_ref = a;
```

二、在传参时避免值拷贝

```cpp
// pointer param
void func(int* param);

// reference param
void func(const int& param);
```

在传参时避免值拷贝，是指针和引用作为参数被传递的重要原因之一。当然，这样传递参数有一个潜在的危险之处，即指针或引用的值可能会被悄悄修改。那么，如何避免这样的风险呢？这个时候，引用的优势就体现出来了。

对于引用来说，直接使用 `const type&` 作为参数即可。这就明确地说明了参数是不可修改的。

而对于指针来说，则十分麻烦。`const type*` 只能避免指针不指向别的对象，但是不能阻止对指针指向的内存区域的值的修改，如

```cpp
void func(const int* int_ptr){
*int_ptr = 1;
}
```

除了以上的情形，与指针相比，引用还有一个优点。指针是可以为空的，即 `NULL`，那么此时如果对指针取值，则会报错。因此，指针是具有很大的潜在风险的。而引用就没有这个风险，引用在定义的时候就必须初始化，也就是说，引用总是引用某个存在的变量。实际上，引用内部的实现就是通过 `const type*` 来实现的。

综上，何时该用指针？何时该用引用呢？在 StackOverflow 上看到了一句建议：

> Use reference when you can, use pointer when you must

毕竟有些地方指针的作用还是难以替代的，例如多态中的强制类型转换。



