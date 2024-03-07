---
title: Mysql一主二从 （4）- MHA 安装
date: 2024-03-07 15:08:53
tags:
categories: Mysql
thumbnail:
toc: true
---

- MHA 简介
- 安装 和 配置 MHA
- 验证故障转移

<!--more-->

## MHA 简介

MHA 目前在 MYSQL 高可用方面是一个相对成熟的解决方案，是一套在 MYSQL 高可用性环境下进行故障切换和主从提升的优秀软件。在 MYSQL 故障切换的过程中，最大程度地保证数据的完整性和一致性，以达到真正意义上的高可用。

### 工作原理

![图 1](mha_arch.png)

#### 传统复制：

1. 从宕机崩溃的 Master 保存二进制日记事件（BINLOG event）。
2. 识别含有最新更新的 slave。
3. 应用差异的中继日志（Relay Log）到其他 Slave。
4. 应用从 Master 保存的二进制日志事件。
5. 提升一个 Slave 为新的 Master。
6. 使其他 Slave 连接到新的 Master 进行复制。

#### GTID 复制模式：

1. 识别含有最新更新的 Slave。
2. 复制差异的 BINLOG 数据到其他 Slave。
3. 提升一个 Slave 为新的 Master。
4. 使其他 Slave 连接到新的 Master 进行复制。

### 组件

{% codeblock "MHA Manager" lang: >folded %}
masterha_check_ssh：MHA SSH 免密通信环境检测工具
masterha_check_repl：Mysql 复制环境检测工具
masterha_manager：MHA 服务主程序
masterha_check_status：MHA 运行状态检测工具
masterha_master_monitor：Mysql master 节点性能检测工具
masterha_master_swich：master 节点切换工具
masterha_conf_host：添加或者删除配置的节点
masterha_stop：关闭 MHA 服务的工具
{% endcodeblock %}


{% codeblock "MHA Node" lang: >folded %}
save_binary_logs：保存和复制 master 的二进制日志
apply_diff_replay_logs：识别差异的中继日志事件并应用到其他 slave 节点
filter_mysqlbinlog：去除不必要的 Rollback 事件（MHA 已经不使用这个工具）
purge_replay_logs：清楚中继日志（不会阻塞 sql 线程）
{% endcodeblock %}


{% codeblock "自定义扩展" lang: >folded %}
secondary_check_script：通过多条网络路由检测 master 的可用性。
master_ip_failover_script：检测 application 使用的 masterip。
shutdown_script：强制关闭 master 节点。
init_conf_load_script：加载初始配置参数。
master_ip_online_change_script：更新 master 节点 ip 地址。
{% endcodeblock %}

### 使用要点

- MHA Manager 可以单独部署在一台服务器，可以同时管理多个  MYSQL  集群。
- MHA Manager 也可以部署在其中一台**从节点**上，但是不能在**主节点**上，否则一尸两命。
- MHA Manager 在进行故障转移后，会停止自身进程，需要运维人员再次启动，如果 MHA Manager 节点被提升为主节点，需要在其他从节点启动 MHA Manager。

### MHA Manager 高可用

MHA 目前不支持多个 Manager 同时监控一个 MYSQL 主从集群，因此，如果 MHA Manager 宕机，就无法启动主节点的故障转移。但是，MHA Manager 本身不参与业务，因此其单点故障的潜在威胁比较低，唯一需要避免的就是主节点和 MHA Manager 同时宕机。

可以考虑下面两种方法来提高 MHA 的可用性：

- 对 MHA Manager 进行监控，发生单点故障时能及时通知运维人员进行处理。

- 可以选择使用 Pacemaker 等常见的主/备作为其高可用方案。

## 安装 MHA 

### 安装 MHA Node

> 注意：每个节点都需要安装 MHA Node，包括 MHA Manager


{% codeblock "1. 安装 epel 源" lang:shell >folded %}
[root@node4 ~]# yum install -y epel-release
[root@node4 ~]# yum clean all
[root@node4 ~]# yum makecache
{% endcodeblock %}


