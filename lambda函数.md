---
title: C++ lambda函数
date: 2021-02-21 02:21:25
tags:
  - C++
  - 语法
keywords: C++
categories: 实操
---

# lambda函数

>  **为什么要lambda函数**

> 匿名函数是许多编程语言都支持的概念，有函数体，没有函数名。1958年，lisp首先采用匿名函数，匿名函数最常用的是作为回调函数的值。正因为有这样的需求，c++引入了lambda  函数，你可以在你的源码中内联一个lambda函数，这就使得创建快速的，一次性的函数变得简单了。例如，你可以把lambda函数可在参数中传递给std::sort函数。

~~~c++
#include "stdafx.h"
#include <algorithm>   //标准模板库算法库
#include <cmath>       //数学库
#include <iostream>
using namespace std;

//绝对值排序
void abssort(float* x, unsigned n) 
{
    //模板库排序函数
    std::sort(x, x + n,
        // Lambda 开始位置
        [](float a, float b) 
        {
            return (std::abs(a) < std::abs(b));
        } // lambda表达式结束
    );
}

int _tmain(int argc, _TCHAR* argv[])
{
    float a[5] = { 2.1f, 3.5f, 4.0f, 5.2f, 3.3f };
    abssort(a, 5);
    for (auto& x : a)
    {
        cout << x << endl;
    }
    system("pause");
    return 0;
}
~~~

![1079669-20180714211548261-953167390](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200713124322.png)

## lambda函数的语法

基本形式如下：

~~~C++
[capture](parameters)->return-type {body}
~~~

- []叫做捕获说明符，表示一个lambda表达式的开始。接下来是参数列表，即这个匿名的lambda函数的参数。
- parameters，普通参数列表
- ->return-type表示返回类型，如果没有返回类型，则可以省略这部分。这涉及到c++11的另一特性，参见自动类型推导，最后就是函数体部分。

![1079669-20180714213530219-2011197285](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200713124417.png)

- capture clause（捕获）
- lambda-parameter-declaration-list （变量列表）
- mutable-specification （捕获的变量可否修改）
- exception-specification （异常设定）
- lambda-return-type-clause （返回类型）
- compound-statement （函数体）

外部变量的捕获规则

默认情况下，即捕获字段为 [] 时，lambda表达式是不能访问任何外部变量的，即表达式的函数体内无法访问当前作用域下的变量。

如果要设定表达式能够访问外部变量，可以在 [] 内写入 & 或者 = 加上变量名，其中 & 表示按引用访问，=  表示按值访问，变量之间用逗号分隔，比如 [=factor, &total] 表示按值访问变量 factor，而按引用访问 total。

不加变量名时表示设置默认捕获字段，外部变量将按照默认字段获取，后面在书写变量名时不加符号表示按默认字段设置，比如下面三条字段都是同一含义：

[&total, factor]
[&, factor]
[=, &total]

~~~c++
#include <functional>
#include <iostream>
using namespace std;int _tmain(int argc, _TCHAR* argv[])
{
    //lambd函数对象
    auto fl = [](int x, int y){return x + y; };
    cout << fl(2, 3) << endl;

    function<int(int, int)>f2 = [](int x, int y){return x + y; };
    cout << f2(3, 4) << endl;
    system("pause");
    return 0;
}
~~~

不能访问任何局部变量

![1079669-20180714215759584-612883621](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200713124513.png)



如何传进去局部变量

~~~C++
int test = 100;
    //lambd函数对象捕获局部变量
    auto fl = [test](int x, int y){return test +x + y; };
    cout << fl(2, 3) << endl;
~~~

默认访问所有局部变量

~~~C++
int test = 100;
    //lambd函数对象捕获所有局部变量
    auto fl = [=](int x, int y){return test +x + y; };
    cout << fl(2, 3) << endl;
~~~

在C++11中这一部分被成为捕获外部变量

## 捕获外部变量

[captures] (params) mutable-> type{...} //lambda 表达式的完整形式

在 lambda 表达式引出操作符[ ]里的“captures”称为“捕获列表”，可以捕获表达式外部作用域的变量，在函数体内部直接使用，这是与普通函数或函数对象最大的不同（C++里的包闭必须显示指定捕获，而lua语言里的则是默认直接捕获所有外部变量。）

捕获列表里可以有多个捕获选项，以逗号分隔，使用了略微“新奇”的语法，规则如下

