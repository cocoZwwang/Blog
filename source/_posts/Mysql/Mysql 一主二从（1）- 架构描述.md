---
title: Mysql 一主二从（1）- 架构描述
date: 2024-03-07 15:00:42
tags:
categories: Mysql
thumbnail:
---

- 需求描述
- 架构图
- 架构实现
- 部署图

<!--more-->

## 需求描述

- MYSQL 复制：
  - 保证业务请求已经正确返回的数据一定被写入硬盘。
  - 同时保证集群中任何时候都至少有两个节点拥有完整的数据集。
- MYSQL 高可用：
  - 主节点宕机时，能自动进行故障转移，包括提升新的主节点和所有剩余从节点的复制源切换。
  - 故障转移过程中需要保证集群数据的完整性。
  - 故障转移过程对 Client 端透明。
- MYSQL 故障恢复：宕机节点恢复时，能尽快追赶复制进度并加入集群。
- MYSQL 读写分离：写请求由主节点负责，读请求负载均衡到各个节点，充分利用集群的资源。
- 负载均衡器：
  - 健康检测：某个 MYSQL 节点宕机时，能自动将该节点从负载均衡列表中去除，反之如果节点重新加入集群能自动加入负载均衡列表。
  - 高可用：主负载均衡器宕机时，能自动快速地切换到备用负载均衡器，并且切换过程对 Client 端透明。
- 只有三台机器，所有需求都需要在三台机器上实现。

## 架构图

![图 1](1_master_2_slave_arch.png)

## 架构实现

- MYSQL 复制
  - 双 1 设置：`sync_binlog=1`  和 `innodb_flush_log_at_trx_commit=1` 
  - 二进制日志格式：`binlog_format = row`
  - 增强半同步复制
  - GTID 模式
- MYSQL 高可用：
  - 使用 MHA 实现 MYSQL 的高可用。
  - `浮动 IP`:  由 MHA Manager 的故障转移脚本 `master_ip_failover` 实现，故障转移时会自动转移到新的主节点上。

- 负载均衡：
  - 基于 keepalived + LVS/DR 的主备高可用负载均衡器
  - Keepalived 对外提供唯一的`虚拟 IP`，主负载均衡器宕机时，`虚拟 IP` 会自动转移到备负载均衡器上。
- Client 端双数据源实现读写分离：
  - 写请求绑定到 MYSQL 主节点的`浮动 IP`。
  - 读请求绑定到负责均衡器的`虚拟 IP`。

## 部署

### 部署图

![图 2](1_master_2_slave_deploy.png)

- 操作系统：CentOS Linux release 7.7.1908 (Core)
- MYSQL：mysql-8.0.27-linux-glibc2.12-x86_64
- MHA Node：mha4mysql-node-0.58
- MHA Manager：mha4mysql-manager-0.58
- Keepalived：Keepalived v1.3.5