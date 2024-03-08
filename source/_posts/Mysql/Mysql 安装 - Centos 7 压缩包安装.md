---
title: Mysql 安装 - Centos 7 二进制包安装
toc: true
date: 2024-03-08 09:28:24
tags: Mysql
categories:
thumbnail:
---

二进制包安装可以指定任意安装路径，灵活性好，同时一台服务器可以安装多个版本的 Mysql 软件。

- 操作系统：CentOS Linux release 7.7.1908
- MYSQL：mysql-8.0.27
- 安装过程和配置仅仅作为参考，你应该根据自己的环境进行调整。
- 文章最后提供二进制包自动安装脚本

<!--more-->

## 创建 mysql 用户

 ```shell
groupadd -r mysql
useradd -g mysql -r -M mysql
chown -R mysql:mysql /data/mysql
 ```

## 下载和安装

```shell
# 下载到安装包到 /tmp
wget -P /tmp https://cdn.mysql.com/archives/mysql-8.0/mysql-8.0.27-linux-glibc2.12-x86_64.tar.xz
# 解压安装包到 /usr/local
tar -xvf /tmp/mysql-8.0.27-linux-glibc2.12-x86_64.tar.xz -C /usr/local
# 设置软链接：
ln -sv /usr/local/ $base_dir 
# 修改 MYSQL 文件属主
cd /usr/local/mysql
chown -R mysql:mysql ./
```


### 配置


{% codeblock "创建 MYSQL 相关目录和文件" lang:shell >folded %}
mkdir /data/mysql -pv # 数据目录
mkdir /var/log/mysql -pv # mysqld 日志目录
touch /var/log/mysql/mysqld.log # error-log-file
mkdir /var/lib/mysql -pv 
mkdir /var/run/mysqld -pv
# 修改文件目录所属
chown -R mysql:mysql /data/mysql
chown -R mysql:mysql /var/lib/mysql
chown -R mysql:mysql /var/log/mysql
chown -R mysql:mysql /var/run/mysqld
{% endcodeblock %}


{% codeblock "配置环境变量" lang:shell >folded %}
vim /etc/profile.d/mysql.sh

export MYSQL_HOME=$base_dir
export PATH=$MYSQL_HOME/bin:$PATH
{% endcodeblock %}

{% codeblock "刷新环境变量" lang:shell >folded %}
source /etc/profile.d/mysql.sh
{% endcodeblock %}



{% codeblock "配置 systemd" lang:shell >folded %}
cp /usr/local/mysql/support-files/mysql.server  /etc/rc.d/init.d/mysqld
chmod +x /etc/rc.d/init.d/mysqld
/sbin/chkconfig --add mysqld
/sbin/chkconfig mysqld on
{% endcodeblock %}

{% codeblock "配置 mysql lib" lang:shell >folded %}
ln -sv /usr/local/mysql/include  /usr/include/mysql 
echo "/usr/local/mysql/lib" > /etc/ld.so.conf.d/mysql.conf 
ldconfig
{% endcodeblock %}

{% codeblock "配置 /etc/my.cnf" lang:ini >folded %}

[client]
socket=/var/lib/mysql/mysql.sock
port=3306

[mysql]
prompt="\\u@\\h \\d \\r:\\m:\\s>"
auto-rehash


[mysqld]
port=3306
basedir=/usr/local/mysql
datadir=/data/mysql/data
socket=/var/lib/mysql/mysql.sock
pid-file=/var/run/mysqld/mysqld.pid

# mysql service log
log_error=/var/log/mysql/mysqld.log
slow_query_log=ON
slow_query_log_file=/var/log/mysql/slow-query.log
long_query_time=10
log_queries_not_using_indexes=1
log_slow_admin_statements=1

# binary log
log_bin=/data/mysql/binlog/mysql-bin
log_bin_index=/data/mysql/binlog/mysql-bin.index
binlog_format=row
binlog_rows_query_log_events=on
sync_binlog=1 #表示每次事务的 binlog 都会fsync持久化到磁盘，MySQL 5.7.7 之后默认为1，之前的版本默认为0
innodb_flush_log_at_trx_commit=1 #表示每次事务的 redo log 都直接持久化到磁盘，默认值为1
{% endcodeblock %}

## 初始化 MYSQL

