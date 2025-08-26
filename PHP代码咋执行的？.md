---
title: PHP代码咋执行的？
date: 2024-05-07 04:51:06
tags: 
  - PHP
  - 刨根问底
keywords: 
  - PHP
  - C++ main
  - ELF
categories: 实操
---



# PHP代码咋执行的？

> PHP是门解释型语言，所以理论上来说，和Java、Python的执行过程差不多
>
> 如果，发现错误的话直接编辑改掉就好，如果闲的话可以加条勘误日志在<a href="#mistakeDiary">最后</a>
>
> 文章中可能会有一些扩展出去的知识点，不看也不会影响主体问题的了解，所以，看你时间了



> 下面的流程以PHP7举例，一是因为PHP7开始引入了AST，二是我懒得切其他版本了，当然，也有一小点原因是因为：搜的资料都是PHP7的



## 铺垫

* 解释型语言、编译型语言？
  * 通俗来讲：
  * 编译型语言（*<a href="#compile">详细了解点我，推荐看一下</a>*）：**代码执行之前**，就将代码“翻译”成汇编语言，再根据软硬件环境编译成目标文件。干这活的我们称为`编译器`
  * （还记得大学学过的 g++ 1.cpp -o 1.out吧，这就是一次完成的编译过程）
  * 解释型语言（<a href="#mainContent">看正文就行，以PHP代码为例</a>）：**代码执行时**，才将代码“翻译”成机器语言，一般每执行一次“翻译”一次，所以执行效率低。这里干“翻译”这个活的我们称为`解释器`
* 总结：对编译型语言与解释型语言的区别的理解，立足于源代码被编译成目标平台CPU指令的时机。对于编译型语言，编译结果已经是针对当前CPU体系的指令；而解释型语言，需要先编译成中间代码，再经由该解释型语言的特定虚拟机，翻译成特定CPU体系的指令被执行。解释型语言是在运行过程中，翻译为目标平台的指令。常说解释型语言“慢”，主要也是慢在这里



## <div id="mainContent">正文</div>

* 整体流程：在PHP 7中，源代码首先进行`词法分析`，将源代码切割为多个`字符串单元`，分割后的字符串称为`Token`。而一个一个独立的Token是无法表达完整语义的，需经过`语法分析`阶段，将Token转换为`抽象语法树`（简称AST）。之后，抽象语法树被转换为`机器指令`执行。在PHP中，这些指令称为`opcode`（opcode可以先简单理解为CPU指令，后续会讲他到底是个啥（题外话：XDebug好多功能就是通过在用户程序的opcode前后追加自己的opcode实现的））。

* 到AST的生成这一步，编译型语言和解释型语言经历的过程相似（是吧，你还是需要看一下编译型语言的编译过程），从这步之后，开始产生差异。如下图

  ![image-20230602014505264](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306020145290.png)

  * PHP代码执行前两步，和编译型语言大差不差
    * 第一步：词法分析将PHP代码转换为有意义的标识Token。该步骤的词法分析器使用`Re2c`实现。
    * 第二步：语法分析将Token和符合文法规则的代码生成抽象语法树。语法分析器基于`Bison`(还有个Yacc)实现。语法分析使用了BNF（Backus-Naur Form，巴科斯范式）来表达文法规则，Bison借助状态机、状态转移表和压栈、出栈等一系列操作，生成抽象语法树。
      * 第三步：上步的抽象语法树生成对应的opcode，并被虚拟机执行。opcode是PHP 7定义的一组指令标识，指令对应着相应的handler（处理函数）。当虚拟机调用opcode，会找到opcode背后的处理函数，执行真正的处理。以常见的echo语句为例，其对应的opcode便是ZEND_ECHO。
    * **这里为了便于理解词法分析和语法分析过程，将两者分开描述。但实际情况下，出于效率考虑，两个过程并非完全独立**。

* 可以看下图，大体梳理PHP代码执行的过程：

  ![image-20230602021218961](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306020212990.png)