- [ ]     ：无捕获，函数体内不能访问任何外部变量 
- [ =]    ：以值（拷贝）的方式捕获所有外部变量，函数体内可以访问，但是不能修改。
- [ &]    ：以引用的方式捕获所有外部变量，函数体内可以访问并修改（需要当心无效的引用）；
- [ var]  ：以值（拷贝）的方式捕获某个外部变量，函数体可以访问但不能修改。
- [ &var] ：以引用的方式获取某个外部变量，函数体可以访问并修改
- [ this]  ：捕获this指针，可以访问类的成员变量和函数，
- [ =，&var] ：引用捕获变量var，其他外部变量使用值捕获。
- [ &，var]：只捕获变量var，其他外部变量使用引用捕获。

下面代码示范了这些捕获列表的用法：

~~~C++
int x = 0,y=0;
~~~

- auto f1 = [=](){ return x; };            //以值方式捕获使用变量，不能修改
- auto f2 = [&](){ return ++x; };          //以引用方式捕获所有变量，可以修改，但要当心引用无效
- auto f3 = [x](){ return x; };            //以值方式捕获x，不能修改
- auto f4 = [x,&y](){ y += x; };           //以值方式捕获x，以引用方式捕获y，y可以修改
- auto f5 = [&,y](){ x += y;};            //以引用方式捕获y之外所有变量，y不能修改
- auto f6 = [&](){ y += ++x;};           //以引用方式捕获所有变量，可以修改
- auto f7 = [](){ return x ;};            //无捕获，不能使用外部变量，编译错误

值得注意的是变化的捕获发生在了lambda表达式的声明之时，如果使用值方式捕获，即使之后变量的值发生变化，lambda表达式也不会感知，仍然使用最初的值。如果想要使用外部变量的最新值就必须使用引用的捕获方式，但也需要当心变量的生命周期，防止引用失效。

刚才的lambda表达式运行结果是：

- f1();           //以值方式捕获，x,y不发生变化
- f2();           //函数内部x值为0，之后变为1，y没有被修改，值仍然是0；
- f3();           //函数内部x值仍然为0，即f3()==0;
- f4();           //x,y均是0，运算后y仍然是0；
- f5();           //y是0；引用捕获的x是1，运算后x仍然为1；
- f6();          //x,y均引用捕获，运算后x,y均是2

~~~C++

#include <functional>
#include <iostream>
 
int main()
{
   using namespace std;
 
   int i = 3;
   int j = 5;
 
   // The following lambda expression captures i by value and
   // j by reference.
   function<int (void)> f = [i, &j] { return i + j; };
 
   // Change the values of i and j.
   i = 22;
   j = 44;
 
   // Call f and print its result.
   cout << f() << endl;
}

~~~

![1079669-20180722150524346-1922280250](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200713124630.png)

可以看到i是拷贝值，j是引用值，所以是24，结果26

![1079669-20180722182549771-1052608375](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200713124645.png)

## 把lambda表达式当作参数传送

~~~c++
#include <list>
#include <algorithm>
#include <iostream>
int main()
{
    using namespace std;
    // Create a list of integers with a few initial elements.
    list<int> numbers;
    numbers.push_back(13);
    numbers.push_back(17);
    numbers.push_back(42);
    numbers.push_back(46);
    numbers.push_back(99);

    // Use the find_if function and a lambda expression to find the 
    // first even number in the list.
    const list<int>::const_iterator result = 
        find_if(numbers.begin(), numbers.end(),[](int n) { return (n % 2) == 0; });//查找第一个偶数

    // Print the result.
    if (result != numbers.end())
     {
        cout << "The first even number in the list is " << *result << "." << endl;
    } else 
    {
        cout << "The list contains no even numbers." << endl;
    }
}
~~~

## lambda表达式嵌套使用

~~~C++
#include <iostream>

int main()
{
    using namespace std;

    // The following lambda expression contains a nested lambda
    // expression.
    int timestwoplusthree = [](int x) { return [](int y) { return y * 2; }(x) + 3; }(5);

    // Print the result.
    cout << timestwoplusthree << endl;
}
~~~

## lambda表达式使用在高阶函数里

~~~C++
#include <iostream>
#include <functional>

int main()
{
    using namespace std;

    auto addtwointegers = [](int x) -> function<int(int)> { 
        return [=](int y) { return x + y; }; 
    };

    auto higherorder = [](const function<int(int)>& f, int z) { 
        return f(z) * 2; 
    };

    // Call the lambda expression that is bound to higherorder. 
    auto answer = higherorder(addtwointegers(7), 8);

    // Print the result, which is (7+8)*2.
    cout << answer << endl;
}
~~~

