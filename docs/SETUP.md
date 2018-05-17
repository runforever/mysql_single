# 安装特定版本的 MySQL 并调优

## 背景
现有的 MySQL 服务是用 Docker 搭建的单点，版本 5.7.17，没有开启 Binlog 和 GTID，无法使用 xtrabackup 工具进行备份，现有数据库文件大小 114G，使用机械硬盘，初步计划将其迁移到 SSD 服务器上。

## 安装 MySQL
不管迁移方案如何，最先可以做的事是将 SSD 服务器初始化好，为了降低风险，需要在 SSD 服务器上安装 5.7.17 版本的数据库，5.7 的版本中最近的版本是 5.7.21。

Google 没有搜索到如何安装特定版本的 MySQL，查看 [官方文档](https://dev.mysql.com/doc/mysql-apt-repo-quick-guide/en/#repo-qg-apt-replace-direct) 经过实验，使用 MySQL deb 包安装可行，具体步骤：

1. 下载 MySQL [配置](https://dev.mysql.com/downloads/repo/apt/)，使用 `sudo dpkg -i /PATH/version-specific-package-name.deb` 进行配置，MySQL 版本选择 5.7。
2. 执行 `apt-get update`。
3. 执行 `apt-get build-dep mysql-server` 安装依赖。
4. 去 MySQL [下载页](https://downloads.mysql.com/archives/community/) 选择操作系统的版本和 MySQL 的版本并下载 DEB Bundle。
5. 解压后执行 `dpkg -i mysql-{common,community-client,client,community-server,server}_*.deb` 进行安装即可。

## 调优配置

### 修改字符集
修改 `/etc/mysql/conf.d/mysql.cnf` 添加：
```
[mysql]
default-character-set=utf8
```

修改 `/etc/mysql/mysql.conf.d/mysqld.cnf` 添加：
```
[mysqld]
character-set-server=utf8mb4
```

### 修改数据库目录
```
[mysqld]
datadir     = /data/mysql
log-error   = /data/log/error.log
```
修改 `/etc/apparmor.d/tunables/alias`，添加：
```
alias /var/lib/mysql/ -> /data/mysql/,
alias /var/log/mysql/ -> /data/log/,
```

### 修改监听 Ip
```
[mysqld]
bind-address    = 0.0.0.0
```

### innodb 调优
```
[mysqld]
innodb_buffer_pool_size=3G
innodb_buffer_pool_instances=4
innodb_buffer_pool_chunk_size=256M

innodb_flush_method=O_DIRECT

innodb_log_file_size=4G
innodb_io_capacity=1000
innodb_max_dirty_pages_pct=25
```

#### innodb_buffer_pool_size
这个是 innodb 调优最重要的参数，必须为 `innodb_buffer_pool_instances` * `innodb_buffer_pool_chunk_size=256M` 的倍数，例如上面的配置 `3G = 4 * 256 * 3`。

其他参数参考：[MySQL性能调优 – 你必须了解的15个重要变量](https://www.centos.bz/2016/11/mysql-performance-tuning-15-config-item/)

### 打开 GTID 和 Binlog
由于要配置主从同步，GTID 和 Binlog 必须要打开，配置如下：
```
[mysqld]
server_id = 3
gtid_mode = on
enforce_gtid_consistency = on

log-bin = mysqlbin
binlog_format = row
sync_binlog = 1
binlog-rows-query-log-events = 1

expire_logs_days = 14
max_binlog_size = 512M

skip_slave_start = 1
relay_log_recovery = 1
log-slave-updates = 1
```

*binlog 和数据文件最好放在不同的磁盘上，提高数据库磁盘的性能。*

#### server_id
保证每台 MySQL 的配置不一样，其他参数根据自身需要配置。

## Linux 运行环境调优
修改 CPU 运行模式，磁盘 IO 调度算法，关闭 swap 和最大文件数，硬盘挂载添加 noatime，nobarrier。

### 关闭 NUMA
Ubuntu 14.04 和 16.04 编辑 `/etc/default/grub`
```
GRUB_CMDLINE_LINUX_DEFAULT="numa=off"
```

### 修改磁盘 IO 调度
SSD 修改为 noop

### 修改磁盘 IO 调度
1. SSD 修改为 noop。
2. 机械硬盘修改为 deadline。

```
echo deadline > /sys/block/sda/queue/schedule
```
**注意硬盘命名是不是 sda**

永久修改：
```
GRUB_CMDLINE_LINUX_DEFAULT="elevator=noop numa=off"
```

### 关闭 swap
运行 `echo "vm.swappiness = 0" >>/etc/sysctl.conf` 并执行 `sysctl -p` 生效。

### 最大文件数
```
# maximum capability of system
user@ubuntu:~$ cat /proc/sys/fs/file-max
708444

# available limit
user@ubuntu:~$ ulimit -n
1024

# To increase the available limit to say 200000
user@ubuntu:~$ sudo vim /etc/sysctl.conf

# add the following line to it
fs.file-max = 200000

# run this to refresh with new config
user@ubuntu:~$ sudo sysctl -p

# edit the following file
user@ubuntu:~$ sudo vim /etc/security/limits.conf

# add following lines to it
* soft     nproc          200000
* hard     nproc          200000
* soft     nofile         200000
* hard     nofile         200000
root soft     nproc          200000
root hard     nproc          200000
root soft     nofile         200000
root hard     nofile         200000

# edit the following file
user@ubuntu:~$ sudo vim /etc/pam.d/common-session

# add this line to it
session required pam_limits.so

# logout and login and try the following command
user@ubuntu:~$ ulimit -n
200000
```

### 硬盘挂载添加 noatime，nobarrier
编辑 `/etc/fstab`，数据库文件所在硬盘修改为如下：
```
/dev/sda3   /data   ext4    defaults,noatime,nobarrier       0   0
```

## 参考资料
+ [MySQL性能调优 – 你必须了解的15个重要变量](https://www.centos.bz/2016/11/mysql-performance-tuning-15-config-item/)
+ [mysql性能调优的关键点](https://riverdba.github.io/2017/03/27/Performance-tuning-key-points-of-mysql/)
+ [MySQL InnoDB Buffer Pool](http://www.ywnds.com/?p=9886)
+ [LINUX上MYSQL优化三板斧](https://linux.cn/article-3479-1.html)
+ [SSD 下的 MySQL IO 优化尝试](https://linux.cn/article-6115-1.html#4_13597)
+ [MySQL 运行环境优化(Linux)](http://liaoph.com/mysql-optimize-in-linux/)
+ [GTID的主从复制的配置](http://www.cnblogs.com/abobo/p/4244059.html)
+ [Increate max no of open files limit in Ubuntu 16.04 for Nginx](https://gist.github.com/luckydev/b2a6ebe793aeacf50ff15331fb3b519d)