{% codeblock "2. 下载并安装 MHA Node" lang:shell >folded %}
[root@localhost tmp]# wget -P /tmp https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
[root@node4 ~]# yum install -y /tmp/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
{% endcodeblock %}

{% codeblock "3. 安装完成后会在 /usr/bin 目录生成如下脚本" lang:shell >folded %}
/usr/bin/save_binary_logs #保存和复制 master 的二进制日志
/usr/bin/apply_diff_replay_logs #识别差异的中继日志事件并应用到其他 slave 节点
/usr/bin/filter_mysqlbinlog #去除不必要的 Rollback 事件（MHA 已经不使用这个工具）
/usr/bin/purge_replay_logs #清楚中继日志（不会阻塞 sql 线程）
{% endcodeblock %}


### 安装 MHA Manager

####  配置主库


{% codeblock "1. 在主节点（node1）上创建 mha 的监测账号" lang:sql >folded %}
# MHA 默认使用 mysql_native_password 认证方式
root@localhost (none) :37: >create user 'mhauser'@'192.168.3.%' identified with mysql_native_password by 'mhapass';
Query OK, 0 rows affected (0.03 sec)

root@localhost (none) :40: >flush privileges;
Query OK, 0 rows affected (0.01 sec)

root@localhost (none) :40: >grant all privileges on *.* to 'mhauser'@'192.168.3.%';
Query OK, 0 rows affected (0.01 sec)
{% endcodeblock %}



{% codeblock "2. 检测账号是否同步完毕" lang:sql >folded %}
# 由于前面配置了主从复制，因此我们不需要在从节点上重复创建，也一定不要创建（会冲突）
[root@localhost tmp]# ssh node2
[root@node2 ~]# mysql -uroot -p -e "select user,host from mysql.user where user='mhauser';"
Enter password:
+---------+-------------+
| user    | host        |
+---------+-------------+
| mhauser | 192.168.3.% |
+---------+-------------+

[root@localhost tmp]# ssh node3
[root@node3 ~]# mysql -uroot -p -e "select user,host from mysql.user where user='mhauser';"
Enter password:
+---------+-------------+
| user    | host        |
+---------+-------------+
| mhauser | 192.168.3.% |
+---------+-------------+
{% endcodeblock %}



{% codeblock "3. 配置主节点的浮动 IP: 192.168.3.200/24" lang:shell >folded %}
[root@localhost tmp]# ip addr add 192.168.3.200/24 dev ens33
[root@localhost tmp]# ip addr list ens33
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ca:2b:a0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.3.131/24 brd 192.168.3.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    # 在网络接口 ens33 上新配置的 ip 地址
    inet 192.168.3.200/24 scope global secondary ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::25c7:c0dc:4aeb:8e4/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
   {% endcodeblock %}

#### 配置 MHA Manager 节点

> 我们选择从节点 node3 部署 MHA Manager


{% codeblock "1. 下载 rpm 包并安装" lang:shell >folded %}
# 下载 MHA Manager rpm 包
[root@localhost tmp]# wget -P /tmp https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm

# 安装 MHA Manager
yum install -y /tmp/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
{% endcodeblock %}


{% codeblock "1. 创建 MHA Manager 的配置文件夹" lang:shell >folded %}
[root@node3 ~]# mkdir -pv /etc/masterha
# app1 为 MYSQL 集群的名称（任意）
[root@node3 ~]# mkdir -pv /data/masterha/app1
# 由于 MHA Manager 也有可能在其他从节点运行，因此同样在其他节点也顺便创建配置文件夹
[root@node3 ~]# ssh node1 'mkdir -pv /data/masterha/app1'
[root@node3 ~]# ssh node2 'mkdir -pv /data/masterha/app1'
{% endcodeblock %}


