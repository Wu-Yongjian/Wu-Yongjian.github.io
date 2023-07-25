
---
layout: default
title: Yj
---


## 准备

6个节点或者端口分别安装redis软件，并启动redis
## 配置文件

```shell
[root@localhost system]# cat /opt/redis-cluster1/etc/redis.conf 
bind 0.0.0.0
port 6381
masterauth 123456 
#建议配置，否则后期的master和slave主从复制无法成功，还需再配置
requirepass 123456
cluster-enabled yes 
#取消此行注释,必须开启集群，开启后 redis 进程会有cluster标识
cluster-config-file /opt/redis-cluster1/nodes-6381.conf 
#取消此行注释,此为集群状态数据文件,记录主从关系及slot范围信息,由redis cluster 集群自动创建和维护
cluster-require-full-coverage no 
#默认值为yes,设为no可以防止一个节点不可用导致整个cluster不可用
```
注意修改每个节点对应的redis软件配置文件位置与端口号
##  service 文件
```shell
[root@localhost system]# cat /usr/lib/systemd/system/rediscluster1.service 
[Unit]
Description=Redis persistent key-value database
After=network.target
[Service]
ExecStart=/opt/redis-cluster1/bin/redis-server /opt/redis-cluster1/etc/redis.conf --supervised systemd
ExecStop=/bin/kill -s QUIT $MAINPID
Type=notify 
#如果支持systemd可以启用此行
User=redis
Group=redis
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
LimitNOFILE=1000000 
#指定此值才支持更大的maxclients值
[Install]
WantedBy=multi-user.target
[root@localhost system]# systemctl start rediscluster
Job for rediscluster.service failed because the control process exited with error code.
See "systemctl status rediscluster.service" and "journalctl -xeu rediscluster.service" for details.
[root@localhost system]# vi /var/log/messages
[root@localhost system]# vi /opt/redis-cluster/etc/redis.conf 
```

##  3 检查端口16379
```shell
[root@localhost system]# systemctl start rediscluster1.service 
[root@localhost system]# systemctl start rediscluster2.service 
[root@localhost system]# systemctl start rediscluster3.service 
[root@localhost system]# systemctl start rediscluster4.service 
[root@localhost system]# systemctl start rediscluster4.service 
[root@localhost system]# systemctl start rediscluster5.service 
[root@localhost system]# systemctl start rediscluster6.service 
[root@localhost system]# netstat -antpl|grep 638
tcp        0      0 0.0.0.0:16382           0.0.0.0:*               LISTEN      3978/redis-server 0 
tcp        0      0 0.0.0.0:16383           0.0.0.0:*               LISTEN      3985/redis-server 0 
tcp        0      0 0.0.0.0:16381           0.0.0.0:*               LISTEN      3971/redis-server 0 
tcp        0      0 0.0.0.0:6382            0.0.0.0:*               LISTEN      3978/redis-server 0 
tcp        0      0 0.0.0.0:6383            0.0.0.0:*               LISTEN      3985/redis-server 0 
tcp        0      0 0.0.0.0:6381            0.0.0.0:*               LISTEN      3971/redis-server 0 
tcp        0      0 0.0.0.0:6386            0.0.0.0:*               LISTEN      4008/redis-server 0 
tcp        0      0 0.0.0.0:6384            0.0.0.0:*               LISTEN      3992/redis-server 0 
tcp        0      0 0.0.0.0:6385            0.0.0.0:*               LISTEN      4001/redis-server 0 
tcp        0      0 0.0.0.0:16386           0.0.0.0:*               LISTEN      4008/redis-server 0 
tcp        0      0 0.0.0.0:16384           0.0.0.0:*               LISTEN      3992/redis-server 0 
tcp        0      0 0.0.0.0:16385           0.0.0.0:*               LISTEN      4001/redis-server 0
```

