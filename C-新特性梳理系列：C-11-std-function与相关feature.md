从本文开始，后面的文章将介绍C++11 的Library features。

本文将重点介绍std::function和std::bind，期间可能会提到其他相关的特性，提到的时候再细说。在正式介绍他们之前，先回想一下在C++11之前，我们如何传递一个函数？答案是通过**函数指针**。

> 如果各位对于下面关于函数指内容非常熟悉了，可以直接跳过。

### 函数指针

函数指针就是指向函数的指针，即本身是一个指针，只是他的内容是一个函数的地址，语法如下：

```c++
//正常声明
return-type (*pointer-name)(param-list)
    
//typedef的时候
typedef return-type (*new-type-name)(param-list)
```

下面是一个demo:

```c++
int func(int i) {
    return i * 10;
}

typedef int (*MyFunc)(int);

//1.
int (*f)(int) = func;
f(5);
//2.
MyFunc f = func;
f(5);
```

既然是一个指针，那么就可以作为函数参数或者保存在类或者结构体中，总之，和普通指针一样。下面是一个传递函数指针的例子：

```c++
void callFunc(MyFunc myFunc, int param) {
    myFunc(param);
} 

callFunc(func, 5);
```

例子中callFunc接受一个MyFunc类型的参数，这个参数我们知道其实是一个函数指针的，再调用callFunc的时候，我们可以直接传递func这个函数名字作为实参，再callFunc函数里可以使用func，这就完成了一次传递。

函数指针大量用在回调以及使用动态库的场景，例如使用dlsym通过符号获取动态库暴露的接口实例，一般就需要用到函数指针了来绑定接口实例，从而调用。需要注意的是，函数指针是类型不安全的，即可以任意被转换成其他类型的指针，所以使用起来得非常小心。

## std::function

函数指针不仅定义式看起来复杂，使用起来也比较麻烦，普通指针有的问题，函数指针也都有。C++11提供了一种新的方式来封装函数实体，即std::function。std::function是一个类模板，他可以封装所以**可调用对象**，包括普通函数、仿函数、lambda表达式甚至是函数指针等，由于std::function本身也是一个类，所以也具有普通类的特性。其声明如下：

```c++
template< class R, class... Args >
class function<R(Args...)>;
```

下面是一些使用的例子：

```c++
class Functor
{
public:
    double operator()(double d) {
        return d * 10;
    }
};

int compareInt(int lhs, int rhs) {
    if (lhs == rhs) return 0;
    else if (lhs < rhs) return -1;
    else return 1;
}

int func(int i) {
    return i * 10;
}

typedef int (*MyFunc)(int);

class Test
{
public:
    Test(int i)
        :m_i{i} {}

    void printDouble(double d) {
        std::cout << d << std::endl;
    }

    int m_i;
};

int main()
{
    //普通函数
    std::function<int(int,int)> f_compareInt = compareInt;
    std::cout << f_compareInt(5,6) << std::endl;

    //仿函数
    Functor functor;
    std::function<double(double)> f_functor = functor;
    std::cout << f_functor(1.5) << std::endl;

    //lambda
    std::function<void(int)> f_printInt = [](int num) { std::cout << num << std::endl; };
    f_printInt(1);

    //函数指针
    MyFunc myFunc = func;
    std::function<int(int)> f_myFunc = myFunc;
    std::cout << f_myFunc(4) << std::endl;

    //成员函数，注意成员函数的第一个参数实际上是this对象，所以这里的参数列表需要把Test类型加上，同时要传递一个Test对象
    std::function<void(Test&, double)> f_T_printDouble = &Test::printDouble;
    Test t(1.4);
    f_T_printDouble(t, 4.5);

    return 0;
}
```

总之，std::function的提供了一种统一的使用可调用对象的方式，可以在大部分场景下替代函数指针。并且由于std::function本身是一个类，所以在基于类架构的体系中，组合使用std::function对象会比使用函数指针更加和谐一些。

## std::bind

本节将介绍一个经常和std::function搭配使用的工具，std::bind。std::bind是一个模板函数，其声明式如下：

```c++
template< class F, class... Args >
function-object bind( F&& f, Args&&... args );

template< class R, class F, class... Args >
function-object bind( F&& f, Args&&... args );
```

std::bind的作用就是将传入的函数与参数绑定起来返回一个新的函数对象。这么说可能有点抽象，下面是一些例子，看了之后就很容易理解了。

```c++
void nm(int i, int j, int k) {
    std::cout << i << " " << j << " " << k << " " << std::endl;
}

//占位符
using std::placeholders::_1;
using std::placeholders::_2;
using std::placeholders::_3;

// (1)
auto fnm1 = std::bind(nm, 1, 2, 3);
fnm1();

// (2)
auto fnm2 = std::bind(nm, 1, 2, _1);
fnm2(3);

// (3)
auto fnm3 = std::bind(nm, _2, 2, _1);
fnm3(3, 1);

// (4)
auto bPrintDouble = std::bind(&Test::printDouble, t, _1);
bPrintDouble(1.1);
```

解释一下上述4个不同的使用方式的意义：

