# 备份方案

## 方案迷思
1. 使用 mysqldump。
2. 使用 xtrabackup。
3. 使用 mydumper。
4. 搭建从库并且设置延时 3 个小时同步主库。

## 背景
数据库文件有 113G，做表 migrate 的时候需要先备份数据库，要求尽快备份完成。

## mysqldump
mysqldump 适合数据量比较小的时候使用，会锁表，pass。

## mydumper
不祥

## 搭建从库备份
搭建时间较长，考虑以后做。

## xtrabackup
支持热备份，不锁表，支持增量备份，就它了。

### 安装
根据 [安装文档](https://www.percona.com/doc/percona-xtrabackup/LATEST/installation.html) 将工具安装好。

### 全量备份
执行一次全量备份，并且将备份压缩
```
xtrabackup --user=youruser --password=yourpassword --backup --compress --compress-threads=6 --target-dir=/backup/compressd/1
```

### 恢复备份
解压
```
xtrabackup --user=youruser --password=yourpassword --parallel=6 --decompress --target-dir=/backup/compressd/1
```

### 准备备份
```
xtrabackup --user=youruser --password=yourpassword --prepare --target-dir=/backup/compressd/1
```

### 恢复备份
```
xtrabackup --user=youruser --password=yourpassword --copy-back --target-dir=/backup/compressd/1
```

## 当前环境的方案选择
全量备份 5 分钟左右，时间可以接受，暂时不考虑引入增量备份来提高复杂度，将备份命令写成脚本，每天凌晨执行一次全量备份，备份保留时间为 14 天。

## 参考
[Percona XtraBackup - Documentation](https://www.percona.com/doc/percona-xtrabackup/LATEST/index.html)
