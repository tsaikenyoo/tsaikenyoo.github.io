---
title: 如何退出C++程序
date: 2017-12-15 20:11:51
tags: 
- CPP
categories: 代码随录
---

在平时写 C++ 程序的时候，希望程序在条件异常的时候直接退出。而条件异常的捕获通常不是在 main 函数中完成的，那么此时该如何退出程序呢？

如果在 main 函数中捕获了异常，直接 `return 1;` 即可，但是若在非 main 函数中，调用 `return;`则只会退出该函数。

比较直观的一种做法就是直接调用 `exit(0);`，表示程序非正常退出。但实际上，这种做法并不安全。我们知道，在程序结束时，栈中所有的对象都会调用析构函数来释放内存空间。但是 `exit(0);` 则会直接退出程序，并不会去管栈中尚未释放的对象。

要保证栈中的对象所占的内存空间被释放，则必须在 main 函数中调用 `return` 函数。因此最合理的做法是，在需要退出程序的地方调用 `throw exception();`，在 main 函数中则使用 try、catch，在 catch 块中调用 `return EXIT_FAILURE;`

其中 EXIT_FAILURE 在 `<cstdlib>` 头文件中定义：

```cpp
#define EXIT_SUCCESS 0
#define EXIT_FAILURE 1
```

示例如下：

```cpp
void a(){
    throw exception();
}

int main(){
    try{
        a();
    }
    catch(const exception&){
        return EXIT_FAILURE;
    }
    return 0;
}
```
