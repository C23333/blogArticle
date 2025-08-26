---
title: 构建STM32最小系统板注意事项
date: 2022-04-02 08:19:22
tags:
  - STM32
  - PCB
keywords: STM32
categories: 实操

---





# 构建STM32最小系统板注意事项



## 1.VBAT引脚

>  在主流的设计中，VBAT与0欧的电阻串联，接至3.3V。

## 2. OSC32_IN与OSC32_OUT

> 32.768k的rtc时钟用于精确定时，待机唤醒时钟。根据您的需要判断是否添加。如果您不需要待机状态的定时功能的话，可以不用外接晶振。

![886966-20181230112253910-1551765500](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200613012319.png)

![886966-20181230113647984-2042922198](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200613012343.png)



## 3.XTAL_IN与XTAL_OUT。

> 外部时钟晶振不是必须要接8M，官方数据写的是4-16MHz，然后经过pll倍频后给其它外设提供时钟信号。
>
> 比如说系统最大主频就是由它倍频得到的。

![886966-20181230111833022-463224457](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200613012542.png)

![886966-20181230113553465-210385334](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200613012548.png)



## 4.BOOT0与BOOT1配置启动方式

> BOOT1=x BOOT0=0 从用户闪存启动，这是正常的工作模式。
> BOOT1=0 BOOT0=1 从系统存储器启动，这种模式启动的程序功能由厂家设置。
> BOOT1=1 BOOT0=1 从内置SRAM启动，这种模式可以用于调试。
>
> 实际设计中，BOOT0设计为可以调节的方式。
>
> ​           BOOT1设计为0。
>
> （我不理解的是，为什么要经过10k电阻接地呢？欢迎交流，有文章说是为了改善emc）

![1](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200613012704.png)

![2](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200613012709.png)

![886966-20181230114219243-141809961](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200613012818.png)



## 5.SWD下载方式：

>   SWD下载方式只需要NRST（复位），TCLK（时钟），TMS（信号），GND四个引脚。个人习惯了这种下载方式。再简单一点的话，NRST也是可以省掉的，下载完程序可以手动复位。

![886966-20181230114219243-141809961](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200613012855.png)



## 6.NRST系统复位

> 复位的方式有很多种，这里就不一一叙述了。

![1](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200613012938.png)

![批注 2020-06-13 013037](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200613013048.png)

## 7.供电

> VDDA，VDD1，VDD2，VDD3 该供电3V3的就供电3V3
>
>   VSSA，VSS1，VSS2，VSS3  该接地的就接地。
>
>   同时，VDD 与 VSS 之间需要滤波。

![886966-20181230114946267-1839413024](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200613013134.png)