{% codeblock "3. 创建自动故障转移时的回调脚本`/usr/local/bin/master_ip_failover`" lang:perl >folded %}
 #!/usr/bin/env perl
 #  Copyright (C) 2011 DeNA Co.,Ltd.
 #
 #  This program is free software; you can redistribute it and/or modify
 #  it under the terms of the GNU General Public License as published by
 #  the Free Software Foundation; either version 2 of the License, or
 #  (at your option) any later version.
 #
 #  This program is distributed in the hope that it will be useful,
 #  but WITHOUT ANY WARRANTY; without even the implied warranty of
 #  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 #  GNU General Public License for more details.
 #
 #  You should have received a copy of the GNU General Public License
 #   along with this program; if not, write to the Free Software
 #  Foundation, Inc.,
 #  51 Franklin Street, Fifth Floor, Boston, MA  02110-1301  USA
 
 ## Note: This is a sample script and is not complete. Modify the script based on your environment.

 # 脚本官方样例：https://github.com/yoshinorim/mha4mysql-manager/blob/master/samples/scripts/master_ip_failover，下面脚本是在官方脚本的基础上做的修改，修改的地方有中文注释说明
 
 use strict;
 use warnings FATAL => 'all';
 
 use Getopt::Long;
 use MHA::DBHelper;
 
 my (
   $command,        $ssh_user,         $orig_master_host,
   $orig_master_ip, $orig_master_port, $new_master_host,
   $new_master_ip,  $new_master_port,  $new_master_user,
   $new_master_password
 );
 
 ##### 浮动 ip 配置
 my $vip = '192.168.3.200/24'; # 修改成你需要的 virtual ip 
 my $net_dev = "ens33"; # 修改成你的网络设备
 my $ssh_start_vip = "ip addr add $vip dev $net_dev"; 
 my $ssh_stop_vip = "ip addr del $vip dev $net_dev";

 GetOptions(
   'command=s'             => \$command,
   'ssh_user=s'            => \$ssh_user,
   'orig_master_host=s'    => \$orig_master_host,
   'orig_master_ip=s'      => \$orig_master_ip,
   'orig_master_port=i'    => \$orig_master_port,
   'new_master_host=s'     => \$new_master_host,
   'new_master_ip=s'       => \$new_master_ip,
   'new_master_port=i'     => \$new_master_port,
   'new_master_user=s'     => \$new_master_user,
   'new_master_password=s' => \$new_master_password,
 );
 
 exit &main();
 
 sub main {
   if ( $command eq "stop" || $command eq "stopssh" ) {
 
     # $orig_master_host, $orig_master_ip, $orig_master_port are passed.
     # If you manage master ip address at global catalog database,
     # invalidate orig_master_ip here.
     my $exit_code = 1;
     eval {
       # updating global catalog, etc
       # 故障转移时，旧 Master 节点的回调处理
       print "disabling the vip on old master: $orig_master_host \n";
       &stop_vip();
 
       $exit_code = 0;
     };
     if ($@) {
       warn "Got Error: $@\n";
       exit $exit_code;
     }
     exit $exit_code;
   }
   elsif ( $command eq "start" ) {
 
     # all arguments are passed.
     # If you manage master ip address at global catalog database,
     # activate new_master_ip here.
     # You can also grant write access (create user, set read_only=0, etc) here.
     my $exit_code = 10;
     eval {
       ## Update master ip on the catalog database, etc
       ## 故障转移时，新 Master 节点的回调处理
       print "Enabling the VIP - $vip on the new master - $new_master_host\n";
       &start_vip();
 
       $exit_code = 0;
     };
     if ($@) {
       warn $@;
 
       # If you want to continue failover, exit 10.
       exit $exit_code;
     }
     exit $exit_code;
   }
   elsif ( $command eq "status" ) {
 
     # do nothing
     exit 0;
   }
   else {
     &usage();
     exit 1;
   }
 }
 
 # 启动浮动 IP
 sub start_vip(){
     `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
 }
 
 # 关闭浮动 IP
 sub stop_vip(){
     `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
 }
 
 sub usage {
   print
 "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
 }
 {% endcodeblock %}
    



{% codeblock "4. 创建 MYSQL 集群的配置文件：`/etc/masterha/app1.cnf`" lang:ini >folded %}
# 服务默认配置
[server default]

# ssh
ssh_user=root
ssh_port=22

# mha 工作目录
manager_workdir=/data/masterha/app1
manager_log=/data/masterha/app1/manager.log
remote_workdir=/data/masterha/app1

