---
title: C++ gdb调试
date: 2021-02-21 02:21:25
tags:
  - C++
  - gdb
  - 调试
keywords: 
  - C++
  - gdb
  - 调试
categories: 实操
---



# GCC 和 GDB调试 总结

## 一、GCC：

gcc和g++是c/c++的编译器。

**格式**： gcc [options] file……

**主要options：**

| 选项   | 含义                                                         |
| ------ | ------------------------------------------------------------ |
| **-v** | 查看gcc编译器的版本，显示gcc执行时的详细过程；               |
| **-o** | 指定输出文件名为file，这个名称不能跟源文件名同名；           |
| **-E** | 只预处理，不会编译、汇编、链接；                             |
| **-S** | 只编译，不会汇编、链接；                                     |
| **-c** | 编译和汇编，不会链接；                                       |
| **-g** | 产生符号调试工具（GNU的gdb）所必要的符号信息，想要对源代码进行调试，就必须加入这个选项。 |

**具体过程：**一个C/C++文件要经过预处理（preprocessing）、编译（compliation）、汇编（assembly）和连接（linking）才能变成可执行文件。

1. 预处理,生成.i的文件[预处理器cpp] 
2. 将预处理后的文件转换成汇编语言,生成文件.s[编译器egcs] 
3. 有汇编变为目标代码(机器代码)生成.o的文件[汇编器as] 
4. 连接目标代码,生成可执行程序[链接器ld] 

**源文件：**

~~~c++
main.c
    #include <stdio.h>
    int main()
    {
        int temp;
        printf ("&d\n",temp);
        return 0;
    }
~~~

**（1）预处理**

g++ -E -o GCC.i GCC.c

预处理是将包含（include）的文件插入源文件中，将宏定义展开、根据条件编译命令选择要使用的代码，最后将代码输出到一个“ .i ”文件中等待进一步处理。

GCC.i 文件内容如下（值列出了部分内容，看行数就看出来了。。。）：
![20180805115725777](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200713123354.png)

**(2)编译：**

gcc -S -o GCC.s GCC.i

编译就是把 c/c++ 代码（比如上面的“ .i ”文件）翻译成汇编代码，文件内容如下所示：

![20180805120019152](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200713123437.png)

**（3）汇编**

gcc -c -o GCC.o GCC.s

汇编是将第二步输出的汇编代码翻译成符合一定格式的机器代码，在Linux系统上一般表现为ELF目标文件（OBJ文件），文件部分内容如下所示：

![20180805120300866](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200713123456.png)

**（4）链接**

gcc -o GCC GCC.o

链接是将汇编生成的OBJ文件、系统库的OBJ文件、库文件链接起来，最终生成可以在特定平台运行的可执行程序。

如果想一步操作成功的话可以用：gcc GCC.c （未指明输出文件，默认是a.out）或 gcc -o GCC GCC.c

**gcc和g++的区别：**

先说一下相关概念：GCC : GNU Compiler Collection(GUN 编译器集合)，它可以编译C、C++、JAV、Fortran、Pascal、Object-C、Ada等语言。

gcc是GCC中的GUN C Compiler（C 编译器）

g++是GCC中的GUN C++ Compiler（C++编译器）

**主要区别：**

1. 对于 *.c和*.cpp文件，gcc分别当做c和cpp文件编译（c和cpp的语法强度是不一样的）；

2. 对于 *.c和*.cpp文件，g++则统一当做cpp文件编译；

3. 使用g++编译文件时，g++会自动链接标准库STL，而gcc不会自动链接STL；

4. gcc在编译C文件时，可使用的预定义宏是比较少的；

5. gcc在编译cpp文件时/g++在编译c文件和cpp文件时（这时候gcc和g++调用的都是cpp文件的编译器），会加入一些额外的宏，这些宏如下：

#define __GXX_WEAK__ 1
#define __cplusplus 1
#define __DEPRECATED 1
#define __GNUG__ 4
#define __EXCEPTIONS 1
#define __private_extern__ extern

6.        在用gcc编译c++文件时，为了能够使用STL，需要加参数 –lstdc++ ，但这并不代表 gcc –lstdc++ 和 g++等价，它们的区别不仅仅是这个；

二、GDB调试：

如果想进行GDB调试的话，需要这样：gcc -g -o GCC GCC.c

~~~txt
gdb 常用命令
(1) gdb 可执行文件 : 表示对某个文件进行调试
(2) b 函数名/行数  :  在某个函数名或行数前设置断点
(3) run/r          : 表示开始运行，如果是正在调试的程序的话，表示再次进行调试
(4) n/next         : 表示执行下一行语句
(5) l/list         : 列出源码默认10行（当前位置的上下共10行）
    list 行号      : 列出行号上下共10行的源码
    list 函数名    : 列出函数名上下共10行的源码
(6) s/step         : 表示单步执行，进入函数
(7) p /x 变量名    : 按16进制输出变量的值
      /d           : 按10进制
      /o           : 按八进制
(8) set var 变量名=值 : 设置变量的值
(9) bt(backtrace)  : 查看各级函数调用及参数,简写bt
(10)q/quit         : 退出
(11)finish         : 连续运行到当前函数返回为止，然后停下来等待命令
(12)continue/c     : 跳转到下个断点，或者跳转到观察点
(13)ptype 变量名   : 可以查看变量的类型，简写为pt
(14)watch
    作用：一般用来观察某个变量/内存地址的状态(也可以是表达式），
          如可以监控该变量/内存值是否被程序读/写情况。
    有三种方法：
    1.watch expr（指定变量/内存地址/表达式）
    一旦expr的值有变化时，将停住程序。
    2.rwatch expr
    当expr被读时，停住程序。
    3.awatch expr
    当expr被读或被写时，停住程序。
    watch使用步骤：
        1. 使用break在要观察的变量所在处设置断点；
        2. 使用run执行，直到断点；
        3. 使用watch设置观察点；
        4. 使用continue观察设置的观察点是否有变化。
(15)start            : 开始执行程序，停在main函数第一行语句前面等待命令
(16)info watchpoints : 列出所有观察点
    info breakpoints : 查看当前设置的所有断点
(17)d/delete [breakpoinsts num] [rang...]         
    d/delete         : 删除所有断点
    d/delete num     : 删除breakpoints为num的断点
    d/delete num1-num2 : 删除breakpoints为num1-num2的断点
(18)enable num       : 启用num号断点
(19)disable num      : 关闭num号断点
(20)u/until          : 结束当前循环
~~~



Reference：

    https://www.cnblogs.com/zhangsir6/articles/2956798.html
    https://blog.csdn.net/czg13548930186/article/details/78331692
    https://www.cnblogs.com/oxspirt/p/6847438.html
