---
layout: default
title: Yj
---


## 安装kibana
```shell

[root@localhost opt]# ls kibana-7.17.10-linux-aarch64/
bin  config  data  LICENSE.txt  node  node_modules  NOTICE.txt  package.json  plugins  README.txt  src  x-pack
[root@localhost opt]# cat kibana-7.17.10-linux-aarch64/config/kibana.yml |grep -v '#'
server.port: 5601
server.host: "172.16.17.131"
elasticsearch.hosts: ["http://172.16.17.131:9200"]
[root@localhost opt]# cat /usr/lib/systemd/system/kibana.service
[Unit]
Description=Kibana
Documentation=https://www.elastic.co/guide/en/kibana/current/index.html
Wants=network-online.target
After=network-online.target

[Service]
Environment=NODE_OPTIONS="--max-old-space-size=4096"
Environment=NODE_ENV=production
EnvironmentFile=-/etc/default/kibana
WorkingDirectory=/opt/kibana-7.17.10-linux-aarch64
User=jian
Group=jian
ExecStart=/opt/kibana-7.17.10-linux-aarch64/bin/kibana
StandardOutput=journal
StandardError=inherit
LimitNOFILE=65536
TimeoutStopSec=20
TimeoutStartSec=20
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

```


## 安装elasticsearch
```shell
[root@localhost opt]# cat elasticsearch-7.17.10/config/elasticsearch.yml |grep -v '#'
network.host: 172.16.17.131
http.port: 9200
discovery.seed_hosts: ["172.16.17.131", "127.0.0.1"]
cluster.initial_master_nodes: ["node-1"]
[root@localhost opt]# cat /usr/lib/systemd/system/elasticsearch.service
[Unit]
Description=Elasticsearch
Documentation=https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
Wants=network-online.target
After=network-online.target

[Service]
Environment=JAVA_HOME=/opt/jdk1.8
Environment=ES_HOME=/opt/elasticsearch-7.17.10
Environment=ES_PATH_CONF=/opt/elasticsearch-7.17.10/config
EnvironmentFile=-/etc/default/elasticsearch
User=jian
Group=jian
ExecStart=/opt/elasticsearch-7.17.10/bin/elasticsearch -p /opt/elasticsearch-7.17.10/elasticsearch.pid --quiet
StandardOutput=journal
StandardError=inherit
LimitMEMLOCK=infinity
LimitNOFILE=65536
TimeoutStopSec=20
TimeoutStartSec=20
Restart=always
RestartSec=10
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target

```


## 安装filebeat
```shell
[root@localhost opt]# ls /opt/filebeat-7.17.0/
data  fields.yml  filebeat  filebeat.reference.yml  filebeat.yml  filebeat.yml.bak  filebeat.yml.es  kibana  LICENSE.txt  logs  module  modules.d  NOTICE.txt  README.md
[root@localhost filebeat-7.17.0]# cat filebeat.yml
filebeat.inputs:
- type: log
  paths:
    - /var/log/messages
  include_lines: ['error', 'exception', 'fatal']  # 字符串列表，匹配包含 'error'、'exception'、'fatal' 的日志行
- type: log
  paths:
    - "/opt/apache-tomcat-9.0.70/logs/localhost_access_log.2023-07-*"

output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["172.16.17.131:9200"]

setup.template.name: "filebeat-message"  # 指定索引模板名称
setup.template.pattern: "filebeat-message-*"  # 指定匹配索引的模式

setup.ilm.enabled: false  # 禁用索引生命周期管理

[root@localhost opt]# cat  /usr/lib/systemd/system/filebeat.service
[Unit]
Description=Filebeat
Documentation=https://www.elastic.co/products/beats/filebeat
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/opt/filebeat-7.17.0/filebeat -c /opt/filebeat-7.17.0/filebeat.yml
Restart=always

[Install]
WantedBy=multi-user.target
```


## 启动服务
```shell
Systemctl start Kibana
Systemctl start elastic search
Systemctl start filbeat
```