## Mysql
# mha manager 用例管理其他 mysql 服务的超级权限账号
user=mhauser
password=mhapass
# 复制账号
repl_user=repluser
repl_password=replpass
# mha 监控心跳周期
ping_interval=1
ping_type=connect
# 自动 failover 时候的切换脚本
master_ip_failover_script=/usr/local/bin/master_ip_failover
# 手动切换时候的切换脚本
master_ip_online_change_script=/usr/local/bin/master_ip_online_change
# 故障转移后发送告警通知脚本
report_script=/usr/local/bin/send_report
# 一旦 mha 到 node1 的监控之间网络出现了问题，mha manager 将会尝试从 node2 登录到 node1
secondary_check_script=/usr/bin/masterha_secondary_check -s node2 --user=root --master_host=node1 --master_ip=192.168.3.131 --master_port=3306
# 故障发送后关闭故障主机脚本，防止发生脑裂
shutdown_script=""

[server1]
hostname=node1
candidate_master=1

[server2]
hostname=node2
candidate_master=1

[server3]
hostname=node3
candidate_master=1
{% endcodeblock %}

### 启动 MHA Manager



{% codeblock "1. 检测集群 SSH 互通状态" lang:shell >folded %}
[root@node3 ~]# masterha_check_ssh --conf=/etc/masterha/app1.cnf
Thu Feb 29 18:33:15 2024 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Thu Feb 29 18:33:15 2024 - [info] Reading application default configuration from /etc/masterha/app1.cnf..
Thu Feb 29 18:33:15 2024 - [info] Reading server configuration from /etc/masterha/app1.cnf..
Thu Feb 29 18:33:15 2024 - [info] Starting SSH connection tests..
Thu Feb 29 18:33:16 2024 - [debug]
Thu Feb 29 18:33:15 2024 - [debug]  Connecting via SSH from root@node1(192.168.3.131:22) to root@node2(192.168.3.132:22)..
Thu Feb 29 18:33:16 2024 - [debug]   ok.
Thu Feb 29 18:33:16 2024 - [debug]  Connecting via SSH from root@node1(192.168.3.131:22) to root@node3(192.168.3.133:22)..
Thu Feb 29 18:33:16 2024 - [debug]   ok.
Thu Feb 29 18:33:17 2024 - [debug]
Thu Feb 29 18:33:15 2024 - [debug]  Connecting via SSH from root@node2(192.168.3.132:22) to root@node1(192.168.3.131:22)..
Thu Feb 29 18:33:16 2024 - [debug]   ok.
Thu Feb 29 18:33:16 2024 - [debug]  Connecting via SSH from root@node2(192.168.3.132:22) to root@node3(192.168.3.133:22)..
Thu Feb 29 18:33:17 2024 - [debug]   ok.
Thu Feb 29 18:33:17 2024 - [debug]
Thu Feb 29 18:33:16 2024 - [debug]  Connecting via SSH from root@node3(192.168.3.133:22) to root@node1(192.168.3.131:22)..
Thu Feb 29 18:33:17 2024 - [debug]   ok.
Thu Feb 29 18:33:17 2024 - [debug]  Connecting via SSH from root@node3(192.168.3.133:22) to root@node2(192.168.3.132:22)..
Thu Feb 29 18:33:17 2024 - [debug]   ok.
Thu Feb 29 18:33:17 2024 - [info] All SSH connection tests passed successfully.
{% endcodeblock %}


