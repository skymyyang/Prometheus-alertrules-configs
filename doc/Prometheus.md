## Prometheus整体概述



### 概述

Prometheus(由go语言(golang)开发)是一套开源的监控&报警&时间序列数据库的组合。适合监控docker容器。因为kubernetes(俗称k8s)的流行带动了prometheus的发展。

Prometheus算是一个全能型选手，原生支持容器监控，也支持传统应用的监控。所有监控系统的基本流程，数据采集--->数据处理--->数据存储--->数据展示--->告警。

### 特点

- 多维数据模型：由度量名称和键值对标识的时间序列数据
- PromSQL: —种灵活的查询语言，可以利用多维数据完成复杂的查询
- 不依赖分布式存储，单个服务器节点可直接工作
- 基于HTTP的pull方式釆集时间序列数据
- 推送时间序列数据通过PushGateway组件支持
- 通过服务发现或静态配罝发现目标
- 多种图形模式及仪表盘支持(grafana)



### 任务分析

1. 为什么需要监控？

   进行实时的数据收集，通过报警及时发现问题，并进行处理。收集的数据为优化也可以提供依据。

2. 监控的对象

   - 主机状态，操作系统
   - 服务，应用
   - 资源 CPU,内存，硬盘
   - url 以及端口

3. 用什么进行监控   Prometheus， node_exporter,mysqld_exporter,blackbox_exporter；  zabbix-server zabbix-agent 等。
4. 什么时间监控      7X24
5. 报警给谁              管理员



### Prometheus的组成与架构



| 名称              | 说明                                       |
| ----------------- | ------------------------------------------ |
| Prometheus Server | 收集指标和存储时间序列数据，并提供查询接口 |
| Push Gateway      | 短期存储指标数据，主要用于临时性任务       |
| Exporters         | 采集已有的三方服务监控指标并暴露metrics    |
| Alertmanager      | 告警组件                                   |
| Web UI            | 简单的WEB控制台                            |



Prometheus官网架构图

