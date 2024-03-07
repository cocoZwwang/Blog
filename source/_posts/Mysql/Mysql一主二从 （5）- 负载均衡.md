---
title: Mysql一主二从 （5）- 负载均衡
date: 2024-03-07 15:08:59
tags:
categories: Mysql
thumbnail:
---

- LVS 简介
- Keepalived  + LVS/DR 的主备负载均衡集群

<!--more-->

## LVS 简介

如果想了解 LVS/DR 的工作原理可以参考文章：《01 LVS-简介》

如果想了解如何基于 Keepalived 部署一个最小化的标准 LVS/DR 负载均衡集群，可以参考文章：《02 LVS 负载均衡器搭建》，《03 Keepalived + LVS-DR 高可用负载均衡器搭建》

## 部署 Keepalived + LVS/DR 的主备负载均衡集群

### 部署图

![图 1](1_master_2_slave_deploy.png)

这里无法直接套用文章《》中的配置，因为上述图中只有三台机器，因此一个节点上会同时存在 Director Server 和 Real Server，这种情况下需要解决一个额外的问题：

- LVS 备用节点存在 IPVS 规则时会产生流量转发的死循环问题。

### 流量转移死循环问题

#### 为什么会产生流量转发的死循环？

我们可以假设这么一种情景：存在两台互为主备的 LVS-DR 节点，并且它们同时也是 Real Server。

![图 2](two_box_lvs_while_true.png)

1. Client 向 VIP:Port 发送请求，由于只有 LVS Master 节点上的 VIP 是对外暴露的，因此该请求会被转发到 LVS Master 节点上。
2. LVS Master 节点接收数据后，会首先由 IPVS 进行处理，IPVS 根据规则选择一个 Real Server 进行转发，假如选择了 Real Server2。
3. LVS Backup节点接收数据后， 由于其上面也存在 IPVS 规则，因此 IPVS 也会根据规则选择一个 Real Server 进行转发，假如选择了 Real Server1。
4. LVS Master 节点再次接收到同一个连接的数据后，IPVS 会根据缓存表直接将数据再次转发给 Real Server2。
5. LVS Backup 节点再次接收到同一个连接的数据后，IPVS 同样也会根据缓存表直接将数据再次转发给 Real Server1。
6. 最终形成了死循环，当请求流量大时，不断形成的死循环会形成流量风暴，导致网络接口不能正常工作。

在当前场景中，我是通过 Keepalived 来实现 LVS 的高可用，而 Keepalived 不管主/备都会生成 IPVS 规则，这有利于快速进行故障转移，但是也会造成上述的死循环问题。

#### 解决死循环问题

容易想到的两个思路是：

- LVS Backup 节点上不加载 IPVS 规则，只有转为主节点时才加载 IPVS 规则。
- 想办法识别 LVS Master 转发过的流量，并让 LVS 不处理这种流量。

##### 动态加载 IPVS 规则

由于 Keepalived 主备状态都默认生成 IPVS 规则，因此需要借助通知脚本来实现该功能，Keepalived 的配置文件中有三个参数：

-  notify_master：当前节点切换为主节点时，执行指定的脚本
-  notify_backup ：当前节点切换为备节点时，执行指定的脚本
-  notify_fault：当前节点切换为故障状态时，执行指定的脚本

因此：

- 指定 notify_master 执行脚本，并通过其加载 IPVS 规则。
- 指定 notify_backup 和 notify_fault，并通过其清空当前 IPVS 规则。

这种方式思路非常简单，但也会有一些问题：

- 动态加载 IPVS 规则增加了主备切换的时间成本。
- 虽然 Keepalived 内置了对 LVS 的支持，但是 LVS 本身和 Keepalived 并不是一个组件，因此如果 Keepalived Master 被意外杀死而没有调用通知脚本，此时当 Keepalived Backup 转为 Master 状态时，就会再次出现死循环问题。

##### 使用 iptables + lvs fwmark

lvs fwmark 功能可以对被 iptables 标记的流量进行识别，因此假如我们有两个 LVS 节点  node1 和 node2：

- 在 node1 上配置 iptable 规则

  ```shell
  iptables -t mangle -I PREROUTING -d $VIP -p tcp --dport $VPORT -m mac ! --mac-source $MAC_NODE2 -j MARK --set-mark 0x1
  ```

  如果请求 VIP:Port 的流量的源 MAC 地址不等于 node2 MAC 地址，则表明该流量为客户端请求流量，并将其 mark 值设置为 1。

