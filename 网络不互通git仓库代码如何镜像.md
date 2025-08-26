---
title: 网络不互通git仓库代码如何镜像？
date: 2024-02-02 02:22:22
tags: Git
keywords: Git
categories: 实操
---



# 网络不互通git仓库代码如何镜像？

> 这里的网络不互通示意如下
>
> ![image-20240301091502806](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202403010915871.png)

**仓库操作如下：**

```bash
# 假设现在要从 repoA 迁移仓库到repoB
# 完整克隆要迁移的仓库，包含分支和代码
# 使用--mirror选项克隆，它会克隆所有的分支、标签以及提交记录，创建一个裸仓库（bare repository）。这意味着这个克隆包含了仓库的所有Git数据，但没有工作目录的文件
git clone --mirror <repoA-URL> old.git
 你 
# 在 repoB 上创建一个新的代码库，保存仓库URL

# 修改 repoA 代码库的远端地址
cd old.git
git remote add repoB <repoB-URL>

# 推送仓库到 repoB
# --mirror选项会推送所有的引用（包括分支、标签等）
git push --mirror repoB
```

**开发人员修改远端仓库地址**

*不保留旧远端地址*

~~~bash
git remote set-url origin <repoB-URL>
~~~



*保留旧远端地址*

~~~bash
# 备份原来远端地址
git remote rename origin old-origin

# 添加新的远端地址
git remote add origin <repoB-URL>

# （可选）删除旧仓库远端地址
git remote remove old-origin
~~~

