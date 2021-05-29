---
title: C++新特性梳理系列：C++11 chrono库和ratio库
date: 2021-05-29 18:43:45
tags:
- chronon
- ratio
---

## 前言

本文给大家介绍C++11新增的两个好用的工具库：chrono和ratio库。

chrono库提供了对时间日期进行操作的API，C++11之前我们只能使用C风格的时间库，即`<time.h>`或者其他一些第三方库。众所周知，`<time.h>`使用起来并不方便，而有些三方库虽然好用，但在一些场景下，其引入的成本不小，例如C++底层编程或者嵌入式编程，引入三方库的成本可能会比较大，chrono是C++11的标准库，只要平台支持C++11，就可以直接使用他提供的简单方便的API。

ratio库提供了对分数进行操作的API，分数计算的一个好处是可以避免浮点型数值计算导致的精度丢失。在C++11之前，如果要进行分数运算，一般会选择自己写一套符合需求的分数计算函数，然后在项目中使用，或者使用第三方库。但在C++11中，分数计算已经被标准支持，我们可以不用再重复造轮子了，直接使用标准库ratio提供的API即可。

## chrono

理解chrono的使用需要理解如下3个类型（他们都定义在std::chrono命名空间内）：

- clocks，表示时钟，即有一个起点和tick rate组成的整体。例如从1970年1月1日作为起点，tick rate是1秒，他们俩组合起来表示的就是一个clock。所以一个clock是包含了起点时间和tick rate两个信息的。
- time points，表示某个具体的时间点，例如当前时间是2021/05/29 19:17。类似的，时间辍也是时间点。
- durations，表示某一段时间，例如一个小时，一天，一周这样。

> C++20还新增了几个类型，有兴趣的朋友可以自行查阅文档。后续介绍C++20新特性的时候不会再作介绍。

下面是一个示例：

```c++
long fib(int n) {
    if (n < 2) return n;
    return fib(n-1) + fib(n-2);
}

template<typename T>
constexpr T convertTime(std::chrono::duration<double> originTime)
{
    return std::chrono::duration_cast<T>(originTime);
}

void chronoTest()
{
    //获取当前时间
    const std::chrono::time_point<std::chrono::system_clock> now = std::chrono::system_clock::now();
    //time_since_epoch获取从1970-01-01 00:00:00到现在的纳秒数
    std::chrono::nanoseconds nano = now.time_since_epoch();
    std::cout << nano.count() << std::endl;

    //转换成微秒数
    std::chrono::microseconds micro = std::chrono::duration_cast<std::chrono::microseconds>(nano);
    std::cout << micro.count() << std::endl;

    //转换成毫秒数
    std::chrono::milliseconds mill = std::chrono::duration_cast<std::chrono::milliseconds>(nano);
    std::cout << mill.count() << std::endl;

    //转换成秒
    std::chrono::seconds sec = std::chrono::duration_cast<std::chrono::seconds>(nano);
    std::cout << sec.count() << std::endl;
    //还可以转换成小时，天等，不演示了


    //统计计算耗时
    auto start = std::chrono::system_clock::now();
    fib(30);
    auto end = std::chrono::system_clock::now();

    //默认单位是s
    std::chrono::duration<double> runTime = end - start;
    std::cout << "fib(20) need: " << convertTime<std::chrono::microseconds>(runTime).count() << "us" << std::endl;
    std::cout << "fib(20) need: " << convertTime<std::chrono::milliseconds>(runTime).count() << "ms" << std::endl;
    std::cout << "fib(20) need: " << runTime.count() << "s" << std::endl;
}
```

更多的使用各位自行查阅文档吧，本来只是个工具库，看看文档就知道怎么用了。

## ratio

`<ratio>`头文件里主要就是std::ratio类，这是一个模板类，有两个模板参数，第一个是分子，第二个是分母。如下：

```c++
template<
    std::intmax_t Num,
    std::intmax_t Denom = 1
> class ratio;
```

这个类只有两个开放的字段，num和den，分别获取其分子和分母。`<ratio>`头文件还包含了几个用于计算加减乘除以及比较的模板偏特化版本，这里就不一一列出了，有兴趣请查看文档，很简单的。

下面是一个使用示例：

```c++
void ratioTest()
{
    using one_two = std::ratio<1,2>;
    using two_three = std::ratio<2,3>;

    std::ratio_add<one_two, two_three> res;
    std::cout << "1/2 + 2/3 = " << res.num << "/" << res.den << std::endl;

    std::cout << "1/2 > 2/3 ? " << std::ratio_greater<one_two, two_three>::value << std::endl;

}
```

