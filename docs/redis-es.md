---
layout: default
title: Yj
---

## Redis 服务器配置

```shell
[root@localhost redis]# cat /opt/redis/redis.conf


# 监听的地址和端口
bind 0.0.0.0
port 6379

# 数据库文件的位置
# dbfilename dump.rdb

# 快照配置
save 900 1
save 300 10
save 60 10000

# 数据库数量
databases 16

# RDB 持久化
# save 900 1
# save 300 10
# save 60 10000

# AOF 持久化
appendonly yes
appendfilename "appendonly.aof"

# 安全配置
requirepass  123456

# 日志配置
loglevel notice
logfile /opt/redis/redis-server.log

# 最大客户端连接数
maxclients 10000

# 内存优化
maxmemory-policy noeviction

# 主从复制
# slaveof <masterip> <masterport>

# 集群模式
# cluster-enabled yes
# cluster-config-file nodes.conf
# cluster-node-timeout 15000
```
```shell
[root@localhost redis]# cat /usr/lib/systemd/system/redis.service
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
Type=simple
ExecStart=/opt/redis/redis-server /opt/redis/redis.conf
ExecStop=/usr/local/bin/redis-cli shutdown
Restart=always

[Install]
WantedBy=multi-user.target


[root@localhost redis]# systemctl start redis
```



## Filebeat

```shell
[root@localhost filebeat-7.17.0]# cat /opt/filebeat-7.17.0/filebeat.yml|grep -v '#'
filebeat.inputs:
- type: log
  paths:
    - /var/log/nginx/access.log



filebeat.config.modules:
  path: "/opt/filebeat-7.17.0/modules.d/*.yml"

  reload.enabled: false

output.logstash:
  hosts: ["172.16.17.131:5044"]

setup.kibana:

  host: "172.16.17.131:5601"

[root@localhost filebeat-7.17.0]# systemctl start filebeat
```

## Logstash

```
[root@localhost ~]# cat /opt/logstash-7.17.0/config/logstash.conf
input {
  beats {
    port => 5044
# 替换为实际的 Filebeat 端口号
  }
}

filter {
  if [message] =~ ".*GET /favicon.ico.*" {
    drop { }
# 丢弃匹配 favicon.ico 请求的记录
  }

  grok {
    match => {
       "message" => '%{IPORHOST:[nginx][clientip]} - - \[%{HTTPDATE:[nginx][time]}\] "%{WORD:[nginx][method]} %{URIPATHPARAM:[nginx][request]} HTTP/%{NUMBER:[nginx][httpversion]}" %{NUMBER:[nginx][response]} %{NUMBER:[nginx][bytes]} "-" "%{DATA:[nginx][agent]}"'
    }
    remove_field => "message"
  }

  date {
    match => [ "[nginx][time]", "dd/MMM/YYYY:H:m:s Z" ]
    target => "[nginx][time]"
  }
}

output {
  redis {
    host => "172.16.17.131"
    port => 6379
    data_type => "list"
# 或者 "channel"
    key => "nginxaccesslog"
    password => "123456"
  }
  elasticsearch {
    hosts => ["172.16.17.131:9200"]
    index => "filebeat-message-%{+YYYY.MM.dd}"
  }
}
[root@localhost logstash-7.17.0]# cat /usr/lib/systemd/system/logstash.service
[Unit]
Description=Logstash
Documentation=https://www.elastic.co/products/logstash

[Service]
Type=simple
ExecStart=/opt/logstash-7.17.0/bin/logstash -f /opt/logstash-7.17.0/config/logstash.conf
TimeoutStopSec=30
TimeoutStartSec=0
Restart=always

[Install]
WantedBy=multi-user.target

[root@localhost logstash-7.17.0]# systemctl start logstash
```



## Redis 查看数据

