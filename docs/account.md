# MySQL 账号管理

## 背景
目前，我们使用 MySQL 都只用了 root 账号，为了安全起见，应该对每种业务场景建立相应的账号并控制好权限，场景如下：

1. Xtrabackup 备份和恢复账号。
2. 主从同步账号。
3. 应用系统访问数据库账号。

另外，修改 Root 账号密码，并且慎用 Root 账户。

## MySQL 账户管理

创建账号
```
CREATE USER 'backup'@'localhost' IDENTIFIED BY 'xxxxx';
```

授权
```
GRANT RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'bkpuser'@'localhost';
FLUSH PRIVILEGES;
```

修改密码
```
ALTER USER 'root'@'localhost' IDENTIFIED BY 'password';
```

删除账户
```
DROP USER 'jeffrey'@'localhost';
```

查看所有账户
```
SELECT USER, HOST FROM mysql.user;
```

查看账户授权
```
show grants for'cactiuser'@'%';
```

修改账号
```
RENAME USER 'jeffrey'@'localhost' TO 'jeff'@'127.0.0.1';
```

### 创建 Xtrabackup 全量备份账户
根据 [Xtrabackup 文档](https://www.percona.com/doc/percona-xtrabackup/LATEST/innobackupex/privileges.html#permissions-and-privileges-needed)，如图：

![xtrabackup](http://cdn.defcoding.com/36F3B40B-39C5-4F0D-9899-B77BA1039030.png)
```
mysql> CREATE USER 'bkpuser'@'localhost' IDENTIFIED BY 's3cret';
mysql> GRANT RELOAD, LOCK TABLES, REPLICATION CLIENT, PROCESS ON *.* TO 'bkpuser'@'localhost';
mysql> FLUSH PRIVILEGES;
```

### 主从同步账号创建
```
create user 'ReplicaUser'@'%' identified by 'xxxxxx';
grant replication slave on *.* to 'ReplicaUser'@'%' identified by 'xxxxxx';
flush privileges;
```

### 应用读写数据库账号创建
```
create user 'youruser'@'%' identified by 'xxxxxx';
grant all privileges on yourdb.* to 'youruser'@'%' identified by 'xxxxxx';
flush privileges;
```

### zabbix 监控账号创建
```
create user 'monitor'@'localhost' identified by 'xxxxxx';
grant SELECT,PROCESS,SUPER,REPLICATION CLIENT on *.* to 'monitor'@'localhost' identified by 'xxxxxx';
flush privileges;
```

### 修改 root 密码
```
ALTER USER 'root'@'%' IDENTIFIED BY 'password';
```

## 总结
为了提高安全性，创建新账号时最好限定访问 IP 地址，不要使用 '%'，例如：

1. 限定单个 IP：`CREATE USER 'foo'@'192.168.1.1' IDENTIFIED BY 'xxxxx';`
2. 限定网段：`CREATE USER 'foo'@'192.168.1.%' IDENTIFIED BY 'xxxxx';`

## 参考
+ [Connection and Privileges Needed](https://www.percona.com/doc/percona-xtrabackup/LATEST/innobackupex/privileges.html#permissions-and-privileges-needed)
+ [MySQL User Account Management](https://dev.mysql.com/doc/refman/5.7/en/user-account-management.html)
