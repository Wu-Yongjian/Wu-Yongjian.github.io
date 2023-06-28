---
layout: default
title: Yj
---

### dockerfile

```shell
[root@localhost dockerfile]# cat dockerfile 
ARG MYSQLUSER=mysql

FROM centos:7
LABEL maintainer="yongjian.wu <469697232@qq.com>"

ONBUILD RUN echo "welcome user yongjian's mariadb image"

VOLUME /jian/data/db

ENV MYSQLHOME=/jian/data/db
ENV MYSQLUSER=root

RUN echo '[mariadb]' > /etc/yum.repos.d/mariadb.repo \
&& echo 'name = MariaDB' >> /etc/yum.repos.d/mariadb.repo \
&& echo 'baseurl = http://yum.mariadb.org/10.3/centos7-amd64' >> /etc/yum.repos.d/mariadb.repo \
&& echo 'gpgcheck=0' >> /etc/yum.repos.d/mariadb.repo \
&& yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm \
&& yum install -y MariaDB-server MariaDB-devel MariaDB-backup galera which socat bc sysvinit-tools \
&& sed -i '/\[mysqld\]/a server-id=1\nlog-bin=jian\nbinlog_format=ROW\ngtid_domain_id=10' /etc/my.cnf.d/server.cnf \
&& cp /etc/my.cnf.d/server.cnf /etc/my.cnf \
&& mkdir -p $MYSQLHOME 

WORKDIR /

COPY mysqlinitmaster-entrypoint.sh mysqlinitmaster-entrypoint.sh
ENTRYPOINT ["/mysqlinitmaster-entrypoint.sh"]
EXPOSE 3306
HEALTHCHECK --interval=5s --timeout=3s CMD netstat -antpl|grep 3306

```

### entrypoint

```shell

[root@localhost dockerfile]# cat mysqlinitmaster-entrypoint.sh 
#!/bin/bash
mysql_install_db --user=$MYSQLUSER --defaults-file=/etc/my.cnf --datadir=$MYSQLHOME
chown mysql:mysql -R /jian/data/db
/etc/init.d/mysql start
sleep 2
mysql -u${MYSQLUSER} -e  "create user jian@'%' identified by '123456';grant all privileges on *.* to jian@'%';reset master;"
/etc/init.d/mysql stop
mysqld_safe
/bin/bash
```

### 创建volumn
```shell
[root@localhost dockerfile]# mkdir /data
```

### 运行docker并连接数据库
```shell
[root@localhost dockerfile]# docker run -itd -v /data:/jian/data/db -p 3306:3306 

[root@localhost dockerfile]# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED        STATUS                           PORTS                                       NAMES
8b60f3eba8e8   jian-mariadb   "/mysqlinitmaster-en…"   23 hours ago   Up 1 second (health: starting)   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp   sweet_ritchie

[root@localhost dockerfile]# mysql -ujian -p -h192.168.198.136 -P3306
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 10
Server version: 10.3.39-MariaDB-log MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
+--------------------+
4 rows in set (0.003 sec)

```
