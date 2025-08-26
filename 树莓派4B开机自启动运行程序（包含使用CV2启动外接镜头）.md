---
title: 树莓派4B开机自启动运行程序（包含使用CV2启动外接镜头）
date: 2021-05-21 06:21:55
tags:
  - 树莓派
  - Linux
  - OpenCV
keywords: 树莓派
categories: 实操
---



# 树莓派4B开机自启动运行程序（包含使用CV2启动外接镜头）

> 转自：https://blog.csdn.net/cxxxxxxxxxxxxx/article/details/109369981

#### 对于文献【2】中的方法一：向rc.local文件添加启动代码

自启动无效

#### 对于文献【2】中的方法二：将程序作为服务启动

自启动无效

#### 对于文献【2】中的方法三：通过桌面启动

根据提供的代码启动.sh文件，.sh文件运行有效，但重启后自启动无效。
参考【1】中的.desktop文件参数设置，并直接将 “Exce = /home/pi/…/XXX.sh”更换为想要运行的.py脚本，不再通过启动shell脚本启动python脚本。（路径保证为绝对路径）
程序自启动成功，相机启动正常。

此外，有些会遇见shell脚本多次启动的问题，上述方法只启动一次，仍将解决方法记录如【3】

参考博文：

### 1.树莓派开机启动脚本

1 开机启动 [python](https://so.csdn.net/so/search?from=pc_blog_highlight&q=python) 脚本

  一般脚本，可在 **/home/pi/.config/autostart** 路径下新建 .desktop 文件，文件主要内容如下：

![1](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/1.png)

  此种方案与 windows 的 开始菜单  启动中添加 程序类似，会在系统桌面加载完成后启动。并且此文件可直接拖放至桌面，类似于应用程序，可双击执行。

2 开机启动terminal

上述方案的问题是，不能在开机时启动terminal，也就是如果python脚本没有界面，则开机之后看似没有任何反应，但通过ps 可查询到相应的脚本在运行，如图

![2](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/2.png)

分析原因，主要原因是树莓派的terminal 是 lxterminal，那么解决方案如下：

（1） 建立desktop 文件，开机执行 lxterminal ，经过此更改后，发现开机会启动terminal， desktop 如下图：

![3](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/3.png)

2）但怎么在terminal中执行脚本呢？查询terminal 参数

![4](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/4.png)

根据以上参数，经过测试，以下脚本可正常开机执行

Exec=lxterminal  --working-directory=/home/pi/test/ --command=./test.sh

注意：必须先设置  --working-directory 不能直接 -e 或者 --command= 这样并没有正确执行脚本

那么怎么执行python 脚本呢 ？当然是写在 test.sh 里喽，不要忘记加权限。附：test.sh

 

#!/bin/bash
echo "run test!"

python /home/pi/test/test.py

树莓派 开机启动脚本 python 命令行

### 2树莓派程序开机自启动方法总结



刚上手树莓派，也因此接触[Linux](https://so.csdn.net/so/search?from=pc_blog_highlight&q=Linux)，对Linux系统很多机制都不熟悉，先前想把自己写的一个程序随树莓派开机启动，搜寻并尝试了网上各种方法，经过一番折腾，总结了四种实现开机自启动的方法。

# 制作测试脚本

首先我们需要制作一个脚本来测试自启动是否有效。在终端下输入并回车新建脚本文件testboot.sh

> pi@raspberry:~ $ nano testboot.sh

testboot.sh文件内容如下：

> \#!/bin/sh
>
> touch /home/pi/testboot.txt
>
> chmod 777 /home/pi/testboot.txt
>
> echo "hello pi~" >> /home/pi/testboot.txt

测试脚本将打印字符串到文件中。按ctrl+o保存文件，再按ctrl+x退出编辑器。

给脚本文件添加执行权限：

> pi@raspberry:~ $ chmod 777 testboot.sh

测试一下脚本功能：

> pi@raspberry:~ $ ./testboot.sh

执行正常的话会在当前目录（pi）生成一个testboot.txt的文本文件。显示文件内容：

> pi@raspberry:~ $ cat testboot.txt

![img](https://upload-images.jianshu.io/upload_images/1338443-122bc6d0326ed883?imageMogr2/auto-orient/strip%7CimageView2/2/w/563)

# 

# 添加自启动

#### 方法一：向rc.local文件添加启动代码

修改rc.local文件，在终端输入并回车：

> pi@raspberry:~ $ sudo nano /etc/rc.local

在打开的文本中找到exit 0，在此之前添加的代码在启动时都会被执行，在exit 0 之前添加一行代码：

> su pi -c "exec /home/pi/testboot.sh"

ctrl+o保存，ctrl+x退出，然后在终端输入：sudo reboot ,重启系统测试。

su命令是指定在pi用户下执行这条命令，-c 表示执行完这条命令之后恢复原来的用户。

**注意：系统启动时在执行这段代码时是使用root用户权限的，如果不指定pi用户，可能会因为权限问题导致脚本执行失败。**

### 方法二：将程序作为服务启动

在/etc/init.d/目录下新建一个服务脚本文件。在终端输入并回车

> pi@raspberry:~ $ sudo nano /etc/init.d/testboot

在空白文件中输入以下内容：

> \#!/bin/sh
>
> \#/etc/init.d/testboot
>
> \### BEGIN INIT INFO
>
> \# Provides:testboot
>
> \# Required-Start:$remote_fs $syslog
>
> \# Required-Stop:$remote_fs $syslog
>
> \# Default-Start:2 3 4 5
>
> \# Default-Stop:0 1 6
>
> \# Short-Description: testboot
>
> \# Description: This service is used to start my applaction
>
> \### END INIT INFO
>
> case "$1" in
>
>    start)
>
>    echo "start your app here."
>
>    su pi -c "exec ~/testboot.sh"
>
>    ;;
>
>    stop)
>
>    echo "stop your app here."
>
>    ;;
>
>    *)
>
>    echo "Usage: service testboot start|stop"
>
>    exit 1
>
>    ;;
>
> esac
>
> exit 0

