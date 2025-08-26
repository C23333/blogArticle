---
title: Docker小知识
date: 2024-01-02 22:55:25
tags: Docker
keywords: Docker
categories: 理论
---



## Docker小知识



### docker镜像下载到本地，并导入其他机器docker

* 拥有镜像的机器

  * `docker images` 查看本地镜像

    ![image-20221227155440387](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202212271554453.png)

  * `docker save mysql:5.7.38-debian > /home/liuyongkai/mysql5.7.38-debian`
    mysql:5.7.38-debian为镜像Name

  * 拷贝镜像文件到本地

* 目标机

  * 上传文件到本地
  * `docker load < mysql5.7.38-debian`

* 查看目标及镜像，看是否load成功

  ![image-20221227155920535](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202212271559575.png)

* 如果镜像的`REPOSITORY`和`TAG`为none，则通过`docker tag c39b1bfebc65 mysql:5.7.38-debian`来修改标签





### DockFile

#### 1.Dockerfile是干啥的

可以这么理解

* Dockfile是原材料
* Docker镜像是软件的交付品
* Docker容器则可以认为是软件镜像的运行态，即依照镜像运行的容器实例
* Dockfile面向开发，Docker镜像则是交付标准，Docker容器则涉及部署和运维，三者缺一不可

#### 2.Dockfile基础机制

* **每条保留字指令都必须为大写字母**且后面跟随至少一个参数
* 指令按照从上到下，顺序执行
* #表示注释
* 每条指令都会创建一个新的镜像层并对镜像层进行提交

#### 4.常用保留字

* FROM：基础镜像，当前新镜像是基于那个镜像的，指定一个已经存在的镜像作为模板，第一条必须是FROM

* MAINTAINER：镜像维护者的姓名和邮箱地址

* RUN：容器构建时需要运行的命令

  * 两种格式：

    * shell格式：

      ```dockerfile
      RUN apt install vim
      ```

    * exec格式：

      ```dockerfile
      RUN ["可执行文件", "参数1", "参数2"]
      # 例如：
      # RUN ["./test.php", "dev", "offline"] 等价于 RUN ./test.php dev offline
      ```

  * RUN是在`docker build`时运行的

* EXPOSE：当前容器对外暴露出的端口

  ```dockerfile
  EXPOSE 8080
  .......
  # 声明容器运行时提供8080端口
  ```

  * 格式为 `EXPOSE <端口1> [<端口2>...]`。

    `EXPOSE` 指令是声明容器运行时提供服务的端口，**这只是一个声明，在容器运行时并不会因为这个声明应用就会开启这个端口的服务**。在 Dockerfile 中写入这样的声明有两个好处，一个是帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；另一个用处则是在运行时使用随机端口映射时，也就是 `docker run -P` 时，会自动随机映射 `EXPOSE` 的端口。

    要将 `EXPOSE` 和在运行时使用 **`-p <宿主端口>:<容器端口>`** 区分开来。`-p`，是映射宿主端口和容器端口，换句话说，就是**将容器的对应端口服务公开给外界访问**，而 **`EXPOSE` 仅仅是声明容器打算使用什么端口而已，并不会自动在宿主进行端口映射**

* WORKDIR：指定在创建容器后，终端默认登录进来的工作目录，落脚点

* USER：指定该镜像以什么样的用户去执行，如果不指定，则**默认为root**

* ENV：用来在构建镜像过程中设置环境变量

  ```dockerfile
  ENV PHP_PATH /etc/php/7.2
  # 这个环境变量可以在后续的任何RUN指令中使用，这就如同在命令前面指定了环境变量前缀一样；也可以在其他指令中直接使用这些环境变量
  # 比如
  WORKDIR PHP_PATH
  ```

* VOLUME：容器数据卷，用于数据保存和持久化工作

  * 格式为：

    * VOLUME ["<路径1>", "<路径2>"...]
    * VOLUME <路径>

  * 之前我们说过，**容器运行时应该尽量保持容器存储层不发生写操作**，对于数据库类需要保存动态数据的应用，其数据库文件应该保存于卷(volume)中，后面的章节我们会进一步介绍 Docker 卷的概念。为了防止运行时用户忘记将动态文件所保存目录挂载为卷，在 `Dockerfile` 中，我们可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据

    ```dockerfile
    VOLUME /data
    ```

    这里的 `/data` 目录就会在容器运行时自动挂载为匿名卷，任何向 `/data` 中写入的信息都不会记录进容器存储层，从而保证了容器存储层的无状态化。当然，运行容器时可以覆盖这个挂载设置。比如：

    ```dockerfile
    $ docker run -d -v mydata:/data xxxx
    ```

    在这行命令中，就使用了 `mydata` 这个命名卷挂载到了 `/data` 这个位置，替代了 `Dockerfile` 中定义的匿名卷的挂载配置。

* ADD：将宿主机目录下的文件拷贝进镜像中，并且会自动处理URL和解压tar压缩包 