```shell
[root@localhost logstash-7.17.0]# /opt/redis-7.0.12/src/redis-cli -a 123456
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
127.0.0.1:6379> keys *
1) "nginxaccesslog"
127.0.0.1:6379> LRANGE nginxaccesslog 0 9
1) "{\"ecs\":{\"version\":\"1.12.0\"},\"tags\":[\"beats_input_codec_plain_applied\",\"_grokparsefailure\"],\"host\":{\"name\":\"localhost.localdomain\"},\"agent\":{\"hostname\":\"localhost.localdomain\",\"version\":\"7.17.0\",\"name\":\"localhost.localdomain\",\"id\":\"b2612462-6d5d-411e-a218-0d50523a6c30\",\"type\":\"filebeat\",\"ephemeral_id\":\"a208c8ba-12e7-49fb-b7f4-976a59da10fa\"},\"@version\":\"1\",\"@timestamp\":\"2023-08-09T06:20:04.362Z\",\"message\":\"172.16.17.1 - - [09/Aug/2023:15:20:00 +0900] \\\"GET /testjian HTTP/1.1\\\" 502 559 \\\"-\\\" \\\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/115.0.0.0 Safari/537.36\\\" \\\"-\\\"\",\"input\":{\"type\":\"log\"},\"log\":{\"file\":{\"path\":\"/var/log/nginx/access.log\"},\"offset\":2616}}"
2) "{\"ecs\":{\"version\":\"1.12.0\"},\"tags\":[\"beats_input_codec_plain_applied\",\"_grokparsefailure\"],\"host\":{\"name\":\"localhost.localdomain\"},\"agent\":{\"hostname\":\"localhost.localdomain\",\"version\":\"7.17.0\",\"name\":\"localhost.localdomain\",\"id\":\"b2612462-6d5d-411e-a218-0d50523a6c30\",\"type\":\"filebeat\",\"ephemeral_id\":\"a208c8ba-12e7-49fb-b7f4-976a59da10fa\"},\"@version\":\"1\",\"@timestamp\":\"2023-08-09T06:20:04.366Z\",\"message\":\"172.16.17.1 - - [09/Aug/2023:15:20:03 +0900] \\\"GET / HTTP/1.1\\\" 502 559 \\\"-\\\" \\\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/115.0.0.0 Safari/537.36\\\" \\\"-\\\"\",\"input\":{\"type\":\"log\"},\"log\":{\"file\":{\"path\":\"/var/log/nginx/access.log\"},\"offset\":3064}}"
3) "{\"ecs\":{\"version\":\"1.12.0\"},\"tags\":[\"beats_input_codec_plain_applied\",\"_grokparsefailure\"],\"host\":{\"name\":\"localhost.localdomain\"},\"agent\":{\"hostname\":\"localhost.localdomain\",\"version\":\"7.17.0\",\"name\":\"localhost.localdomain\",\"id\":\"b2612462-6d5d-411e-a218-0d50523a6c30\",\"ephemeral_id\":\"a208c8ba-12e7-49fb-b7f4-976a59da10fa\",\"type\":\"filebeat\"},\"@version\":\"1\",\"@timestamp\":\"2023-08-09T06:20:22.369Z\",\"message\":\"172.16.17.1 - - [09/Aug/2023:15:20:15 +0900] \\\"GET / HTTP/1.1\\\" 502 559 \\\"-\\\" \\\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/115.0.0.0 Safari/537.36\\\" \\\"-\\\"\",\"input\":{\"type\":\"log\"},\"log\":{\"file\":{\"path\":\"/var/log/nginx/access.log\"},\"offset\":3496}}"

```


## rabbitmq
[root@localhost opt]# wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.12.2/rabbitmq-server-generic-unix-3.12.2.tar.xz --no-check-certificate
[root@localhost opt]# wget https://github.com/erlang/otp/releases/download/OTP-25.3.2/otp_src_25.3.2.tar.gz  --no-check-certificate
[root@localhost opt]# tar zxf otp_src_25.3.2.tar.gz
[root@localhost fop]# wget  https://dlcdn.apache.org//xmlgraphics/fop/binaries/fop-2.8-bin.tar.gz
[root@localhost fop]# ln -s /opt/fop-2.8/fop/fop /usr/bin/fop

## 安装 erlang 
[root@localhost opt]# cd otp_src_25.3.2/
[root@localhost opt]# yum -y install gcc glibc-devel make ncurses-devel openssl-devel xmlto perl gtk2-devel binutils-devel  wxBase-devel mesa-libGL-devel mesa-libGLU-devel
[root@localhost otp_src_25.3.2]# ./configure --prefix=/usr/local/erlang 
[root@localhost otp_src_25.3.2]# make
[root@localhost otp_src_25.3.2]# make install

   没有error的话报错暂时忽略
