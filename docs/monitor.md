# 监控相关

本地使用zabbix 3.4 来部署的监控，针对mysql的监控使用的percona来进行监控，percona提供的模板可以将mysql的大部分监控项全部列入。

## 大致步骤
1、zabbix-server端导入percona的mysql监控模板。  
2、zabbix-client安装percona模块。  
3、zabbix-server添加mysql的监控服务。

## 详细步骤
###zabbix-server端导入percona的mysql监控模板
 1、目前针对percona自带mysql监控模板只支持zabbix-2.x版本，针对3.0及以上版本percona自带mysql监控模板无法导入到zabbix-server。  
 2、针对zabbix-3.0以上版本，目前mysql模本配置文件可参考:
  	http://jaminzhang.github.io/soft-conf/Zabbix/zbx_percona_mysql_template.xml  
 3、本地下载后直接导入到zabbix-server。  
  1)打开zabbix-server 页面，点击配置-->模板-->导入

![gtid](https://img-blog.csdn.net/20171212142220342?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzE2MTMwNTU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![gtid](https://img-blog.csdn.net/20171212142302026?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzE2MTMwNTU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

###zabbix-client安装percona模块
1、首先percona运行需要依赖PHP环境，本地首先安装php： apt-get install php php-mysql  

2、登录percona官网下载安装包： https://www.percona.com/downloads/percona-monitoring-plugins/  
   根据操作系统版本来选择源安装包下载。  

3、下载源包后进行安装：  
   dpkg -i percona-zabbix-templates_1.x.x-1.trusty_all.deb  
   apt-get update  
   apt-get install percona-zabbix-templates  

4、安装完后,将percona的配置文件复制到zabbix-client的zabbix_agentd.d目录:    
   cp /var/lib/zabbix/percona/templates/userparameter_percona_mysql.conf /etc/zabbix/zabbix_agentd.d/userparameter_percona_mysql.conf  

5、配置mysql同clinet间的连接  
   1)修改配置文件:/var/lib/zabbix/percona/scripts/ss_get_mysql_stats.php  

```
$mysql_user = 'zabbixmysql';  
$mysql_pass = 'zabbixmysql';  
$mysql_port = 3306;  

```   
  2)修改脚本文件:/var/lib/zabbix/percona/scripts/get_mysql_stats_wrapper.sh

```

HOST=x.x.x.x  #(zabbix-client IP)  
DIR=`dirname $0`  
CMD="/usr/bin/php -q $DIR/ss_get_mysql_stats.php --host $HOST --items gg"  
CACHEFILE="/tmp/$HOST-mysql_cacti_stats.txt"  

if [ "$ITEM" = "running-slave" ]; then  
    # Check for running slave  
    RES=`HOME=~/usr/bin/mysql -e 'SHOW SLAVE STATUS\G' | egrep '(Slave_IO_Running|Slave_SQL_Running):' | awk -F: '{print $2}' | tr '\n' ','`  

```

6、测试连接是否正常。
 
```
[root@centos6 main]# /var/lib/zabbix/percona/scripts/get_mysql_stats_wrapper.sh gg  
66
#(此处返回数字正常）
```

7、重启zabbix-client。
  service zabbix-agent restart

###zabbix-server添加mysql的监控服务

在zabbix hosts里面的templates里面添加percona mysql 的连接，就可以加入percona mysql模版监控了  

![gtid](https://img-blog.csdn.net/20160518214820622?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

然后去监控界面查看监控效果图  

![gtid](https://img-blog.csdn.net/20160518214832989?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