## filebeat定义索引名
```shell
filebeat.inputs:
  - type: log
    paths:
      - /var/log/messages
    fields:
      input_type: system_logs

  - type: log
    paths:
      - /path/to/application_logs.log
    fields:
      input_type: application_logs

output.elasticsearch:
  hosts: ["your_elasticsearch_host:9200"]
  index: >
    {%- if "system_logs" in [fields.input_type] -%}
      system-logs-%{+yyyy.MM.dd}
    {%- elseif "application_logs" in [fields.input_type] -%}
      application-logs-%{+yyyy.MM.dd}
    {%- else -%}
      other-logs-%{+yyyy.MM.dd}
    {%- endif -%}


```




## 安装mongodb

master 目录位置 mongodb-master
slave  目录位置 mongodb-slave
arbiter 目录位置 mongodb-arbiter
```shell
 [root@localhost opt]# wget https://fastdl.mongodb.org/linux/mongodb-linux-aarch64-rhel90-6.0.8.tgz
 [root@localhost opt]# tar zxf mongodb-linux-aarch64-rhel90-6.0.8.tgz
 [root@localhost opt]# mv mongodb-linux-aarch64-rhel90-6.0.8/ mongodb-master
 [root@localhost opt]# mkdir mongodb-master/data
 [root@localhost opt]# cp -ra mongodb-master/ mongodb-slave
 [root@localhost opt]# cp -ra mongodb-master/ mongodb-arbiter
 [root@localhost mongodb-master]# pwd
 /opt/mongodb-master
 [root@localhost opt]# wget https://downloads.mongodb.com/compass/mongosh-1.10.3-linux-arm64.tgz
 [root@localhost opt]# tar zxf mongosh-1.10.3-linux-arm64.tgz
 [root@localhost opt]# cp mongosh-1.10.3-linux-arm64/bin/mongosh /opt/mongodb-master/bin/
 [root@localhost opt]# cp mongosh-1.10.3-linux-arm64/bin/mongosh /opt/mongodb-slave/bin/
 [root@localhost opt]# cp mongosh-1.10.3-linux-arm64/bin/mongosh /opt/mongodb-arbiter/bin/
```
## 准备配置文件
```shell
 [root@localhost mongodb-master]# cat mongo.conf
 #master.conf
 dbpath=/opt/mongodb-master/data
 logpath=/opt/mongodb-master/master.log
 pidfilepath=/opt/mongodb-master/master.pid
 directoryperdb=true
 logappend=true
 replSet=jianrs
 bind_ip=0.0.0.0
 port=28017
 oplogSize=10000
 fork=true
 [root@localhost mongodb-slave]# pwd
 /opt/mongodb-slave
 [root@localhost mongodb-slave]# cat mongo.conf
 #slave.conf
 dbpath=/opt/mongodb-slave/data
 logpath=/opt/mongodb-slave/slave.log
 pidfilepath=/opt/mongodb-slave/slave.pid
 directoryperdb=true
 logappend=true
 replSet=jianrs
 bind_ip=0.0.0.0
 port=28018
 oplogSize=10000
 fork=true
 [root@localhost mongodb-arbiter]# pwd
 /opt/mongodb-arbiter
 [root@localhost mongodb-arbiter]# cat mongo.conf
 #arbiter.conf
 dbpath=/opt/mongodb-arbiter/data
 logpath=/opt/mongodb-arbiter/arbiter.log
 pidfilepath=/opt/mongodb-arbiter/arbiter.pid
 directoryperdb=true
 logappend=true
 replSet=jianrs
 bind_ip=0.0.0.0
 port=28019
 oplogSize=10000
 fork=true

```