{% codeblock "执行初始化命令" lang:shell >folded %}
# $MYSQL_HOME:/usr/local/mysql
[root@node1 ~]# /usr/local/mysql/bin/mysqld --initialize --user=mysql
{% endcodeblock %}

{% codeblock "查看 'root'@'localhost' 的初始密码" lang:shell >folded %}
# 初始密码打印在 log_error 文件中，该路径配置在 /etc/my.cnf 
[root@node1 ~]# tail -30 /var/log/mysql/mysqld.log

2024-03-03T12:13:39.218978Z 0 [System] [MY-013169] [Server] /usr/local/mysql/bin/mysqld (mysqld 8.0.27) initializing of server in progress as process 2943
2024-03-03T12:13:39.232758Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2024-03-03T12:13:39.761340Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2024-03-03T12:13:40.766867Z 0 [Warning] [MY-013746] [Server] A deprecated TLS version TLSv1 is enabled for channel mysql_main
2024-03-03T12:13:40.766883Z 0 [Warning] [MY-013746] [Server] A deprecated TLS version TLSv1.1 is enabled for channel mysql_main
# 冒号后面的内容就是 root@localhost 的初始化密码
2024-03-03T12:13:40.798911Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: 1iwve;lr)vZ4  
....
{% endcodeblock %}

{% codeblock "或者使用 grep 命令直接读取初始化密码" lang:shell >folded %}
grep 'A temporary password is generated for root@localhost' /var/log/mysql/mysqld.log | tail -1 | grep -o 'root@localhost: .*' | awk '{print $2}'
{% endcodeblock %}

{% codeblock "启动 mysql" lang:shell >folded %}
service mysqld start
{% endcodeblock %}

{% codeblock "使用初始密码登录 MYSQL，并修改 'root'@'localhost' 密码：" lang:sql >folded %}
# mysql 8.0 账号默认使用 'caching_sha2_password' 认证插件
alter user 'root'@'localhost' identified by 'yourpassword'

# 指定 mysql_native_password
alter user 'root'@'localhost' identified with mysql_native_password by 'yourpassword'
{% endcodeblock %}


{% codeblock "关闭 mysql" lang:shell >folded %}
service mysqld stop
{% endcodeblock %}

退出，使用新密码重新登录，安装完成~

## 安装脚本
- 安装配置：mysql_config，根据安装环境的需要修改
- 安装命令：mysql_install
    - 使用前：安装 wget
    - 使用后：使用命令 `source /etc/profile.d/mysql.sh` 刷新当前窗口的环境变量
- 卸载命令：mysql_uninstall
- 辅助脚本：script_helper.sh
- 脚本存放路径：所有脚本需要放在同一路径之下


{% codeblock "mysql_config" lang:shell >folded %}
#!/bin/bash

package_name=mysql-8.0.27-linux-glibc2.12-x86_64
package_suffix=.tar.xz
package_file=$package_name$package_suffix
package_url=https://cdn.mysql.com/archives/mysql-8.0/$package_file

# 输入 base_dir_prefix ，使用默认值(/usr/local)，直接回车
read -p "base_dir_prefix(default /usr/local): " base_dir_prefix
base_dir_prefix=${base_dir_prefix:-"/usr/local"}
# 安装目录
install_dir=$base_dir_prefix/$package_name
# MYSQL_HOME
base_dir=$base_dir_prefix/mysql

# 输入 data_dir_prefix，使用默认值(/data/mysql)，直接回车
read -p "data_dir_prefix(default /data/mysql): " base_dir_prefix
data_dir_prefix=${data_dir_prefix:-"/data/mysql"}
# 表数据存储目录
data_dir=$data_dir_prefix/data
# 二进制日志存储目录
binlog_dir=$data_dir_prefix/binlog

log_dir=/var/log/mysql
error_log_file=$log_dir/mysqld.log
lib_dir=/var/lib/mysql
run_dir=/var/run/mysqld
{% endcodeblock %}



{% codeblock "mysql_install" lang:shell >folded %}
#!/bin/bash
source ./script_helper.sh
source ./mysql_config
LOG_LEVEL=2

# read variables
# 输入端口，使用默认值(3306)，直接回车
read -p "mysqld port(default 3306): " port
port=${port:-3306}
# 输入 server_id，使用默认值(1)，直接回车
read -p "server_id(default 1): " server_id
server_id=${server_id:-1}
# 输入需要设置的 root@localhost 的密码，使用默认值(初始化自动生成的密码)，直接回车
read -s -p "root@locahost's password(default init password): " rootpass

