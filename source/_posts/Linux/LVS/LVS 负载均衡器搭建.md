---
title: LVS 负载均衡器搭建
toc: true
date: 2024-03-08 11:43:29
tags:
- Linux
- LVS
- Load Balancer
categories:
thumbnail:
---
本文将通过虚拟机环境，描述如果一步步搭建一个简单标准的 LVS 负载均衡器，包括 LVS-DR 和 LVS-NAT 两种网络模型。
<!--more-->

# LVS-DR

## 部署图

![一个标准的小型 DR 网络模型，VIP 和 RIP 同在一个 IP 网络](lvs_dr_deploy_simple1.png)

> VMware 在创建虚拟网络是会自动创建虚拟交换机，上图中的三台虚拟机通过虚拟交换机链接在同一个 VMnet。

## 搭建虚拟机环境

参考文章[《VMware 安装 Centos 7》](/blog/2024/03/08/Linux/VMWare%20Player%20安装Centos%207/) 搭建三个虚拟机节点：

- director，real1 和 real2。
- 三个节点的网络适配器都选择桥接模式。

## Director Server


{% codeblock "1. 配置 DIP: 192.168.3.130" lang:shell >folded %}
[root@localhost ~]# vim /etc/sysconfig/network-scripts/ifcfg-ens33
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static # 静态
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=e0abc9ba-44a4-4570-b0bd-b7ee3e519bac 
DEVICE=ens33
IPADDR=192.168.3.130 # DIP：192.168.3.130 
NETMASK=255.255.255.0 # 子掩码
GATEWAY=192.168.3.1 # 网关 
DNS1=192.168.3.1 
ONBOOT=yes # 开机自动加载

# 重启网络服务
[root@director ~]# systemctl restart network
{% endcodeblock %}

{% codeblock "2. 配置 VIP: 192.168.3.10" lang:shell >folded %}
[root@director ~]# ip addr add 192.168.3.10/32 broadcast 192.168.3.10 dev ens33
[root@director ~]# ip route add 192.168.3.10 via 192.168.3.10 dev ens33
{% endcodeblock %}

{% codeblock "3. 安装 ipvsadm" lang:shell >folded %}
[root@director ~]# yum -y install ipvsadm
{% endcodeblock %}

