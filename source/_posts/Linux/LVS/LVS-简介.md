---
title: LVS-简介
toc: true
date: 2024-03-08 11:43:19
tags: 
- Linux
- LVS
- Load Balancer
categories:
thumbnail:
---

LVS（Linux Virtual Server）是建立在真实服务器集群基础上的高可扩展性和高可用性虚拟服务器，整个真实服务集群系统对终端用户而言就好像只是一台高性能的虚拟服务器。

LVS 可用于构建 Linux 高性能的负载均衡服务器集群，使集群的并行服务以虚拟服务的形式出现在单个 IP 地址上，请求分派使用 IP 负载平衡技术。系统的可扩展性是通过在集群中透明地添加或删除节点来实现的。通过检测节点或守护进程故障并适当地重新配置系统，可提供高可用性。

<!--more-->

# 关键词

- Director Server：负载均衡节点
- RealServer：处理请求的真实服务节点
- VIP：Virtule IP，主要用于外网通信
- DIP：Director IP，主要用于内网通信
- RIP：Real IP，RealServer 的 IP 地址
- CIP：Client IP，外网客户端 IP


# LVS 工作模式

- LVS-NAT
- LVS-DR(direct routint)
- LVS-TUN(ip tunnelling)
- LVS-FULLNET

## LVS-NAT

通过 NAT 虚拟服务器的优点是，真实服务器可以运行任何支持 TCP/IP 协议的操作系统，真实服务器可以使用专用互联网地址，负载平衡器只需要一个 IP 地址。

缺点是通过 NAT 虚拟服务器的可扩展性有限。当服务器节点（一般 PC 服务器）增加到 20 个左右或更多时，负载平衡器可能会成为整个系统的瓶颈，因为请求数据包和响应数据包都需要由负载平衡器重写。

**LVS-NAT 一些需要注意的点**

- RIP 应该和 DIP 应该使用私网地址。
- RIP 和 DIP必须在同一IP网络，且 Real Server 的网关需要指向 DIP。
- 请求和响应报文都要经由 director 转发，极高负载的场景中，director 可能会成为系统瓶颈。
- 支持端口映射，VIP 和 RIP 不需要监听在同一端口。
- Real Server 可以使用任意OS。

![lvs-nat](lvs_nat.png)

1. 客户端请求（src：CIP，dest：VIP:80）
2. LVS 调度和重写数据包并将数据包通过 DIP 发送到 Real Server（src：CIP，dest：RIP1:80）
3. Real Server1 接受并处理请求
4. Real Server1 回复 DIP（src：RIP1:80，dest：CIP）
5. DIP 接受数据后将数据包的源地址和端口修改为 VIP:80 并转发到 VIP （src：VIP:80，des：CIP）
6. VIP 将数据返回到 CIP（src：VIP:80，des：CIP）




## LVS-DR

DR 模式相对于 NAT 来说有着更好的负载能力，因为它只需要处理连接的请求数据，而回复则由真实服务器直接返回给客户端。

**DR 模式对网络拓扑有以下要求：**

- DR 模式是通过修改 MAC 地址来进行直接转发，因此 Director Server 和 Real Server 必须有一个接口通过 `HUB/交换机`进行物理连接。
- Diretor Server 和 Real Server 都同时拥有 VIP，前者使用 VIP 接受请求，后者则需要使用 VIP 进行回复。
- Diretor Server 的 VIP 必须暴露给路由器，而 Real Server 的 RIP 则必须被隐藏，否则将会出现 IP 冲突。
- Director Server 和 Real Server 至少存在一个物理网口连接在同一个集线器/交互机上。

下面是一个简单`LVS-DR`负载均衡的网络模型：
![lvs-dr: vip/rip 在同一个网段](lvs_dr.png)

1. 客户端向 VIP:Port 发送请求报文。
2. 由于只有 Direcotr 的 VIP 暴露给了路由，因此路由将请求报文的目标 MAC 指向了 Director。
3. 交互机将请求报文转发给 Director。
4. Directory 接受请求报文，IPVS 会在 netfilter 中进行拦截，发现和 IPVS 规则匹配，因此根据调度算法选择一台 Real Server 进行转发：将请求报文的目标 MAC 修改为该 Real Server 的MAC，并重新转发给交换机。
5. 交换机将请求报文转发给 Real Server。
6. Real Server 接受并解析请求报文，发现目标地址为 VIP，因此转发给 lo:0 进行处理，并使用 VIP 作为源地址对客户端直接进行回复。
7. 交换机将回复报文转发给路由器
8. 路由器将回复报文转发给客户端。