1. nm函数有3个参数，std::bind的第一个参数是nm函数，后面的3个参数是实参，std::bind将nm函数和这3个实参绑定起来生成了一个新的函数对象fnm1，这个函数对象实际上就包含了实参信息，这时候nm函数的3个参数已经被固定了，在调用的时候不需要也不能再接受参数了。
2. 这里使用了**_1**占为符，假设我们还不知道或者不想固定nm函数的参数k，但由于nm函数需要接受3个参数，所以为了能顺利的生成新的函数对象，这里传递一个占位符给std::bind，最终生成的函数对象只固定了原函数nm的i和j参数，对于k参数则是在调用时确定。
3. 同2，只不过这里使用了两个占位符，占位符的名字是包含了顺序的意义的。例如`_1`表示第一个参数，`_2`表示第2个参数。在本里中，形参j被固定了。i的位置占位符是`_2`，即表示新函数对象的第2个参数将会被传递到原函数nm的i这个位置。j的位置同理。
4. bind也可以作用在成员函数，规则和上述几个规则一样，需要注意的是，要记得传递一个类对象。

各位花点时间去实际写一下代码就能理解了。同时，各位可能会觉得std::bind很神奇，占位符是怎么生效的？这就得深入std::bind的具体实现了，以后有机会我会再写一篇文章介绍一下bind的实现。

这里再多提一个比较有意思的东西，即**lambda可以模拟bind的功能**。例如:

```c++
auto lambda_nm = [](int i) { nm(i, 2, 3); };
lambda_nm(1);
```

其中2,3还可以是利用lambda的参数捕获特性来获取。

至于使用lambda还是使用bind，网上有很多对比的文章，主流的观点就是尽量使用lambda，因为std::bind生成的可调用对象类型也是一个普通的类，很难进行优化，以及占位符的上限是20个，很难进行扩展。而我个人的想法是，其实所谓的编译优化对整体的性能并没有很大提升，性能的优化应该优先去优化性能消耗大头，而不是纠结这点编译优化，std::bind还是比lambda简单一些的，调用个函数就完事了，我会在需要的场景优先选择std::bind。

std::bind在回调的时候也非常有用，假设现在有如下代码：

```c++
class ResultProcessor
{
public:
    void setCallback(const std::function<void(int, int)>& callback) {
        m_callback = callback;
    }

    void doProcess(int i) {
        //do something
        int res = i * 100;
        int status = 0; //mean OK
        m_callback(res, status);
    }
private:
    std::function<void(int, int)> m_callback;
};

void callback1(int res, int status) {
    std::cout << "callback1 : " << res << " status : " << status << std::endl;
}

ResultProcessor rp;
rp.setCallback(callback1);
rp.doProcess(1);
```

代码很简单，就是设置callback，调用doProcess的时候要触发callback函数。现在考虑一个场景，如果我想使用另一个callback函数，例如有新的callback函数:

```c++
void callback2(int res, int status, double whatever) {
    std::cout << "callback2 : " << res << " status : " << status << " and " << whatever << std::endl;
}
```

并且其实第三个参数对我们的ResultProcessor来说并不是很重要，如何修改代码使得ResultProcessor可以使用这个callback2呢？一种直接的方法是修改ResultProcessor的m_callback成员的类型为`std::function<void(int, int, int)>`，但这不是好办法，如果又要换回callback1，那这里又得修改。而且如果在代码中动态的切换callback函数，这样的修改是无法达到目的的。std::bind可以解决这个问题，很简单，只需要这样就行：

```c++
ResultProcessor rp;
rp.setCallback(callback1);
rp.doProcess(1);

//使用std::bind生成一个具有void(int,int)签名的函数对象
//至于这里的第3个参数，在实际场景则可以变得更有意义一些，这里只是demo。
rp.setCallback(std::bind(callback2, _1, _2, 1));
rp.doProcess(2);
```

这样一来，既可以动态切换callback，又不用修改ResultProcessor类的声明，一举两得。

## 总结

std::function是一种统一的可以封装各种可调用对象（包括函数，lambda，函数指针，成员函数，仿函数等等）的方法，在C++11中用std::function来替代以前的函数指针是比较推荐的做法，这也是C++11建议的做法。std::bind将传入的函数与参数绑定起来返回一个新的函数对象，经常和std::function配合使用，可以提供极大的灵活性，例如上一小节中callback的例子。lambda也可以模拟std::bind，使用上来说没有std::bind简单，而且实际上他们俩的通途完全不同，所以请在合适的地方使用他们。

## 进度

- [ ] [atomic operations library](https://en.cppreference.com/w/cpp/atomic)
- [ ] `emplace()` and other use of rvalue references throughout all parts of the existing library
  - [ ] [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)
  - [ ] [std::move_iterator](https://en.cppreference.com/w/cpp/iterator/move_iterator)

- [ ] [std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)
- [ ] [stateful](https://en.cppreference.com/w/cpp/named_req/Allocator#Stateful_and_stateless_allocators) and [scoped](https://en.cppreference.com/w/cpp/memory/scoped_allocator_adaptor) allocators
- [ ] [std::forward_list](https://en.cppreference.com/w/cpp/container/forward_list)
- [ ] [chrono library](https://en.cppreference.com/w/cpp/chrono)
- [ ] [ratio library](https://en.cppreference.com/w/cpp/numeric/ratio)
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
- [ ] [thread library](https://en.cppreference.com/w/cpp/thread)
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