ctrl+o保存，ctrl+x退出。

设置脚本可执行权限：

> pi@raspberry:~ $ sudo chmod 777 /etc/init.d/testboot

最后将该脚本作为服务设置开机自动加载：

> pi@raspberry:~ $ sudo update-rc.d testboot defaults

sudo reboot 重启测试。

#### 方法三：通过桌面启动

此方法是在加载了桌面后再启动我们自定义的程序，因此需要安装带有桌面的版本，如果不是请跳过。

在/home/pi/.config/目录下新建一个名为 autostart 的文件夹：

> pi@raspberry:~ $ mkdir .config/autostart

在 autostart 目录下新建testboot.desktop （经测试名字任意，但后缀必须是.desktop）：

> pi@raspberry:~ $ nano .config/autostart/testboot.desktop

文件内容如下：

> [Desktop Entry]
>
> Type=Application
>
> Name=testboot
>
> NoDisplay=true
>
> Exec=/home/pi/testboot.sh

sudo reboot 重启测试。

**注意：这个方法除了依赖桌面之外，如果开启了多个桌面则会导致自定义的程序多次启动。比如系统启动桌面会调用一次testboot.sh脚本，如果再用远程桌面登录到树莓派，脚本会再执行一次。**

### 方法四：使用systemctl设置服务

在/usr/lib/systemd/system/ 下新建文件testboot.service:

> pi@raspberry:~ $ sudo nano /usr/lib/systemd/system/testboot.service

如果目录system不存在，请自行创建：

> pi@raspberry:~ $ sudo mkdir /usr/lib/systemd/system

testboot.service文件内容如下：

> [Unit]
>
> Description=testboot
>
> [Service]
>
> Type=oneshot
>
> ExecStart=/home/pi/testboot.sh
>
> [Install]
>
> WantedBy=multi-user.target

这里直接指定启动文件的路径，无法指定到pi用户执行，所以只能在root用户下执行。

设置服务自启动：

> pi@raspberry:~ $ sudo systemctl enable testboot.service

**注意：这个方法与方法二类似都是通过服务启动，所以如果两种方法同时使用要注意不能使用同个服务名。**

# 总结

除了通过桌面启动以外，其他方式在执行启动代码的时候都是用root用户在执行的，所以需要特别注意权限的问题，最好就全部都指定到pi用户去执行。除了可以执行脚本之外，也可以启动自己写的程序或者python脚本，需要注意的是如果自启动的程序有依赖于其他服务则必须等待其他服务加载完毕才能正常启动，保险的做法延时后再启动。



### 如何避免shell脚本被同时运行多次

转自：http://www.etwiki.cn/linux/2786.html

比如说有一个周期性(cron)备份mysql的脚本，或者rsync脚本，

如果出现意外，运行时间过长，
很有可能下一个备份周期已经开始了，当前周期的脚本却还没有运行完，
显然我们都不愿意看到这样的情况发生。

其实只要对脚本自身做一些改动，就可以避免它被重复运行。


\#!/bin/bash

LOCK_NAME="/tmp/my.lock"
if [[ -e $LOCK_NAME ]] ; then
echo "re-entry, exiting"
exit 1
fi

\### Placing lock file
touch $LOCK_NAME
echo -n "Started..."

\### 开始正常流程
\### 正常流程结束

\### Removing lock
rm -f $LOCK_NAME

echo "Done."


当脚本开始运行时， 创建 /tmp/my.lock文件，
这时如果再次运行此脚本，发现存在my.lock，就退出，
脚本运行结束时删除这个文件。

大多数情况下，这样做都没有什么问题。
意外1) 如果同时运行二次此脚本， 二个进程都会发现my.lock不存在，然后都可以继续执行。
意外2) 如果脚本在运行过程中意外退出， 没有来得及删除 my.lock文件， 那么就悲剧了。

修改如下：


\#!/bin/bash

LOCK_NAME="/tmp/my.lock"
if ( set -o noclobber; echo "$$" > "$LOCK_NAME") 2> /dev/null; 
then
trap 'rm -f "$LOCK_NAME"; exit $?' INT TERM EXIT

\### 开始正常流程
\### 正常流程结束

\### Removing lock
rm -f $LOCK_NAME
trap - INT TERM EXIT
else
echo "Failed to acquire lockfile: $LOCK_NAME." 
echo "Held by $(cat $LOCK_NAME)"
exit 1
fi



echo "Done."


set -o noclobber 的意思：


If set, bash does not overwrite an existing file with the >, >&, and <> redirection operators.


这样就能保证my.lock只能被一个进程创建出来。比touch靠谱多了。

trap 可以捕获各种信号，然后做出处理：
INT 用来处理 ctrl+c取消脚本执行的情况。
TERM 用来处理 kill -TERM pid 的情况。
EXIT 不清楚

另外，对于 kill -9 无效。。

还记得N年前，在php群里面，草人也问过这个问题，
我们给的答案是 ps aux|grep filename |wc -l ，哈哈，真2。