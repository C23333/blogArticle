```
title: g++编译时如何链接opencv库
date: 2021-05-18 03:25:09
tags:
  - g++
  - OpenCV
keywords: 编译链接OpenCV
categories: 实操
```



# g++编译时如何链接opencv库

只需在命令后添加如下语句即可

~~~C++
$(pkg-config --cflags --libs opencv)
~~~

如

~~~C++
g++ holeFill.cpp -o test $(pkg-config --cflags --libs opencv)
~~~

如果使用CMakeLists.txt，在版本的选择上可能更方便。