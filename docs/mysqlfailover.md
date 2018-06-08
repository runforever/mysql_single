# 高可用方案

## 方案迷思
Google 一搜，MySQL 的高可用方案有 5 种之多，我们需要结合自身业务选择最合适的方案，我们的业务量不是很大，开发运维人员也不多，所以我们要保证方案尽可能简单易懂，实际调研下来，使用 keepalived + MySQL 双主方案是最简单的。



## MySQL双主模式

生产环境的双主模式搭建，整个过程中MySQL服务持续未中断。

#### 前提

​	生产环境MySQL服务节点（别名A）开启了binlog和GTID服务

#### 准备步骤

​	基于运行中的A节点数据搭建从库节点（别名B）（参考“搭建从库”章节）

#### 双主配置

1. 检查和保证一致性：鉴于B为A的从库，理论上只要节点B没有新数据写入，和节点A的一致性是有保证的。若需要可以利用工具[pt-table-checksum](https://www.percona.com/doc/percona-toolkit/LATEST/pt-table-checksum.html)检查主从一致性。

   【唧唧歪歪】有文章指出可以利用两个从库检测一致性（参考http://keithlan.github.io/2016/05/25/pt_table_checksum/），但是pt-table-checksum基于复制关系做一致性检测（默认使用本地库），没有找到指定非主从关系的两个库做检测的方法？

2. 若双主节点都可能涉及写入，最好各自配置数据的ID自增步长和移位，降低数据非一致风险。

   自增步长：auto_increment_increment

   移位：auto_increment_offset

   运行中的服务可以在shell下设置，A节点：

   ```
   set global auto_increment_increment=2;
   set global auto_increment_offset=1;
   ```

   B节点：

   ```
   set global auto_increment_increment=2;
   set global auto_increment_offset=2;
   ```

3. 创建复制用户：

   若节点之间已建立复制关系则利用已存在的复制用户即可，否则需要在两个节点分别创建用户

   ```
   create user 'replicate'@'%' identified by 'user_pass'
   ```

4. 配置双节点互为从库：

   由于B已经为A的从库，故仅需将A配置为B的从库：

   ```
   change master to master_host='B_host',master_user='replicate',master_password='user_pass',master_auto_position=1;
   start slave;
   ```

完成后可查看下A的slave状态：

```
show slave status\G
```



## keepalived搭建和配置

安装

配置

测试