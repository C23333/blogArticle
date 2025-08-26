---
title: markdown插入网易云播放模块
date: 2022-10-31 20:53:27
tags: markdown
keywords: markdown
categories: 实操
---



# markdown插入网易云播放模块



> 转自：https://blog.csdn.net/weixin_60625619/article/details/122761864



## 核心代码

~~~HTML
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width="100%" height="100" src="https://music.163.com/outchain/player?type=2&amp;id=38018486&amp;auto=1&amp;height=100"></iframe>
~~~



## 一、获取歌曲id

![](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20220319190214.png)

**从复制的链接中得到id，将核心代码中的id替换为获得的**



## 检查元素后加入代码

** 其中，height为插入模块的高度，auto为 1 时，为自动播放模式*（但某些浏览器出于某些考量仍不会自动播放）。为 0 时为非自动播放。



## 最终效果

![image-20220319191307895](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/20220319191307.png)