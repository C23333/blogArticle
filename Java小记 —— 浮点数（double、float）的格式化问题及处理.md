---
title: Java小记 —— 浮点数（double、float）的格式化问题及处理
date: 2022-05-05 09:21:42
tags:
  - Java
keywords: Java
categories: 实操
---



# Java小记 —— 浮点数（double、float）的格式化问题及处理

平时常会面临浮点数的格式处理问题，下面就举例说一说常见的问题及处理：

  **1，科学计数法问题**

  一个浮点数123456789.10，在打印的时候变成了1.234567891E8，处理起来很简单，如：

```java
double d = 123456789.10;



System.out.println(d);//1.234567891E8



NumberFormat nf = NumberFormat.getNumberInstance();



nf.setGroupingUsed(false);



System.out.println(nf.format(d));//打印结果：123456789.10
```

  使用NumberFormat的时候要setGroupingUsed(false)，否则结果就会变成123,456,789.1。

  再有直接转为BigDecimal更简便：

```java
System.out.println(new BigDecimal(d));//打印结果：123456789.10
```

  

  **2，指定小数位的位数**

  指定浮点数1.010515的小数位的位数：

```java
double d = 1.010515;



NumberFormat nf = NumberFormat.getNumberInstance();



nf.setMaximumFractionDigits(10);



System.out.println(nf.format(d));//打印结果：1.010515



nf.setMinimumFractionDigits(10);



System.out.println(nf.format(d));//打印结果：1.0105150000
```

  使用NumberFormat格式化的时候可以设置最大和最小的小数位数，如果要求必须有多少位，就要将最大和最小位数保持一致了。

  再有，转为BigDecimal也很简单，会自动补0：

```java
System.out.println(new BigDecimal(d).setScale(10,BigDecimal.ROUND_HALF_UP));//打印结果：1.0105150000
```

  使用BigDecimal的时候要选择舍入模式，接下来就说说这个问题。

  

  **3，四舍五入**

  接着看上面的例子，对1.010515进行小数位截取，进行四舍五入：

```java
nf.setMaximumFractionDigits(3);



System.out.println(nf.format(d));//打印结果：1.011 四舍五入



nf.setMaximumFractionDigits(5);



System.out.println(nf.format(d));//打印结果：1.01052 四舍五入
```

  不像BigDecimal，使用NumberFormat指定小数位的时候，不需要指定舍入方式，我们看到结果已经舍入了，但是我们将d+1，然后再看一下：

```java
d = 2.010515;



nf.setMaximumFractionDigits(3);



System.out.println(nf.format(d));//打印结果：2.011 四舍五入



nf.setMaximumFractionDigits(5);



System.out.println(nf.format(d));//打印结果：2.01051 没有四舍五入
```

  改变d的值后再进行小数位截取，会发现有的时候会四舍五入，有的时候不四舍五入，这时候就会想到指定舍入方式，NumberFormat 有很多舍入模式：UP、DOWN、CEILING、FLOOR、HALF_UP、HALF_DOWN、HALF_EVEN、UNNECESSARY，这些模式其实和BigDecimal的模式是一样的，NumberFormat的舍入模式就是对BigDecimal的做了一下封装，并没有什么不同，比如，RoundingMode.HALF_UP就等于BigDecimal.ROUND_HALF_UP，至于这么多模式我就不再多解释了，平时使用的时候根据实际的需求选择一种模式即可，对于我们平时理解的四舍五入对应的模式就是RoundingMode.HALF_UP，我们看代码：

```java
nf.setRoundingMode(RoundingMode.HALF_UP);



System.out.println(nf.format(d));//打印结果：2.01051 没有四舍五入
```

  是不是觉得奇怪，为什么没有四舍五入，换BigDecimal试一下：

```java
System.out.println(new BigDecimal(d).setScale(5,BigDecimal.ROUND_HALF_UP));//打印结果：2.01051 没有四舍五入
```

  还是没有四舍五入，再来试试下面的代码：

