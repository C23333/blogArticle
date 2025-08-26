---
title: 配置Typora+PicGo+gitee过程中个别问题的解决
date: 2029-11-25 02:25:47
tags:
  - PicGo
  - Typora
  - 图床
keywords: 图床
categories: 实操
---



# 配置Typora+PicGo+gitee过程中个别问题的解决



> 最近学习使用Typora，在安装配置中遇到了一些问题，分享一下个人的解决过程。

## 问题一：验证图片上传失败

* 问题：

​       在Typora设置好PicGo上传服务后，无法验证图片上传选项。（PicGo没有问题）

![11927101-3fa4371e5a53c599](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200614215108.png)

* 出现验证失败。

![11927101-cb31b1b1b684a650](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200614215131.png)

**分析：**

* 可能是PicGo的服务器默认端口与Typora不一致（注：程序运行结果是：**Failed to fetch**）或图床上已经有验证过的默认图片（注：程序运行结果是：**{“success”,false}**）。

**解决：**

* 查看"设置PicGo-Server"窗口的监听端口是否一致，如果不一致则修改。

![11927101-d47a9bb26903217f](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200614215215.png)



![11927101-17d13e13df0c29be](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200614215243.png)

* 检查图床上是否已经有下面两个文件，如有删除后再验证就行了。

![11927101-f67e53e4e5e48373](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200614215302.png)

**成功验证：**

![11927101-25ae6c9d258eaab7](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200614215318.png)



## 问题二：无法粘贴图片到Typora文档

* 问题：

​        当图床正确配置后，在粘贴图片到Typora文档中时出现“Error”弹窗错误：



![11927101-a676b7dbca756f94](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200614215414.png)

**分析：**

* 这是因为当前用户对“C:\Program Files\Typora”文件夹没有“创建文件夹”操作权限造成。

**解决：** 

* 给当前用户配置对文件夹Typora足够权限就可以了。在文件夹Typora属性的“**安全**”选项卡，单击“**编辑**”按钮，打开“**Typora权限**”窗口。

![11927101-a7a350a5ffe4d154](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200614215454.png)



添加用户并设置其权限即可。

![11927101-602eba68ce63b4ea](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200614215504.png)

## 问题三：粘贴图片时“image load failed”错误

* 问题：

  在文档中粘贴图片时出现“**image load failed**”错误，如下图：

  ![11927101-15484ad6a6fe5aa7](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200614215612.png)

**分析：**

* 这里问题有可能是图片在粘贴后，程序需要先在upload文件夹中创建和保存它，但未成功。经分析可能是路径出了问题。

**解决：**

* 打开Typora的“**编好设置**”，在图像配置中“**插入图片时...**”选项里，勾选“**优先使用相对路径**”即可。

![11927101-7144903dd5f75675](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200614215839.png)



