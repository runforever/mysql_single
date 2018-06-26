# mysql监控方案

## 需求背景

由于mysql调优的需要，希望以更直观的方式收集和展示性能指标，并支持数据自定义和设置报警，以便于分析和快速定位问题，故以此为前提考虑部署一套mysql监控系统。



## 监控系统调研

运维方面有搭建zabbix，并由Percona Monitoring Plugins获取到mysql部分指标数据（据反映数据不全），但图形展示不友好且网上未找到现成的解决方案，目测学习成本较高，难以低成本达到理想效果。同时，团队已存在一个基于grafana+graphite+statsD的[监控服务](http://10.9.255.71:82)，部署方式采用现成的第三方[docker镜像](https://github.com/kamon-io/docker-grafana-graphite)。该监控系统基本满足了对django REST web后端服务的访问数据采集和展示，虽然还存在一些性能问题（见x.x），但理论上可以基于这套系统做到mysql监控。



###方案A：基于现有的grafana+graphite+statsd系统

1. graphite源码为python，数据收集开发比较简单，对于团队的技术栈比较友好

2. mysql 数据收集可选用第三方[mysql-statsd包](https://github.com/db-art/mysql-statsd)，但最新更新时间为二年前，且文档说明仅支持mysql5.5，而我们的mysql为5.7

3. grafana+graphite+statsd系统对硬件写入性能要求较高（且iops随着retention的调整增高，默认为10s:7d，改为10s:1d, 1m:30d后也会增长，目前原因不明，猜测iops同时受到采样间隙，数据文件大小和采样间隙阶梯数目的影响）

4. grafana+graphite+statsd方案比较古老，且优势在于水平扩展方案和历史数据留存周期，对于我们来说没有明显的吸引力（参考见底部）

5. 第三方dashboard也很古老和简陋，例如[Grafana dashboards for measuring MySQL performance with Graphite](https://grafana.com/dashboards/235)

   ​

### 方案B：搭建新的监控系统

1. 参考方案：Prometheus，InfluxDB，OpenTSDB，PMM（Percona Monitoring and Management）
2. Prometheus VS InfluxDB VS OpenTSDB：目测prometheus最新，一站式平台功能完备，且文档丰富新鲜（参考见底部）；
3. [PMM（Percona Monitoring and Management）](https://www.percona.com/doc/percona-monitoring-and-management/architecture.html)由prometheus和QAN system组成，QAN（Query Analytics）用于收集和分析mysql查询语句性能Metrics，经评估对我们当前mysql调优帮助非必需，故略过。
4. 邻近团队有在开始使用prometheus部分功能，目测顺畅；
5. 由于比较熟悉且使用感良好继续沿用grafana，完整方案为grafana+prometheus；
6. 调研是否存在现成的性能调优参考图标，目前最全的还是[Grafana dashboards for MySQL and MongoDB monitoring using Prometheus](https://github.com/percona/grafana-dashboards)，但是都没有明确的性能指标建议，但似乎支持自定义[阈值线](https://github.com/grafana/grafana/issues/9084)




##grafana+prometheus搭建

可选择第三方集成的[docker镜像](https://github.com/vegasbrianc/prometheus)，但从长远考虑为了熟悉系统选择了从零开始搭建。



###创建新项目

```
mkdir /opt/grafana-prometheus
cd /opt/grafana-prometheus
```



###配置docker-compose.yml

images：[prometheus](https://hub.docker.com/r/prom/prometheus/) & [grafana](https://hub.docker.com/r/grafana/grafana/)

docker-compose.yml demo：

```
version: '3'
services:
  prometheus:
    image: prom/prometheus:v2.2.1
    container_name: prom-prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - '9090:9090'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana-grafana
    environment:
        - GF_SECURITY_ADMIN_PASSWORD=${admin_pass}
    volumes:
      - ./grafana/data:/var/lib/grafana
      - ./grafana/grafana.ini:/etc/grafana/grafana.ini
    depends_on:
      - prometheus
    ports:
      - '3000:3000'
```



### 配置prometheus准备

prometheus/prometheus.yml 默认配置如下：

```
global:
  scrape_interval: 5s
  external_labels:
    monitor: 'default-monitor'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

后续新增exporter后需要更新该配置。



### 配置grafana

启动grafana和prometheus服务：

```
docker-compose up -d
```

浏览器访问：http://localhost:3000

登陆用户：admin/${GF_SECURITY_ADMIN_PASSWORD}，若未特别定义GF_SECURITY_ADMIN_PASSWORD默认为admin

配置数据源：http://localhost:3000/datasources

​	其中Type选择Prometheus，URL由于docker-compose.yml配置过depends_on可填写sevice name。

​	![image-20180524110122458](/var/folders/pp/sbc9hlss70bc4h_xl8xg0nz40000gn/T/abnerworks.Typora/image-20180524110122458.png)

导入dashboards：http://localhost:3000/dashboard/import

​	可通过官方grafana.com提供的[dashboard模板](https://grafana.com/dashboards?dataSource=prometheus)的url或id，或上传json文件的方式导入dashboard。也可采用第三方提供的dashboard，例如percona的[Grafana dashboards for MySQL and MongoDB monitoring using Prometheus](https://github.com/percona/grafana-dashboards)。

​	【注意】percona提供的Install dashboards方法，亲测无法生效，原因不明。grafana官方也没有提供类似的批量导入方案，故依照官方手册一个个手动导入的json文件。



### 安装和配置Exporter

#### mysql_exporter

安装：

​	[docker image方式](https://hub.docker.com/r/prom/mysqld-exporter/)

​	[二进制包方式](https://github.com/prometheus/mysqld_exporter/releases)

​	[github源码方式](https://github.com/prometheus/mysqld_exporter)

​	由于mysql生产环境经过调优cpu模式不支持docker服务，故未采用docker方式。而非docker方式依赖golang环境，安装参考[golang install](https://golang.org/doc/install)。

​	若采用二进制包方式，下载解压即可。若采用源码生产二进制文件则较麻烦：

​		git clone ${remote_repo}

​		make

​			golang官方步骤为直接make，但是根据Makefile*来看它只会进行语法检测，并不会下载依赖包（懵逼），故按照如下流程进行的安装：

​				下载依赖：go get -d ./...

​				下载测试依赖：go get -t ./…

​				万恶的GFW造成部分包无法顺利下载，可以找到相应包的github地址并下载

​				若go vet ./…通过，则执行make，否则处理错误直至通过

​				若make进行到staticcheck步骤报错SA1019（推测原因为mysql exporter的某行代码语法过时，官方建议更新，但makefile中有指明忽略这个错误，不明白为什么还会提示），可到Makefile*中去掉staticcheck继续进行



配置：

​	mysql创建专用监控账号：

```
mysql> CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'XXXXXXXX' WITH MAX_USER_CONNECTIONS 3;
mysql> GRANT PROCESS, REPLICATION CLIENT, SELECT ON . TO 'exporter'@'localhost';
```

​	配置环境变量：

​		tcp方式：

​			export DATA_SOURCE_NAME='exporter:XXXXXXXX@tcp(localhost:3306)/${db_name}'

​		Unix domain sockets方式：

​			export DATA_SOURCE_NAME='exporter:XXXXXXXX@unix(XXX/mysqld.sock)/'

​		生产环境实测无法以tcp的方式访问localhost，推测原因为tcp方式需要将localhost解析成127.0.0.1，而我们的mysql禁用了dns解析（skip-name-resolve = ON）。



启动：./mysqld_exporter <flags>

​	导入的dashboard可能需要一些mysqld_exporter的非默认参数数据，这时需要在启动时配置flags，例如：./mysqld_exporter —collect.binlog_size（参考 https://github.com/prometheus/mysqld_exporter 的Collector Flags）



####  node_exporter

安装：

​	选择相应版本的二进制包：https://github.com/prometheus/node_exporter/releases

​	tar -xzf node_exporter-0.16.0.linux-amd64.tar.gz

​	cd node_exporter-0.16.0.linux-amd64/

启动：./node_exporter

【注意】最新的node_exporter0.16.0，对Metric name指标名定义做了大范围更新，目前没有相配套的dashboards，需要自行替换更新（参考 https://github.com/prometheus/node_exporter/issues/830 ）。如果嫌麻烦且不介意指标数据出现定义断层，可以选择下载上个版本0.15.0，并等待需要的第三方dashboards提供兼容版本后再一起更新。



## 配置prometheus

确认数据采集源exporter后可在scrape_configs下新增job，例如配置mysql指标和myql服务器指标后如下：

```
global:
  scrape_interval: 5s
  external_labels:
    monitor: 'default-monitor'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'mysql'
    scrape_interval: 10s
    static_configs:
      - targets: ['${node_a}:9104', '${node_a}:9104']

  - job_name: 'node'
    scrape_interval: 10s
    static_configs:
      - targets: ['${node_b}:9104', '${node_b}:9104']
```



##  问题

来自同一台服务器但不同instance（也就是ip相同，exporter端口不同）的Metrics的图表配置还存在问题，例如dashboard "MySQL Overview"的" I/O Activity"图表。



## 报警通知

采用Prometheus配套的报警管理组件Alertmanager。介绍和详细配置参考[官方报警概述](https://prometheus.io/docs/alerting/overview/)。

其中prometheus根据rules发送报警数据给alertmanager，而后alertmanager根据配置规则处理报警，例如发出通知。

alertmanager安装及配置

docker-compose.yml

```
version: '3'
services:
  ...
  alertmanager:
    image: prom/alertmanager
    container_name: prom-alertmanager
    ports:
      - 9093:9093
    volumes:
      - ./alertmanager/:/etc/alertmanager/
    restart: always
    command:
      - '--config.file=/etc/alertmanager/config.yml'
      - '--storage.path=/alertmanager'
```

alertmanager/config.yml

```
global:
  resolve_timeout: 5m
  smtp_from: from_email
  smtp_smarthost: smtp_sever:port
  smtp_auth_username: smtp_user
  smtp_auth_password: smtp_passwd

route:
  receiver: default
  group_wait: 5s
  group_interval: 10s
  repeat_interval: 20s
  group_by: [alertname]

receivers:
- name: default
  email_configs:
  - to : to_email
    send_resolved: true
```

prometheus与alertmanager通讯配置

prometheus/prometheus.yml

```
...

rule_files:
    - /etc/prometheus/rules.yml

alerting:
  alertmanagers:
    - static_configs:
      - targets: ["alertmanager:9093"]
```

docker-compose.yml

```
version: '3'
services:
  ...
  prometheus:
    ...
    depends_on:
      - alertmanager
```

prometheus报警规则配置

prometheus/rules.yml

```
groups:
- name: test-rule
  rules:
  - alert: MysqlServiceDown
    expr: mysql_up == 0
    for: 10s
    labels:
      team: mysql
      severity: page
    annotations:
      summary: "{{ $labels.instance }}: Service is down"
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 2 minutes."
```

docker-compose.yml

```
version: '3'
services:
  ...
  prometheus:
    ...
    volumes:
      ...
      - ./prometheus/rules.yml:/etc/prometheus/rules.yml
    ...
```

启动alertmanager，重新生成container并重启prometheus，即可。

如需测试，可以通过修改rules.yml的expr条件，或手动向alertmanager发送报警数据

```
curl -H "Content-Type: application/json" -d '[{"labels":{"alertname":"TestAlert"}}]' localhost:9093/api/v1/alerts
```

【注意】

1. 若prometheus成功载入rules，可在prometheus的[alters页面](http://localhost:9090/alerts)下看见载入规则相关信息。否则可能是rules文件没有找到（prometheus不会因此报错），处理参考[这里](https://github.com/prometheus/prometheus/issues/2406)
2. 启动prometheus时若报错“Opening storage failed mkdir xxx: permission denied”，处理参考[这里](https://github.com/coreos/prometheus-operator/issues/830)



## 参考

​	[时序数据库比较](http://liubin.org/blog/2016/02/25/tsdb-list-part-1/)

​	[Prometheus VS 其它监控系统](https://prometheus.io/docs/introduction/comparison/)

​	[Prometheus Getting Started](https://prometheus.io/docs/prometheus/latest/getting_started/)

​	[Graphing MySQL performance with Prometheus and Grafana](https://www.percona.com/blog/2016/02/29/graphing-mysql-performance-with-prometheus-and-grafana/)

​	[Prometheus Related Download](https://prometheus.io/download/)