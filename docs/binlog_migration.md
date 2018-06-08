# mysql binlog目录迁移

为了节省ssd资源，将二进制文件binlog目录迁移到普通硬盘目录。

#### 手动方式

​	手动将binlog和binlog index迁移到新的目录下，并且更新binlog index文件定义的binlog位置，采用绝对路径。

#### 使用工具[mysqlbinlogmove](https://dev.mysql.com/doc/mysql-utilities/1.6/en/mysqlbinlogmove.html)

1. 安装[ MySQL Utilities](https://dev.mysql.com/doc/mysql-utilities/1.6/en/mysql-utils-install.html)

   [下载地址](https://downloads.mysql.com/archives/utilities/) 下载时注意下载文件名是否正确，个人遇到过筛选返回了MySQL Utilities以外的deb文件。

2. 迁移binlog：

   1. mysql运行状态下：mysqlbinlogmove --server=user:pass@localhost:3306 target_dir

      【注意】据官方文档描述，这种方式下正在使用的binlog无法迁移

   2. mysql关闭状态下：mysqlbinlogmove --binlog-dir=source_dir target_dir

      【注意】若binlog的basename非默认，需要定义--bin-log-basename参数，即： mysqlbinlogmove  --bin-log-basename=your_basename  --binlog-dir=target_dir。

3. 迁移binlog index：

由于上面仅迁移了binlog，binlog index还留在source目录下。

若binlog 与binlog index分居两地，则需要在mysql conf下配置参数bin-log-index=$target。如果希望它们待在一起，可以选择手动移动到target目录下。

1. 重启mysql若报错，则在/etc/apparmor.d/usr.sbin.mysqld 新增对target目录的安全权限配置：

```
  /backup/mysql_binlog/ r,
  /backup/mysql_binlog/** rw,
```