* 下面通过一段代码来演示一些细节：

  ```php
      <? php
      echo "hello world";
  ```

  * 首先，这段代码会被切割为Token

    * Token：是PHP代码被切割成有意义的标识。PHP7共有137中Token（自行查阅参照表或者拉一下php-src库看一下/Zend/zend_language_parser.h）

    * PHP提供了一个函数：token_get_all()函数获取PHP代码被切割后的Token，可以通过下面语句查看一句简单的PHP语句被切割后的Token

      ```bash
      php -r 'print_r(token_get_all("<?php echo \"hello world\"; "));'
      ```

      输出结果为：

      ```bash
      #其中，二维数组的每个成员数组的第一个值为Token对应的枚举值。第二个值为Token对应的原始字符串内容。第三个值为代码对应的行号。可以看出，词法解析器将“<? php echo "hello world"; ”这段文本内容切分成了4部分
      Array
      (
          [0] => Array
              (
                  [0] => 379
                  [1] => <? php
                  [2] => 1
              )
          [1] => Array
              (
                  [0] => 328
                  [1] => echo
                  [2] => 1
              )
          [2] => Array
              (
                  [0] => 382
                  [1] =>
                  [2] => 1
              )
          [3] => Array
              (
                  [0] => 323
                  [1] => "hello world"
                  [2] => 1
              )
          [4] => ;
      )
      ```

      * 文本“<? php”，切割后对应的Token值为379，参考PHP 7中的源码：

        ```php
            #define T_OPEN_TAG 379
        ```

        可以看出，它是PHP代码的起始tag，也就是<? php标识。

      * echo对应的Token是T_ECHO：

        ```php
            #define T_ECHO 328
        ```

      * 源码中的空格，对应的Token为T_WHITESPACE，值为382：

        ```php
            #define T_WHITESPACE 382
        ```

      * 字符串"hello world"，对应的Token值为323：

        ```php
            #define T_CONSTANT_ENCAPSED_STRING 323
        ```

  * 也就是说，Token就是一个个的“词块”，但是单独存在的词块不能表达完整的语义，还需要借助规则进行组织串联。语法分析器就是这个组织者。它会检查语法，匹配Token，对Token进行关联。

    PHP 7中，组织串联的产物就是AST（Abstract Syntax Tree，抽象语法树）。

  * AST

    * AST是PHP 7版本新特性。在这之前的版本中，PHP代码的执行过程中是没有生成AST这一步的。PHP 7对抽象语法树的支持，实现了PHP编译器和解释器解耦，有效提升了可维护性。
    * 这里简单理解下就好，具体细节后续再写
    * 简单来说，抽象语法树具有树状结构。AST的节点分为多种类型，对应着PHP语法。我们可以认为节点类型是对语法规则的抽象，例如赋值语句，生成的抽象语法树节点为ZEND_AST_ASSIGN。而赋值语句的左右操作数又将作为ZEND_AST_ASSIGN类型节点的孩子。通过这样的节点关系，构建出抽象语法树。
    * 如果你想查看AST的话，可以通过PHP-Parser

  * opcodes

  * AST扮演了源码到中间代码的临时存储介质的角色，还需要将其转换为opcode，才能被引擎直接执行。opcode只是单条指令，opcodes是opcode的集合形式，是PHP执行过程中的中间代码，类似Java中的字节码。opcode生成之后由虚拟机执行

  * PHP工程优化措施中有一个比较常见的“开启opcache”，指的就是这里的opcodes的缓存（opcodes cache）。通过省去从源码到opcode的阶段，引擎可以直接执行缓存的opcode，以此提升性能。

  * 如下图，可以查看一段PHP代码生成的opcode：

    ![image-20230602022204473](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306020222509.png)



## <div id="compile">编译型语言的编译过程</div>

* 这里我们以一段 C 代码为例，总体流程为 预编译 -- 编译 -- 汇编 -- 链接 --> 可执行二进制文件

  ```c++
  #include <stdio.h>
  int main()
  {
      printf("How I Compile ?");
      return 1;
  }
  //这段代码会打印个字符串，main是入口函数
  ```

如果你想了解，为什么c和c++他们的入口函数是main，以及，能不能是别的入口函数，<a href="#entryFunc">看这里吧那</a>

* 第一步：预处理器预处理：这一步会进行依赖处理、宏替换等操作。如上述代码，`#include <stdio.h>`会被替换，stdio.h文件内容会被引入。
* 第二步：编译器编译：编译器会把C语言翻译成汇编语言程序，一条C语言通常编译为多条汇编代码。同时编译器会对程序进行优化，生成目标汇编程序。
* 第三步：汇编器汇编：编译器会把C语言翻译成汇编语言程序，一条C语言通常编译为多条汇编代码。同时编译器会对程序进行优化，生成目标汇编程序。
* 第四步：链接器链接：程序中往往包含一些共享目标文件，如示例程序中的printf()函数，位于静态库，需要经过链接器（如Uinx连接器ld）进行链接。
* 第五步：装载器装载代码：将可执行程序加载到内存并进行执行。

* 整体流程如下图所示：

  ![image-20230602012846094](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306020128124.png)



## <div id="entryFunc">为啥是你-main？</div>

* 通俗来讲：当我们执行编译后的可执行文件时，必须有一个**入口函数**来告诉**链接器**，指明代码从哪里开始执行。目前，大部分链接器的默认函数入口都是main函数。However，你也可以显式指明你的入口函数，入口函数的名称没有特殊要求，只需要在链接的时候，告诉链接器就可以了。

* 即：**C、C++等高级语言的程序main函数不是必须的，只不过链接器默认指明的程序入口是main函数而已**