{% codeblock "2. 检测集群同步状态" lang:shell >folded %}
[root@node3 ~]# masterha_check_repl --conf=/etc/masterha/app1.cnf
Thu Feb 29 19:39:17 2024 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Thu Feb 29 19:39:17 2024 - [info] Reading application default configuration from /etc/masterha/app1.cnf..
Thu Feb 29 19:39:17 2024 - [info] Reading server configuration from /etc/masterha/app1.cnf..
Thu Feb 29 19:39:17 2024 - [info] MHA::MasterMonitor version 0.58.
Thu Feb 29 19:39:18 2024 - [info] GTID failover mode = 1
Thu Feb 29 19:39:18 2024 - [info] Dead Servers:
Thu Feb 29 19:39:18 2024 - [info] Alive Servers:
Thu Feb 29 19:39:18 2024 - [info]   node1(192.168.3.131:3306)
Thu Feb 29 19:39:18 2024 - [info]   node2(192.168.3.132:3306)
Thu Feb 29 19:39:18 2024 - [info]   node3(192.168.3.133:3306)
Thu Feb 29 19:39:18 2024 - [info] Alive Slaves:
Thu Feb 29 19:39:18 2024 - [info]   node2(192.168.3.132:3306)  Version=8.0.27 (oldest major version between slaves) log-bin:enabled
Thu Feb 29 19:39:18 2024 - [info]     GTID ON
Thu Feb 29 19:39:18 2024 - [info]     Replicating from 192.168.3.131(192.168.3.131:3306)
Thu Feb 29 19:39:18 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Thu Feb 29 19:39:18 2024 - [info]   node3(192.168.3.133:3306)  Version=8.0.27 (oldest major version between slaves) log-bin:enabled
Thu Feb 29 19:39:18 2024 - [info]     GTID ON
Thu Feb 29 19:39:18 2024 - [info]     Replicating from 192.168.3.131(192.168.3.131:3306)
Thu Feb 29 19:39:18 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Thu Feb 29 19:39:18 2024 - [info] Current Alive Master: node1(192.168.3.131:3306)
Thu Feb 29 19:39:18 2024 - [info] Checking slave configurations..
Thu Feb 29 19:39:18 2024 - [info] Checking replication filtering settings..
Thu Feb 29 19:39:18 2024 - [info]  binlog_do_db= , binlog_ignore_db=
Thu Feb 29 19:39:18 2024 - [info]  Replication filtering check ok.
Thu Feb 29 19:39:18 2024 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Thu Feb 29 19:39:18 2024 - [info] Checking SSH publickey authentication settings on the current master..
Thu Feb 29 19:39:18 2024 - [info] HealthCheck: SSH to node1 is reachable.
Thu Feb 29 19:39:18 2024 - [info]
node1(192.168.3.131:3306) (current master)
 +--node2(192.168.3.132:3306)
 +--node3(192.168.3.133:3306)

Thu Feb 29 19:39:18 2024 - [info] Checking replication health on node2..
Thu Feb 29 19:39:18 2024 - [info]  ok.
Thu Feb 29 19:39:18 2024 - [info] Checking replication health on node3..
Thu Feb 29 19:39:18 2024 - [info]  ok.
Thu Feb 29 19:39:18 2024 - [info] Checking master_ip_failover_script status:
Thu Feb 29 19:39:18 2024 - [info]   /usr/local/bin/master_ip_failover --command=status --ssh_user=root --orig_master_host=node1 --orig_master_ip=192.168.3.131 --orig_master_port=3306
Thu Feb 29 19:39:19 2024 - [info]  OK.
Thu Feb 29 19:39:19 2024 - [warning] shutdown_script is not defined.
Thu Feb 29 19:39:19 2024 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.
{% endcodeblock %}



{% codeblock "3. 启动" lang:shell >folded %}
[root@node3 ~]# nohup masterha_manager --conf=/etc/masterha/app1.cnf --ignore_last_failove < /dev/null >/data/masterha/app1/manager.log 2>&1 &
[1] 15176
[root@node3 ~]#
{% endcodeblock %}