# create mysql user
log_info "create user mysql:mysql..."
groupadd -r mysql >& /dev/null &&
useradd -g mysql -r -M mysql &>/dev/null
id mysql
[ "$?" -ne 0 ] && exit 1;


# load mysql
log_info "loading mysql..."
wget -P /tmp $package_url &&
tar -xvf /tmp/$package_file -C $base_dir_prefix &&
ln -sv $install_dir $base_dir &&
rm -rf /tmp/$package_file
[ "$?" -ne 0 ] && exit 1;

# create dir and file
log_info "create mysql-related files and directories...."
mkdir -pv $data_dir $binlog_dir $log_dir $lib_dir $run_dir &&
touch $error_log_file &&
chown -R mysql:mysql $data_dir_prefix $log_dir $lib_dir $run_dir
[ "$?" -ne 0 ] && exit 1;


# setting mysql profile
log_info "setting mysql profile..."
cat > /etc/profile.d/mysql.sh << EOF
export MYSQL_HOME=$base_dir
export PATH=\$MYSQL_HOME/bin:\$PATH
EOF
source /etc/profile.d/mysql.sh
[ $? -ne 0 ] && exit 1;


# configuring systemd for mysqld
log_info "configuring systemd for mysqld..."
cp $base_dir/support-files/mysql.server  /etc/rc.d/init.d/mysqld &&
chmod +x /etc/rc.d/init.d/mysqld &&
/sbin/chkconfig --add mysqld &&
/sbin/chkconfig mysqld on
[ $? -ne 0 ] && exit 1;

# add mysql man
log_info "add mysql man..."
cat >> /etc/man.config << EOF
MANPATH  $base_dir/man
EOF
[ $? -ne 0 ] && log_warn "cp mysql.server to init.d failed";


# configurate mysql lib
log_info "config mysql lib..."
ln -sv $base_dir/include  /usr/include/mysql &&
echo "$base_dir/lib" > /etc/ld.so.conf.d/mysql.conf &&
ldconfig
[ $? -ne 0 ] && log_warn "configurate mysql lib failed";


####################### setting /etc/my.cnf ################################################
log_info "setting my.cnf..."
cat > /etc/my.cnf << EOF

[client]
socket=$lib_dir/mysql.sock
port=$port

[mysql]
prompt="\\u@\\h \\d \\r:\\m:\\s>"
auto-rehash


[mysqld]
port=$port
basedir=$base_dir
datadir=$data_dir
socket=$lib_dir/mysql.sock
pid-file=$run_dir/mysqld.pid

# mysql service log
log_error=$error_log_file
slow_query_log=ON
slow_query_log_file=$log_dir/slow-query.log
long_query_time=10
log_queries_not_using_indexes=1
log_slow_admin_statements=1

# binary log
log_bin=$binlog_dir/mysql-bin
log_bin_index=$binlog_dir/mysql-bin.index
binlog_format=row
binlog_rows_query_log_events=on
sync_binlog=1 #表示每次事务的 binlog 都会fsync持久化到磁盘，MySQL 5.7.7 之后默认为1，之前的版本默认为0
innodb_flush_log_at_trx_commit=1 #表示每次事务的 redo log 都直接持久化到磁盘，默认值为1

EOF
[ $? -ne 0 ] && exit 1;

######################## initalize mysql ######################################################################
log_info "initalize mysql..."
$MYSQL_HOME/bin/mysqld --initialize --user=mysql

tmppwd=`grep 'A temporary password is generated for root@localhost' $error_log_file | tail -1 | grep -o 'root@localhost: .*' | awk '{print $2}'`
log_debug "tmp password:$tmppwd"

service mysqld start
mysql -uroot -p${tmppwd} --connect-expired-password  -e "alter user 'root'@'localhost' identified by '$rootpass';"
service mysqld stop
systemctl enable mysqld
{% endcodeblock %}



{% codeblock "mysql_uninstall" lang:shell >folded %}
#!/bin/bash
source ./script_helper.sh
source ./mysql_config
LOG_LEVEL=2

