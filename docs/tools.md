# 常用工具

## pt-mysql-summary
[pt-mysql-summary](https://www.percona.com/doc/percona-toolkit/3.0/pt-mysql-summary.html) 可以非常友好的输出 MySQL 的一些情况，我用它来关注 MySQL Innodb buffer pool 的使用情况。

如图：

![innodb buffer](http://cdn.defcoding.com/FA21F466-2039-4B81-AE1A-B862873F0426.png)

## pt-query-digest
[pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html) 用于分析 MySQL 的慢查询日志。

## MySQLTuner-perl
[MySQLTuner-perl](https://github.com/major/MySQLTuner-perl/) 可以对当前的运行的 MySQL 提供优化建议，对于新手来说帮助挺大的。

### 安装
`git clone https://github.com/major/MySQLTuner-perl.git`

### 使用
`perl mysqltuner.pl --host targetDNS_IP --user admin_user --pass admin_password`

内存信息：

![Performance Metrics](http://cdn.defcoding.com/862A5046-1D0B-4E0A-B765-8F62EF617F48.png)

优化建议：

![Recommendations](http://cdn.defcoding.com/6A397F1F-B59F-4762-9149-DB46C140EF4D.png)