{% codeblock "4. 查看启动状态" lang:shell >folded %}
[root@node3 ~]# tail -f /data/masterha/app1/manager.log
Fri Mar  1 01:42:36 2024 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Fri Mar  1 01:42:36 2024 - [info] Reading application default configuration from /etc/masterha/app1.cnf..
Fri Mar  1 01:42:36 2024 - [info] Reading server configuration from /etc/masterha/app1.cnf..
Fri Mar  1 01:42:36 2024 - [info] MHA::MasterMonitor version 0.58.
Fri Mar  1 01:42:37 2024 - [info] GTID failover mode = 1
Fri Mar  1 01:42:37 2024 - [info] Dead Servers:
Fri Mar  1 01:42:37 2024 - [info] Alive Servers:
Fri Mar  1 01:42:37 2024 - [info]   node1(192.168.3.131:3306)
Fri Mar  1 01:42:37 2024 - [info]   node2(192.168.3.132:3306)
Fri Mar  1 01:42:37 2024 - [info]   node3(192.168.3.133:3306)
Fri Mar  1 01:42:37 2024 - [info] Alive Slaves:
Fri Mar  1 01:42:37 2024 - [info]   node2(192.168.3.132:3306)  Version=8.0.27 (oldest major version between slaves) log-bin:enabled
Fri Mar  1 01:42:37 2024 - [info]     GTID ON
Fri Mar  1 01:42:37 2024 - [info]     Replicating from 192.168.3.131(192.168.3.131:3306)
Fri Mar  1 01:42:37 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Fri Mar  1 01:42:37 2024 - [info]   node3(192.168.3.133:3306)  Version=8.0.27 (oldest major version between slaves) log-bin:enabled
Fri Mar  1 01:42:37 2024 - [info]     GTID ON
Fri Mar  1 01:42:37 2024 - [info]     Replicating from 192.168.3.131(192.168.3.131:3306)
Fri Mar  1 01:42:37 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Fri Mar  1 01:42:37 2024 - [info] Current Alive Master: node1(192.168.3.131:3306)
Fri Mar  1 01:42:37 2024 - [info] Checking slave configurations..
Fri Mar  1 01:42:37 2024 - [info] Checking replication filtering settings..
Fri Mar  1 01:42:37 2024 - [info]  binlog_do_db= , binlog_ignore_db=
Fri Mar  1 01:42:37 2024 - [info]  Replication filtering check ok.
Fri Mar  1 01:42:37 2024 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Fri Mar  1 01:42:37 2024 - [info] Checking SSH publickey authentication settings on the current master..
Fri Mar  1 01:42:37 2024 - [info] HealthCheck: SSH to node1 is reachable.
Fri Mar  1 01:42:37 2024 - [info]
node1(192.168.3.131:3306) (current master)
 +--node2(192.168.3.132:3306)
 +--node3(192.168.3.133:3306)

Fri Mar  1 01:42:37 2024 - [info] Checking master_ip_failover_script status:
Fri Mar  1 01:42:37 2024 - [info]   /usr/local/bin/master_ip_failover --command=status --ssh_user=root --orig_master_host=node1 --orig_master_ip=192.168.3.131 --orig_master_port=3306
Fri Mar  1 01:42:37 2024 - [info]  OK.
Fri Mar  1 01:42:37 2024 - [warning] shutdown_script is not defined.
Fri Mar  1 01:42:37 2024 - [info] Set master ping interval 1 seconds.
Fri Mar  1 01:42:37 2024 - [info] Set secondary check script: /usr/bin/masterha_secondary_check -s node2 --user=root --master_host=node1 --master_ip=192.168.3.131 --master_port=3306
Fri Mar  1 01:42:37 2024 - [info] Starting ping health check on node1(192.168.3.131:3306)..
Fri Mar  1 01:42:37 2024 - [info] Ping(CONNECT) succeeded, waiting until MySQL doesn't respond..
{% endcodeblock %}

### 模拟故障转移



{% codeblock "1. 在 windows 主机通过浮动 IP 登录并查看 server_id" lang:cmd >folded %}
D:\Works\mysql-8.0.15-winx64\bin>mysql -umhauser -p -h 192.168.3.200 -e "select @@server_id;"
Enter password: *******
+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
{% endcodeblock %}



{% codeblock "2. 停掉当前主节点（node1）上的 MYSQL 实例" lang:shll >folded %}
# 停止 mysql 实例
[root@localhost tmp]# systemctl stop mysqld
[root@localhost tmp]# systemctl status mysqld
● mysqld.service - LSB: start and stop MySQL
   Loaded: loaded (/etc/rc.d/init.d/mysqld; bad; vendor preset: disabled)
   Active: inactive (dead) since 五 2024-03-01 01:44:15 CST; 5s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 47061 ExecStop=/etc/rc.d/init.d/mysqld stop (code=exited, status=0/SUCCESS)
  Process: 45134 ExecStart=/etc/rc.d/init.d/mysqld start (code=exited, status=0/SUCCESS)