## 启动服务
```shell
 [root@localhost opt]# /opt/mongodb-master/bin/mongod --config /opt/mongodb-master/mongo.conf
 about to fork child process, waiting until server is ready for connections.
 forked process: 15399
 child process started successfully, parent exiting
 [root@localhost opt]# /opt/mongodb-slave/bin/mongod --config /opt/mongodb-slave/mongo.conf
 about to fork child process, waiting until server is ready for connections.
 forked process: 15468
 child process started successfully, parent exiting
 [root@localhost opt]# /opt/mongodb-arbiter/bin/mongod --config /opt/mongodb-arbiter/mongo.conf
 about to fork child process, waiting until server is ready for connections.
 forked process: 15531
 child process started successfully, parent exiting

```


## 配置集群
```shell
[root@localhost mongodb-master]# /opt/mongodb-master/bin/mongosh 172.16.17.131:28017
test> use admin
switched to db admin

admin> config={_id:"jianrs",members:[{_id:0,host:'172.16.17.131:28017',priority:2},{_id:1,host:'172.16.17.131:28018',priority:1},{_id:2,host:'172.16.17.131:28019',arbiterOnly:true}]};
{
  _id: 'jianrs',
  members: [
    { _id: 0, host: '172.16.17.131:28017', priority: 2 },
    { _id: 1, host: '172.16.17.131:28018', priority: 1 },
    { _id: 2, host: '172.16.17.131:28019', arbiterOnly: true }
  ]
}
admin> rs.initiate(config)
{ ok: 1 }


jianrs [direct: secondary] admin> rs.status()
{
  set: 'jianrs',
  date: ISODate("2023-08-04T08:15:42.636Z"),
  myState: 1,
  term: Long("1"),
  syncSourceHost: '',
  syncSourceId: -1,
  heartbeatIntervalMillis: Long("2000"),
  majorityVoteCount: 2,
  writeMajorityCount: 2,
  votingMembersCount: 3,
  writableVotingMembersCount: 2,
  optimes: {
    lastCommittedOpTime: { ts: Timestamp({ t: 1691136936, i: 1 }), t: Long("1") },
    lastCommittedWallTime: ISODate("2023-08-04T08:15:36.120Z"),
    readConcernMajorityOpTime: { ts: Timestamp({ t: 1691136936, i: 1 }), t: Long("1") },
    appliedOpTime: { ts: Timestamp({ t: 1691136936, i: 1 }), t: Long("1") },
    durableOpTime: { ts: Timestamp({ t: 1691136936, i: 1 }), t: Long("1") },
    lastAppliedWallTime: ISODate("2023-08-04T08:15:36.120Z"),
    lastDurableWallTime: ISODate("2023-08-04T08:15:36.120Z")
  },
  lastStableRecoveryTimestamp: Timestamp({ t: 1691136896, i: 1 }),
  electionCandidateMetrics: {
    lastElectionReason: 'electionTimeout',
    lastElectionDate: ISODate("2023-08-04T08:14:16.058Z"),
    electionTerm: Long("1"),
    lastCommittedOpTimeAtElection: { ts: Timestamp({ t: 1691136845, i: 1 }), t: Long("-1") },
    lastSeenOpTimeAtElection: { ts: Timestamp({ t: 1691136845, i: 1 }), t: Long("-1") },
    numVotesNeeded: 2,
    priorityAtElection: 2,
    electionTimeoutMillis: Long("10000"),
    numCatchUpOps: Long("0"),
    newTermStartDate: ISODate("2023-08-04T08:14:16.102Z"),
    wMajorityWriteAvailabilityDate: ISODate("2023-08-04T08:14:16.677Z")
  },
  members: [
    {
      _id: 0,
      name: '172.16.17.131:28017',
      health: 1,
      state: 1,
      stateStr: 'PRIMARY',
      uptime: 1051,
      optime: { ts: Timestamp({ t: 1691136936, i: 1 }), t: Long("1") },
      optimeDate: ISODate("2023-08-04T08:15:36.000Z"),
      lastAppliedWallTime: ISODate("2023-08-04T08:15:36.120Z"),
      lastDurableWallTime: ISODate("2023-08-04T08:15:36.120Z"),
      syncSourceHost: '',
      syncSourceId: -1,
      infoMessage: 'Could not find member to sync from',
      electionTime: Timestamp({ t: 1691136856, i: 1 }),
      electionDate: ISODate("2023-08-04T08:14:16.000Z"),
      configVersion: 1,
      configTerm: 1,
      self: true,
      lastHeartbeatMessage: ''
    },
    {
      _id: 1,
      name: '172.16.17.131:28018',
      health: 1,
      state: 2,
      stateStr: 'SECONDARY',
      uptime: 97,
      optime: { ts: Timestamp({ t: 1691136936, i: 1 }), t: Long("1") },
      optimeDurable: { ts: Timestamp({ t: 1691136936, i: 1 }), t: Long("1") },
      optimeDate: ISODate("2023-08-04T08:15:36.000Z"),
      optimeDurableDate: ISODate("2023-08-04T08:15:36.000Z"),
      lastAppliedWallTime: ISODate("2023-08-04T08:15:36.120Z"),
      lastDurableWallTime: ISODate("2023-08-04T08:15:36.120Z"),
      lastHeartbeat: ISODate("2023-08-04T08:15:42.158Z"),
      lastHeartbeatRecv: ISODate("2023-08-04T08:15:41.167Z"),
      pingMs: Long("0"),
      lastHeartbeatMessage: '',
      syncSourceHost: '172.16.17.131:28017',
      syncSourceId: 0,
      infoMessage: '',
      configVersion: 1,
      configTerm: 1
    },
    {
      _id: 2,
      name: '172.16.17.131:28019',
      health: 1,
      state: 7,
      stateStr: 'ARBITER',
      uptime: 97,
      lastHeartbeat: ISODate("2023-08-04T08:15:42.158Z"),
      lastHeartbeatRecv: ISODate("2023-08-04T08:15:42.157Z"),
      pingMs: Long("0"),
      lastHeartbeatMessage: '',
      syncSourceHost: '',
      syncSourceId: -1,
      infoMessage: '',
      configVersion: 1,
      configTerm: 1
    }
  ],
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1691136936, i: 1 }),
    signature: {
      hash: Binary(Buffer.from("0000000000000000000000000000000000000000", "hex"), 0),
      keyId: Long("0")
    }
  },
  operationTime: Timestamp({ t: 1691136936, i: 1 })
}
```

