---
title: 飞桨高层API训练营-Day3
date: 2022-10-03 21:25:36
tags:  PaddlePaddle
keywords: PaddlePaddle
categories: 实操
---



# Day3

> 学习内容 ：人脸**关键点检测**

[TOC]

### 基础概念

* CHW图像
  * C ： channel 通道数
  * H ： 高度
  * W ： 宽度

* 损失函数：图像分类VS人脸关键点检测

![image-20210205205251034](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20210205205251.png)

* 评估指标1

![image-20210205205354854](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20210205205355.png)





### API

* paddle.set_device('GPU')
  * 设置AIStudio使用GPU
* **对于图像的操作API** 含于paddle.visoin.transform



### 扩展

* 更高精度模型
  * ![image-20210205215542102](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20210205215542.png)
  * 