# 查看 ip 地址状态
[root@localhost tmp]# ip addr list ens33
# 浮动 ip "192.168.3.200/24" 已经被转移
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ca:2b:a0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.3.131/24 brd 192.168.3.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::25c7:c0dc:4aeb:8e4/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
[root@localhost tmp]#
{% endcodeblock %}



{% codeblock "3. 再次在 windows 主机通过浮动 IP 登录并查看 server_id" lang:cmd >folded %}
D:\Works\mysql-8.0.15-winx64\bin>mysql -umhauser -p -h 192.168.3.200 -e "select @@server_id;"
Enter password: *******
# server_id 已经由 1 变成了 2，也就说明了 node2 成为了新的主节点，并且浮动 IP 也已经转移到 node2。
+-------------+
| @@server_id |
+-------------+
|           2 |
+-------------+
{% endcodeblock %}

{% codeblock "4. 查看 node3 的主从状态" lang:shell >folded %}
root@localhost (none) :58: >show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 192.168.3.132 # 主节点也已经切换到 node2
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 2264
               Relay_Log_File: node3-relay-bin.000002
                Relay_Log_Pos: 418
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
{% endcodeblock %}

MHA Manager 故障转移成功，并且整个过程对于 Client 端都是透明的。

### 模拟故障恢复

下面再用一个简单的情景模拟一下 MHA 故障转移和故障恢复，已经其是否能保障数据的完整性。

#### 故障转移：


{% codeblock "1. 在主节点创建一张 student 表 " lang:shell >folded %}
root@localhost (none) :52: >create database test;
Query OK, 1 row affected (0.04 sec)
root@localhost (none) :52: >create table test.student( studentId int primary key, name varchar(50), age int);
Query OK, 0 rows affected (0.07 sec)
{% endcodeblock %}



{% codeblock "2. Client 端使用 10 条线程插入 10w 条数据" lang:java >folded %}
// 插入采用重试策略，插入返回失败会继续重试到成功为止
@SpringBootApplication
public class MHADemo implements ApplicationListener<ApplicationReadyEvent> {
    @Autowired
    JdbcTemplate jdbcTemplate;

    public static void main(String[] args) {
        SpringApplication.run(MHADemo.class,args);
    }

    @Override
    public void onApplicationEvent(ApplicationReadyEvent applicationReadyEvent) {
        for(int i=1;i<=10;i++){
            // 每条线程插入 1w 条数据
            new Thread(task(i),"Thread-"+i).start();
        }
    }

    private Runnable task(final int threadId){
        return ()->{
            for(int studentId = threadId;studentId <=100_000;studentId+=10){
                Map<String,String> student = new HashMap<>();
                student.put("studentId",studentId + "");
                student.put("name","name-"+studentId);
                student.put("age",(studentId % 100)+"");
                while (true) {
                    try {
                        jdbcTemplate.update("insert into test.student (studentId,name,age) values(?,?,?)",
                                student.get("studentId"), student.get("name"), student.get("age"));
                        System.out.printf("[Thread-%s]: insert studentId: %s\n",threadId,studentId);
                        break;
                    } catch (Exception e) {
                        System.err.printf("[Thread-%s]: insert failed, studentId: %s\n",threadId,studentId);
                    }
                }
            }
            System.out.printf("[Thread-%s]: insert completed\n",threadId);
        };
    }
}
{% endcodeblock %}


{% codeblock "3. 在插入的过程中关闭主节点的 MYSQL 实例，此时 Client 端会插入失败，并且不断重试。" lang:shell >folded %}
{% endcodeblock %}

![图 2](mha_failover_2_1.png)


{% codeblock "4. MHA 故障转移成功，Client 重试成功，继续完成后面的数据插入。" lang:shell >folded %}
{% endcodeblock %}