> 上面网络拓扑 Real Server 可以直接回复 Client 是因为 VIP 和 RIP 同处一个网段，并且 RIP 的网关指向了图中的路由器。 

# LVS 组件

## ipvs

ipvs 工作在 Linux 内核中的 netfilter INPUT 钩子上，支持 TCP，UPD，AH，EST，AH_EST，SCTP 等诸多协议。

- 一个 ipvs 主机可以同时定义多个 cluster service
- 一个 cluster service 上至少有一个 real service
- 定义时可以指明 lvs-type，缺省为 dr
- 定义时可以指明lvs scheduler，缺省为 wlc
  - rr：轮询
  - wrr：加权轮询
  - lc：最少连接
  - wlc：加权最少连接
  - lblc：基于局部性的最少链接调度
  - lblcr：带复制的基于局部性最少链接调度
  - dh：目标地址散列调度
  - sh：源地址散列调度

## ipvsadm

ipvsadm 是一个用户空间的命令行工具，用于管理集群服务。

### 管理 Director Server

添加一个 Director Server

```shell
# ipvsadm 添加 tcp 192.168.3.10:80 调度算法 轮询
ipvsadm   -A   -t  192.168.3.10:80 -s       rr

# ipvsadm 添加 udp 192.168.3.2:5000 调度算法 加权最少连接数
ipvsadm   -A   -u  192.168.3.2:5000 -s       wlc
```

修改一个 Director Server

```shell
# ipvsadm 修改 tcp 192.168.3.10:80 调度算法 加权轮询
ipvsadm   -E   -t  192.168.3.10:80 -s       wrr
```

删除一个 Director Server

```shell
ipvsadm -D -t 192.168.3.100:80
```

### 管理 Real Server： 

添加一个 Real Server 

```shell
# ipvsadm  添加  tcp  director server    real server           dr  权重   1
ipvsadm    -a    -t   192.168.3.10:80   -r 192.168.3.101:8080  -g  -w     1
ipvsadm    -a    -t   192.168.3.10:80   -r 192.168.3.102:80    -g  -w     1

# ipvsadm  添加  udp  director server    real server            nat  权重   2
ipvsadm    -a    -u   192.168.3.2:5000   -r 192.168.3.103:5000  -m   -w     2
ipvsadm    -a    -u   192.168.3.2:5000   -r 192.168.3.104:5001  -m   -w     1
```

### 查看和清空

查看 ipvs 规则：

```shell
# -L：列出所有 ipvs 规则
# -n：地址和端口通过数字显示
[root@node4 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.3.10:80 wrr
  -> 192.168.3.101:80             Route   1      0          0
  -> 192.168.3.102:80             Route   1      0          0
UDP  192.168.3.2:5000 wlc
  -> 192.168.3.103:5000           Masq    2      0          0
  -> 192.168.3.104:5001           Masq    1      0          0
```

清空 ipvs 规则

```shell
ipvsadm -C
```

### 保存和恢复

保存 ipvs 规则到指定文件：

```shell
[root@node4 ~]# ipvsadm-save -n > /etc/sysconfig/ipvsadm-config
[root@node4 ~]# cat /etc/sysconfig/ipvsadm-config
-A -t 192.168.3.10:80 -s wrr
-a -t 192.168.3.10:80 -r 192.168.3.101:80 -g -w 1
-a -t 192.168.3.10:80 -r 192.168.3.102:80 -g -w 1
-A -u 192.168.3.2:5000 -s wlc
-a -u 192.168.3.2:5000 -r 192.168.3.103:5000 -m -w 2
-a -u 192.168.3.2:5000 -r 192.168.3.104:5001 -m -w 1
```

恢复 ipvs 规则：

```shell
# 清空 ipvs 规则
[root@node4 ~]# ipvsadm -C
[root@node4 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
[root@node4 ~]#
# 恢复 ipvs 规则
[root@node4 ~]# ipvsadm-restore < /etc/sysconfig/ipvsadm-config
[root@node4 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.3.10:80 wrr
  -> 192.168.3.101:80             Route   1      0          0
  -> 192.168.3.102:80             Route   1      0          0
UDP  192.168.3.2:5000 wlc
  -> 192.168.3.103:5000           Masq    2      0          0
  -> 192.168.3.104:5001           Masq    1      0          0
```


# 参考资料

[LVS 官方文档](http://www.linuxvirtualserver.org/Documents.html)
[LVS-HOWTO](http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/index.html)