{% codeblock "4. 添加 ipvs 规则" lang:shell >folded %}
[root@director ~]# ipvsadm -A -t 192.168.3.10:80 -s rr
[root@director ~]# ipvsadm -a -t 192.168.3.10:80 -r 192.168.3.131 -g -w 1
[root@director ~]# ipvsadm -a -t 192.168.3.10:80 -r 192.168.3.132 -g -w 1
[root@director ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.3.10:80 rr
  -> 192.168.3.131:80             Route   1      0          0
  -> 192.168.3.132:80             Route   1      0          0
[root@director ~]#
{% endcodeblock %}

## Real Server

> 分别对两个 Real Server 进行同样的配置


{% codeblock "1. 配置 RIP1: 192.168.3.131" lang:shell >folded %}

vi /etc/sysconfig/network-scripts/ifcfg-ens33

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static # 修改为静态
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=8b0071db-3302-5df1-a61b-4542df285eb2
DEVICE=ens33
IPADDR=192.168.3.131 # RIP1
NETMASK=255.255.255.0 # 子网掩码
GATEWAY=192.168.3.1 # 网关 
DNS1=192.168.3.1 
ONBOOT=yes # 修改启动自动加载

# 重启网络服务
[root@director ~]# systemctl restart network
{% endcodeblock %}

{% codeblock "2. 配置 VIP (noarp): 192.168.3.10" lang:shell >folded %}

# 防止其他网络接口通过 arp 暴露 lo 上的 VIP
[root@real1 cdrom]# echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore  
[root@real1 cdrom]# echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce 
# 配置 VIP
[root@real1 cdrom]# ip addr add 192.168.3.10/32 broadcast 192.168.3.10 dev lo label lo:0
[root@real1 cdrom]# ip route add 192.168.3.10 via 192.168.3.10 dev lo:0
[root@real1 cdrom]# ip addr list lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    # VIP
    inet 192.168.3.10/32 brd 192.168.3.10 scope global lo:0
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
# 路由信息
[root@real1 cdrom]# ip route show
default via 192.168.3.1 dev ens33 proto static metric 100
192.168.3.0/24 dev ens33 proto kernel scope link src 192.168.3.131 metric 100
192.168.3.10 via 192.168.3.10 dev lo
{% endcodeblock %}

{% codeblock "3. 安装 httpd：" lang:shell >folded %}
# 安装 httpd
yum -y install httpd
# 创建首页
echo "<h1>I am Real Server 1</h1>" > /var/www/html/index.html
# 启动 httpd
 systemctl start htppd
# 访问首页
[root@real1 cdrom]# curl localhost
<h1>I am real server 1</h1>
{% endcodeblock %}

## 验证负载均衡效果

{% codeblock "1. 分别关闭 Director Server 和 两个 Real Server的 防火墙" lang:shell >folded %}
[root@director ~]# systemctl stop firewalld
[root@director ~]# systemctl disable firewalld
{% endcodeblock %}


{% codeblock "2. 多次请求 VIP:80 效果" lang:shell >folded %}
# 浏览器的 Http 连接是设置了 Keep-Alived 的长连接，因此 rr 的负载均衡的演示效果不够明显，因为多次刷新可能使用的都是同一条连接。
C:\Windows\system32>curl http://192.168.3.10
<h1> I am real server 2</h1>

C:\Windows\system32>curl http://192.168.3.10
<h1>I am real server 1</h1>

C:\Windows\system32>curl http://192.168.3.10
<h1> I am real server 2</h1>

C:\Windows\system32>curl http://192.168.3.10
<h1>I am real server 1</h1>
{% endcodeblock %}

# LVS-NAT

## 部署图

![一个简单 LVS-NAT 网络模型，VIP 和 DIP/RIP 同处不同的网络](lvs_nat_deploy_simple1.png)

>注意：LVS-NAT 模型下 VIP/DIP 是一组浮动 IP，当主 Director 宕机时，它们需要同时转移到备 Direcotr 上，而 LVS-DR 模型下只需要转移 VIP 即可。

## 搭建虚拟机环境

> 注意：我们修改虚拟机配置的时候，最好先关闭虚拟机

- 我们继续使用上面 LVS-DR 的三个节点：director，real1 和 real2。
- 修改网络拓扑：
  - director 网络适配器-》网络连接-》桥接模式（和上面 LVS-DR 小节的配置一样）
  - director 节点额外添加一个网络适配器2：编辑虚拟机设置-》添加-》网络适配器-》完成。
  - director 网络适配器2-》网络连接-》自定义-》VMnet2
  - real1 网络适配器-》网络连接-》自定义-》VMnet2
  - real2 网络适配器-》网络连接-》自定义-》VMnet2

## Director Server
{% codeblock "1. 查看网络接口名称" lang:shell >folded %}
[root@director ~]# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
# 网络适配器
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:cf:dc:08 brd ff:ff:ff:ff:ff:ff
# 网络适配器2
3: ens38: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:cf:dc:12 brd ff:ff:ff:ff:ff:ff
{% endcodeblock %}

{% codeblock "2. 配置 DIP: 192.168.4.1" lang:shell >folded %}
# 在 lvs-nat 模型下 dip 也是浮动 ip 
[root@director ~]# ip addr add 192.168.4.1/24 broadcast 192.168.4.255 dev ens38
{% endcodeblock %}

{% codeblock "3. 配置 VIP: 192.168.3.10" lang:shell >folded %}
[root@director ~]# ip addr add 192.168.3.10/32 broadcast 192.168.3.10 dev ens33
[root@director ~]# ip route add 192.168.3.10/32 via 192.168.3.10 dev ens33
{% endcodeblock %}

{% codeblock "4. 打开 ipv4 转发" lang:shell >folded %}
echo 1 > /proc/sys/net/ipv4/ip_forward
{% endcodeblock %}


{% codeblock "5. 配置 IPVS 规则" lang:shell >folded %}
# 清除 ipvs 规则
[root@director ~]# ipvsadm -C

# 添加 ipvs 规则
[root@director ~]# ipvsadm -A -t 192.168.3.10:80 -s rr
# 添加两个 Real Server nat模式 权重为1
[root@director ~]# ipvsadm -a -t 192.168.3.10:80 -r 192.168.4.101 -m -w 1
[root@director ~]# ipvsadm -a -t 192.168.3.10:80 -r 192.168.4.102 -m -w 1
# 查看 ipvs 规则
[root@director ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.3.10:80 rr
  -> 192.168.4.101:80             Masq    1      0          0
  -> 192.168.4.102:80  
{% endcodeblock %}

## Real Server

> 分别对两个 Real Server 进行同样的配置

{% codeblock "1. 配置 RIP: 192.168.4.101" lang:shell >folded %}
vi /etc/sysconfig/network-scripts/ifcfg-ens33

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static # 修改为静态
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=02497b7f-4f6c-4ac4-a0cd-0c435f1c6898
DEVICE=ens33
IPADDR=192.168.4.101 # RIP1
NETMASK=255.255.255.0 # 子网掩码
GATEWAY=192.168.4.1 # 网关必须指向 Directoer 的 DIP
ONBOOT=yes # 修改启动自动加载

# 重启网络服务
[root@real1 ~]# systemctl restart network
{% endcodeblock %}

> 注意：此时 Real Server 和你的宿主机不在同一个网段，因此可能无法使用 ssh 终端进行登录，一个的方法是使用 director 作为跳板，先使用终端登录 director，再通过 ssh 命令登录 real server。

## 验证负载均衡效果 

{% codeblock "多次请求 VIP:80 效果" lang:shell >folded %}
# 浏览器的 Http 连接是设置了 Keep-Alived 的长连接，因此 rr 的负载均衡的演示效果不够明显，因为多次刷新可能使用的都是同一条连接。
C:\Windows\system32>curl http://192.168.3.10
<h1> I am real server 2</h1>

C:\Windows\system32>curl http://192.168.3.10
<h1>I am real server 1</h1>

C:\Windows\system32>curl http://192.168.3.10
<h1> I am real server 2</h1>

C:\Windows\system32>curl http://192.168.3.10
<h1>I am real server 1</h1>
{% endcodeblock %}

## 如何离线安装 Httpd

在 LVS-NAT 模式下直接使用 yum 源安装 Httpd 可能遇到失败的情况，因为此时 Real Server 不能访问外网。

一个方式是使用 ISO 镜像作为本地的 yum 源：

{% codeblock "1. 先备份 Base 源" lang:shell >folded %}
[root@real2 ~]# mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
{% endcodeblock %}

{% codeblock "2. 修改 etc/yum.repos.d/CentOS-Media.repo 文件，如果没有则手动创建" lang:shell >folded %}
[c7-media]
name=CentOS-$releasever - Media
baseurl=file:///media/CentOS/
       file:///media/cdrom/
       file:///media/cdrecorder/
gpgcheck=1
enabled=1 # 该参数默认值为 0，修改为 1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
{% endcodeblock %}


{% codeblock "3. 挂载镜像" lang:shell >folded %}
[root@real2 ~]# mkdir /media/cdrom
[root@real2 ~]# mount -r /dev/cdrom /media/cdrom
[root@real2 ~]# cd /cdrom
{% endcodeblock %}


{% codeblock "4. 重新加载 yum 缓存" lang:shell >folded %}
[root@real2 ~]# yum clean all
[root@real2 ~]# yum makecache
{% endcodeblock %}


{% codeblock "5. 安装 httpd" lang:shell >folded %}
[root@real2 cdrom]# yum -y install httpd
{% endcodeblock %}


# 参考资料

[LVS 官方文档](http://www.linuxvirtualserver.org/Documents.html)
[LVS-HOWTO](http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/index.html)