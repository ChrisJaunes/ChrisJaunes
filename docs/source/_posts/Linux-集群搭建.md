---
title: Linux 集群搭建
date: 2021-05-28 12:05:14
categories: Linux
tags: Linux
excerpt: Linux
---



# 简介

出于各种原因，我们手头可能就一台计算机，但我们又需要模拟多台计算机的场景，因此在本地虚拟计算机集群有很有必要了。

为了使得我们能够去虚拟计算机集群，需要用到虚拟机这个概念。虚拟机(Virtual Machine) 是在计算机科学中的体系结构里，是指一种特殊的软件，可以在计算机平台和终端用户间创建一种环境，而终端用户则是基于虚拟机这个软件创建的环境来操作其他软件。虚拟机(VM) 是计算机系统的仿真器，通过软件模拟具有完整硬件系统功能的、运行在一个完全隔离环境中的完整计算机系统，能提供物理计算机的功能。[^1] 

## 安装

### 安装VM

我们需要安装系统虚拟机，常见的系统虚拟机有VMware Workstation。本篇文章使用的虚拟机是VMware Workstation pro 16， 安装没有什么特别需要注意的地方，根据自己的需求进行选择就可以了。如果囊中羞涩，也可以在网上找一些许可证。[^2]

### 添加虚拟机

首先我们需要一个操作系统镜像，本文使用的是Linux发行版 Centos 8。[下载地址](http://mirrors.aliyun.com/centos/8.3.2011/isos/x86_64/CentOS-8.3.2011-x86_64-dvd1.iso)

如果不太熟悉虚拟机，可以考虑简易安装。点击菜单栏的 "文件"->"新建虚拟机"，然后在弹窗中选择"典型(推荐)"，接着"下一步"，选择"安装程序光盘镜像文件(iso)(M)", 点击"下一步", 然后输入账号信息进行简易安装。

如果对于虚拟机比较熟悉，可以考虑自定义的方式去设置虚拟机，很多东西可以去配置。

首先虚拟出一台计算机。点击文件->新建虚拟机，然菜单栏后在新建虚拟机的弹窗中进行配置，其配置取决于我们的需求，而且后期是可以修改的。

下一步是安装虚拟系统，本文使用的虚拟系统是Linux的发行版Centos 8。可以考虑简易安装，也可以设置CD/DVD(右键点击虚拟机名称->设置->硬件->CD/DVD)里的使用ISO镜像文件然后从镜像安装。安装同样没有特别要注意的地方。

下一步是配置网络。主要设置静态的ip地址，配置文件位于:  /etc/sysconfig/network-scripts, 然后修改网卡里的配置信息即可。

```shell
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=f81c8a21-dc9b-4276-b4b3-9136497b6def
DEVICE=ens33
ONBOOT=yes
NM_CONTROLLED=yes
BOOTPROTO=static
IPADDR=192.168.42.129
GATEWAY=192.168.42.1
BROADCAST=192.168.42.255
DNS1=114.114.114.114
DNS2=8.8.8.8
```

现在我们有了一台虚拟的计算机了。

所谓计算机集群，其实就是很多台计算机而已，他们通过网络进行通信。

### 创建多台虚拟机

使用克隆功能就可以了

右键点击我们需要克隆的虚拟机， "管理" -> "克隆"， 进入克隆界面。

"虚拟机中的当前状态"->"创建完整的克隆" -> 填写名称和位置 -> 克隆



管理多台机器

```shell
sudo yum -y install epel-release

sudo yum -y install ansible
```







[^1]: https://zh.wikipedia.org/wiki/%E8%99%9B%E6%93%AC%E6%A9%9F%E5%99%A8
[^2]:  VMware-workstation-full-16.1.2-17966106的一个许可证是ZF3R0-FHED2-M80TY-8QYGC-NPKYF

