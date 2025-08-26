```
title: PHP小知识点
date: 2024-05-07 02:16:21
tags: 
  - PHP
  - 刨根问底
keywords: PHP
categories: 实操
```



# PHP小知识点



## 基础东西

### CLI

* CLI模式下运行PHP代码的三种方式：

  * 运行文件：

    ```php
    php source.php
    ```

  * 直接运行php代码：

    ```shell
    php -r 'phpinfo()';
    ```

  * 交互模式

    ```shell
    php -a #进入交互模式
    ```



### CGI模式

* 啥是CGI？

  * 简单来说：
    * 背景：早期Web服务器都只能响应静态资源，为了使服务器能直接运行动态脚本，即解决Web服务器和外部应用程序（CGI程序）之间的数据交互，出现了CGI（Common Gateway Interface，通用网关接口）
    * 简单理解：CGI是Web服务器和运行在其上的应用程序进行“交流”的一种约定
  * 具体可以看这个文章，带着如下问题去看可以：[CGI是什么](https://www.jianshu.com/p/c4dc22699a42)
    * CGI如何实现数据的输入输出
    * CGI 的的缺点？
    * 如果使用Apache作为web服务器，是如何调用php解释器的？为什么每次修改php.ini文件都需要重启Apache呢？
    * FastCGI和CGI啥关系？一句话概括下是啥？
    * FastCGI和PHP-FPM啥关系？
    * PHP-FPM创建多个CGI子进程，当有请求进入时，怎么分配谁处理？如果有CGI嘎掉了，FPM管不管？
    * Swoole和FPM区别在哪？举个栗子？

* 4种PHP Web环境的配置

  * 内置Web服务器：5.4.0以上的版本内置了个服务器，默认的Web根目录是当前目录，`-t`显式指定Web根目录

    ```shell
    $ php -S localhost:8000 -t ~/www
         PHP 7.1.19 Development Server started at Wed Jan 23 15:11:37 2019
         Listening on http://localhost:8000
         Document root is /Users/david/www
         Press Ctrl-C to quit.
    ```

  * 集成开发环境：XMAPP，集合了如下软件的安装包

    * X：支持跨平台
    * A：Apache
    * M：MySQL或MariaDB
    * P：PHP
    * P：Perl

  * LNMP：Linux、Nginx、MySQL、PHP

  * Docker



### 预定义常量和变量作用域

* 预定义常量：

| 名称      | 说明                           |
| --------- | ------------------------------ |
| $GLOBALS  | 引用全局作用域种可用的全部变量 |
| $_SERVER  | 服务器和执行环境信息           |
| $_GET     | HTTP GET 变量                  |
| $_POST    | HTTP POST 变量                 |
| $_FILES   | HTTP 文件上传变量              |
| $_REQUEST | HTTP Request 变量              |
| $_SESSION | Session变量                    |
| $_ENV     | 环境变量                       |
| $_COOKIE  | HTTP Cookies                   |

$_SERVER包含信息：

![image-20230611020957038](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110209123.png)

* 作用域：

  * 文件域：定义在两个不同文件的变量，其作用域限制在文件内部

    ```php
    # 文件a.php中定义了$a
    <?php
        $a = 1;
    ```

    ```php
    # 文件b.php引入a.php，此时a.php中的 $a对于b.php是不可访问的
    <?php
        include a.php
        var_dump($a);  // $a = NULL
    ```

  * 函数域：定义在函数内部的变量，其作用域限制在函数内部

    例如函数foo里定义了$a=1，但其生效范围仅限于函数foo里，所以函数foo之外的$a=NULL

  * 全局变量：全局变量可以在任意地方生效，上表中的PHP数据类型里的预定义变量就是全部变量

    可以使用`global`来改变变量的作用域



### 常量

* 常量定义之后，值无法改变，且不能被unset掉

* 定义方式：

  * define

    ```php
    defind('FOO', 'foo');
    ```

  * const

    ```
    const FOO = 'foo';
    ```

* PHP定义的一些魔术变量：

  | 名称              | 说明                                                         |
  | ----------------- | ------------------------------------------------------------ |
  | \_\_LINE\_\_      | 此语句在文件中的当前行号（可能因为include等，行号与单文件不一致） |
  | \_\_FILE\_\_      | 此语句所在文件的完整路径和文件名                             |
  | \_\_DIR\_\_       | 此语句文件所在的目录                                         |
  | \_\_FUNCTION\_\_  | 此语句所在的函数名称                                         |
  | \_\_CLASS\_\_     | 此语句所在的类名称                                           |
  | \_\_TRAIT\_\_     | 此语句所在Trait名称（trait 和 Class 相似，但仅仅旨在用细粒度和一致的方式来组合功能） |
  | \_\_METHOD\_\_    | 类的方法名                                                   |
  | \_\_NAMESPACE\_\_ | 当前命名空间的名称                                           |





### 循环控制

* break跳出循环。break可以接受一个可选的数字参数来决定跳出几重循环
* continue跳过本次循环，不再执行continue之后的剩余代码。continue也可以接受一个参数来决定跳过几重循环到结尾。



### 可变长度参数

![image-20230611015104613](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110151660.png)



### 数组

* PHP的数组除了传统意义的数组（如C语言里的数组）外，还可以起到链表、队列、栈、map的作用
* 数组分为两种，其一是纯数组，其下标为数字，叫作压缩数组（packed array）；其二是哈希数组（hash array），类似于其他语言的map



### 类和对象

* PHP的类是单继承的，即最多只能有一个父类。可以实现多个接口，用逗号来分隔多个接口的名称11

  ![image-20230611015223519](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110152564.png)





### 异常处理

* PHP5中异常处理的流程是try...catch...finally。
* PHP7中引入了一个与Exception同级别的结构Error，将一些Fatal Error当作Error异常抛出。这种Error异常也可以像Exception异常一样被第一个匹配的try/catch块所捕获。如果没有匹配的catch块，则调用异常处理函数（事先通过set_exception_handler()注册）进行处理。如果尚未注册异常处理函数，则按照传统方式处理：被报告为一个致命错误（Fatal Error）

例如：

![image-20230611015352612](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110153648.png)



### 命名空间

* 命名空间是一种对类的层级结构的一种封装方式，类似于操作系统的目录。在不同的命名空间下，用户不用担心类／函数／常量的名字冲突，在引入第三方类库时，也不用担心名字冲突



### 数据类型

#### 10种数据类型：

* 4种基本类型：布尔、整形、浮点、字符串
* 4种复合类型：数组、对象、回调函数、迭代器
* 2种特殊类型：资源(Resource)、NULL


#### 其他类型转换为bool

![image-20230611015613321](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110156350.png)



#### PHP整形数值常量

![image-20230611015642090](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110156116.png)

* 在涉及大数运算时，可以使用BCMath（任意精度数学，手册链接：http://php.net/manual/zh/book.bc.php）扩展



#### 浮点型

* 浮点型（float或double）表示的变量属于实数的集合。在C语言里，float和double是不同的类型，**但在PHP语言里，两者没有区分，都是float类型**。PHP的浮点数采用IEEE二进制浮点数算术标准（IEEE 754），通常最大值是1.8e308并具有14位十进制数字的精度（64位IEEE格式）
* 一般来说，对于64位的双精度数字，只有前15位的有效数字是有意义的，这保证了最大误差一般不大于10-16



#### 字符串

* string有4种表示方法：单引号、双引号、heredoc、nowdoc

![image-20230611015803987](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110158014.png)

* 字符串所能表示的最大长度是多少？

  PHP 7之前的版本（5.x），string所能表示的最大长度为2GB。PHP 7之后，字符串就没有这种限制了。出现这种情况的原因，是因为PHP底层对于字符串设计的改变。

  ![image-20230611015840685](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110158720.png)

* 注意：
  * 对于Java、PHP等语言，字符串每次赋值都会生成临时字符串，造成内存浪费



#### 回调函数

* 回调函数（callbacks）类型可以将一个函数作为变量或参数传递给其他函数使用。在PHP中，诸如call_user_func、call_user_func_array、usort、uasort、uksort等函数，都可以接收一个函数作为参数
* call_user_func()和call_user_func_array()区别
  * call_user_func()函数不支持引用参数，而call_user_func_array()支持引用参数
  * 参数数目不同。call_user_func可以接收多个参数，包括可变数量的参数；而call_user_func_array可接收一个数组作为参数



#### 迭代器

* 迭代器是啥？
  * 迭代器（Iterable）是PHP 7.1引入的一种新的数据类型，可用于数组或其他实现Traversable接口的对象，主要用于遍历内部元素。迭代器最多的用处是foreach和yield（生成器相关）



#### Resource

* ![image-20230611020435390](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110204428.png)



#### NULL

* NULL是一种特殊的值，表示变量没有值。判断一个变量是否为NULL，仅有三种可能：
  * 一个变量从未被赋值过。
  * 主动给变量赋值为NULL。
  * 对变量使用unset。



#### isset和empty区别

* isset判断一个变量是否被设置或非null，即如果不是null就返回true，否则返回false。判断变量为null的方法就是上面所讲的3种情况。
* empty判断一个变量是否为0、0.0、空字符串、null、false、空数组等，若是则返回true，否则返回false



### 变量

#### 指针和引用

* 在C语言里，指针是一个强大的存在，它可以通过传递指针的方式，将一个变量或函数的内存地址传递出去。直接操作内存是一个高效行为，但过于危险和不可控制，所以在一些语言，如Java、PHP等，不允许直接传递指针，而采用变量引用的方式来实现

* PHP的变量引用，是指不同的变量访问同一变量的内容，语法为&+变量名。例如，将$a的引用赋值给$b，即$a和$b指向同一个“地址”，当任何一个变量变化时，都会影响到另一个变量

* 引用的取消：可以用unset来取消其引用，即销毁引用变量

  ![image-20230611020723545](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110207583.png)



* foreach引用陷阱

  ![image-20230611020756183](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110207219.png)

​		![image-20230611020809750](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110208788.png)

* 结合上图，我们分析一下为什么出现这种情况。
  * 第2行，定义的数组[1,2]，如图3-3所示的第1步。
  * 第3至5行，将数组的每个元素都加1，所以$nums = [2,3]，同时数组的第2个元素引入了$num的引用。这造成了后面的意想不到的情况。第2个foreach处理第1个元素时，$num被赋值为2，同时$num又和第2个元素共享地址，所以第2个元素的值由3变为2。第2个foreach处理第2个元素时，$num被赋值为2，同时$num又和第2个元素共享地址，相当于把变量本身的值重新赋值给自己，所以第2个元素的值仍然为2。综上所述，$nums变为了[2,2]。
  * 究其原因，是由于数组最后一个元素的$num引用在foreach循环之后仍会保留。建议使用unset()来将其销毁
* 解决方案：
  * 方法1：不使用变量引用，而用$nums[$index]取出要改变的元素
  * 方法2：既然第1个和第2个foreach的变量名相等，那么不妨将第2个foreach的变量更名为$num2，就规避了变量引用的问题，也能输出正确的结果
  * unset调引用变量
  * 也可使用函数式编程解决



### 垃圾回收机制

* 带着问题来吧：
  * 什么样的变量会被回收？
  * 什么时机进行回收操作？
  * 回收步骤是什么？
  * 垃圾回收机制解决什么问题？

#### 回收规则

* 变量回收的基本规则
  * 变量引用计数增加（被使用或被引用），就不会被回收‘
  * 引用计数减少到零，所在变量容器将被清除（free），直接清理不用进入回收机制
  * 仅仅在引用计数减少到非零值时，才会产生垃圾周期（garbage cycle），可能会被回收



#### 回收时机

![image-20230611021209877](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202306110212918.png)



#### 回收步骤

* 模拟删除。根缓冲区（root buffer）里的每个变量zval的refcount减1
* 模拟恢复。如果refcount不为0，则refcount加1；如果为0，则不加1

* 真的删除。删除所有refcount为0的zval



#### 机制特性

* 垃圾回收机制首先是可解决内存泄漏和循环引用问题的（如回收时机的例子2）。注意，并不是每次refcount减少时都进入回收周期，只有根缓冲区满额后在开始垃圾回收。这是性能与功能的tradeoff。其次是将内存泄漏控制在一个阈值以下

