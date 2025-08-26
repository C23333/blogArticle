---
title: 树莓派Paddle-Lite从模型训练到部署全流程
date: 2021-05-20 23:21:42
tags:
  - 树莓派
  - 边缘化部署
  - Paddle-Lite
keywords: Paddle-Lite
categories: 实操
---



# 树莓派Paddle-Lite从模型训练到部署全流程

> 此教程默认树莓派已安装Paddle-Lite推离库
>
> 若**未安装**请参考此教程：https://blog.csdn.net/weixin_40973138/article/details/114780090

[TOC]



## 一、通过PaddleX进行模型可视化训练

### 1.PaddleX前置环境准备

* N卡<a gref="https://zhuanlan.zhihu.com/p/94220564">安装CUDA和cuDNN</a>

* 安装<a href="https://www.paddlepaddle.org.cn/install/quick?docurl=/documentation/docs/zh/install/pip/linux-pip.html">PaddlePaddle</a>
* 安装<a href="https://www.paddlepaddle.org.cn/paddle/paddleX">PaddleX</a>

### 2.PaddleX-建立数据集

> 一般来说，图像分类至少需要各类100张，目标检测至少需要80张

**下面以创建目标检测数据集举例**

* 点击“新建数据集”，选择任务类型，点击“创建”

![image-20211230170640531](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230170640531.png)

* 在右侧有数据集存放的详细说明

![image-20211230170928353](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230170928353.png)

* 导入完成后，在**数据分析**处切分数据集（此图已分割完成），在右侧可以看到数据集预览

![image-20211230171019446](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230171019446.png)

* 至此，数据集创建完成

### 3.PaddleX-模型训练

> 请选择GPU好的电脑进行训练，GPU算力越高训练越快

* 新建项目，任务类型记得和数据集相同

![image-20211230171929300](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230171929300.png)

* 1.选择数据集

  2.模型参数设置和数据集增强处理选择

![image-20211230172123721](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230172123721.png)

* 可实时观察训练进度

![image-20211230172306065](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230172306065.png)

* 通过模型评估可以看到各epoch准确率
* **混淆矩阵**可以查看**预测错误图片**

![image-20211230172430629](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230172430629.png)

* 最后选择文件夹，进行“模型发布”，即将模型保存到本地

![image-20211230172617175](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230172617175.png)

### 4.PaddleX-模型通过OPT工具优化

> 详细资料可参考<a href="https://paddle-lite-pjc.readthedocs.io/zh/latest/user_guides/model_optimize_tool.html">官方文档</a>

**OPT版本请选择与Paddle-Lite推离库相同版本**

~~~txt
#通过以下指令转换， = 后根据实际情况填写
#--model_file 填写模型.pdmodel文件路径
#--param_file 填写模型.pdiparams文件路径
#optimize_out 填写导出后模型名称
./opt_linux --model_file=./model.pdmodel --param_file=./model.pdiparams --optimize_out=num1203
~~~

**导出成功后可获得.nb模型文件**

### 5.编写预测程序，实现部署

> 可参照<a herf="https://paddle-lite-pjc.readthedocs.io/zh/latest/demo_guides/cpp_demo.html">官方示例</a>

* 推荐从官方示例进行改写，节省开发时间

## 其他

### 1.PaddleX可视化训练失败怎么办？

* 进入目录下，手动运行.py文件进行训练

![image-20211230192316236](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230192316236.png)

![image-20211230192332897](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230192332897.png)

### 2.效果展示

* 目标检测（21年电赛F题示例）

![image-20211230210603763](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230210604681.png)

![image-20211230210625801](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230210625801.png)

### 3.自己电脑训练实在太慢怎么办？

* 可使用GPU租赁网站在线训练，也可使用<a href="https://aistudio.baidu.com/aistudio/index">AIStudio</a>通过完成任务（截止到写此教程时，任务很简单，关注几十个用户便可达到）获得免费算力

### 4.目标检测数据集标注有什么软件吗？

* 推荐使用<a href="http://www.jinglingbiaozhu.com/">精灵标注助手</a>
* **注意：**PaddleX目标检测数据集，截至此教程(2021/12/30、PaddleX-Ver2.0.0)仅支持矩形标注框
* 导出时选择Pascal VOC格式导出

![image-20211230200951637](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/image-20211230200951637.png)

### 5.待续