* 上代码：

  ```c
  #include <stdio.h>
  #include <stdlib.h>
  
  int main()
  {
      printf("I'm main!\n");
      return 0;
  }
  
  int test()
  {
      printf("I'm test!\n");
      exit(0);
  
      /* 这里不能return，而是直接exit，因为我们自己执行入口函数之后，需要同步指明，链接时不使用标准系统的启动文件（原因见下面解释）。 */
      //return 0;
  }
  
  ```

  * 不指明入口函数，默认走main![image-20230602000424834](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306020004876.png)

  * 指明入口函数，走我们显式指明的入口函数

    ![image-20230602002837880](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306020028904.png)



* 为啥自己指定入口函数的时候，需要配合 `-nostartfiles` 参数编译

  * 要搞明白这个，我们首先需要搞明白下面这个问题：上面我们说过，main函数作为入口函数，是链接器默认的行为。那么这个默认行为是在哪里写的呢？可不可以理解成，main函数作为默认入口函数这一行为，是被硬编码到链接阶段的初始化代码中的？

  * 答案是可以，对于我们来说，程序是从入口函数开始执行的，但是对于链接器来说，真正的入口函数是`_start函数`

  * 怎么看呢？

    * 明确一个概念：C的标准系统启动文件对应的库叫 C run-time library，简称CRT、C运行时库，对应的目标文件是crt1.o和crti.o。_start函数就在crt1.o中。main函数的调用就是被硬编码在这个函数中的

    * 在明确一个概念，我们刚才gcc xx 那一套，就生成了一个可执行文件，也就是一个目标文件，结构如下：

      ![img](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306020140976.png)

    * `readelf -s crt1.o`查看符号表

      ![image-20230602010153458](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306020101528.png)

  * 现在再回到最开始的问题：为啥自己指定入口函数的时候，需要配合 `-nostartfiles` 参数编译

    * 因为：如果使用标准系统库，则还是会走_start函数，此时链接器会去寻找用户程序中的main函数

    * **在上面的实例程序中，我们同时定义了main函数和test函数，所有链接器能在符号表中找到，而如果是下面的代码，则找不到，即产生报错**

      ```c
      #include <stdio.h>
      #include <stdlib.h>
      
      int test()
      {   
          printf("test\n");
          exit(0);
      }
      ```

      ![image-20230602005652859](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306020056895.png)

* 为什么要exit而不能return呢？

  * main函数可以return，是因为标准系统库会帮你在return之后，进行exit和清理工作，但是不使用标准系统库后，无人会替你exit了。此时如果还是return，程序可以正常执行，但是会有段错误，因为程序没有结束。（我承认，这段有我自己猜的成分）

* 如果你想了解，链接器为什么知道找不找的到main函数，以及链接器是干什么的，<a href="#linker">看这里吧</a>

* 扩展一个操作，我们其实可以重写_start函数，来指定入口函数（需要配合 `-nostartfiles` 参数编译），代码如下：

  ```c
  #include <stdio.h>
  #include <stdlib.h>
  
  _start(void)
  {
      printf("I'm _start");
      exit(0);
  }
  ```

  ![image-20230602011558390](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306020115424.png)

​		![image-20230602011705486](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306020117539.png)



## <div id="linker">彻底搞定链接器<div>

* 这个已经有现成的、成体系的文章了，讲的比我强，不再赘述了，大家自己看吧

  > <a href="https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg4OTYzODM4Mw==&action=getalbum&album_id=1923374391426416642&scene=173&from_msgid=2247485635&from_itemidx=1&count=3&nolastread=1#wechat_redirect">彻底理解链接器</a>

* 防止url挂掉，可以使用如下关键字搜索：`彻底理解链接器`

## <div id="mistakeDiary">勘误日志</div>





## 参考

**感谢：**（排名不分先后）

> <a href="https://blog.csdn.net/justlinux2010/article/details/11621087">CSDN-C语言中没有main函数生成可执行程序的几种方法</a>
>
> <a href="https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg4OTYzODM4Mw==&action=getalbum&album_id=1923374391426416642&scene=173&from_msgid=2247485635&from_itemidx=1&count=3&nolastread=1#wechat_redirect">WX-彻底搞定链接器</a>
>
> <a href="https://zhuanlan.zhihu.com/p/349873222">知乎-C语言必须写main函数？最简单的 Hello world 你其实一点都不懂！</a>
>
> <a href="https://blog.csdn.net/poject/article/details/84031102">CSDN-关于程序的入口函数（main _start...）</a>
>
> <a href="https://blog.csdn.net/superSmart_Dong/article/details/116891431">Linux C: 为什么C都必须有一个main函数</a>
>
> 《PHP7底层设计与源码实现》
>
> [C编译器、链接器、加载器详解](https://www.cnblogs.com/oubo/archive/2011/12/06/2394631.html)