- 在 node2 配置 iptable 规则

  ```shell
  iptables -t mangle -I PREROUTING -d $VIP -p tcp --dport $VPORT -m mac ! --mac-source $MAC_NODE1 -j MARK --set-mark 0x1
  ```

  如果请求 VIP:Port 的流量的源 MAC 地址不等于 node1 MAC 地址，则表明该流量为客户端请求流量，并将其 mark 值设置为 1。

- keepalived 上配置 lvs fwmark：  

  ```shell
  virtual_server fwmark 1 {
       delay_loop 10
       lb_algo rr
        lb_kind DR
       ....
  }
  ```

这种方式需要利用 iptables 规则，但是可以允许主/备 LVS 同时存在 IPVS 规则，这里我也是选择了这种解决方案。

### Real Server 配置

> 三台节点都需要同样的配置

设置隐藏 VIP: 192.168.3.100

```shell
# 设置 vip 到 lo 网口
ip addr add 192.168.3.100/32 broadcast 192.168.3.100 dev lo
ip route add 192.168.3.100/32 via 192.168.3.100 dev lo
# 防止 lo 上地址被其他网口通过 arp 广播暴露出去
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
```

### LVS 主节点配置（node2）

安装 keepalived 

```shell
yum install -y keepalived
```

配置 `/etc/keepalived/keepalived.conf`：

```shell
global_defs {
   notification_email {
    	xxxxx@163.com
    	yyyyy@163.com
   }
   notification_email_from lvs_node1@domain.com
   smtp_server 192.168.3.20
   smtp_connect_timeout 30
    
   router_id node2 # 修改为当前节点名称
   script_user root
   enable_script_security
}

#VIP 配置
vrrp_instance VI_1 {
    state BACKUP # 采用 Backup -> Backup 的非抢占模式
    nopreempt # 非抢占模式
    interface ens33 # vip 所在网络接口
    virtual_router_id 3
    priority 150 # 优先级
    advert_int 1
    authentication {
        auth_type AH
        auth_pass k@l!ve1
    }
    # 虚拟ip 地址
    virtual_ipaddress {
        192.168.3.10
    }
	
	# 通知脚本
    notify_master "/etc/keepalived/notify.sh master"
    notify_backup "/etc/keepalived/notify.sh backup"
    notify_fault "/etc/keepalived/notify.sh fault"
}


# 健康监测 和 LVS 负载均衡
virtual_server fwmark 1 {
    delay_loop 3 # 监控检测间隔
    lb_kind DR # 负载均衡的转发模式

    # 下面配置为 rr 和 0 是因为测试效果明显，能明显看到每次请求都分发到不同服务器
    lb_algo rr # 负载均衡算法
    persistence_timeout 0 #会话保持时间

    protocol TCP

    #真实服务器
    real_server 192.168.3.131 3306 {
    	weight 1
    	# 这里只做简单的健康检测
        TCP_CHECK {
            connect_timeout 3
        }
    }

    real_server 192.168.3.132 3306 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
        }
    }

    real_server 192.168.3.133 3306 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
        }
    }
}
```

设置 iptalbes 规则：

```shell
# ping 一下 RIP2
ping 192.168.3.133

# 获取 node3 的 mac 地址
[root@node2 ~]# arp -a 192.168.3.133
node3 (192.168.3.133) at 00:0c:29:e9:27:55 [ether] on ens33

# 设置 iptables 规则：vip：192.168.3.10 
[root@node2 ~]# iptables -t mangle -I PREROUTING -d 192.168.3.10 -p tcp --dport 3306 -m mac ! --mac-source 00:0c:29:e9:27:55 -j MARK --set-mark 0x1
[root@node2 ~]# iptables -t mangle -L -n
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
MARK       tcp  --  0.0.0.0/0            192.168.3.10         tcp dpt:3306 MAC ! 00:0C:29:E9:27:55 MARK set 0x1
```

### LVS 备节点配置（node3）

安装 keepalived 

```shell
yum install -y keepalived
```

配置 `/etc/keepalived/keepalived.conf`：

