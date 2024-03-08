---
title: Mysql一主二从 （3）- MYSQL 集群搭建
date: 2024-03-07 15:08:44
tags:
categories: Mysql
thumbnail:
toc: true
---

本文将描述如何从零开始搭建一个一主二从的 MYSQL 集群，该集群通过以下特性来满足第一章 {% post_link "Mysql 一主二从（1）- 架构描述" "《Mysql 一主二从（1）- 架构描述》" %} 中提出的需求：
- 双一配置：保证业务请求已经正确返回的数据一定被写入硬盘。
- 增强半同步复制：同时保证集群中任何时候都至少有两个节点拥有完整的数据集。
- GTID 复制：故障转移和恢复变得更加安全，方便，快速。
<!--more-->

# 安装 MYSQL

分别在 node1，node2 和 node3 上安装 MYSQL

- yum 安装参考《Mysql 安装 - yum 安装》
- 压缩包安装参考《Mysql 安装 - Centos 7 压缩包安装》
- 脚本自动安装参考《Centos7 压缩包安装脚本》

# 关闭防火墙

在每个节点上分别执行：

```shell
[root@localhost bin]# systemctl stop firewalld
[root@localhost bin]# systemctl disable firewalld
```

# 配置主从

> 注意：下面以 node1 为例，每个节点都需要同样配置一次

## 双 1 设置

确认 `/etc/my.cnf` 中下列参数的配置：

```ini
[mysqld]
#表示每次事务的 binlog 都会fsync持久化到磁盘，MySQL 5.7.7 之后默认为1，之前的版本默认为0
sync_binlog=1 
#表示每次事务的 redo log 都直接持久化到磁盘，默认值为1
innodb_flush_log_at_trx_commit=1 
```

## 开启 GTID 复制

删除 $datadir/auto.cnf

```shell
rm -rf /data/mysql/data/auto.cnf
```

在 /etc/my.cnf 追加参数：

```ini
#每个节点的 server_id 必须不同
server_id=1
#手动启动同步复制，节点恢复时需要人工处理并确认无误后，再手动启动复制
skip_slave_start=ON
#启动默认为只读
read_only=ON

# 开启 GTID 
gtid_mode=ON
enforce_gtid_consistency=ON
# GTID 模式下从库更新写入 binlog
# ON：写入 OFF：不写入
log_slave_updates=ON
# log_slave_updates=OFF 时起作用
# 表示每处理完成 1000 个事务，就对表 mysql.gtid_executed 进行压缩
gtid_executed_compression_period=1000
```

重启 MYSQL 实例 `systemctl restart mysql`，检测 MYSQL 参数：

{% codeblock "demo" lang:shell >folded %}
root@localhost (none) :45: >select @@server_id;
+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)

root@localhost (none) :46: >select @@skip_slave_start;
+--------------------+
| @@skip_slave_start |
+--------------------+
|                  1 |
+--------------------+
1 row in set, 1 warning (0.00 sec)

root@localhost (none) :47: >show variables like '%gtid%';
+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| binlog_gtid_simple_recovery      | ON        |
| enforce_gtid_consistency         | ON        |
| gtid_executed                    |           |
| gtid_executed_compression_period | 0         |
| gtid_mode                        | ON        |
| gtid_next                        | AUTOMATIC |
| gtid_owned                       |           |
| gtid_purged                      |           |
| session_track_gtids              | OFF       |
+----------------------------------+-----------+
9 rows in set (0.02 sec)

root@localhost (none) :47: >
{% endcodeblock %}

## 使用增强半同步模式

1. 判断 Mysql 服务是否支持动态增加插件：

```sql
[root@localhost bin]# mysql -uroot -p -e "select @@have_dynamic_loading;"
Enter password:
+------------------------+
| @@have_dynamic_loading |
+------------------------+
| YES                    |
+------------------------+
```

2. 确认是否存在主库 `semisync_master.so` 和 从库插件 `semisync_slave.so`：

