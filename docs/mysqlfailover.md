# 高可用方案

## 方案迷思
Google 一搜，MySQL 的高可用方案有 5 种之多，我们需要结合自身业务选择最合适的方案，我们的业务量不是很大，开发运维人员也不多，所以我们要保证方案尽可能简单易懂，实际调研下来，使用 keepalived + MySQL 双主方案是最简单的。



## MySQL双主模式

可应用于生产环境的双主模式搭建，整个过程中MySQL服务持续无须中断。

#### 原理

​	双节点相互复制，互为主库Master，理论上两个节点都支持写入，但是需要为潜在的数据冲突做配置。为了降低复杂性，我们采用了同时仅允许一个节点写入的模式。

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

5. 完成后可查看下A的slave状态：

   ```
   show slave status\G
   ```



## keepalived搭配

#### 原理

​	[keepalived和vrrp协议](https://liangshuang.name/2017/11/16/keepalived/)

#### [安装](http://www.keepalived.org/doc/installing_keepalived.html)

#### 配置

1. 配置/etc/keepalived/keepalived.conf

   其中vip表示虚拟IP，rip表示真实IP，其它参数意义可参考[官方](http://www.keepalived.org/doc/configuration_synopsis.html)

   ```
   global_defs {
      router_id mysqld-node1
   }

   vrrp_instance VI_1 {
       state BACKUP
       interface ens160
       virtual_router_id 51
       priority 100
       advert_int 1
       nopreempt
       authentication {
           auth_type PASS
           auth_pass 12344321
       }
       virtual_ipaddress {
           vip
       }
   }

   virtual_server vip 3306 {
       delay_loop 2
       lb_algo rr
       lb_kind NAT
       persistence_timeout 50
       protocol TCP

       real_server rip 3306 {
           weight 1
       notify_down /etc/keepalived/mysql.sh
       TCP_CHECK{
               connect_timeout 3
               retry 3
               delay_before_retry 3
               connect_port 3306
           }
       }
   }
   ```

   为了实现故障自动转移，所以节点都需要设置为BACKUP。若需要默认设置某节点为主节点，可将其priority设置高于其它节点，这样在所有节点启动时会自动选出优先级最高的节点作为主节点。

   但考虑到故障节点恢复后的数据时差和可能存在的冲突，我们希望它不会由于优先级更高而被自动推举为主节点，因此设置nopreempt允许低优先级节点作为主节点。这种情况下，如果希望手动恢复故障节点为主节点，可将nopreempt暂时去掉，若其优先级更高则会自动接管为主节点。之后再加上nopreempt，并reload。

2. keepalived自杀脚本：/etc/keepalived/mysql.sh

   ```
   #!/bin/bash
   pkill keepalived
   ```

#### 调试

1. 守护进程启动：

   ```
   keepalived
   ```

	keepalived默认配置可参考[官网](http://www.keepalived.org/doc/programs_synopsis.html)。若需要打印具体的配置信息和错误，可加上参数-d（Dump the configuration data）。启动后可查看日志（默认记入/var/log/syslog）确认启动是否成功。

2. 检查vip：

   ```
   ip address show ens160
   ```

	若在配置的网络设备下出现新增的vip，则表示成功。

3. 节点间通讯：

   所有节点配置/etc/iptables.rules

   ```
   -A INPUT -p vrrp -j ACCEPT
   ```

   所有节点启动keepalived守护进程，并查看vip是否在其中一个节点正常注册，若出现多个节点同时存在注册的vip，可能是由于节点间的vrrp不通造成的。

#### systemctl服务配置

关于linux服务管理的更新换代可[参考](https://wizardforcel.gitbooks.io/vbird-linux-basic-4e/content/148.html)。这里ubuntu版本为16.04，故采用更新的systemctl方式管理daemon服务。

1. 配置/etc/systemd/system/keepalived.service

   ```
   #
   # keepalived control files for systemd
   #
   # Incorporates fixes from RedHat bug #769726.

   [Unit]
   Description=LVS and VRRP High Availability monitor
   After=network.target
   ConditionFileNotEmpty=/etc/keepalived/keepalived.conf

   [Service]
   Type=simple
   # Ubuntu/Debian convention:
   EnvironmentFile=/etc/default/keepalived
   ExecStart=/usr/local/sbin/keepalived --dont-fork
   ExecReload=/bin/kill -s HUP $MAINPID
   # keepalived needs to be in charge of killing its own children.
   KillMode=process

   [Install]
   WantedBy=multi-user.target
   ```

2. 配置开机启动

   ```
   systemctl enable keepalived
   ```

3. 服务启动&关闭&状态查询

   ```
   systemctl start keepalived
   systemctl stop keepalived
   systemctl status keepalived
   ```

#### 测试

mysq访问地址改为vip前，可模拟测试下mysql故障转移是否生效。

关闭主节点mysql服务。查看以下项目若均实现则表示成功：

​	主节点vip被注销，且keepalived进程被杀死；

​	备用节点vip被注册，升级为新主节点；

​	局域网其他节点查看地址解析缓存（命令arp -a）的vip解析为新主节点；

【注意】由于本文基于A节点服务不中断的前提，因此故障转移测试节点均为新增的B节点。

#### 报警

待完善...