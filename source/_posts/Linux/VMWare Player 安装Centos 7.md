---
title: VMWare Player 安装Centos 7
toc: true
date: 2024-02-27 10:53:23
tags:
- VMware
- CentOS
categories: Linux
thumbnail:
---

 对于学习技术来说，VMware Player  +  CentOS 可以算得上一对相当不错的免费工具组合，本文就是描述如何在 VMware Player 安装 CentOS 7，并且如何通过手动的方式克隆虚拟机。

- 虚拟机：[VMware Workstation 16 Player](https://www.vmware.com/products/workstation-player/workstation-player-evaluation.html)
- 操作系统：[CentOS Linux release 7.7.1908](https://mirrors.aliyun.com/centos/?spm=a2c6h.13651104.0.0.456712b2U8WrtC) 

<!--more-->

## 创建新虚拟机实例

- 点击创建新虚拟机
- 选择安装来源：稍后安装操作系统
- 选择客户机操作系统：Linux
- 设置该虚拟机实例名称和工作目录（存储该虚拟机相关的所有文件）
- 设置硬盘大小：
  -  参考：40 G；选择“将虚拟机硬盘拆分成多个文件”
- 点击完成
- 点击编辑虚拟机实例设置
  - 网络适配器：桥接模式
  - CD/DVD：使用 IOS 影像文件 -》 选择上面下载的 CentOS 7 ISO 文件

## 安装 CentOS 7

### 安装系统

- 选择虚拟机实例，点击播放虚拟机
- 进入安装界面：Install CentOS 7
- 进入安装语言界面：中文 -》 简体中文（中国）-》 继续
- 进入安装信息摘要界面：点击安装位置 -》 选择我要配置分区
- 进入手动分区界面：点击自动创建他们
- 我的手动分区参考：
  - /boot: 1024
  - /：25 GiB
  - swap: 3968 MiB
  - /home：10 GiB
- /home 分区需要手动改创建
  - 先把 / 分区减少 10 G 空间
  - 点击左下 + 号按钮
  - 挂载点：下拉选择 /home
  - 期望容量： 10 GiB
  - 点击添加挂载点
- 手动分区完成，返回安装信息摘要界面，点击开始安装
- 安装过程中会出现配置界面，点击 Root 密码配置选项，设置 root 密码
- 等待安装完成 -》 点击完成配置 -》 点击重启

### 配置网络

重启进入登录界面，输入root账号和密码登录，并查看当前网络接口名称：

```shell
[root@localhost ~]# ip addr list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ca:2b:a0 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::25c7:c0dc:4aeb:8e4/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

CentOS 7 默认的网络接口名称是 ens33，编辑该网络接口配置文件：

```shell
[root@localhost ~]# vi /etc/sysconfig/network-scripts/ifcfg-ens33
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static # 配置为静态 IP
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=70257dc8-749f-4109-a705-574684642ab8
DEVICE=ens33
IPADDR=192.168.3.131 # IP
NETMASK=255.255.255.0 # 掩码
GATEWAY=192.168.3.1 # 网关
DNS1=192.168.3.1 # DNS 可以配多个，比如：DNS2，DNS3
ONBOOT=yes # 开机自动加载
```

重启网络服务：

```shell
[root@localhost ~]# systemctl restart network
```

可以跟尝试跟同网段的其他主机互 PING，测试是否能相互通信。

> 注意：windows 10 防火墙可能会拦截 PING 请求，测试时需要关闭防火墙。

## 使用SSH客户端登录 CentOS

Linux的连接终端有很多，我这里使用的是开源的putty

![images](./images/putty.png)

输入账号密码登录成功：

![images](./images/clone_login_successful.png)

##  VMware Workstation Player 克隆虚拟机实例

如果你使用的是 VMware Workstation Pro，可以直接使用其提供的克隆功能，但是如果你使用的是 VMware Workstation Player，则可以通过文件拷贝的方式来进行虚拟机克隆。

### 拷贝工作目录

- 如果需要拷贝的虚拟机正在运行，则先关闭。
- 打开 VMware Workstation Player，主页虚拟机列表-》选择需要拷贝的虚拟机-》编辑虚拟机设置 -》 选项 -》 确认工作目录路径。
- 拷贝工作目录，并对新的工作目录进行重命名。

### 启动拷贝的虚拟机

- VMware Workstation Player 主页 -》 点击打开虚拟选项。
- 选择刚刚拷贝的文件夹，点击进去会发现一个 后缀为“.vmx”的文件，选择打开。
  ![image](./images/vmx.png)
- 此时 VMware Workstation Player 主页虚拟机列表会出现一个新的同名虚拟机。
- 选择新的同名虚拟机 -》 编辑虚拟机 -》 选项 -》 虚拟机名称 -》 重新命名虚拟机。
- 选择虚拟机 -》 播放虚拟机 -》 弹出对话框 -》 我已复制该虚拟机。

### 重新配置虚拟机网络

使用 root 账号登录，密码和源虚拟机一样，编辑网络接口配置文件：

```shell
[root@localhost ~]# vi /etc/sysconfig/network-scripts/ifcfg-ens33
...
IPADDR=192.168.3.132 # 修改 IP 
...
```

由于是克隆，因此需要修改 ens33 的 UUID：

```shell
sed -i "/UUID=/cUUID=$(uuidgen ens33)" /etc/sysconfig/network-scripts/ifcfg-ens33
```

修改主机名称：

```shell
hostnamectl set-hostname node2
```

重启网络服务：

```shell
systemctl restart network
```

虚拟机克隆完毕。