![](https://prometheus.io/assets/architecture.png)

集成了数据的采集，处理，存储，展示，告警一系列流程都已经具备了。

其大概的工作流程是：

1. Prometheus server 定期从配置好的 jobs 或者 exporters 中拉 metrics，或者接收来自 Pushgateway 发过来的 metrics，或者从其他的 Prometheus server 中拉 metrics。
2. Prometheus server 在本地存储收集到的 metrics，并运行已定义好的 alert.rules，记录新的时间序列或者向 Alertmanager 推送警报。
3. Alertmanager 根据配置文件，对接收到的警报进行处理，发出告警。
4. 在图形界面中，可视化采集数据。



### Prometheus数据模型

Prometheus将所有的数据存储为时间序列，具有相同度量名称以及标签的属于同个指标。

Prometheus从数据源拿到数据之后都会存到内置的TSDB数据库中，这里存储的就是时间序列数据，它存储的数据会有一个度量名称，譬如你现在监控一个nginx，首先你要给他起个名字，这个名称就是度量名，还会有N个标签，你可以理解为表名，标签为字段，所以每个时间序列都由度量标准名称和一组键值对作为唯一标识。



时间序列的格式如下：

```
<metrice name> {<label name>=<label value>,...}
```

`metrice name`指的就是度量标准名称，`label name`也就是标签名，这个标签可以有多个:

```
nginx_http_access{method="GET",uri="/index.html"}
```

这个度量名称为`nginx_http_access`，后面是两个标签，和他们各对应的值，当然你还可以继续指定标签，你指定的标签越多查询的维度就越多。



### Prometheus指标类型



| 类型名称  | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| Counter   | 递增计数器，适合收集接口请求次数                             |
| Guage     | 可以任意变化的数值，适用CPU使用率                            |
| Histogram | 对一段时间内数据进行采集，并对有所数值求和于统计数量,可以对观察结果采样，分组及统计 |
| Summary   | 与Histogram类型类似，典型的应用如：请求持续时间，响应大小,提供观测值的 count 和 sum 功能,提供百分位的功能，即可以按百分比划分跟踪结果 |



### Prometheus安装部署

这里我们先从二进制部署入手，先监控传统型应用。

Prometheus下载地址：`https://github.com/prometheus/prometheus/releases`

找到适合自己平台的release版本，下载即可。

```bash
cd /usr/local/
export VER="2.21.0"
wget https://github.com/prometheus/prometheus/releases/download/v${VER}/prometheus-${VER}.linux-amd64.tar.gz
ln -s prometheus-2.21.0.linux-amd64 prometheus
echo "PATH=/usr/local/prometheus/bin:$PATH:$HOME/bin" >> /etc/profile
source /etc/profile
```

prometheus.yml就是他的配置文件。

启动项有很多，主要是以下两点。

```
--config.file=/usr/local/prometheus/config/prometheus.yml #指定配置文件位置
--storage.tsdb.path=/usr/local/prometheus/data   #数据存储目录
--storage.tsdb.retention=60d  #数据存储时间，默认是15天
```

TSDB不太适合长期去存储数据，数据量大了支持并不是很好，这里其实是可以引入外部存储的。譬如说使用InfluxDB.

 写入开机启动项

```bash
vim /usr/lib/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/
After=network.target


[Service]
Type=simple
ExecStart=/usr/local/prometheus/bin/prometheus --config.file=/usr/local/prometheus/prometheus.yml --storage.tsdb.path=/usr/local/prometheus/data --storage.tsdb.retention=60d
Restart=on-failure


[Install]
WantedBy=multi-user.target

systemctl daemon-reload && systemctl start prometheus.service
```

这样就启动成功了，去访问`http://yourip:9090`就可以看到Prometheus的webui界面了。



### Prometheus配置文件详解

prometheus.yml，官方说地址：`https://prometheus.io/docs/prometheus/latest/configuration/configuration/`

```yaml
global:
  [ scrape_interval: <duration> | default = 1m ]      ##采集间隔
  [ scrape_timeout: <duration> | default = 10s ]      ##采集超时时间
  [ evaluation_interval: <duration> | default = 1m ]  ##告警评估周期
  external_labels:                                    ##外部标签             
    [ <labelname>: <labelvalue> ... ]
```

指定告警规则

```yaml
rule_files:
  [ - <filepath_glob> ... ]
```

配置被监控端

```yaml
scrape_configs:
  [ - <scrape_config> ... ]
```

配置告警方式

```yaml
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]
```

指定远程存储

```yaml
remote_write:
  [ - <remote_write> ... ]
remote_read:
  [ - <remote_read> ... ]
```



1. scape_configs

   这里就是我们需要监控的内容，以下是我们所需要用到的常用配置

   ```yaml
   job_name: <job_name>  ##指定job名字
   [ scrape_interval: <duration> | default = <global_config.scrape_interval> ]
   [ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]  ##这两段指定采集时间，默认继承全局
   [ metrics_path: <path> | default = /metrics ]  ##metrics路径，默认metrics
   [ honor_labels: <boolean> | default = false ]  ##默认附加的标签，默认不覆盖
   [ scheme: <scheme> | default = http ]  ## 默认使用http方式去访问
   params:
     [ <string>: [<string>, ...] ]        ## 配置访问时携带的参数
   basic_auth:
     [ username: <string> ]
     [ password: <secret> ]
     [ password_file: <string> ]          ## 配置访问接口的用户名密码
   [ bearer_token: <secret> ]
   [ bearer_token_file: /path/to/bearer/token/file ]  ##指定认证token
   tls_config:
     [ <tls_config> ]                     ## 指定CA证书
   [ proxy_url: <string> ]                ## 使用代理模式访问目标
   
   consul_sd_configs:                   ##通过consul去发现
     [ - <consul_sd_config> ... ]
   dns_sd_configs:                      ##通过DNS去发现
     [ - <dns_sd_config> ... ]
   file_sd_configs:                   ##通过文件去发现
     [ - <file_sd_config> ... ]
   kubernetes_sd_configs:               ##通过kubernetes去发现
     [ - <kubernetes_sd_config> ... ]
     
   static_configs:
     [ - <static_config> ... ]     ##静态配置被监控端
   ```

    监控Prometheus自己的配置：

   ```yaml
   scrape_configs:
     - job_name: 'prometheus'
       static_configs:
       - targets: ['localhost:9090']
   ```

   最后的标签配置：

   ```yaml
   relabel_configs:
     [ - <relabel_config> ... ]          ##在数据采集前对标签进行重新标记
   metric_relabel_configs:
     [ - <relabel_config> ... ]          ##在数据采集之后对标签进行重新标记
   [ sample_limit: <int> | default = 0 ] ##采集样本数量，默认0
   ```





2. relabel_configs

   这个是用来重新打标记的，对于Prometheus数据模型最关键点就是一个指标名称和一组标签来组成一个多维度的数据模型，想要完成一个复杂的查询就需要有多维度，relabel_configs就是对标签进行处理的，能够帮助你在数据采集之前对任目标的标签进行修改，重打标签的意义就是如果标签有重复的可以帮你重命名。

   ![target-no1](D:\myworkspace\img\target-no1.png)

   现在instance是他默认给我加的标签，relabel_configs也可以重打标签，也可以删除标签，也可以过滤标签。，具体配置段如下:

   ```yaml
   relabel_configs: 
     [ source_labels: '[' <labelname> [, ...] ']' ]   ##源标签，指定对哪个现有标签进行操作
     [ separator: <string> | default = ; ]            ##多个源标签时连接的分隔符
     [ target_label: <labelname> ]                    ##要将源标签换成什么名字
     [ regex: <regex> | default = (.*) ]              ##怎么来匹配源标签，默认匹配所有
     [ modulus: <uint64> ]                            ##不怎么会用到
     [ replacement: <string> | default = $1 ]         ##替换正则表达式匹配到的分组，分组引用$1,$2,$3
     [ action: <relabel_action> | default = replace ] ##基于正则表达式匹配执行的操作，默认替换
   ```

   2.1 添加标签

   ```yaml
   - targets:
     - "192.168.227.132:9100"
     - "192.168.227.133:9100"
     labels:
       server: 'c6-node'
   ```

   重启Prometheus，就可以看到添加的标签了。

   

   然后可以根据这个标签去查了，语法是这样的，内置函数。

   ```
   sum(process_cpu_seconds_total{server="c6-node"})
   ```

   2.2 标签重命名

   就是将一个已有的标签重命名一个新的标签。

   如上图所示的，现在要将job="DMC_HOST" 改为  rmhost="c6-node1",下面开始用relabel进行重命名，改完之后的配置是这样的，

   ```yaml
   relabel_configs:
       - action: replace
         source_labels: ['job'] ##源标签
         regex: (.*)            ##正则，会匹配到job值，也就是DMC_HOST
         replacement: 'c6-node1' ##引用正则匹配到的内容，也就是c6-node1
         target_label: 'rmhost'  ##赋予新的标签，名为rmhost
   ```

   这样修改就可以了。新的数据已经有了，之前的标签还会保留，因为没有配置删除他，这样就可以了，现在就可以聚合了

   action重新打标签动作

   如下表所示：

   | **值**    | **描述**                                                     |
   | --------- | ------------------------------------------------------------ |
   | replace   | 默认，通过正则匹配source_label的值，使用replacement来引用表达式匹配的分组 |
   | keep      | 删除regex于链接不匹配的目标source_labels                     |
   | drop      | 删除regex与连接匹配的目标source_labels                       |
   | labeldrop | 匹配Regex所有标签名称                                        |
   | labelkeep | 不匹配regex所有标签名称                                      |
   | hashmod   | 设置target_label为modulus连接的哈希值source_lanels           |
   | labelmap  | 匹配regex所有标签名称，复制匹配标签的值分组，replacement分组引用(${1},${2})替代 |







### 基于文件的服务发现

如下所示：

```yaml
- job_- job_name: 'DMC_HOST'
  file_sd_configs:
    - files: ['/usr/local/prometheus/files_sd_configs/*.yml']

```

重启，然后在改目录下创建yml文件即可。





## Prometheus 监控

 ### node_exporter

node_exporter的导出器，会帮你收集系统指标和一些软件运行的指标，并且把指标暴漏出去，这样Prometheus就可以去采集了。

官方的GitHub地址：`https://github.com/prometheus/node_exporter`

```bash
cd /usr/local
wget https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz
ln -s node_exporter-1.0.1.linux-amd64 /usr/local/node_exporter
cd /usr/local/node_exporter
./node_exporter --help
```

配置开机启动脚本

Centos7 下：

```bash
[Unit]
Description=node_exporter
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/node_exporter/node_exporter
Restart=on-failure

[Install]
WantedBy=mulser.target

```

Centos6 下：

这里我们推荐使用supervisord

如果我们使用`yum install supervisor` 来进行安装，安装的版本是2.1.9；2.x版本有很多问题，可以启动supervisord进程，但是无法正常使用supervisorctl这个命令。

用pip install supervisor，默认装的是最新的4.0.3版本，但是centos6.5默认的只有python2.6，4.0.3的supervisor跑不起来，具体错误没有记录了，可以升级到python2.7，比较麻烦。

所以这里建议制定安装supervisor3.1.3，这个版本可以用python2.6，直接装了就能用.

```bash
yum install python-pip
pip install supervisor==3.1.3
#创建相关目录
mkdir /var/run/supervisor
mkdir /etc/supervisor
mkdir /etc/supervisor/supervisord.d
mkdir /var/log/supervisor
#创建配置文件
echo_supervisord_conf > /etc/supervisor/supervisord.conf
vim /etc/supervisor/supervisord.conf #基本都是默认配置，只需改最后一行，包含的子配置文件
```

```bash
; Sample supervisor config file.

[unix_http_server]
file=/var/run/supervisor/supervisor.sock   ; (the path to the socket file)
;chmod=0700                 ; sockef file mode (default 0700)
;chown=nobody:nogroup       ; socket file uid:gid owner
;username=user              ; (default is no username (open server))
;password=123               ; (default is no password (open server))

;[inet_http_server]         ; inet (TCP) server disabled by default
;port=127.0.0.1:9001        ; (ip_address:port specifier, *:port for all iface)
;username=user              ; (default is no username (open server))
;password=123               ; (default is no password (open server))

[supervisord]
logfile=/var/log/supervisor/supervisord.log  ; (main log file;default $CWD/supervisord.log)
logfile_maxbytes=50MB       ; (max main logfile bytes b4 rotation;default 50MB)
logfile_backups=10          ; (num of main logfile rotation backups;default 10)
loglevel=info               ; (log level;default info; others: debug,warn,trace)
pidfile=/var/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
nodaemon=false              ; (start in foreground if true;default false)
minfds=1024                 ; (min. avail startup file descriptors;default 1024)
minprocs=200                ; (min. avail process descriptors;default 200)
;umask=022                  ; (process file creation umask;default 022)
;user=chrism                 ; (default is current user, required if root)
;identifier=supervisor       ; (supervisord identifier, default is 'supervisor')
;directory=/tmp              ; (default is not to cd during start)
;nocleanup=true              ; (don't clean up tempfiles at start;default false)
;childlogdir=/tmp            ; ('AUTO' child log dir, default $TEMP)
;environment=KEY=value       ; (key value pairs to add to environment)
;strip_ansi=false            ; (strip ansi escape codes in logs; def. false)

; the below section must remain in the config file for RPC
; (supervisorctl/web interface) to work, additional interfaces may be
; added by defining them in separate rpcinterface: sections
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///var/run/supervisor/supervisor.sock ; use a unix:// URL  for a unix socket
;serverurl=http://127.0.0.1:9001 ; use an http:// url to specify an inet socket
;username=chris              ; should be same as http_username if set
;password=123                ; should be same as http_password if set
;prompt=mysupervisor         ; cmd line prompt (default "supervisor")
;history_file=~/.sc_history  ; use readline history if available

; The below sample program section shows all possible program subsection values,
; create one or more 'real' program: sections to be able to control them under
; supervisor.

;[program:theprogramname]
;command=/bin/cat              ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=999                  ; the relative start priority (default 999)
;autostart=true                ; start at supervisord start (default: true)
;autorestart=true              ; retstart at unexpected quit (default: true)
;startsecs=10                  ; number of secs prog must stay running (def. 1)
;startretries=3                ; max # of serial start failures (default 3)
;exitcodes=0,2                 ; 'expected' exit codes for process (default 0,2)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (default 10)
;stdout_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10     ; # of stderr logfile backups (default 10)
;stderr_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;environment=A=1,B=2           ; process environment additions (def no adds)
;serverurl=AUTO                ; override serverurl computation (childutils)

; The below sample eventlistener section shows all possible
; eventlistener subsection values, create one or more 'real'
; eventlistener: sections to be able to handle event notifications
; sent by supervisor.

;[eventlistener:theeventlistenername]
;command=/bin/eventlistener    ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;events=EVENT                  ; event notif. types to subscribe to (req'd)
;buffer_size=10                ; event buffer queue size (default 10)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=-1                   ; the relative start priority (default -1)
;autostart=true                ; start at supervisord start (default: true)
;autorestart=unexpected        ; restart at unexpected quit (default: unexpected)
;startsecs=10                  ; number of secs prog must stay running (def. 1)
;startretries=3                ; max # of serial start failures (default 3)
;exitcodes=0,2                 ; 'expected' exit codes for process (default 0,2)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (default 10)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups        ; # of stderr logfile backups (default 10)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;environment=A=1,B=2           ; process environment additions
;serverurl=AUTO                ; override serverurl computation (childutils)

; The below sample group section shows all possible group values,
; create one or more 'real' group: sections to create "heterogeneous"
; process groups.

;[group:thegroupname]
;programs=progname1,progname2  ; each refers to 'x' in [program:x] definitions
;priority=999                  ; the relative start priority (default 999)

; The [include] section can just contain the "files" setting.  This
; setting can list multiple files (separated by whitespace or
; newlines).  It can also contain wildcards.  The filenames are
; interpreted as relative to this file.  Included files *cannot*
; include files themselves.

[include]
files = /etc/supervisor/*.ini
```

supervisord的开机启动文件`/etc/init.d/supervisord`：

```bash
#!/bin/bash
#
# supervisord   This scripts turns supervisord on
#
# Author:       Mike McGrath <mmcgrath@redhat.com> (based off yumupdatesd)
#
# chkconfig:    - 95 04
#
# description:  supervisor is a process control utility.  It has a web based
#               xmlrpc interface as well as a few other nifty features.
# processname:  supervisord
# config: /etc/supervisord.conf
# pidfile: /var/run/supervisord.pid
#

# source function library
. /etc/rc.d/init.d/functions
PIDFILE=/var/run/supervisord.pid
RETVAL=0

start() {
        echo -n $"Starting supervisord: "
        daemon "supervisord --pidfile=$PIDFILE -c /etc/supervisor/supervisord.conf"
        RETVAL=$?
        echo
        [ $RETVAL -eq 0 ] && touch /var/lock/subsys/supervisord
}

stop() {
        echo -n $"Stopping supervisord: "
        killproc supervisord
        echo
        [ $RETVAL -eq 0 ] && rm -f /var/lock/subsys/supervisord
}

restart() {
        stop
        start
}

case "$1" in
  start)
        start
        ;;
  stop)
        stop
        ;;
  restart|force-reload|reload)
        restart
        ;;
  condrestart)
        [ -f /var/lock/subsys/supervisord ] && restart
        ;;
  status)
        status supervisord
        RETVAL=$?
        ;;
  *)
        echo $"Usage: $0 {start|stop|status|restart|reload|force-reload|condrestart}"
        exit 1
esac

exit $RETVAL



#添加到开机启动项
chkconfig supervisord on
```



配置node_exporter子配置文件,

```bash
[root@c6-node1 supervisor]# cat /etc/supervisor/supervisord.d/node_exporter.ini
[program:node_exporter]
# 启动程序的命令;
command = /usr/local/node_exporter/node_exporter
# 在supervisord启动的时候也自动启动;
autostart = true
# 程序异常退出后自动重启;
autorestart = true
# 启动5秒后没有异常退出，就当作已经正常启动了;
startsecs = 5
# 启动失败自动重试次数，默认是3;
startretries = 3
# 启动程序的用户;
user = root
# 把stderr重定向到stdout，默认false;
redirect_stderr = true
# 标准日志输出;
stdout_logfile=/usr/local/node_exporter/logs/out.log
# 错误日志输出;
stderr_logfile=/usr/local/node_exporter/logs/err.log
# 标准日志文件大小，默认50MB;
stdout_logfile_maxbytes = 20MB
# 标准日志文件备份数;
stdout_logfile_backups = 20

#启动
/etc/init.d/supervisord start
supervisorctl restart node_exporter
```

然后可以通过`curl -s 127.0.0.1:9100/metrics | grep head`  进行测试。

配置Prometheus监控此主机

```yaml
# vim prometheus.yml
  - job_name: "nodes"
    file_sd_configs:
      - files: ['/usr/local/prometheus/node_sd_configs/*.yml']
        refresh_interval: 5s
```

```yaml
# cd /usr/local/prometheus/ && mkdir nodes_sd_configs
# 重新加载Prometheus配置
# ps aux | grep prometheus.yml  | grep -v grep  | awk {'print $2'} | xargs kill -hup
# vim nodes1.yml
- targets: ['192.168.227.132:9100'] 
  labels:
    name: server132
```

此时可以去Prometheus的webui界面进行验证。

### mysqld_exporter

官网地址：`https://github.com/prometheus/mysqld_exporter`

1. 创建MySQL相关权限账户

   ```mysql
   mysql> CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'exporter';
   Query OK, 0 rows affected (0.02 sec)
   
   mysql> GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
   Query OK, 0 rows affected (0.00 sec)
   
   mysql> flush privileges;
   Query OK, 0 rows affected (0.00 sec)
   
   mysql> select user,host from mysql.user;
   ```

2. 下载mysqld_exporter并进行安装配置

   ```bash
   cd /usr/local
   wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.12.1/mysqld_exporter-0.12.1.linux-amd64.tar.gz
   ln -s mysqld_exporter-0.12.1.linux-amd64 mysqld_exporter
   ```

   centos7 下的开机启动脚本

   ````bash
   [Unit]
   Description=mysqld_exporter
   Documentation=https://prometheus.io/
   After=network.target
   
   [Service]
   Type=simple
   ExecStart=/usr/local/mysqld_exporter/mysqld_exporter --collect.info_schema.processlist.processes_by_host --collect.info_schema.processlist.processes_by_user --collect.info_schema.innodb_tablespaces --collect.info_schema.innodb_metrics --collect.perf_schema.tableiowaits --collect.perf_schema.indexiowaits --collect.perf_schema.tablelocks --collect.engine_innodb_status --collect.perf_schema.file_events --collect.binlog_size --collect.info_schema.clientstats --collect.perf_schema.eventswaits --config.my-cnf=/etc/.my.cdf
   Restart=on-failure
   
   [Install]
   WantedBy=mulser.target
   ````

   centos6 下依然使用supervisrd进行管理

   创建相关配置文件：

   ```bash
   [program:mysqld_exporter]
   # 启动程序的命令;
   command = /usr/local/mysqld_exporter/mysqld_exporter --collect.info_schema.processlist.processes_by_host --collect.info_schema.processlist.processes_by_user --collect.info_schema.innodb_tablespaces --collect.info_schema.innodb_metrics --collect.perf_schema.tableiowaits --collect.perf_schema.indexiowaits --collect.perf_schema.tablelocks --collect.engine_innodb_status --collect.perf_schema.file_events --collect.binlog_size --collect.info_schema.clientstats --collect.perf_schema.eventswaits --config.my-cnf=/etc/my.cnf
   # 在supervisord启动的时候也自动启动;
   autostart = true
   # 程序异常退出后自动重启;
   autorestart = true
   # 启动5秒后没有异常退出，就当作已经正常启动了;
   startsecs = 5
   # 启动失败自动重试次数，默认是3;
   startretries = 3
   # 启动程序的用户;
   user = root
   # 把stderr重定向到stdout，默认false;
   redirect_stderr = true
   # 标准日志输出;
   stdout_logfile=/usr/local/mysqld_exporter/logs/out.log
   # 错误日志输出;
   stderr_logfile=/usr/local/mysqld_exporter/logs/err.log
   # 标准日志文件大小，默认50MB;
   stdout_logfile_maxbytes = 20MB
   # 标准日志文件备份数;
   stdout_logfile_backups = 20
   ```

3. 然后进行启动，

   访问MySQL的metrics进行验证 `http://192.168.227.133:9104/metrics`

4. 在Prometheus的配置文件中，进行添加

   ```yaml
     - job_name: 'MySQL'
       file_sd_configs:
         - files: ['./mysql.yml']
           refresh_interval: 15s
           
   # mysql.yml      
   - targets:
     - "192.168.227.133:9104"
   
   ```

5. 也可以在Prometheus 的webui中targets中进行验证。



### jmx_exporter

官网地址：`https://github.com/prometheus/jmx_exporter`

由于官方只有jmx的exporter，所以这里我们在监控tomcat的时候，需要修改tomcat的JAVA_OPTS。

1. 安装jmx_exporter

   ```bash
   cd /usr/local/
   mkdir jmx_exporter && cd jmx_exporter
   wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.14.0/jmx_prometheus_javaagent-0.14.0.jar
   #下载tomcat的yml配置文件
   wget https://raw.githubusercontent.com/prometheus/jmx_exporter/master/example_configs/tomcat.yml
   ```

2. 启动

   ```bash
   ## JMX_exporter支持jar包启动直接添加javaagent，但是这里监控的是tomcat，一般是修改path/bin/catalina.sh添加JAVA_OPTS，让jar包跟随tomcat启动。
   ## 根据tomcat启动顺序，不建议直接修改catalina.sh，可以新建path/bin/setenv.sh写入内容，启动tomcat会直接加载，
   ## 这里我们监控tomcat使用38080端口，如果有多个tomcat实例，请注意每一个实例需要有不同的监控端口
   # 新建添加内容，path为你tomcat目录
   vi path/bin/setenv.sh
   export JAVA_OPTS="-javaagent:/usr/local/jmx_exporter/jmx_prometheus_javaagent-0.14.0.jar=38080:/usr/local/jmx_exporter/tomcat.yml"
   # 重新启动tomcat
   cd path/bin
   sh shutdown.sh
   sh startup.sh
   ```

3. 然后进行访问测试

   ```bash
   curl http://127.0.0.1:38080/metrics
   ```

4. 配置Prometheus

   ```yaml
     - job_name: "java"
       file_sd_configs:
         - files: ['./tomcat.yml']
           refresh_interval: 15s
   #配置tomcat.yml
   - targets:
     - "192.168.50.204:38080"
   ```

5. 重新加载Prometheus，查看target内容即可。

6. 添加agent之后，tomcat无法正常shutdown；手动添加shutdown脚本。

   ````bash
   #!/bin/sh
   
   
   tomcat_base=/usr/local/server/tomcat
   TOMCAT_PATH=${tomcat_base}/bin
   
   echo "TOMCAT_PATH is $TOMCAT_PATH"
   
   PID=`ps aux | grep ${tomcat_base} | grep java | awk '{print $2}'`
   
   if [ -n "$PID" ]; then
           echo "Try to shutdown Tomcat: $PID"
           sh "$TOMCAT_PATH/shutdown.sh"
                   sleep 1
   fi
   
   for((i=0;i<10;i++))
   do
           PID2=`ps aux | grep ${tomcat_base} | grep java | awk '{print $2}'`
               
           if [ -n "$PID2" ]; then
                           if [ $i -ge 9 ] ; then
                                   echo "Try to kill Tomcat: $PID2"
                                   ((i--))
                                   kill -9 $PID2
                           else
                                   echo "wait to kill Tomcat: $PID2"
                           fi
                           sleep 1
           else 
                   echo "Tomcat is closed"
                   break
           fi
   done
   ````

   



### redis_exporter

官方地址：`https://github.com/oliver006/redis_exporter`

1. 安装

   ```bash
   cd /usr/local/
   wget https://github.com/oliver006/redis_exporter/releases/download/v1.11.1/redis_exporter-v1.11.1.linux-amd64.tar.gz
   ln -s redis_exporter-v1.11.1.linux-amd64 redis_exporter			
   ```

2. 配置开机启动项

   Centos 7下：

   ```bash
   [root@c2 local]# cat /usr/lib/systemd/system/redis_exporter.service 
   [Unit]
   Description=redis_exporter
   After=network.target
   
   [Service]
   Restart=on-failure
   ExecStart=/usr/local/redis_exporter/redis_exporter -redis.addr 192.168.50.182:6379
   
   [Install]
   WantedBy=multi-user.target
   ```

3. Prometheus配置

   ```yaml
     - job_name: 'Redis'
       static_configs:
       - targets: ['localhost:9121']
   ```

4. 重新加载Prometheus即可





### nginx-prometheus-exporter

官网地址：`https://github.com/nginxinc/nginx-prometheus-exporter`

1. nginx开启stub_status

   ```nginx
   server {
       listen localhost:38080;
       location /metrics {
         stub_status on;
       }
   }
   
   ```

2. 安装并配置nginx-prometheus-exporter

   ```bash
   cd /usr/loca/
   mkdir nginx-prometheus-exporter && cd nginx-prometheus-exporter
   wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v0.8.0/nginx-prometheus-exporter-0.8.0-linux-amd64.tar.gz
   #centos7下添加到开机启动项
   [root@camp204 local]# cat /usr/lib/systemd/system/nginx_prometheus_exporter.service 
   [Unit]
   Description=NGINX Prometheus Exporter
   After=network.target
   
   [Service]
   Type=simple
   ExecStart=/usr/local/nginx-prometheus-exporter/nginx-prometheus-exporter \
       -web.listen-address=192.168.50.204:9113 \
       -nginx.scrape-uri http://127.0.0.1:38080/metrics
   
   SyslogIdentifier=nginx_prometheus_exporter
   Restart=always
   
   [Install]
   WantedBy=multi-user.target
   #centos6下依然采用supervisord
   [program:nginx-prometheus-exporter]
   # 启动程序的命令;
   command = /usr/local/nginx-prometheus-exporter/nginx-prometheus-exporter -web.listen-address=192.168.50.204:9113 -nginx.scrape-uri http://127.0.0.1:38080/metrics
   # 在supervisord启动的时候也自动启动;
   autostart = true
   # 程序异常退出后自动重启;
   autorestart = true
   # 启动5秒后没有异常退出，就当作已经正常启动了;
   startsecs = 5
   # 启动失败自动重试次数，默认是3;
   startretries = 3
   # 启动程序的用户;
   user = root
   # 把stderr重定向到stdout，默认false;
   redirect_stderr = true
   # 标准日志输出;
   stdout_logfile=/usr/local/nginx-prometheus-exporter/logs/out.log
   # 错误日志输出;
   stderr_logfile=/usr/local/nginx-prometheus-exporter/logs/err.log
   # 标准日志文件大小，默认50MB;
   stdout_logfile_maxbytes = 20MB
   # 标准日志文件备份数;
   stdout_logfile_backups = 20
   ```

3. 添加Prometheus配置

   ```bash
     - job_name: "Nginx"
       static_configs:
         - targets: ['192.168.50.204:9113']
   ```

4. 重新加载Prometheus服务。



### blackbox_exporter

blackbox_exporter是Prometheus 官方提供的 exporter 之一，可以提供 http、dns、tcp、icmp 的监控数据采集.

官方地址：`https://github.com/prometheus/blackbox_exporter`

应用场景

1. http测试

   - 定义 Request Header 信息
   - 判断 Http status / Http Respones Header / Http Body 内容

2. TCP测试

   - 业务组件端口状态监听
   - 应用层协议定义与监听

3. ICMP测试

   - 主机探活机制

4. POST测试

   - 接口联通性

5. SSL证书过期时间

   



- 安装并配置开机启动项

  ```bash
  cd /usr/local/
  wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.17.0/blackbox_exporter-0.17.0.linux-amd64.tar.gz
  ln -s blackbox_exporter-0.17.0.linux-amd64 blackbox_exporter
  #centos6下 使用supervisord管理
  [program:blackbox_exporter]
  # 启动程序的命令;
  command = /usr/local/blackbox_exporter/blackbox_exporter --config.file=/usr/local/blackbox_exporter/blackbox.yml
  # 在supervisord启动的时候也自动启动;
  autostart = true
  # 程序异常退出后自动重启;
  autorestart = true
  # 启动5秒后没有异常退出，就当作已经正常启动了;
  startsecs = 5
  # 启动失败自动重试次数，默认是3;
  startretries = 3
  # 启动程序的用户;
  user = root
  # 把stderr重定向到stdout，默认false;
  redirect_stderr = true
  # 标准日志输出;
  stdout_logfile=/usr/local/blackbox_exporter/logs/out.log
  # 错误日志输出;
  stderr_logfile=/usr/local/blackbox_exporter/logs/err.log
  # 标准日志文件大小，默认50MB;
  stdout_logfile_maxbytes = 20MB
  # 标准日志文件备份数;
  stdout_logfile_backups = 10
  
  #centos7下:
  [Unit]
  Description=blackbox exporter
  After=network.target
  
  [Service]
  Type=simple
  ExecStart=/usr/local/blackbox_exporter/blackbox_exporter --config.file=/usr/local/blackbox_exporter/blackbox.yml \
      --web.listen-address=:9115 
  
  SyslogIdentifier=blackbox_exporter
  Restart=always
  
  [Install]
  WantedBy=multi-user.target
  ```

- Prometheus配置

  ```yaml
  #HTTP检测
  - job_name: 'http_get_all'  # blackbox_export module
      scrape_interval: 30s
      metrics_path: /probe
      params:
        module: [http_2xx]
      static_configs:
        - targets:
          - https://frognew.com
      relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: 127.0.0.1:9115 #blackbox-exporter 所在的机器和端口
  #http检测除了可以探测http服务的存活外，还可以根据指标probe_ssl_earliest_cert_expiry进行ssl证书有效期预警。
  
  #监控主机存活状态
  - job_name: node_status
      metrics_path: /probe
      params:
        module: [icmp]
      static_configs:
        - targets: ['10.165.94.31']
          labels:
            instance: node_status
            group: 'node'
      relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - target_label: __address__
          replacement: 172.19.155.133:9115
          
  #监控主机端口存活状态
  - job_name: 'prometheus_port_status'
      metrics_path: /probe
      params:
        module: [tcp_connect]
      static_configs:
        - targets: ['172.19.155.133:8765']
          labels:
            instance: 'port_status'
            group: 'tcp'
      relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: 172.19.155.133:9115
          
          
  #监控网站状态
  - job_name: web_status
      metrics_path: /probe
      params:
        module: [http_2xx]
      static_configs:
        - targets: ['http://www.baidu.com']
          labels:
            instance: user_status
            group: 'web'
      relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - target_label: __address__
          replacement: 172.19.155.133:9115
  ```

  



## Alertmanager

### 安装并配置

官方地址： `https://github.com/prometheus/alertmanager`

下载并配置

```bash
cd /usr/local/src/
wget https://github.com/prometheus/alertmanager/releases/download/v0.21.0/alertmanager-0.21.0.linux-amd64.tar.gz
tar zxvf alertmanager-0.21.0.linux-amd64.tar.gz -C /usr/local/
ln -s alertmanager-0.21.0.linux-amd64 alertmanager
#配置开机启动项
cat /usr/lib/systemd/system/alertmanager.service 
[Unit]
Description=Prometheus: the alerting system
Documentation=http://prometheus.io/docs/
After=prometheus.service

[Service]
ExecStart=/usr/local/alertmanager/alertmanager --config.file=/usr/local/alertmanager/alertmanager.yml
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target

systemctl enable alertmanager && systemctl start alertmanager
```



修改配置文件，这里我们使用的是企业微信告警，如需其他告警可自行百度：

**访问企业微信官网（https://work.weixin.qq.com/），注册企业微信账号（不需要企业认证）**

**登录成功后--->>应用管理--->>创建第三方应用，点击创建应用按钮 -> 填写应用信息：**



```bash
[root@IT-ECS-Prometheus alertmanager]# cat alertmanager.yml 
global:
  resolve_timeout: 2m  #每两分钟查看是否恢复
  wechat_api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'             #企业微信的api
  wechat_api_secret: 'MWqZ2yaIZvBKkVi5lUxPpuEIgl7q2D6tEFadnf7Hcbs'   #第三方企业应用密钥
  wechat_api_corp_id: 'ww5fc93f267b956282'            #企业账户唯一ID，可以在我的企业中查看

route:                           #用来设置报警的分发策略
  group_by: ['alertname']        #采用哪个标签来作为分组依据
  group_wait: 10s                #组告警等待时间。也就是告警产生后等待10s，如果有同组告警一起发出
  group_interval: 10s            #重复告警的间隔时间，减少相同微信告警的发送频率
  repeat_interval: 8h    #重复告警的间隔时间，减少相同微信告警的发送频率
  receiver: 'wechat'     #设置默认接收人

receivers:                  #定义接收者
- name: 'wechat'
  wechat_configs:
  - send_resolved: true
    to_user: 'LiYang'   #需要发送的用户，也可以选择发送到组 to_party
    agent_id: '1000016'  #第三方企业应用的ID
templates:
- /usr/local/alertmanager/template.tmpl  #告警模板
```



配置告警规则，告警规则已上传到GitHub，地址是：`https://github.com/skymyyang/Prometheus-alertrules-configs`

告警规则可参考此项目中的PrometheusConfig 配置。

重启Prometheus服务，然后在Prometheus的WebUI中可以查看到报警规则。

钉钉报警参考链接：`https://mp.weixin.qq.com/s/QRK49Xa-HWjnrw9Vvkmh4A`



## Grafana

1. 安装并配置

   ```bash
   cd /usr/local/src/
   export VER="7.1.5"
   wget https://dl.grafana.com/oss/release/grafana-${VER}-1.x86_64.rpm
   yum localinstall -y grafana-${VER}-1.x86_64.rpm
   ```

2. 启动服务

   ```bash
   systemctl daemon-reload
   systemctl enable grafana-server.service
   systemctl stop grafana-server.service
   systemctl restart grafana-server.service
   ```

3. 安装相关插件

   ```bash
   #需要安装饼图的插件
   grafana-cli plugins install grafana-piechart-panel
   #安装consul数据源插件
   grafana-cli plugins install sbueringer-consul-datasource
   #安装pmm-singlestat-panel插件
   #这里我在使用percona的MySQLdashboard 需要用到这个插件，但是一直安装失败
   cd /usr/local/src/
   wget https://jira.percona.com/secure/attachment/22830/22830_pmm-singlestat-panel.tgz
   tar xvf pmm-singlestat-panel.tgz
   mv pmm-singlestat-panel /var/lib/grafana/plugins/
   systemctl restart grafana-server
   ```

4. 导入相关展示图表

   ```
   https://grafana.com/grafana/dashboards/11074  基础监控-new
   
   https://grafana.com/dashboards/8919   基础监控
   
   https://grafana.com/dashboards/7362   数据库监控
   https://github.com/percona/grafana-dashboards/blob/master/dashboards/MySQL_Overview.json #percona MySQL图表
   ```

   图表中遇到的问题，由于部分机器是centos6 的操作系统，系统内核版本过低，在metrics中没有`MemAvailable`指标，这里我们只能使用`node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes` 来接近显示内存使用率。

   可以编辑图表，修改其中内存使用率的计算表达式为：

   ```promql
   (1 - ((node_memory_MemFree_bytes{job=~"$job"} + node_memory_Buffers_bytes{job=~"$job"} + node_memory_Cached_bytes{job=~"$job"} ) / (node_memory_MemTotal_bytes{job=~"$job"})))* 100
   ```

   修改完成之后，图表显示正常。

5. 安装blackbox_exporter 的dashboard 展示看板

   grafana的模板ID为:9965

6. Redis的dashboard  

   grafana的模板ID为：763