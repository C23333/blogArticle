---
title: 玩客云（armv7hf）源码编译安装OpenCV
date: 2021-05-25 09:56:24
tags:
  - Arm
  - 玩客云
  - OpenCV
keywords: 玩客云
categories: 实操
---



# 玩客云（armv7hf）源码编译安装OpenCV

> 最近手痒，从咸鱼买了个玩客云，这里记录一下折腾记录，同时电赛没有树莓派没有搞定的数字识别这里也一并研究一下

> 感谢来自<a href="http://zhaoxuhui.top/blog/2019/06/04/OpenCVContribEnvCPP.html">此博主的帮助</a>



## 一、下载源码

这里选择安装OpenCV基础包和Contrib两部分：[OpenCV](https://github.com/opencv/opencv)和[OpenCV Contrib](https://github.com/opencv/opencv_contrib)，分别点击去Github下载即可。 在各自的项目主页里点击”Releases”，如下。

![image-20220407195747969](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20220407195748.png)

选择想要的版本，下载Source Code即可。需要注意的是**OpenCV和Contrib的代码版本要一致**



## 二、配置代码目录

下载代码后，分别解压，将解压后的Contrib文件夹整体放入OpenCV的文件夹中，如下如

![image-20220407201503238](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20220407201503.png)

然后可以在OpenCV中新建一个文件夹，用来存放生成的文件。我这里建的是`build`文件夹



## 三、配置CMake

这一步我用的是CMake的GUI进行的。熟练的话也可以直接在终端中输入参数。在终端中输入

~~~bash
cmake-gui
~~~

即可打开CMake的GUI界面

在`Where is the source code`里选择OpenCV目录，在`Where to build the binaries`选择刚刚我们建立的`build`目录。 因为这里需要编译的是Contrib，因此还需要修改一些变量，如果普通安装的话那么这里不需要修改任何参数，直接Configure然后Generate就可以了。 需要修改的参数主要有以下几个：

如图进行配置

![image-20220407202146353](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20220407202146.png)

### 编译Contrib需要修改的参数变量

### OPENCV_EXTRA_MODULES_PATH

![image-20220407202345951](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20220407202346.png)

这个参数指定的是额外模块的路径，指定了之后CMake就会自动寻找是否有额外模块。需要注意的是路径是contrib目录下的`modules`，例如`/root/softwares/opencv-3.4.6/opencv_contrib-3.4.6/modules`，而不能只写到`/root/softwares/opencv-3.4.6/opencv_contrib-3.4.6`，会提示找不到模块。



### OPENCV_ENABLE_NONFREE

![image-20220407202417183](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20220407202417.png)

以上便是两个使用Contrib模块必须要修改的参数，其它没有特殊要求默认即可。配置好后，点击”Configure”按钮，CMake即开始配置，不出意外的话会会下载一些文件，这些文件都是外网的，还比较大，因此经常失败。

### Configure环节下载文件失败解决

> 我在下载过程中没有遇到过，这里提供参考博文中给出的解决办法

思路为：

* 手动下载多次下载失败的文件，自行更改文件所在文件夹下的`CMakeLists.txt`文件，将网络源地址更换为自己的本地文件地址
* 这里给出`face_landmark_model.dat`文件的<a href=:https://raw.githubusercontent.com/opencv/opencv_3rdparty/8afa57abc8229d611c4937165d20e2a2d9fc5a12/face_landmark_model.dat>下载地址</a>（<a href="/usr/uploads/2022/face_landmark_model.dat">备用地址</a>）

使用方式

![image-20220407204633504](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20220407204633.png)

![image-20220407204659215](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20220407204659.png)

### Configure和Generate

这样基本就不会再出问题了，点击`Configure`如下显示”Configuring done”。

![image-20220407204721573](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20220407204721.png)

再点击`Generate`就可以完成CMake配置了。



## 四、编译代码

在新建的`build`文件夹里打开终端，直接输入`make`就可以，至于用几个线程看你的电脑配置，我在玩客云上用三个线程`make -j3`，之前在树莓派上编译的时候只能用一个线程。 然后就是漫长的等待了，如下图。

![image-20220407195606483](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20220407195606.png)

这一步受编译环境的影响，可能会出现各种各样的问题。遇到问题多搜索，总会有和你一样问题的人。底部提供一些问题供参考，来自参考博文。

如果顺利到100%，恭喜，编译完成了。



## 五、安装代码

在完成`make`之后，直接`make install`即可将相关头文件拷贝到系统中去，之后就可以调用了。 `make install`以后，源文件不建议删除，因为如果以后还需要卸载的话，直接在`build`目录下打开终端输入`make uninstall`即可，否则的话只能手动删除目录文件了。

### 编译参考问题

#### 1.GTK2和3版本不兼容

补充一下，在使用ROS调用OpenCV的时候有时候会报GTK2和3版本不兼容、不能同时使用的问题。这个问题其实也非常好解决，在CMake的时候，找到`WITH_GTK_2_X`，打勾，再重新编译安装OpenCV即可。更多可以参考[这个网页](https://blog.csdn.net/weixin_34365635/article/details/94274099)。

## 六、参考资料

- [1]https://blog.csdn.net/u010739369/article/details/79966263
- [2]https://blog.csdn.net/CSDN330/article/details/86747867
- [3]https://www.cnblogs.com/needybeerlxy/p/8979238.html
- [4]https://blog.csdn.net/zxj_yantai/article/details/78779880
- [5]https://blog.csdn.net/u011361393/article/details/83210824
- [6]https://www.cnblogs.com/leoking01/p/8306935.html
- [7]http://zhaoxuhui.top/blog/2019/06/04/OpenCVContribEnvCPP.html