![图 3](mha_failover_2_2.png)

{% codeblock "5. 查询 node2 和 node3 的数据完整性" lang:shell >folded %}
# 都有全部 10w 条数据
[root@node1 ~]# ssh node2
Last login: Thu Feb 29 18:50:37 2024 from node1
[root@node2 ~]# mysql -uroot -p -e 'select count(*) from test.student;'
Enter password:
+----------+
| count(*) |
+----------+
|   100000 |
+----------+
[root@node2 ~]# exit;
登出
Connection to node2 closed.

[root@node1 ~]# ssh node3
Last login: Thu Feb 29 18:52:20 2024 from node1
[root@node3 ~]# mysql -uroot -p -e 'select count(*) from test.student;'
Enter password:
+----------+
| count(*) |
+----------+
|   100000 |
+----------+
{% endcodeblock %}



{% codeblock "5. 查询 node1 的数据" lang:cmd >folded %}
# node1 只有故障前的部分数据
[root@node1 ~]# systemctl start mysqld
[root@node1 ~]# mysql -uroot -p -e 'select count(*) from test.student;';
Enter password:
+----------+
| count(*) |
+----------+
|    31189 |
+----------+
{% endcodeblock %}

#### 故障回复：

{% codeblock "1. 重新启动 node1" lang:sql >folded %}
```shell
systemctl start mysqld
```
{% endcodeblock %}

{% codeblock "2. 把 node1 设置为 node2 的从节点加入集群" lang:sql >folded %}
root@localhost (none) :17: >stop slave;
Query OK, 0 rows affected, 2 warnings (0.00 sec)

root@localhost (none) :17: >reset slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)

root@localhost (none) :17: >change master to
-> master_host='192.168.3.132',
    -> master_user='repluser',
    -> master_password='replpass',
    -> master_auto_position=1;
Query OK, 0 rows affected, 7 warnings (0.06 sec)

root@localhost (none) :18: >start slave;
Query OK, 0 rows affected, 1 warning (0.04 sec)

root@localhost (none) :18: >show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 192.168.3.132
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000009
          Read_Master_Log_Pos: 40520183
               Relay_Log_File: node1-relay-bin.000002
                Relay_Log_Pos: 2527803
        Relay_Master_Log_File: mysql-bin.000009
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
1 row in set, 1 warning (0.01 sec)
{% endcodeblock %}



{% codeblock "3. node1 成功从新的主节点上复制剩余的数据：" lang:shell >folded %}
root@localhost (none) :18: >select count(*) from test.student;
+----------+
| count(*) |
+----------+
|    53421 |
+----------+
1 row in set (0.01 sec)

root@localhost (none) :18: >select count(*) from test.student;
+----------+
| count(*) |
+----------+
|    66588 |
+----------+
1 row in set (0.00 sec)

root@localhost (none) :19: >select count(*) from test.student;
+----------+
| count(*) |
+----------+
|   100000 |
+----------+
1 row in set (0.00 sec)
{% endcodeblock %}



{% codeblock "4. 修改 `/etc/masterha/app1.cnf` 的 `secondary_check_script` 参数" lang:shell >folded %}
# 修改为 master_host=node2， master_ip=192.168.3.132， -s node1
secondary_check_script=/usr/bin/masterha_secondary_check -s node1 --user=root --master_host=node2 --master_ip=192.168.3.132 --master_port=3306
{% endcodeblock %}



{% codeblock "5. 重新启动 MHA Manager" lang:shell >folded %}
# 启动前先检测集群的复制状态
[root@node3 ~]# masterha_check_repl --conf=/etc/masterha/app1.cnf
...
MySQL Replication Health is OK.
# 启动
[root@node3 ~]# nohup masterha_manager --conf=/etc/masterha/app1.cnf --ignore_last_failove < /dev/null >/data/masterha/app1/manager.log 2>&1 &
[1] 19535
[root@node3 ~]#
{% endcodeblock %}

## 参考资料

《深入浅出 MYSQL》

[《MHA Manager Github》](https://github.com/yoshinorim/mha4mysql-manager)
