---
title: 飞桨高层API训练营-Day5
date: 2022-10-05 21:36:57
tags:  PaddlePaddle
keywords: PaddlePaddle
categories: 实操
---

# Day5

> 学习内容： NLP自动春节对联

[TOC]

## 本节学习内容

![image-20210207204624230](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20210207204624.png)

## Seq2Seq (sequence to sequence)

![image-20210207204952847](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20210207204953.png)

## 注意力机制通俗解释

* 在处理后续信息时，给予**前置信息**不同**权重**

![image-20210207205616738](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20210207205617.png)

## 数据处理流程

![image-20210207210257019](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20210207210257.png)

* 对联有头尾，故须在句子的开头和结尾加上 **begID**和**endID**

![image-20210207210740101](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20210207210740.png)

## 注意力机制（公式）

![image-20210207213354996](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20210207213355.png)

