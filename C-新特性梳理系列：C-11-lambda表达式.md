---
title: C++新特性梳理系列：C++11 lambda表达式
date: 2021-04-17 22:54:32
tags:
- lambda
- c++11
---

## 前言

lambda表达式是函数是编程中的一个概念，如果有同学阅读过《计算机程序的构造与解释》一书，相信对lambda表达式一定不会陌生，这本书使用Lisp编写示例程序，而Lisp语言只能写出lambda表达式，是纯函数式计算机语言。在其他语言中也有对lambda表达式的支持，例如Java(JDK1.8之后)，Python，Js等，还有本系列文章主要介绍的C++。

C++ 11中引入了lambda表达式，官方的定义是：

> Constructs a [closure](https://en.wikipedia.org/wiki/Closure_(computer_science)): an unnamed function object capable of capturing variables in scope.

即构造一个“闭包”，“闭包”在这里的意义不是数学上的那个意义，而是指**一个可以捕获其所在作用域的变量的匿名函数对象**。（关于数学上的定义，请参考[闭包 (数学)]([https://zh.wikipedia.org/wiki/%E9%97%AD%E5%8C%85_(%E6%95%B0%E5%AD%A6)](https://zh.wikipedia.org/wiki/闭包_(数学)))）

## lambda表达式的语法

C++11中有两种(C++20和C++23扩充了语法)：

1. [捕获的参数列表] (参数) 尾置返回类型 {函数体}
2. [捕获的参数列表] (参数) {函数体}

关于"尾置返回类型"，本文后面会介绍。其中“捕获的参数列表”指的是该函数体能可以直接使用的参数集合，而“参数”即函数参数，函数体则函数主要的逻辑部分。下面是一个简单的lambda表达式：

```c++
#include <iostream>
#include <functional>

using namespace std;

//这里的实现逻辑先暂时不用太关心，functional也是C++ 11中的新特性，后续会介绍
template <typename Function, typename ...Args>
auto callFunc(const Function& f, const Args&... args) 
    -> typename std::result_of<Function(Args...)>::type
{
    return f(args...);
}

void func()
{
    int a = 10, b = 20;
	//这里定义了一个lambda表达式，并作为对象传入callFunc
    auto ret = callFunc([a, b](int c) {
        return a + b + c;
    }, 100);
    std::cout << ret << std::endl;
}

int main()
{
    func();
    return 0;
}
```



### 捕获的参数列表

上面的demo中，lambda表达式的定义在第18行，[]里面的就是要捕获的参数列表，捕获的参数列表有多种写法，分别是（以下假设a,b是变量名）：

- 空，即不捕获任何外部参数。
- =，捕获所有外部一层作用域的变量（包括this），按值传递。
- &，捕获所有外部一层作用域的变量（包括this），按引用传递。
- a, 指定捕获某个变量，按值传递。
- &a, 指定捕获某个变量，按引用传递。
- =, &a,&b，指定捕获a，b变量按引用传递，其余变量按值传递。
- &,a,b，指定捕获a,b变量按值传递，其余变量按引用传递。

### 尾置返回类型

其实lambda表达式如果不写返回类型，且函数体内部或者作为参数传递给其他函数且该形参的function对象声明了返回类型，那么lambda表达式会自动推断返回类型，当然能写出返回类型那更好。

lambda表达式使用尾置返回类型的语法表示其返回类型，尾置返回类型也是C++11的新特性，目的是简化编写复杂返回类型（吐槽一下，其实个人觉得也没怎么简化）。

语法就是在函数声明的末尾，函数体之前并以->符号开始，并且需要在传统函数写返回类型的地方使用auto替代，上面的示例程序中callFunc就使用了尾置返回类型。

```c++
template <typename Function, typename ...Args>
auto callFunc(const Function& f, const Args&... args) 
    -> typename std::result_of<Function(Args...)>::type
{
    return f(args...);
}
```

这里其实不用尾置返回类型也没什么太大区别，可以按传统方法写:

```c++
template <typename Function, typename ...Args>
typename std::result_of<Function(Args...)>::type
callFunc(const Function& f, const Args&... args) 
{
    return f(args...);
}
```

结果是一样的。

回到示例程序中的18行，如果要写明返回类型，该如何修改？

```c++
    auto ret = callFunc([a, b](int c) -> int {
        return a + b + c;
    }, 100);
    std::cout << ret << std::endl;
```

是的，是需要在声明式最后加上-> int即可。

## lambda表达式的作用

上面介绍了lambda表达式的语法，那么lambda表达式的作用是什么呢？为什么要引入这个东西？

个人认为lambda表达式一个作用就是利用其“匿名”的特性，考虑这样一个场景：假设现在我们要编写一段逻辑来处理一些数据而且只是一次性的，传统的思路是编写一个新的函数，然后调用他。但仔细思考一下，我们真的需要编写一个新的函数来作这件事吗？这段逻辑只调用了一次，新写一个函数似乎有点小题大作了，lambda表达式就可以解决这个问题，因为他是“匿名”的，不需要为他专门想一个名字，反正就是作这个一次性的处理而已，并不打算复用他或者在外部调用他。

另一个作用就是利用lambda表达式本身是一个对象的特性，既然是对象，那么就可以作为函数参数，返回值以及模板函数参数来使用。

### 作为函数参数

例如标准库里sort函数的第3个参数就是一个comparor，我们可以传入一个lambda表达式来自定义vector里的元素比较逻辑：

```c++
vector<int> v = {1,3,2,4,5};
sort(v.begin(), v.end(), [](const int& lhs, const int& rhs) { 
    return lhs > rhs; 
});
//for_each的第3个参数也接受一个lambda表达式
std::for_each(v.begin(), v.end(), [](const int &item) { std::cout << item << " "; });
```

### 作为返回值

```c++
std::function<int (int)> returnFunc()
{
    return [](int) -> int { return 42; };
}
```

这个例子引出了std::function，实际上，本文最开始的那个callFunc模板函数在调用时，第一个参数也会被推断成std::function。

## 最后

本文简单介绍了lambda表达式的语法以及基本作用，lambda表达式是C++11中很重要的一个新特性，引入了函数式编程的概念，丰富了使用C++编程的方式，使得C++更加的amazing了。正文的最后提到了std::function，这也是C++11函数式编程领域中非常重要的东西，我在在下一篇文章介绍std::function以及functional包里的新特性。

## 进度

- [x] auto and decltype
- [x] defaulted and deleted functions
- [x] final and override
- [x] trailing return type
- [x] rvalue references
- [x] move constructors and move assignment operators
- [ ] scoped enums
- [x] constexpr and literal types
- [ ] list initialization
- [ ] delegating and inherited constructors
- [ ] brace-or-equal initializers
- [x] nullptr
- [x] long long
- [x] char16_t and char32_t
- [ ] type aliases
- [ ] variadic templates
- [ ] generalized (non-trivial) unions
- [ ] generalized PODs (trivial types and standard-layout types)
- [ ] Unicode string literals
- [ ] user-defined literals
- [ ] attributes
- [x] lambda expressions
- [x] noexcept specifier and noexcept operator
- [x] alignof and alignas
- [ ] multithreaded memory model
- [ ] thread-local storage
- [ ] GC interface
- [ ] range-for (based on a Boost library)
- [ ] static_assert (based on a Boost library)



