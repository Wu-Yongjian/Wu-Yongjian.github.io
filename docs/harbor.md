---
layout: default
title: Yj
---


## docker 创建网桥
```shell

[root@localhost ~]# docker network create --subnet=172.18.2.0/24 --driver=bridge test

[root@localhost ~]# docker run -itd --net=test --name=networktest centos:7
797c96e2190b33c568980723e8e812d2935a7843a749245743fd48da2547dd90


[root@localhost ~]# docker exec -it 797c96e2190b cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.18.2.2	797c96e2190b

[root@localhost ~]# ping 172.18.2.2
PING 172.18.2.2 (172.18.2.2) 56(84) bytes of data.
64 bytes from 172.18.2.2: icmp_seq=1 ttl=64 time=0.141 ms
64 bytes from 172.18.2.2: icmp_seq=2 ttl=64 time=0.248 ms
```



## harbor

[配置证书](#heading-ids)	
```shell
[root@localhost harbor]# openssl genrsa -out ca.key 4096
[root@localhost harbor]# ls
ca.key
[root@localhost harbor]# openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=jian.com" \
 -key ca.key \
 -out ca.crt
[root@localhost harbor]# ls
ca.crt  ca.key


[root@localhost harbor]# openssl genrsa -out jian.com.key 4096
[root@localhost harbor]# ls
ca.crt  ca.key  jian.com.key
[root@localhost harbor]# openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=jian.com" \
    -key jian.com.key \
    -out jian.com.csr
[root@localhost harbor]# ls
ca.crt  ca.key  jian.com.csr  jian.com.key


[root@localhost harbor]# cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=jian.com
DNS.2=jian
DNS.3=harborserver
EOF
[root@localhost harbor]# ls
ca.crt  ca.key  jian.com.csr  jian.com.key  v3.ext
[root@localhost harbor]# openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in jian.com.csr \
    -out jian.com.crt
Certificate request self-signature ok
subject=C = CN, ST = Beijing, L = Beijing, O = example, OU = Personal, CN = jian.com
[root@localhost harbor]# ls
ca.crt  ca.key  ca.srl  jian.com.crt  jian.com.csr  jian.com.key  v3.ext

[root@localhost harbor]# openssl x509 -inform PEM -in jian.com.crt -out jian.com.cert
[root@localhost harbor]# ls
ca.crt  ca.key  ca.srl  jian.com.cert  jian.com.crt  jian.com.csr  jian.com.key  v3.ext
```

[修改配置文件](#heading-ids)	
```shell
[root@localhost harbor]# cp harbor.yml.tmpl harbor.yml

[root@localhost harbor]# vi harbor.yml
hostname: reg.jian.com
https:
  port: 8443

  certificate: /opt/harbor
  private_key: /opt/harbor
data_volume: /opt/harbor/data

[root@localhost harbor]# mkdir /opt/harbor/data
···

[停止](#heading-ids)	
```shell
[root@localhost harbor]# ./install.sh 



[root@localhost harbor]# docker-compose down -v
[root@localhost harbor]# docker-compose up -d


[root@localhost harbor]# docker login jian.com:8443
Username: admin
Password: Harbor12345


[root@localhost harbor]# docker tag centos:7 jian.com:8443/yjtest/centos:7

[root@localhost harbor]# docker push jian.com:8443/yjtest/centos:7
The push refers to repository [jian.com:8443/yjtest/centos]
174f56854903: Pushed 
7: digest: sha256:dead07b4d8ed7e29e98de0f4504d87e8880d4347859d839686a31da35a3b532f size: 529
[root@localhost harbor]# docker pull jian.com:8443/yjtest/centos:7
7: Pulling from yjtest/centos
Digest: sha256:dead07b4d8ed7e29e98de0f4504d87e8880d4347859d839686a31da35a3b532f
Status: Image is up to date for jian.com:8443/yjtest/centos:7
jian.com:8443/yjtest/centos:7



docker 忽略证书
/etc/docker/daemon.json
{
  "insecure-registries": ["jian.com"]
}


docker daemon 方式加载证书
mkdir -p /etc/docker/certs.d/jian.com
cp ca.crt /etc/docker/certs.d/jian.com
systemctl restart docker


os级别加载证书
cp ca.crt /etc/pki/ca-trust/source/anchors
update-ca-trust extract
systemctl restart docker


[root@localhost harbor]# yum install haproxy

[root@localhost harbor]# cat /etc/haproxy/haproxy.cfg
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend http-in
    bind *:90
    mode http
    default_backend servers

backend servers
    mode http
    balance roundrobin
    server server1 192.168.198.136:80 check
    server server2 192.168.198.140:80  check

[root@localhost harbor]# systemctl restart haproxy






