---
layout: post 
title:  "由一个rsync问题引发的一系列学习"
date:   2017-08-20 19:28:00 +0800
excerpt_separator: <!--abstract-->
tags: rsync chkconfig systemd rc.d
---

以前对rsync命令的印象，就是可以很方便传数据，具体如何搭建、使用，以及安全方面，知道的比较少。最近帮同事处理rsyncd.conf相关问题，也就顺便研究了下。这一下子不得了，发现好多东西似懂非懂，按下葫芦浮起瓢，就一并都看看吧。rsync、守护进程、xinetd、/etc/init.d/*、checkconfig、rc.local等。<!--abstract-->


### rsync服务端使用

- 启动方式：`rsync --daemon`
- 配置：默认配置为/etc/rsyncd.conf，可以通过参数 `--config=/PATH/TO/CONFIG/FILE` 指定配置文件。
- 配置内容：

```
uid=root
gid=root
max connections=4
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsyncd.lock
log file = /var/log/rsyncd.log
# 默认端口873，可以通过设置修改
port=8873

[module]
	comment = test module
	path = /home/user
	hosts allow = 127.0.0.1/16 192.168.10.10/32
	read only = yes
	timeout = 600
	# 带密码验证，密码文件需改为root拥有，且权限为600，文件内容为user:password
	secrets file = /etc/rsyncd.password
```
使用方式：
```

# 查看文件列表，展示/home/user目录下内容
rsync ip::module  

# 上传文件
rsync file.txt ip::module

# 下载文件
rsync ip::module/file.txt .

# 带安全验证情形下：
rsync user@ip::module 
Password: password

# 或使用password文件，文件内容为密码

rsync --password-file=/PATH/TO/PASSWORD ip::module

```


rsync一般启动方式是：`rsync --daemon --config=FILE`，那么 `--daemon`又是个什么玩意呢？

看了下man rsync，其实就是个rsync命令的参数，囧。。。

但是！这里得有但是。加了`--daemon`之后，程序就相当于一个守护进程，跑在机器上。那么，什么是守护进程？

### 守护进程

守护进程是一个在后台运行，不受任何终端控制的进程。

守护进程又分为独立运行的守护进程和由xinetd管理的守护进程。

独立运行的守护进程，在机器重启后，不会重新启动，需要再次执行命令才能启动；而由xinetd管理的守护进程，如果对应程序配置文件中，disable标记为no的情况下，重启后会自动拉起。

那么，问题又来了，要是我想让某些系统进程，再开机时候启动怎么办呢？

就要说到chkconfig了。

阮一峰博客一篇比较详尽的守护进程博文：[Linux 守护进程的启动方法](http://www.ruanyifeng.com/blog/2016/02/linux-daemon.html)


### chkconfig

chkconfig 提供了一个维护/etc/rc[0~6].d 文件夹的命令行工具，它减轻了系统直接管理这些文件夹中的符号连接的负担。chkconfig主要包括5个原始功能：为系统管理增加新的服务、为系统管理移除服务、列出当前服务的启动信息、改变服务的启动信息和检查特殊服务的启动状态。当单独运行chkconfig命令而不加任何参数时，他将显示服务的使用信息。

为啥有/etc/rc[0~6].d，七个文件夹呢？又得聊聊linux的启动等级了。

```
等级0表示：表示关机
等级1表示：单用户模式
等级2表示：无网络连接的多用户命令行模式
等级3表示：有网络连接的多用户命令行模式
等级4表示：不可用，系统保留
等级5表示：带图形界面的多用户模式
等级6表示：重新启动
```
所以我们一般设置一个服务为开机启动时候，会设置它为 level35，也就是等级3和等级5。

centos6使用chkconfig命令来将服务添加为开机启动。

```
# 增加指定的系统服务，让chkconfig命令管理，服务脚本需在/etc/init.d目录下。
chkconfig --add SERVICE_NAME   

# 执行系统服务要在哪个指定等级中开启或关闭，这里指定等级3和等级5。
# 相当于给/etc/rc.d/rcN.d目录下加/etc/init.d/下脚本的软链，N为level。
chkconfig --level 35 SERVICE_NAME on/off  
```

但是！问题又来了，当我们在centos 6上面运行 `chkconfig` 命令的时候，会发现：

```
xinetd based services:
	rsync:         	off
```

这说明，rsync服务是不通过chkconfig管理的，而是通过xinetd服务管理的。如果在centos 6上，调用 `chkconfig --level 35 rsyncd on`，会看到如下报错：

```
~$ chkconfig --level 35 rsyncd on
error reading information on service rsyncd: No such file or directory
```

同时，运行 `service rsyncd start` 会报错。

service命令会去`/etc/init.d/`目录下，查看是否有rsyncd这个脚本可以执行，没有的情况下，就会报错。

所以，在centos 6系统上，无法通过chkconfig来设置rsyncd服务开机启动。

我们也可以通过编写一个rsyncd脚本，放到/etc/init.d目录下，这样就可以通过chkconfig命令，设置开机运行rsyncd脚本，而脚本中，需要获取`start|stop|restart`等命令。

而在centos 7系统上，service命令已经被集成在了systemctl命令下，可以通过 `service start SERVICE_NAME` 启动服务和 `chkconfig --level 35 SERVICE_NAME` 配置开机自启动，相当于：

```
systemctl start SERVICE_NAME

systemctl enable SERVICE_NAME
```

systemctl是systemd服务命令集中的一员。详细的systemd介绍，可以看下阮一峰的微博，感觉讲的很详细：

[Systemd 入门教程：命令篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html)

[Systemd 入门教程：实战篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html)

[Node 应用的 Systemd 启动](http://www.ruanyifeng.com/blog/2016/03/node-systemd-tutorial.html)


### rc.xxx

先来聊下rc.local。

Linux的初始化init系统，RHEL 5为SysVinit，RHEL 6为Upstart， RHEL 7为Syetemd

/etc/rc.local 是 /etc/rc.d/rc.local的一个软链，开机时候会在加载完其他初始化脚本后，即/etc/rc.d/rcN.d/目录下那些初始化脚本顺序执行后，加载rc.local。

而/etc/rc.d/init.d/ 目录，会软链到/etc/init.d目录，这个也是`chkconfig --level N on`的时候，软链到/etc/rc.d/rcN.d/ 目录下的那些系统服务启动脚本。

在centos 6系统，`cat /etc/rc.local` 可以看到：
```
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.
```
这里，系统还建议我们将希望在启动时执行，但是不想写成Sys V型的初始化任务写在这里。

而centos 7系统的 /etc/rc.local 文件中，写到：
```
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.
```
这个文件留在centos 7系统是为了稳定起见，建议我们创建systemd类型的服务，systemd类型服务。如果仍需使用/etc/rc.local来执行初始化任务，则需要将/etc/rc.d/rc.local文件改为可执行，否则将不会执行。


### 总结

所以，七七八八写了这么多，rsync使用起来还是很方便的，可以加到/etc/rc.local里面，开机时候启动，还可以写个rsyncd启动脚本，扔到/etc/rc.d/init.d目录下，通过chkconfig或systemctl来使其开机自启动。同时也了解到了Linux初始化服务的更替，学习了一些systemctl使用方法。虽然以前也知道有启动等级这个事情，但这次更加深刻地记住了不同的level代表什么意思。最后，还理清楚了/etc/rc.xxx到底是什么个层级结构，受益良多。

知识这个东西，感觉还是要翻来覆去的体会、品味的。第一次学习时，大概有个模糊的印象，后面接触多了，再回顾之前的知识，会有一种恍然大悟的感觉。

希望每天都能这样成长着。