```shell
global_defs {
   notification_email {
    	xxxxx@163.com
    	yyyyy@163.com
   }
   notification_email_from lvs_node1@domain.com
   smtp_server 192.168.3.20
   smtp_connect_timeout 30
    
   router_id node3 # 修改为当前节点名称
   script_user root
   enable_script_security
}

#VIP 配置
vrrp_instance VI_1 {
    state BACKUP
    nopreempt 
    interface ens33 
    virtual_router_id 3
    priority 120 ###### node3 优先级 比 node2 低
    advert_int 1
    authentication {
        auth_type AH
        auth_pass k@l!ve1
    }
    # 虚拟ip 地址
    virtual_ipaddress {
        192.168.3.10
    }
	
	# 通知脚本
    notify_master "/etc/keepalived/notify.sh master"
    notify_backup "/etc/keepalived/notify.sh backup"
    notify_fault "/etc/keepalived/notify.sh fault"
}


# 健康监测 和 LVS 负载均衡
virtual_server fwmark 1 {
    delay_loop 3 # 监控检测间隔
    lb_kind DR # 负载均衡的转发模式

    # 下面配置为 rr 和 0 是因为测试效果明显，能明显看到每次请求都分发到不同服务器
    lb_algo rr # 负载均衡算法
    persistence_timeout 0 #会话保持时间

    protocol TCP

    #真实服务器
    real_server 192.168.3.131 3306 {
    	weight 1
    	# 这里只做简单的健康检测
        TCP_CHECK {
            connect_timeout 3
        }
    }

    real_server 192.168.3.132 3306 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
        }
    }

    real_server 192.168.3.133 3306 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
        }
    }
}
```

设置 iptalbes 规则：

```shell
# ping 一下 RIP2
ping 192.168.3.132

# 获取 node2 的 mac 地址
[root@node3 ~]# arp -a 192.168.3.132
node2 (192.168.3.132) at 00:0c:29:3b:63:ce [ether] on ens33

# 设置 iptables 规则：vip：192.168.3.10 
[root@node3 ~]# iptables -t mangle -I PREROUTING -d 192.168.3.10 -p tcp --dport 3306 -m mac ! --mac-source 00:0c:29:3b:63:ce -j MARK --set-mark 0x1
[root@node3 ~]# iptables -t mangle -L -n
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
MARK       tcp  --  0.0.0.0/0            192.168.3.10         tcp dpt:3306 MAC ! 00:0C:29:3B:63:CE MARK set 0x1
```

### 验证

启动 keepalived

```shell
[root@node2 ~]# systemctl start keepalived; ssh node3 'systemctl start keepalived'
```

通过远程主机访问 VIP:3306

```cmd
D:\Works\mysql-8.0.15-winx64\bin>mysql -umhauser -pmhapass -h 192.168.3.10 -e "show variables like 'server_id';"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+

D:\Works\mysql-8.0.15-winx64\bin>mysql -umhauser -pmhapass -h 192.168.3.10 -e "show variables like 'server_id';"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 2     |
+---------------+-------+

D:\Works\mysql-8.0.15-winx64\bin>mysql -umhauser -pmhapass -h 192.168.3.10 -e "show variables like 'server_id';"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
```

关闭 node2 上的 keepalived

```shell
[root@node2 ~]# systemctl stop keepalived
[root@node2 ~]# ip addr list ens33
# VIP 已经发送转移
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:3b:63:ce brd ff:ff:ff:ff:ff:ff
    inet 192.168.3.132/24 brd 192.168.3.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::74a3:7d4f:d23c:4b03/64 scope link noprefixroute
       valid_lft forever preferred_lft forever

```

查看 node3 上的 vip：

```shell
[root@node3 ~]# ip addr list ens33
# VIP 已经别 node3 持有
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:e9:27:55 brd ff:ff:ff:ff:ff:ff
    inet 192.168.3.133/24 brd 192.168.3.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.3.10/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::811d:2f59:9e85:f5c4/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

通过远程主机访问 VIP:3306

```cmd
D:\Works\mysql-8.0.15-winx64\bin>mysql -umhauser -pmhapass -h 192.168.3.10 -e "show variables like 'server_id';"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3     |
+---------------+-------+

D:\Works\mysql-8.0.15-winx64\bin>mysql -umhauser -pmhapass -h 192.168.3.10 -e "show variables like 'server_id';"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 2     |
+---------------+-------+

D:\Works\mysql-8.0.15-winx64\bin>mysql -umhauser -pmhapass -h 192.168.3.10 -e "show variables like 'server_id';"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
```


