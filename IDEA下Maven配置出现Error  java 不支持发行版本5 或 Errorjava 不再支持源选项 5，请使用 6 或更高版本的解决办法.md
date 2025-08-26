```
title: XHProf介绍
date: 2022-05-03 13:24:20
tags:
  - Java
  - IDEA
keywords: IDEA
categories: 实操
```



# IDEA下Maven配置出现Error : java 不支持发行版本5 或 Error:java: 不再支持源选项 5，请使用 6 或更高版本的解决办法 

> 将公司中eclipse代码pull到本地IDEA编译报错
>
> 以下为网络解决办法，亲测有效( 使用的为方法（4）) 2021/4/24



我每次创建一个maven工程，都报错
Error : java 不支持发行版本5 或者是 Error:java: 不再支持源选项 5。请使用 6 或更高版本。
实在忍受不了，这里写篇文章记录一下，不想每次都上网搜解决办法了。
（1）首先，点settings，然后找到图中目录，这里的target bytecode version和project bytecode version都换成你的jdk版本，我的是11
![1](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/1.png)

（2）在settings里搜maven，把这部分设置成图里这样，具体maven的那几个路径看你自己保存在哪了，override图标记得勾上

![2](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/2.png)

（3）点Project Structure的图标，然后把Project SDK和Project language level都换成你的jdk版本

![3](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/3.png)

Modules里也是一样，调好language level

![4](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/4.png)

（4）以上都不起作用的话，在pom.xml里添加如下配置，具体Java版本看你自己安装的是什么

~~~java
<properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.encoding>UTF-8</maven.compiler.encoding>
        <java.version>11</java.version>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
</properties>

~~~