config.status: creating config.mk
config.status: creating c_src/Makefile
./configure: line 5758: test: too many arguments
./configure: line 5761: test: too many arguments
configure: WARNING:
    The requested wxWidgets build couldn't be found.

    The configuration you asked for  requires a wxWidgets
    build with the following settings:
        --unicode
    but such build is not available.
    To see the wxWidgets builds available on this system, please use
    'wx-config --list' command. To use the default build, returned by
    'wx-config --selected-config', use the options with their 'auto'
    default values.
    If you still get this error, then check that 'wx-config' is
    in path, the directory where wxWidgets libraries are installed
    (returned by 'wx-config --libs' command) is in LD_LIBRARY_PATH
    or equivalent variable and wxWidgets version is 3.0.2 or above.
./configure: line 6089: test: too many arguments
./configure: line 6092: test: too many arguments
./configure: line 6373: test: too many arguments
./configure: line 6376: test: too many arguments
configure: WARNING:
        wxWidgets must be installed on your system.

        Please check that wx-config is in path, the directory
        where wxWidgets libraries are installed (returned by
        'wx-config --libs' or 'wx-config --static --libs' command)
        is in LD_LIBRARY_PATH or equivalent variable and
        wxWidgets version is 3.0.2 or above.

*********************************************************************
**********************  APPLICATIONS DISABLED  **********************
*********************************************************************

odbc           : ODBC library - link check failed

*********************************************************************
*********************************************************************
**********************  APPLICATIONS INFORMATION  *******************
*********************************************************************

wx             :
    The requested wxWidgets build couldn't be found.

    The configuration you asked for  requires a wxWidgets
    build with the following settings:
        --unicode
    but such build is not available.
    To see the wxWidgets builds available on this system, please use
    'wx-config --list' command. To use the default build, returned by
    'wx-config --selected-config', use the options with their 'auto'
    default values.
    If you still get this error, then check that 'wx-config' is
    in path, the directory where wxWidgets libraries are installed
    (returned by 'wx-config --libs' command) is in LD_LIBRARY_PATH
    or equivalent variable and wxWidgets version is 3.0.2 or above.
wxWidgets was not compiled with --enable-webview or wxWebView developer package is not installed, wxWebView will NOT be available

        wxWidgets must be installed on your system.

        Please check that wx-config is in path, the directory
        where wxWidgets libraries are installed (returned by
        'wx-config --libs' or 'wx-config --static --libs' command)
        is in LD_LIBRARY_PATH or equivalent variable and
        wxWidgets version is 3.0.2 or above.

[root@localhost bin]# cat ~/.bashrc|grep erl
export PATH="$PATH:/usr/local/erlang/bin"
[root@localhost bin]# which erl
/usr/local/erlang/bin/erl

## 安装 rabbitmaq
[root@localhost opt]# tar Jxf rabbitmq-server-generic-unix-3.12.2.tar.xz
## 启动
[root@localhost rabbitmq_server-3.12.2]# ./sbin/rabbitmq-server -detached
## 查看状态
[root@localhost rabbitmq_server-3.12.2]# ./sbin/rabbitmqctl status
Status of node rabbit@localhost ...
Runtime
开启浏览器访问
[root@localhost rabbitmq_server-3.12.2]# ./sbin/rabbitmq-plugins enable rabbitmq_management
Enabling plugins on node rabbit@localhost:
rabbitmq_management
The following plugins have been configured:
  rabbitmq_management
  rabbitmq_management_agent
  rabbitmq_web_dispatch
Applying plugin configuration to rabbit@localhost...
The following plugins have been enabled:
  rabbitmq_management
  rabbitmq_management_agent
  rabbitmq_web_dispatch

started 3 plugins.
[root@localhost rabbitmq_server-3.12.2]# ./sbin/rabbitmqctl add_user jian 123456
[root@localhost rabbitmq_server-3.12.2]# ./sbin/rabbitmqctl set_user_tags jian administrator
http://172.16.17.131:15672/   Jian 123456
