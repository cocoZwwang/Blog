---
title: Mysql 一主二从（2）- 环境搭建
date: 2024-03-07 15:01:06
tags:
categories: Mysql
thumbnail:
toc: true
---


上一章 {% post_link "Mysql 一主二从（1）- 架构描述" "《Mysql 一主二从（1）- 架构描述》" %}中，给出了架构的实现和部署图，这一章将描述如何通过虚拟机构建实验的基础环境：创建三个节点，并同时配置节点之间的 SSH 免密登录。
<!--more-->

# 创建虚拟机节点

参考文章{% post_link "VMWare Player 安装Centos 7" "《VMWare Player 安装Centos 7》" %}创建三个虚拟机节点：

- 操作系统：CentOS 7 
- 主机名称：node1，node2，node3
- 网络地址：192.168.3.131，192.168.3.132，192.168.3.132

# 主机名解析

在每个节点的 /etc/hosts 都添加如下内容：

```shell
192.168.3.131 node1
192.168.3.132 node2
192.168.3.133 node3
```

# ssh 免密登录

在 node1 上创建脚本  `~/bin/ssh_rsa_gen`：

```shell
#!/bin/bash
# 注意： 需要添加 ~/bin 路径当前环境变量；需要添加 ssh_rsa_gen 的执行权限。
nodes=("node1" "node2" "node3")
ssh_user=root
local_node=`hostname`

# 生成密钥对
ssh-keygen -t rsa -N ''  -f ~/.ssh/id_rsa

# 把公钥 id_rsa.pub 的内容追加到每个节点（包括自身）的 ~/.ssh/authorized_keys 文件
for node in "${nodes[@]}"
do
    scp ~/.ssh/id_rsa.pub $ssh_user@$node:/tmp/
    [ $? -ne 0 ] && exit 1
    ssh $ssh_user@$node "mkdir -pv ~/.ssh &&\
        cat /tmp/id_rsa.pub >> ~/.ssh/authorized_keys &&\
        chmod go= ~/.ssh/authorized_keys &&\
        rm -rf /tmp/id_rsa.pub"
done
```

在 node1 上执行脚本，脚本执行过程中会被多次要求输入密码：

{% codeblock "demo" lang:shell >folded %}
[root@node3 ~]# ssh_rsa_gen
Generating public/private rsa key pair.
Created directory '/root/.ssh'.
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:AeAWRarFpbiVTgJ8tKTCYhatWd9pGdJZnbyAwxxnJbc root@node3
The key's randomart image is:
+---[RSA 2048]----+
|o.ooo+O.==+o.    |
|.o+B.B Xo.o+.    |
|o=B.% o * .E.    |
|=o X . = . .     |
|  o . . S        |
|                 |
|                 |
|                 |
|                 |
+----[SHA256]-----+
The authenticity of host 'node1 (192.168.3.131)' can't be established.
ECDSA key fingerprint is SHA256:JiNjfTMhq+/0CEDeTq1wGH+W6X+2/xYuHhBC3YHsK4w.
ECDSA key fingerprint is MD5:71:0e:23:35:b6:1a:49:81:33:ec:9c:54:46:49:08:ec.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'node1,192.168.3.131' (ECDSA) to the list of known hosts.
root@node1's password:
id_rsa.pub                                                                                                                                 100%  392   264.8KB/s   00:00
root@node1's password:
mkdir: 已创建目录 "/root/.ssh"
The authenticity of host 'node2 (192.168.3.132)' can't be established.
ECDSA key fingerprint is SHA256:JiNjfTMhq+/0CEDeTq1wGH+W6X+2/xYuHhBC3YHsK4w.
ECDSA key fingerprint is MD5:71:0e:23:35:b6:1a:49:81:33:ec:9c:54:46:49:08:ec.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'node2,192.168.3.132' (ECDSA) to the list of known hosts.
root@node2's password:
id_rsa.pub                                                                                                                                 100%  392   294.0KB/s   00:00
root@node2's password:
mkdir: 已创建目录 "/root/.ssh"
The authenticity of host 'node3 (192.168.3.133)' can't be established.
ECDSA key fingerprint is SHA256:JiNjfTMhq+/0CEDeTq1wGH+W6X+2/xYuHhBC3YHsK4w.
ECDSA key fingerprint is MD5:71:0e:23:35:b6:1a:49:81:33:ec:9c:54:46:49:08:ec.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'node3,192.168.3.133' (ECDSA) to the list of known hosts.
root@node3's password:
id_rsa.pub                                                                                                                                 100%  392   485.1KB/s   00:00
root@node3's password:
{% endcodeblock %}

把脚本复制到其他节点，此时 scp 命令已经不需要输入密码：

{% codeblock "demo" lang:shell >folded %}
[root@localhost ~]# scp ~/bin/ssh_rsa_gen node2:~/bin/
ssh_rsa_gen                                   100%  451   234.8KB/s   00:00
[root@localhost ~]# scp ~/bin/ssh_rsa_gen node3:~/bin/
ssh_rsa_gen                                   100%  451   272.8KB/s   00:00
{% endcodeblock %}

远程登录其他节点并执行脚本：

{% codeblock "demo" lang:shell >folded %}
[root@localhost ~]# ssh node2
Last login: Wed Feb 28 22:59:26 2024 from node1
[root@node2 ~]# ssh_rsa_gen
....
[root@node2 ~]# exit
登出
Connection to node2 closed.

[root@localhost ~]# ssh node3
Last login: Wed Feb 28 22:44:57 2024 from node1
[root@node3 ~]# ssh_rsa_gen
...
[root@node3 ~]# exit
登出
Connection to node3 closed.
{% endcodeblock %}