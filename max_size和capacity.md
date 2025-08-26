---
title: C++ max_size和capacity
date: 2021-02-21 02:21:25
tags: C++
keywords: C++
categories: 实操
---



# max_size和capacity



## max——size

### 1.官方文档描述：

* Return maximum size

  Returns the maximum number of elements that the [vector](http://www.cplusplus.com/vector) can hold.

   This is the maximum potential [size](http://www.cplusplus.com/vector::size) the container can reach due to known system or library implementation  limitations, but the container is by no means guaranteed to be able to  reach that size: it can still fail to allocate storage at any point  before that size is reached.

### 2.个人理解：

* 返回此容器所能容纳的最大值，但应注意，可能在达到此数值之前由于剩余内存过小无法继续分配而产生未知错误。



## capacity

### 1.官方文档：

* Return size of allocated storage capacity

  Returns the size of the storage space currently allocated for the [vector](http://www.cplusplus.com/vector), expressed in terms of elements.

   This *capacity* is not necessarily equal to the [vector size](http://www.cplusplus.com/vector::size). It can be equal or greater, with the extra space allowing to  accommodate for growth without the need to reallocate on each insertion.

   Notice that this *capacity* does not suppose a limit on the size of the [vector](http://www.cplusplus.com/vector). When this *capacity* is exhausted and more is needed, it is automatically expanded by the  container (reallocating it storage space). The theoretical limit on the [size](http://www.cplusplus.com/vector::size) of a [vector](http://www.cplusplus.com/vector) is given by member [max_size](http://www.cplusplus.com/vector::max_size).

   The *capacity* of a [vector](http://www.cplusplus.com/vector) can be explicitly altered by calling member [vector::reserve](http://www.cplusplus.com/vector::reserve).

## 2.个人理解：

* 其返回vector的大小，**其返回值不一定等于vector大小，可以等于或大于**，多余容量可以用来容纳增长，**而无需在每次插入式重新分配**
* vector容量可以通过调用成员**vector::reserve**显式更改。



## 示例：

~~~c++
// comparing size, capacity and max_size
#include <iostream>
#include <vector>

int main ()
{
  std::vector<int> myvector;

  // set some content in the vector:
  for (int i=0; i<100; i++) myvector.push_back(i);

  std::cout << "size: " << myvector.size() << "\n";
  std::cout << "capacity: " << myvector.capacity() << "\n";
  std::cout << "max_size: " << myvector.max_size() << "\n";
  return 0;
}
~~~

## 可能的输出（依编译器和操作系统不同可能存在差异）

~~~c++
size: 100
capacity: 128
max_size: 1073741823
~~~