##  开启slave节点可读 
```shell
[root@localhost mongodb-master]# /opt/mongodb-master/bin/mongosh 172.16.17.131:28018
Current Mongosh Log ID:	64ccb41444579cab268db88d
Connecting to:		mongodb://172.16.17.131:28018/?directConnection=true&appName=mongosh+1.10.3
Using MongoDB:		6.0.8
Using Mongosh:		1.10.3

For mongosh info see: https://docs.mongodb.com/mongodb-shell/

------
   The server generated these startup warnings when booting
   2023-08-04T16:58:31.192+09:00: Access control is not enabled for the database. Read and write access to data and configuration is unrestricted
   2023-08-04T16:58:31.192+09:00: You are running this process as the root user, which is not recommended
   2023-08-04T16:58:31.192+09:00: /sys/kernel/mm/transparent_hugepage/enabled is 'always'. We suggest setting it to 'never'
   2023-08-04T16:58:31.192+09:00: Soft rlimits for open file descriptors too low
------

jianrs [direct: secondary] test> db.getMongo().setSecondaryOk()
DeprecationWarning: .setSecondaryOk() is deprecated. Use .setReadPref("primaryPreferred") instead
Setting read preference from "primary" to "primaryPreferred"
```


## 创建管理员
```shel
登录master节点：mongo 127.0.0.1:28017
> use admin
> db.createUser({user:"admin",pwd: "admin",roles:[{role:"root",db:"admin"}]});
> db.grantRolesToUser("admin", ["clusterAdmin"])  #需要给admin用户赋予集群权限，不然后面执行rs.status()会报错："errmsg" : "not authorized on admin to execute command { replSetGetStatus:......
> db.auth("admin", "admin")
> db.system.users.find()
```