################## uninstall mysql #######################################################
function uninstall_mysql(){
kill_process_by_name mysqld;
systemctl disable mysqld &> /dev/null;

test -f $pid_file && rm -rf $pid_file;
test -f $sock_file && rm -rf $sock_file;

test -d $data_dir_prefix && rm -rf $data_dir_prefix;
test -d $log_dir && rm -rf $log_dir;
test -d $lib_dir && rm -rf $lib_dir;
test -d $run_dir && rm -rf $run_dir


if [ -f /etc/profile.d/mysql.sh ]; then
    echo "export PATH=\$PATH" > /etc/profile.d/mysql.sh;
    source /etc/profile.d/mysql.sh;
    rm -rf /etc/profile.d/mysql.sh;
fi

if [ -f "/etc/rc.d/init.d/mysqld" ]; then
/sbin/chkconfig mysqld off;
/sbin/chkconfig --del mysqld;
fi

test -f /etc/rc.d/init.d/mysqld && rm -rf /etc/rc.d/init.d/mysqld;
sed -i  "/MANPATH \${base_dir}/d" /etc/man.config;

test -L /usr/include/mysql && unlink /usr/include/mysql;
test -f /etc/ld.so.conf.d/mysql.conf &&  rm -rf /etc/ld.so.conf.d/mysql.conf;
ldconfig;


test -f /etc/my.cnf && rm -rf /etc/my.cnf;
test -L $base_dir && unlink $base_dir;
test -d $install_dir && rm -rf $install_dir;
echo "uninstall mysql finish"
}

check_dir(){
test -d $1 && log_warn "$1 remain" && exit 1;
}

check_file(){
test -f $1 && log_warn "$1 remain" && exit 1;
}

check_link(){
test -L $1 && log_warn "$1 remain" && exit 1;
}


################## uninstall status #######################################################
function uninstall_status(){
check_dir $base_dir;
check_dir $install_dir;
check_file /etc/my.cnf;
check_link /usr/include/mysql;
check_file /etc/ld.so.conf.d/mysql.conf;
if [ -z "grep  $base_dir/man  /etc/man.config" ]; then
log_warn "mysql man config remain";
exit 1;
fi
check_file /etc/rc.d/init.d/mysqld;
check_file /etc/profile.d/mysql.sh;
check_dir $data_dir_prefix;
check_dir $log_dir;
check_dir $lib_dir;
check_dir $run_dir;
echo "uninstall mysql finished"
}

case "$1" in
status)
    uninstall_status
;;
*)
    uninstall_mysql
;;
esac
{% endcodeblock %}




{% codeblock "script_helper.sh" lang:shell >folded %}
#!/bin/bash
##### 日志打印 #####
LOGG_OPEN=0
#日志级别 debug:1, info:2, warn:3, error:4, always:5
LOG_LEVEL=3

# 调试日志
function log_debug(){
    content="[DEBUG] $(date '+%Y-%m-%d %H:%M:%S') $@";
    [ $LOG_LEVEL -le 1  ] && echo -e "\033[32m"  ${content}  "\033[0m"
}

#信息日志
function log_info(){
    content="[INFO] $(date '+%Y-%m-%d %H:%M:%S') $@";
    [ $LOG_LEVEL -le 2  ] && echo -e "\033[32m"  ${content} "\033[0m"
}
#警告日志
function log_warn(){
    content="[WARN] $(date '+%Y-%m-%d %H:%M:%S') $@";
    [ $LOG_LEVEL -le 3  ] && echo -e "\033[33m" ${content} "\033[0m"
}
#错误日志
function log_err(){
    content="[ERROR] $(date '+%Y-%m-%d %H:%M:%S') $@";
    [ $LOG_LEVEL -le 4  ] && echo -e "\033[31m" ${content} "\033[0m"
}
#一直都会打印的日志
function log_always(){
    content="[ALWAYS] $(date '+%Y-%m-%d %H:%M:%S') $@";
    [ $LOG_LEVEL -le 5  ] && echo -e  "\033[32m" ${content} "\033[0m"
}

##### 进程处理 #####
kill_process_by_name(){

    #根据进程名杀死进程
    if [ $# -lt 1 ]; then
        echo "缺少参数：pro_name"
        exit 1
    fi
    # 第一行是获取进程号
    PROCESS=`ps -ef | grep $1 | grep -v grep | grep -v PPID | awk '{print $2}'`
    # 第二行是遍历进程号使用kill-9 结束进程
    for i in $PROCESS
        do
            echo "Kill the $1 process [ $i ]"
            kill -9 $i
        done
}
{% endcodeblock %}