示例程序就这样的写吧，说实话，有点难使用....剩下的其他操作，各位自行尝试，有兴趣的可以研究一下他这是怎么实现的？看起来是用“类型”在作计算，而不是用变量。

## 小结

本文的内容很简单轻松，因为介绍的两个库只是工具库，正常来说会使用即可，使用的熟练度取决于你对库的熟练度，而对库的熟练度是可以通过查阅文档来完成的，我写作这个系列文章的目的不是像文档一样介绍各个新特性，而是应该包含一些日常使用的经验与对新特性的个人理解等内容，所以就不多废话了。最后建议，学习这种库，最好的方法还是查阅文档，文档永远是最官方的学习资料，网上个人写的甚至是书上写的都有可能有一些错误。

## 进度

- [x] [atomic operations library](https://en.cppreference.com/w/cpp/atomic)
- [x] smart pointer
- [x] [std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)
- [ ] [stateful](https://en.cppreference.com/w/cpp/named_req/Allocator#Stateful_and_stateless_allocators) and [scoped](https://en.cppreference.com/w/cpp/memory/scoped_allocator_adaptor) allocators
- [x] [std::forward_list](https://en.cppreference.com/w/cpp/container/forward_list)
- [x] [chrono library](https://en.cppreference.com/w/cpp/chrono)
- [x] [ratio library](https://en.cppreference.com/w/cpp/numeric/ratio)
- [ ] new [algorithms](https://en.cppreference.com/w/cpp/algorithm):
  - [ ] [std::all_of](https://en.cppreference.com/w/cpp/algorithm/all_any_none_of), [std::any_of](https://en.cppreference.com/w/cpp/algorithm/all_any_none_of), [std::none_of](https://en.cppreference.com/w/cpp/algorithm/all_any_none_of),
  - [ ] [std::find_if_not](https://en.cppreference.com/w/cpp/algorithm/find),
  - [ ] [std::copy_if](https://en.cppreference.com/w/cpp/algorithm/copy), [std::copy_n](https://en.cppreference.com/w/cpp/algorithm/copy_n),
  - [ ] [std::move](https://en.cppreference.com/w/cpp/algorithm/move), [std::move_backward](https://en.cppreference.com/w/cpp/algorithm/move_backward),
  - [ ] [std::random_shuffle](https://en.cppreference.com/w/cpp/algorithm/random_shuffle), [std::shuffle](https://en.cppreference.com/w/cpp/algorithm/random_shuffle),
  - [ ] [std::is_partitioned](https://en.cppreference.com/w/cpp/algorithm/is_partitioned), [std::partition_copy](https://en.cppreference.com/w/cpp/algorithm/partition_copy), [std::partition_point](https://en.cppreference.com/w/cpp/algorithm/partition_point),
  - [ ] [std::is_sorted](https://en.cppreference.com/w/cpp/algorithm/is_sorted), [std::is_sorted_until](https://en.cppreference.com/w/cpp/algorithm/is_sorted_until),
  - [ ] [std::is_heap](https://en.cppreference.com/w/cpp/algorithm/is_heap), [std::is_heap_until](https://en.cppreference.com/w/cpp/algorithm/is_heap_until),
  - [ ] [std::minmax](https://en.cppreference.com/w/cpp/algorithm/minmax), [std::minmax_element](https://en.cppreference.com/w/cpp/algorithm/minmax_element),
  - [ ] [std::is_permutation](https://en.cppreference.com/w/cpp/algorithm/is_permutation),
  - [ ] [std::iota](https://en.cppreference.com/w/cpp/algorithm/iota),
  - [ ] [std::uninitialized_copy_n](https://en.cppreference.com/w/cpp/memory/uninitialized_copy_n)
- [ ] [Unicode conversion facets](https://en.cppreference.com/w/cpp/locale#Locale-independent_unicode_conversion_facets)
- [x] [thread library](https://en.cppreference.com/w/cpp/thread)
- [x] [std::function](https://en.cppreference.com/w/cpp/utility/functional/function)
- [x] std::bind
- [ ] [std::exception_ptr](https://en.cppreference.com/w/cpp/error/exception_ptr)
- [ ] [std::error_code](https://en.cppreference.com/w/cpp/error/error_code) and [std::error_condition](https://en.cppreference.com/w/cpp/error/error_condition)
- [ ] [iterator](https://en.cppreference.com/w/cpp/iterator) improvements:
  - [ ] [std::begin](https://en.cppreference.com/w/cpp/iterator/begin)
  - [ ] [std::end](https://en.cppreference.com/w/cpp/iterator/end)
  - [ ] [std::next](https://en.cppreference.com/w/cpp/iterator/next)
  - [ ] [std::prev](https://en.cppreference.com/w/cpp/iterator/prev)
- [ ] Unicode conversion functions