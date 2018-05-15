# 搭建从库

GTID 和 binlog 已经开启，备份方案也搞定，下面就开始搭建从库了，搭建从库过程不难，只是有些注意点。

## 大致步骤
1. 搭建 MySQL，查看 [安装特定版本的 MySQL 并调优](SETUP.md)。
2. 使用 Xtrabackup 备份数据并且 prepare。
3. 记录备份文件 `/data/mysql/xtrabackup_binlog_info` 中的 GTID。
4. copy 备份到从库所在服务器并放置到 MySQL 所在目录。
5. 打开 MySQL shell 设置主库信息并启动从库复制。

## 详细步骤

### 配置 MySQL
server_id 需要和主库不同。

### 使用 Xtrabackup 备份并 prepare 数据
查看 * [备份 MySQL 数据](backup.md) 中的步骤。

### 查看 GTID
打开 `/data/mysql/xtrabackup_binlog_info` 查看 GTID，如图：

![gtid](http://cdn.defcoding.com/1E67D837-0B5D-4C20-94DD-329A28DE1D09.png)

`gtid` 号：`1175089b-bdf0-11e7-9694-0242ac110002:1-804216`

### 恢复备份到从库中
使用 [迁移 MySQL](migration.md) 的方法迁移备份数据。

### 设置从库
主库上线创建账号和设置权限
```
create user 'ReplicaUser'@'%' identified by 'xxxxxx';
grant replication slave on *.* to 'ReplicaUser'@'%' identified by 'xxxxxx';
flush privileges;
```

从库上设置
```
reset master;
set global gtid_purged='1175089b-bdf0-11e7-9694-0242ac110002:1-804216';

change master to
master_host='master ip',
master_user='ReplicaUser,
master_password='xxxxxx',
master_auto_position=1;

# 开启复制
start slave;

# 查看从库状态
show slave status;
```

![slave status](http://cdn.defcoding.com/EA83BB82-046D-434E-948F-28902823B220.png)

## 参考
+ [MySQL 5.7基于GTID的主从复制实践](https://www.hi-linux.com/posts/47176.html)
+ [GTID的主从复制的配置](http://www.cnblogs.com/abobo/p/4244059.html)
+ [Mysql -- GTID主从方案的学习](https://340starobserver.github.io/2017/03/15/Mysql-Repliset/)