```shell
[root@localhost bin]# ll $MYSQL_HOME/lib/plugin/semisync_master.so
-rwxr-xr-x. 1 7161 31415 1071400 9月  28 2021 /usr/local/mysql/lib/plugin/semisync_master.so

[root@localhost bin]# ll $MYSQL_HOME/lib/plugin/semisync_slave.so
-rwxr-xr-x. 1 7161 31415 691048 9月  28 2021 /usr/local/mysql/lib/plugin/semisync_slave.so
```

3. 安装插件

```shell
# 考虑到主备切换，因此在主从上都安装两个插件
mysql> install plugin rpl_semi_sync_master soname 'semisync_master.so';
Query OK, 0 rows affected, 1 warning (0.20 sec)

mysql> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
Query OK, 0 rows affected, 1 warning (0.00 sec)

# 查看已安装的插件
mysql> select * from mysql.plugin;
+----------------------+--------------------+
| name                 | dl                 |
+----------------------+--------------------+
| rpl_semi_sync_master | semisync_master.so |
| rpl_semi_sync_slave  | semisync_slave.so  |
+----------------------+--------------------+
2 rows in set (0.00 sec)
```

4. 打开增强半同步复制

```sql
mysql> set global rpl_semi_sync_master_enabled=on;
mysql> set global rpl_semi_sync_master_timeout=10000;
mysql> set global rpl_semi_sync_slave_enabled=on;

root@localhost (none) :51: >show variables like '%semi_sync%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | ON         |
| rpl_semi_sync_master_timeout              | 10000      |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
| rpl_semi_sync_slave_enabled               | ON         |
| rpl_semi_sync_slave_trace_level           | 32         |
+-------------------------------------------+------------+
8 rows in set (0.01 sec)

```

5. /etc/my.cnf 追加参数：

```ini
# 这些配置必须在安装插件后才能配置到 my.cnf 文件中，否则会启动失败
# 开启增强半同步复制，因为主从节点会发送切换，这里 master 和 slave 都同时打开
rpl_semi_sync_slave_enabled=ON
rpl_semi_sync_master_enabled=ON
rpl_semi_sync_master_timeout=10000
```

# 开启主从 

- Master：node1
- Slave：node2，node3

## 创建复制账号

在 **Master** 节点上创建复制账号

```sql
# host 需要修改为自己的网络
# 指定认证方式 mysql_native_password，不指定默认为 caching_sha2_password
create user 'repluser'@'192.168.3.%' identified with mysql_native_password by 'replpass';

grant replication slave,replication client on *.* to 'repluser'@'192.168.3.%';
```

> 如果要在同步复制中使用  `caching_sha2_password`认证，可以参考[官方文档](https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html)

## 启动复制

> 两个 slave 节点分别执行相同的操作

```sql
# 保证 Slave 为只读状态 
mysql> set global read_onln=on;
# 配置主节点信息
mysql> change master to
-> master_host='192.168.3.131',
-> master_user='repluser',
-> master_password='replpass',
-> master_auto_position=1;

# 启动复制线程
mysql> start slave; 
```

使用 show processlist 命令查看复制连接状态信息：


{% codeblock "demo" lang:sql >folded %}
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
1 row in set, 1 warning (0.01 sec)
# 以上只显示了部分信息
{% endcodeblock %}

在每个从节点上查看复制账号是否同步过来了，如果同步过来则表示同步配置成功

{% codeblock "demo" lang:shell >folded %}
root@localhost (none) :06: >select user from mysql.user;
+------------------+
| user             |
+------------------+
| repluser         |
| mysql.infoschema |
| mysql.session    |
| mysql.sys        |
| root             |
+------------------+
5 rows in set (0.00 sec)
{% endcodeblock %}

## 设置 Master 为可写状态

```shell
root@localhost (none) :05: >set global read_only=off;
Query OK, 0 rows affected (0.00 sec)

root@localhost (none) :09: >select @@read_only;
+-------------+
| @@read_only |
+-------------+
|           0 |
+-------------+
1 row in set (0.00 sec)
```

一主二从配置完成。