##  创建集群
```shell
[root@localhost system]# redis-cli -a 123456 --cluster create 192.168.198.136:6381 192.168.198.136:6382 192.168.198.136:6383 192.168.198.136:6384 192.168.198.136:6385 192.168.198.136:6386 --cluster-replicas 1
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.198.136:6385 to 192.168.198.136:6381
Adding replica 192.168.198.136:6386 to 192.168.198.136:6382
Adding replica 192.168.198.136:6384 to 192.168.198.136:6383
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: de66af541b69da76bc9d970141930bc087ed5ca2 192.168.198.136:6381
   slots:[0-5460] (5461 slots) master
M: 09f0a4579b4e908e94cb6a1679d3375ffc8031c6 192.168.198.136:6382
   slots:[5461-10922] (5462 slots) master
M: 6a52c0a5457e37fda9b4ab38a6ce511b39b738d0 192.168.198.136:6383
   slots:[10923-16383] (5461 slots) master
S: abea2f84935dd416742a98804ae78c74ccb34d69 192.168.198.136:6384
   replicates 09f0a4579b4e908e94cb6a1679d3375ffc8031c6
S: 39af7e7392368f74c1989b661752d5fc7af32ad8 192.168.198.136:6385
   replicates 6a52c0a5457e37fda9b4ab38a6ce511b39b738d0
S: 34e16a8b93946f22b95a8ad147ab6e671009cab5 192.168.198.136:6386
   replicates de66af541b69da76bc9d970141930bc087ed5ca2
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join

>>> Performing Cluster Check (using node 192.168.198.136:6381)
M: de66af541b69da76bc9d970141930bc087ed5ca2 192.168.198.136:6381
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 34e16a8b93946f22b95a8ad147ab6e671009cab5 192.168.198.136:6386
   slots: (0 slots) slave
   replicates de66af541b69da76bc9d970141930bc087ed5ca2
S: abea2f84935dd416742a98804ae78c74ccb34d69 192.168.198.136:6384
   slots: (0 slots) slave
   replicates 09f0a4579b4e908e94cb6a1679d3375ffc8031c6
S: 39af7e7392368f74c1989b661752d5fc7af32ad8 192.168.198.136:6385
   slots: (0 slots) slave
   replicates 6a52c0a5457e37fda9b4ab38a6ce511b39b738d0
M: 6a52c0a5457e37fda9b4ab38a6ce511b39b738d0 192.168.198.136:6383
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
M: 09f0a4579b4e908e94cb6a1679d3375ffc8031c6 192.168.198.136:6382
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

##  查看集群状态
```shell
[root@localhost system]# redis-cli -a 123456 -p 6381 -h 192.168.198.136  CLUSTER NODES
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
34e16a8b93946f22b95a8ad147ab6e671009cab5 192.168.198.136:6386@16386 slave de66af541b69da76bc9d970141930bc087ed5ca2 0 1690288858529 1 connected
abea2f84935dd416742a98804ae78c74ccb34d69 192.168.198.136:6384@16384 slave 09f0a4579b4e908e94cb6a1679d3375ffc8031c6 0 1690288853479 2 connected
39af7e7392368f74c1989b661752d5fc7af32ad8 192.168.198.136:6385@16385 slave 6a52c0a5457e37fda9b4ab38a6ce511b39b738d0 0 1690288856000 3 connected
6a52c0a5457e37fda9b4ab38a6ce511b39b738d0 192.168.198.136:6383@16383 master - 0 1690288857000 3 connected 10923-16383
09f0a4579b4e908e94cb6a1679d3375ffc8031c6 192.168.198.136:6382@16382 master - 0 1690288857519 2 connected 5461-10922
de66af541b69da76bc9d970141930bc087ed5ca2 192.168.198.136:6381@16381 myself,master - 0 1690288853000 1 connected 0-5460
```



##  配置nginx
```shell
两台机器做相同的操作
添加测试页面
[root@nginx1 system]# cat /var/www/html/index.html 
<!DOCTYPE html>
<html>
<head>
    <title>Welcome to My Website</title>
</head>
<body>
    <h1>Hello, Welcome to My Website!</h1>
    <p>This is the default page served by Nginx.</p>
</body>
</html>
[root@nginx1 system]# cat /etc/nginx/nginx.conf|grep root
        root         /var/www/html;
[root@nginx1 system]# yum install nginx
[root@nginx1 system]# vi /etc/nginx/nginx.conf
[root@nginx1 system]# systemctl restart nginx
访问测试
[root@nginx1 system]# curl 192.168.198.136
<!DOCTYPE html>
<html>
<head>
    <title>Welcome to My Website</title>
</head>
<body>
    <h1>Hello, Welcome to My Website!</h1>
    <p>This is the default page served by Nginx.</p>
</body>
</html>

```

##  配置keepalived
```shell
nginx1-2
[root@nginx1 ]# yum install keepalived

nginx1
[root@nginx1 ]# cat /etc/keepalived/keepalived.conf
global_defs {
    router_id keepalived_hap
}
vrrp_script check-haproxy {
    script "killall -0 haproxy"
    interval 5
    weight -30
}
vrrp_instance VI-nginx {
    state MASTER
    priority 120
    dont_track_primary
    interface ens33
    virtual_router_id 68
    advert_int 3
    track_script {
        check-haproxy
    }
    virtual_ipaddress {
        192.168.198.10
    }
}

nginx2
[root@nginx1 ~]# cat /etc/keepalived/keepalived.conf
global_defs {
    router_id keepalived_hap
}
vrrp_script check-haproxy {
    script "killall -0 haproxy"
    interval 5
    weight -30
}
vrrp_instance VI-nginx {
    state BACKUP
    priority 100
    dont_track_primary
    interface ens33
    virtual_router_id 68
    advert_int 3
    track_script {
        check-haproxy
    }
    virtual_ipaddress {
        192.168.198.10
    }
}
```

## 启动服务并测试访问
``` shell
[root@nginx1]# systemctl start keepalived
[root@nginx2]# systemctl start keepalived


[root@nginx1 system]# curl 192.168.198.10
<!DOCTYPE html>
<html>
<head>
    <title>Welcome to My Website</title>
</head>
<body>
    <h1>Hello, Welcome to My Website!</h1>
    <p>This is the default page served by Nginx.</p>
</body>
</html>