```java
System.out.println(nf.format(BigDecimal.valueOf(d)));//打印结果：2.01052 成功了



System.out.println(BigDecimal.valueOf(d).setScale(5,BigDecimal.ROUND_HALF_UP));//打印结果：2.01052 成功了
```

  哎？怎么又都成功了！呵呵，神不神奇，其实这就是下一个问题了（丢失精度）。





  **4，精度丢失**

  还是接着上面的例子说，对2.010515进行四舍五入的时候，只有最后一次的代码正确的对其进行了四舍五入的格式处理，关键在于BigDecimal.valueOf(d)，将d转为了BigDecimal，但是不是随便转的：

```java
System.out.println(nf.format(BigDecimal.valueOf(d)));	//打印结果：2.01052 成功了



System.out.println(nf.format(new BigDecimal(d)));		//打印结果：2.01051 失败



System.out.println(nf.format(new BigDecimal(String.valueOf(d))));	//打印结果：2.01052 又成功了
```

  通过上边的代码可知，为了避免精度丢失，尽量将浮点数转为BigDecimal，并且，要使用BigDecimal.valueOf()函数，如果要用new函数，就要现将浮点数转为字符串，否则同样会丢失精度。

  接下来我们还是继续上边的例子，我们对d*100进行四舍五入的操作：

```java
nf.setMaximumFractionDigits(3);//保留三位小数



nf.setRoundingMode(RoundingMode.HALF_UP);//四舍五入



System.out.println(nf.format(BigDecimal.valueOf(d*100)));//打印结果：201.051 怎么又失败了



System.out.println(nf.format(new BigDecimal(String.valueOf(d*100)));//打印结果：201.051 失败



System.out.println(BigDecimal.valueOf(d*100).setScale(3,BigDecimal.ROUND_HALF_UP));//打印结果：201.051  使用BigDecimal同样失败
```

  按照之前说的转为BigDecimal，但是还是出现错误，原因出在2.010515*100上，直接打印一下看看:

```java
System.out.println(d*100);//打印结果：201.05149999999998，不是2.010515
```

  结果精度丢失了，变成了201.05149999999998，所以上面的问题不是四舍五入的模式有bug，而是精度又丢失了，就相当于对201.05149999999998保留三位，四舍五入后就变成了201.051，而不是201.052，怎么解决？还是使用BigDecimal：

```java
System.out.println(nf.format(BigDecimal.valueOf(d).multiply(BigDecimal.valueOf(100))));//打印结果：201.052 成功
```

  通过BigDecimal运算函数来替换运算符，可以保证精度的不丢失，所以成功了，但是并不是所有的情况我们都能转化，比如d*100是作为double参数传递到我们的函数中的，我们无法干预函数之外的行为，怎么办？

  对于上面的丢失精度问题，可以先判断一下小数位数，当超多12位的时候认为丢失精度了，先进行一次保留12位小数的四舍五入，然后再进行实际位数的四舍五入（不确定该方法是否通用）：

```java
nf.setMaximumFractionDigits(12);



BigDecimal bd = new BigDecimal(nf.format(BigDecimal.valueOf(d*100)));



nf.setMaximumFractionDigits(3);



System.out.println(nf.format(bd));//打印结果：201.052 达到目的
```

  或者使用BigDecimal：

```java
System.out.println(BigDecimal.valueOf(d*100).setScale(12,BigDecimal.ROUND_HALF_UP).setScale(3,BigDecimal.ROUND_HALF_UP));//打印结果：201.052
```



  **5，正负号问题**

  两个浮点数相减，比如：0.0003-0.0005，取3位小数位：

```java
NumberFormat nf = NumberFormat.getNumberInstance();



nf.setMaximumFractionDigits(3);



nf.setMinimumFractionDigits(nf.getMaximumFractionDigits());



System.out.println(nf.format(0.0003-0.0005));//打印结果：-0.000



System.out.println(BigDecimal.valueOf(0.0003-0.0005).setScale(3,BigDecimal.ROUND_HALF_UP));//0.000
```

  NumberFormat的结果是-0.000，BigDecimal方式的结果是0.000



  对于浮点数相关的问题还有很多其他的，等想起来再补充吧