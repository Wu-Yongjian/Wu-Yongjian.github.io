---
layout: default
title: Yj
---






创建证书
1 所有节点安装docker关闭selinux
```shell

[root@localhost ~]# hostnamectl set-hostname kbs1

[root@localhost ~]# setenforce 0

[root@localhost ~]# vi /etc/sysconfig/selinux

[root@localhost ~]# swapoff -a

[root@localhost ~]# cat /etc/hosts

127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4

::1 localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.135.130 kbs1

192.168.135.131 kbs2

192.168.135.132 kbs3

[root@localhost ~]# mkdir /kbs/tls/ssl -p

[root@localhost ~]# mkdir /kbs/kubernets -p



[root@localhost kbs]# systemctl stop firewalld

[root@localhost kbs]# systemctl disable firewalld

Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.

Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.

[root@localhost kbs]# iptables -F
```

2 Master 节点
1 下载cfssl
```shell
[root@kbs1 tls]# wget

[root@kbs1 tls]# wget

[root@kbs1 tls]# wget

[root@localhost kbs]# chmod +x ./cfssl*

[root@localhost kbs]# cp cfssl-certinfo_linux-amd64 ./tls/cfssl-certinfo

[root@localhost kbs]# cp cfssljson_linux-amd64 ./tls/cfssljson

[root@localhost kbs]# cp cfssl_linux-amd64 ./tls/cfssl

[root@localhost kbs]# vi /root/.bash_profile

[root@localhost kbs]# source /root/.bash_profile

[root@localhost kbs]# cat /root/.bash_profile

# .bash_profile



# Get the aliases and functions

if [ -f ~/.bashrc ]; then

. ~/.bashrc

fi



# User specific environment and startup programs



PATH=$PATH:$HOME/bin:/kbs/tls



export PATH
```


2 创建模板文件
```shell
[root@kbs1 ssl]# cfssl print-defaults config > config.json

[root@kbs1 ssl]# cfssl print-defaults csr > csr.json
```
3 创建ca证书
1 ca证书配置文件
```shell
cat > ca-config.json <<EOF

{

"signing": {

"default": {

"expiry": "87600h"

},

"profiles": {

"kubernetes": {

"usages": [

"signing",

"key encipherment",

"server auth",

"client auth"

],

"expiry": "87600h"

}

}

}

}

EOF
```
2 ca证书签名请求

```shell
[root@kbs1 ssl]# vi ca-csr.json

{

"CN": "kubernetes",

"key": {

"algo": "rsa",

"size": 2048

},

"names": [

{

"C": "CN",

"ST": "BeiJing",

"L": "BeiJing",

"O": "k8s",

"OU": "System"

}

],

"ca": {

"expiry": "87600h"

}

}

```
3 生成ca证书和私钥
```shell
[root@kbs1 ssl]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca

2020/01/15 09:28:51 [INFO] generating a new CA key and certificate from CSR

2020/01/15 09:28:51 [INFO] generate received request

2020/01/15 09:28:51 [INFO] received CSR

2020/01/15 09:28:51 [INFO] generating key: rsa-2048

2020/01/15 09:28:51 [INFO] encoded CSR

2020/01/15 09:28:51 [INFO] signed certificate with serial number 430310401537449018603772405914429363381330908792

[root@kbs1 ssl]# ls

ca-config.json ca.csr ca-csr.json ca-key.pem ca.pem config.json csr.json
```
4 创建kubernets签名请求
```shell
[root@kbs1 ssl]# vi kubernetes-csr.json

[root@kbs1 ssl]# cat kubernetes-csr.json

{

"CN": "kubernetes",

"hosts": [

"127.0.0.1",

"192.168.135.130",

"192.168.135.131",

"192.168.135.132",

"172.16.0.1",

"kubernetes",

"kubernetes.default",

"kubernetes.default.svc",

"kubernetes.default.svc.cluster",

"kubernetes.default.svc.cluster.local"

],

"key": {

"algo": "rsa",

"size": 2048

},

"names": [

{

"C": "CN",

"ST": "BeiJing",

"L": "BeiJing",

"O": "k8s",

"OU": "System"

}

]

}
```
5 生成kubernets证书和私钥
```shell
[root@kbs1 ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes

2020/01/15 09:32:47 [INFO] generate received request

2020/01/15 09:32:47 [INFO] received CSR

2020/01/15 09:32:47 [INFO] generating key: rsa-2048

2020/01/15 09:32:48 [INFO] encoded CSR

2020/01/15 09:32:48 [INFO] signed certificate with serial number 583519535942596979684191213937643094654972057812

2020/01/15 09:32:48 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for

websites. For more information see the Baseline Requirements for the Issuance and Management

of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);

specifically, section 10.2.3 ("Information Requirements").

[root@kbs1 ssl]# ls kubernetes*

kubernetes.csr kubernetes-csr.json kubernetes-key.pem kubernetes.pem
```
6 创建admin签名请求


```shell
[root@kbs1 ssl]# vi admin-csr.json

[root@kbs1 ssl]# cat admin-csr.json

{

"CN": "admin",

"hosts": [],

"key": {

"algo": "rsa",

"size": 2048

},

"names": [

{

"C": "CN",

"ST": "BeiJing",

"L": "BeiJing",

"O": "system:masters",

"OU": "System"

}

]

}
```
7 生成admin证书和私钥
```shell
[root@kbs1 ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

2020/01/15 09:34:33 [INFO] generate received request

2020/01/15 09:34:33 [INFO] received CSR

2020/01/15 09:34:33 [INFO] generating key: rsa-2048

2020/01/15 09:34:33 [INFO] encoded CSR

2020/01/15 09:34:33 [INFO] signed certificate with serial number 377510730900308312316290521080158345526166041001

2020/01/15 09:34:33 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for

websites. For more information see the Baseline Requirements for the Issuance and Management

of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);

specifically, section 10.2.3 ("Information Requirements").

[root@kbs1 ssl]# ls admin

admin.csr admin-csr.json admin-key.pem admin.pem
```

8 创kube-proxy签名请求
```shell
[root@kbs1 ssl]# vi kube-proxy-csr.json

[root@kbs1 ssl]# cat kube-proxy-csr.json

{

"CN": "system:kube-proxy",

"hosts": [],

"key": {

"algo": "rsa",

"size": 2048

},

"names": [

{

"C": "CN",

"ST": "BeiJing",

"L": "BeiJing",

"O": "k8s",

"OU": "System"

}

]

}
```
9 生成kube-proxy签名请求
```shell
[root@kbs1 ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

2020/01/15 09:35:45 [INFO] generate received request

2020/01/15 09:35:45 [INFO] received CSR

2020/01/15 09:35:45 [INFO] generating key: rsa-2048

2020/01/15 09:35:46 [INFO] encoded CSR

2020/01/15 09:35:46 [INFO] signed certificate with serial number 630500717240413048510961254520350338056263994089

2020/01/15 09:35:46 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for

websites. For more information see the Baseline Requirements for the Issuance and Management

of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);

specifically, section 10.2.3 ("Information Requirements").

[root@kbs1 ssl]# ls kube-proxy

kube-proxy.csr kube-proxy-csr.json kube-proxy-key.pem kube-proxy.pem
```

4 分发ca证书
```
[root@localhost ~]# mkdir /kbs/tls/ssl -p

[root@localhost ~]# scp /kbs/tls/ssl/*.pem 192.168.135.131:/kbs/tls/ssl/

[root@localhost ~]# scp /kbs/tls/ssl/*.pem 192.168.135.132:/kbs/tls/ssl/
```
创建 kubeconfig 文件
0 kubectl config
Kubectl 下载地址：


```shell
[root@localhost kbs]# ls

cfssl-certinfo_linux-amd64 kubernetes-server-linux-amd64.tar.gz

cfssljson_linux-amd64 kubernets

cfssl_linux-amd64 tls

[root@kbs1 kbs]# tar zxvf kubernetes-server-linux-amd64.tar.gz

[root@kbs1 kbs]# ls /kbs/kubernetes/server/bin/kubectl

/kbs/kubernetes/server/bin/kubectl

[root@localhost bin]# cat /root/.bash_profile

# .bash_profile



# Get the aliases and functions

if [ -f ~/.bashrc ]; then

. ~/.bashrc

fi



# User specific environment and startup programs



PATH=$PATH:$HOME/bin:/kbs/tls:/kbs/kubernetes/server/bin



export PATH

[root@localhost bin]# source /root/.bash_profile

[root@kbs1 ~]# export KUBE_APISERVER=""

[root@localhost bin]# kubectl config set-cluster kubernetes --certificate-authority=/kbs/tls/ssl/ca.pem --embed-certs=true --server=${KUBE_APISERVER}

Cluster "kubernetes" set.

[root@localhost bin]# kubectl config set-credentials admin --client-certificate=/kbs/tls/ssl/admin.pem --embed-certs=true --client-key=/kbs/tls/ssl/admin-key.pem

User "admin" set.

[root@localhost bin]# kubectl config set-context kubernetes --cluster=kubernetes --user=admin

Context "kubernetes" created.

[root@kbs1 ~]# kubectl config use-context kubernetes

Switched to context "kubernetes".

```

1 bootstrap.kubeconfig
```shell
[root@kbs1 kubectl]#export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')

[root@kbs1 kubectl]#cat > token.csv <<EOF

${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"

EOF

[root@kbs1 kubectl]#export KUBE_APISERVER=""

[root@localhost ssl]# cat token.csv

aa5622bd6249cb663df508585d26e452,kubelet-bootstrap,10001,"system:kubelet-bootstrap"

[root@kbs1 kubectl]# ls /kbs/token/token.csv

/kbs/token/token.csv

# 设置集群参数

[root@localhost ssl]# kubectl config set-cluster kubernetes \

> --certificate-authority=/kbs/tls/ssl/ca.pem \

> --embed-certs=true \

> --server=${KUBE_APISERVER} \

> --kubeconfig=bootstrap.kubeconfig

Cluster "kubernetes" set.



# 设置客户端认证参数

[root@localhost ssl]# kubectl config set-credentials kubelet-bootstrap \

> --token=${BOOTSTRAP_TOKEN} \

> --kubeconfig=bootstrap.kubeconfig

User "kubelet-bootstrap" set.



# 设置上下文参数

[root@localhost ssl]# kubectl config set-context default \

> --cluster=kubernetes \

> --user=kubelet-bootstrap \

> --kubeconfig=bootstrap.kubeconfig

Context "default" created.



# 设置默认上下文

[root@localhost ssl]# kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

Switched to context "default".

[root@kbs1 kubectl]# pwd

/kbs/kubectl

[root@localhost ssl]# ls bootstrap.kubeconfig

bootstrap.kubeconfig

[root@localhost ssl]# pwd

/kbs/tls/ssl

```

2 kube-proxy.kubeconfig
```shell
export KUBE_APISERVER=""

# 设置集群参数

[root@localhost ssl]# kubectl config set-cluster kubernetes \

> --certificate-authority=/kbs/tls/ssl/ca.pem \

> --embed-certs=true \

> --server=${KUBE_APISERVER} \

> --kubeconfig=kube-proxy.kubeconfig

Cluster "kubernetes" set.



# 设置客户端认证参数

[root@localhost ssl]# kubectl config set-credentials kube-proxy \

> --client-certificate=/kbs/tls/ssl/kube-proxy.pem \

> --client-key=/kbs/tls/ssl/kube-proxy-key.pem \

> --embed-certs=true \

> --kubeconfig=kube-proxy.kubeconfig

User "kube-proxy" set.



# 设置上下文参数

[root@localhost ssl]# kubectl config set-context default \

> --cluster=kubernetes \

> --user=kube-proxy \

> --kubeconfig=kube-proxy.kubeconfig

Context "default" created.



# 设置默认上下文

[root@localhost ssl]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

Switched to context "default".

[root@localhost ssl]# ls kube-proxy.kubeconfig

kube-proxy.kubeconfig
```

3 分发文件至node节点
```shell
[root@localhost ssl]# scp ./*.kubeconfig 192.168.135.196:/kbs/tls/ssl/

root@192.168.135.196's password:

Permission denied, please try again.

root@192.168.135.196's password:

bootstrap.kubeconfig 100% 2169 7.2KB/s 00:00

kube-proxy.kubeconfig 100% 6271 26.3KB/s 00:00

[root@localhost ssl]# scp ./*.kubeconfig 192.168.135.197:/kbs/tls/ssl/

root@192.168.135.197's password:

bootstrap.kubeconfig 100% 2169 678.4KB/s 00:00

kube-proxy.kubeconfig 100% 6271 301.8KB/s 00:00
```


etcd
etcd 下载地址：


```shell
[root@localhost kbs]# ls

cfssl-certinfo_linux-amd64 etcd-v3.1.5-linux-amd64.tar.gz tls

cfssljson_linux-amd64 kubernetes

cfssl_linux-amd64 kubernetes-server-linux-amd64.tar.gz

[root@localhost kbs]# tar zxf etcd-v3.1.5-linux-amd64.tar.gz

[root@localhost kbs]# ls

cfssl-certinfo_linux-amd64 etcd-v3.1.5-linux-amd64.tar.gz

cfssljson_linux-amd64 kubernetes

cfssl_linux-amd64 kubernetes-server-linux-amd64.tar.gz

etcd-v3.1.5-linux-amd64 tls

[root@localhost kbs]# mv ./etcd-v3.1.5-linux-amd64 etcd

[root@localhost kbs]# ls

cfssl-certinfo_linux-amd64 etcd-v3.1.5-linux-amd64.tar.gz

cfssljson_linux-amd64 kubernetes

cfssl_linux-amd64 kubernetes-server-linux-amd64.tar.gz

etcd tls

[root@kbs1 ~]# source .bash_profile

[root@kbs1 ~]# cat /root/.bash_profile

# .bash_profile



# Get the aliases and functions

if [ -f ~/.bashrc ]; then

. ~/.bashrc

fi



# User specific environment and startup programs



PATH=$PATH:$HOME/bin:/kbs/tls:/kbs/kubernetes/server/bin:/kbs/etcd



export PATH

```

1 etcd.conf
```shell
[root@kbs1 etcd]# cat /kbs/etcd/etcd.conf

# [member]

ETCD_NAME=etcd1

ETCD_DATA_DIR="/kbs/etcd"

ETCD_LISTEN_PEER_URLS=""

ETCD_LISTEN_CLIENT_URLS=""

#[cluster]

ETCD_INITIAL_ADVERTISE_PEER_URLS=""

ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"

ETCD_ADVERTISE_CLIENT_URLS=



[root@localhost kbs]# cat /kbs/etcd/etcd.conf

# [member]

ETCD_NAME=etcd2

ETCD_DATA_DIR="/kbs/etcd"

ETCD_LISTEN_PEER_URLS=""

ETCD_LISTEN_CLIENT_URLS=""



#[cluster]

ETCD_INITIAL_ADVERTISE_PEER_URLS=""

ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"

ETCD_ADVERTISE_CLIENT_URLS=



[root@localhost kbs]# cat /kbs/etcd/etcd.conf

# [member]

ETCD_NAME=etcd3

ETCD_DATA_DIR="/kbs/etcd"

ETCD_LISTEN_PEER_URLS=""

ETCD_LISTEN_CLIENT_URLS=""



#[cluster]

ETCD_INITIAL_ADVERTISE_PEER_URLS=""

ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"

ETCD_ADVERTISE_CLIENT_URLS=
```


2 etcd.service
```shell
[root@kbs1 etcd]# cat /usr/lib/systemd/system/etcd.service

[Unit]

Description=Etcd Server

After=network.target

After=network-online.target

Wants=network-online.target

Documentation=

[Service]

Type=notify

WorkingDirectory=/kbs/etcd

EnvironmentFile=-/kbs/etcd/etcd.conf

ExecStart=/kbs/etcd/etcd \

--name ${ETCD_NAME} \

--cert-file=/kbs/tls/ssl/kubernetes.pem \

--key-file=/kbs/tls/ssl/kubernetes-key.pem \

--peer-cert-file=/kbs/tls/ssl/kubernetes.pem \

--peer-key-file=/kbs/tls/ssl/kubernetes-key.pem \

--trusted-ca-file=/kbs/tls/ssl/ca.pem \

--peer-trusted-ca-file=/kbs/tls/ssl/ca.pem \

--initial-advertise-peer-urls ${ETCD_INITIAL_ADVERTISE_PEER_URLS} \

--listen-peer-urls ${ETCD_LISTEN_PEER_URLS} \

--listen-client-urls ${ETCD_LISTEN_CLIENT_URLS}, \

--advertise-client-urls ${ETCD_ADVERTISE_CLIENT_URLS} \

--initial-cluster-token ${ETCD_INITIAL_CLUSTER_TOKEN} \

--initial-cluster etcd1= \

--initial-cluster-state new \

--data-dir=${ETCD_DATA_DIR}

Restart=on-failure

RestartSec=5

LimitNOFILE=65536

[Install]

WantedBy=multi-user.target

[root@localhost kbs]# systemctl start etcd


```
3 查看状态
```shell
[root@kbs1 etcd]# etcdctl \

> --ca-file=/kbs/tls/ssl/ca.pem \

> --cert-file=/kbs/tls/ssl/kubernetes.pem \

> --key-file=/kbs/tls/ssl/kubernetes-key.pem \

> cluster-health

2020-01-31 01:45:58.048416 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated

2020-01-31 01:45:58.049186 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated

member 568465f0d31c0e0f is healthy: got healthy result from

member 67e264f60acc1d51 is healthy: got healthy result from

member 6b2797b85d092a15 is healthy: got healthy result from

cluster is healthy

Master
```
1 apiserver
1 创建 kube-apiserver的service配置文件
```shell
[root@kbs1 ~]# vi /usr/lib/systemd/system/kube-apiserver.service

[root@kbs1 ~]# cat /usr/lib/systemd/system/kube-apiserver.service

[Unit]

Description=Kubernetes API Service

Documentation=

After=network.target

After=etcd.service

[Service]

EnvironmentFile=-/kbs/kubernetes/config

EnvironmentFile=-/kbs/kubernetes/apiserver

ExecStart=/kbs/kubernetes/server/bin/kube-apiserver \

$KUBE_LOGTOSTDERR \

$KUBE_LOG_LEVEL \

$KUBE_ETCD_SERVERS \

$KUBE_API_ADDRESS \

$KUBE_API_PORT \

$KUBELET_PORT \

$KUBE_ALLOW_PRIV \

$KUBE_SERVICE_ADDRESSES \

$KUBE_ADMISSION_CONTROL \

$KUBE_API_ARGS

Restart=on-failure

Type=notify

LimitNOFILE=65536

[Install]

WantedBy=multi-user.target
```

2 创建 kube-apiserver的config配置文件
```shell
[root@kbs1 ~]# cat /kbs/kubernetes/config

###

# kubernetes system config

#

# The following values are used to configure various aspects of all

# kubernetes services, including

#

# kube-apiserver.service

# kube-controller-manager.service

# kube-scheduler.service

# kubelet.service

# kube-proxy.service

# logging to stderr means we get it in the systemd journal

KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug

KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers

KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the controller-manager, scheduler, and proxy find the apiserver

KUBE_MASTER="--master="
```

3 创建 kube-apiserver的apiserver配置文件
```shell
[root@kbs1 ~]# cat /kbs/kubernetes/apiserver

###

## kubernetes system config

##

## The following values are used to configure the kube-apiserver

##

#

## The address on the local server to listen to.

#KUBE_API_ADDRESS="--insecure-bind-address=test-001.jimmysong.io"

KUBE_API_ADDRESS="--advertise-address=192.168.135.130 --bind-address=192.168.135.130 --insecure-bind-address=192.168.135.130"

#

## The port on the local server to listen on.

#KUBE_API_PORT="--port=8080"

#

## Port minions listen on

#KUBELET_PORT="--kubelet-port=10250"

#

## Comma separated list of nodes in the etcd cluster

KUBE_ETCD_SERVERS="--etcd-servers="

#

## Address range to use for services

KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

#

## default admission control policies

KUBE_ADMISSION_CONTROL="--admission-control=ServiceAccount,NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota"

#

## Add your own!

KUBE_API_ARGS="--authorization-mode=RBAC --runtime-config=rbac.authorization.k8s.io/v1beta1 --kubelet-https=true --enable-bootstrap-token-auth --token-auth-file=/kbs/token/token.csv --service-node-port-range=30000-32767 --tls-cert-file=/kbs/tls/ssl/kubernetes.pem --tls-private-key-file=/kbs/tls/ssl/kubernetes-key.pem --client-ca-file=/kbs/tls/ssl/ca.pem --service-account-key-file=/kbs/tls/ssl/ca-key.pem --etcd-cafile=/kbs/tls/ssl/ca.pem --etcd-certfile=/kbs/tls/ssl/kubernetes.pem --etcd-keyfile=/kbs/tls/ssl/kubernetes-key.pem --enable-swagger-ui=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/kbs/kubernetes/log --event-ttl=1h"

[root@kbs1 ~]# systemctl start kube-apiserver
```

4 启动
[root@kbs1 ~]# systemctl start kube-apiserver

2 kube-controller-manager
1 创建 kube-controller-manager的serivce配置文件

```shell
[root@kbs1 ~]# cat /usr/lib/systemd/system/kube-controller-manager.service

[Unit]

Description=Kubernetes Controller Manager

Documentation=

[Service]

EnvironmentFile=-/kbs/kubernetes/config

EnvironmentFile=-/kbs/kubernetes/controller-manager

ExecStart=/kbs/kubernetes/server/bin/kube-controller-manager \

$KUBE_LOGTOSTDERR \

$KUBE_LOG_LEVEL \

$KUBE_MASTER \

$KUBE_CONTROLLER_MANAGER_ARGS

Restart=on-failure

LimitNOFILE=65536

[Install]

WantedBy=multi-user.target
```

2 创建controller-manager

```shell
[root@kbs1 ~]# cat /kbs/kubernetes/controller-manager

###

# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# Add your own!

KUBE_CONTROLLER_MANAGER_ARGS="--address=127.0.0.1 --service-cluster-ip-range=10.254.0.0/16 --cluster-name=kubernetes --cluster-signing-cert-file=/kbs/tls/ssl/ca.pem --cluster-signing-key-file=/kbs/tls/ssl/ca-key.pem --service-account-private-key-file=/kbs/tls/ssl/ca-key.pem --root-ca-file=/kbs/tls/ssl/ca.pem --leader-elect=true"
```
3 启动
[root@kbs1 ~]# systemctl start kube-controller-manager

3 kube-scheduler
1 创建 kube-scheduler的serivce配置文件
```shell
[root@kbs1 ~]# cat /usr/lib/systemd/system/kube-scheduler.service

[Unit]

Description=Kubernetes Scheduler Plugin

Documentation=

[Service]

EnvironmentFile=-/kbs/kubernetes/config

EnvironmentFile=-/kbs/kubernetes/scheduler

ExecStart=/kbs/kubernetes/server/bin/kube-scheduler \

$KUBE_LOGTOSTDERR \

$KUBE_LOG_LEVEL \

$KUBE_MASTER \

$KUBE_SCHEDULER_ARGS

Restart=on-failure

LimitNOFILE=65536

[Install]

WantedBy=multi-user.target
```
2 scheduler
```shell
[root@kbs1 ~]# cat /kbs/kubernetes/scheduler

###

# kubernetes scheduler config

# default config should be adequate

# Add your own!

KUBE_SCHEDULER_ARGS="--leader-elect=true --address=127.0.0.1"
```
3启动
[root@kbs1 ~]# systemctl start kube-scheduler

4查看状态
```shell
[root@kbs1 ~]# kubectl get cs -o=go-template='{{printf "|NAME|STATUS|MESSAGE|\n"}}{{range .items}}{{$name := .metadata.name}}{{range .conditions}}{{printf "|%s|%s|%s|\n" $name .status .message}}{{end}}{{end}}'

|NAME|STATUS|MESSAGE|

|controller-manager|True|ok|

|scheduler|True|ok|

|etcd-2|True|{"health": "true"}|

|etcd-1|True|{"health": "true"}|

|etcd-0|True|{"health": "true"}|
```
NODE
Flannel
1下载软件（NODE）

```shell
[root@kbs2 kbs]# tar zxf flannel-v0.11.0-linux-amd64.tar.gz

[root@kbs2 kbs]# ls

etcd flanneld flannel-v0.11.0-linux-amd64.tar.gz kubectl kubernetes kubernetes-server-linux-amd64.tar.gz mk-docker-opts.sh README.md tls

[root@kbs2 kbs]# mkdir flannel

[root@kbs2 kbs]# mv ./flanneld mk-docker-opts.sh README.md ./flannel

[root@kbs2 kbs]# ls

etcd flannel flannel-v0.11.0-linux-amd64.tar.gz kubectl kubernetes kubernetes-server-linux-amd64.tar.gz tls
```
2service配置文件flanneld.service
```shell
[root@kbs2 flannel]# cat /usr/lib/systemd/system/flanneld.service

[Unit]

Description=Flanneld overlay address etcd agent

After=network.target

After=network-online.target

Wants=network-online.target

After=etcd.service

Before=docker.service

[Service]

Type=notify

EnvironmentFile=/kbs/flannel/config

EnvironmentFile=-/kbs/flannel/docker-network

ExecStart=/kbs/flannel/flanneld \

-etcd-endpoints=${FLANNEL_ETCD_ENDPOINTS} \

-etcd-prefix=${FLANNEL_ETCD_PREFIX} \

$FLANNEL_OPTIONS

ExecStartPost=/kbs/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker

Restart=on-failure

[Install]

WantedBy=multi-user.target

RequiredBy=docker.service
```
3 config配置文件
```shell
[root@kbs2 flannel]# cat /kbs/flannel/config

# Flanneld configuration options

# etcd url location. Point this to the server where etcd runs

FLANNEL_ETCD_ENDPOINTS=""

# etcd config key. This is the configuration key that flannel queries

# For address range assignment

FLANNEL_ETCD_PREFIX="/kbs/flannel/network"

# Any additional options that you want to pass

FLANNEL_OPTIONS="-etcd-cafile=/kbs/tls/ssl/ca.pem -etcd-certfile=/kbs/tls/ssl/kubernetes.pem -etcd-keyfile=/kbs/tls/ssl/kubernetes-key.pem"
```

4 在etcd中创建网络配置
```shell
[root@kbs1 ~]# etcdctl --endpoints= --ca-file=/kbs/tls/ssl/ca.pem --cert-file=/kbs/tls/ssl//kubernetes.pem --key-file=/kbs/tls/ssl/kubernetes-key.pem mkdir /kbs/flannel/network

2020-02-24 13:08:55.846954 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated



[root@kbs1 ~]# etcdctl --endpoints= --ca-file=/kbs/tls/ssl/ca.pem --ca-file=/kbs/tls/ssl//ca.pem --cert-file=/kbs/tls/ssl/kubernetes.pem --key-file=/kbs/tls/ssl/kubernetes-key.pem mk /kbs/flannel/network/config '{"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}'

2020-02-24 13:12:48.313917 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated

{"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}
```
5启动
```shell
[root@kbs3 flannel]# systemctl start flanneld

[root@kbs2 ssl]# etcdctl --endpoints= \

> --ca-file=/kbs/tls/ssl/ca.pem \

> --cert-file=/kbs/tls/ssl/kubernetes.pem \

> --key-file=/kbs/tls/ssl/kubernetes-key.pem \

> ls /kbs/flannel/network

2020-02-20 06:53:45.913993 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated

/kbs/flannel/network/config

/kbs/flannel/network/subnets

[root@kbs2 ssl]# etcdctl --endpoints= --ca-file=/kbs/tls/ssl/ca.pem --cert-file=/kbs/tls/ssl/kubernetes.pem --key-file=/kbs/tls/ssl/kubernetes-key.pem ls /kbs/flannel/network/subnets

2020-02-20 06:53:57.368878 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated

/kbs/flannel/network/subnets/172.30.53.0-24

/kbs/flannel/network/subnets/172.30.100.0-24

[root@kbs2 ~]# etcdctl --endpoints= --ca-file=/kbs/tls/ssl/ca.pem --cert-file=/kbs/tls/ssl/kubernetes.pem --key-file=/kbs/tls/ssl/kubernetes-key.pem get /kbs/flannel/network/subnets/172.30.53.0-24

2020-02-20 08:02:49.239782 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated

{"PublicIP":"192.168.135.131","BackendType":"vxlan","BackendData":{"VtepMAC":"f2:9d:1d:b0:36:34"}}
```
Docker



1 下载软件
[root@kbs2 kbs]# tar zxf docker-19.03.3.tgz

[root@kbs2 kbs]# ls

docker docker-19.03.3.tgz etcd flannel flannel-v0.11.0-linux-amd64.tar.gz kubectl kubernetes kubernetes-server-linux-amd64.tar.gz tls

2 docker.service
```shell
[root@kbs2 docker]# cat /usr/lib/systemd/system/docker.service

[Unit]

Description=Docker Application Container Engine

Documentation=

[Service]

Type=notify

NotifyAccess=main

EnvironmentFile=-/etc/sysconfig/docker

EnvironmentFile=-/etc/sysconfig/docker-storage

EnvironmentFile=-/etc/sysconfig/docker-network

EnvironmentFile=-/run/flannel/docker

EnvironmentFile=-/run/docker_opts.env

EnvironmentFile=-/run/flannel/subnet.env

Environment=PATH=/kbs/docker:/usr/bin:/usr/sbin

ExecStart=/kbs/docker/dockerd --log-level=error $DOCKER_NETWORK_OPTIONS

ExecReload=/bin/kill -s HUP $MAINPID

Restart=on-failure

LimitNOFILE=1048576

LimitNPROC=1048576

LimitCORE=infinity

TimeoutStartSec=0

Restart=on-abnormal

KillMode=process

[Install]

WantedBy=multi-user.target

Kubelet，kube-proxy
[root@kbs1 ~]# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap

clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created

[root@kbs1 ~]# kubectl create clusterrolebinding kubelet-nodes --clusterrole=system:node --group=system:nodes

clusterrolebinding.rbac.authorization.k8s.io/kubelet-nodes created

```

1下载软件
```shell

tar -xzvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes

tar -xzvf kubernetes-src.tar.gz

cp -r ./server/bin/{kube-proxy,kubelet} /usr/local/bin/

mkdir -p /var/lib/kubelet
```
2 kubelet.service
```shell
[root@kbs2 kubernetes]# cat /usr/lib/systemd/system/kubelet.service

[Unit]

Description=Kubernetes Kubelet Server

Documentation=

After=docker.service

Requires=docker.service

[Service]

WorkingDirectory=/var/lib/kubelet

EnvironmentFile=-/kbs/kubernetes/config

EnvironmentFile=-/kbs/kubernetes/kubelet

ExecStart=/kbs/kubernetes/server/bin/kubelet \

$KUBE_LOGTOSTDERR \

$KUBE_LOG_LEVEL \

$KUBELET_ADDRESS \

$KUBELET_PORT \

$KUBELET_HOSTNAME \

$KUBE_ALLOW_PRIV \

$KUBELET_POD_INFRA_CONTAINER \

$KUBELET_ARGS

Restart=on-failure

[Install]

WantedBy=multi-user.target
```

3 /kbs/kubernetes/kubelet
```shell
[root@kbs3 kubectl]# cat /kbs/kubernetes/kubelet

###

## kubernetes kubelet (minion) config

#

## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)

KUBELET_ADDRESS="--address=192.168.135.132"

#

## The port for the info server to serve on

#KUBELET_PORT="--port=10250"

#

## You may leave this blank to use the actual hostname

KUBELET_HOSTNAME="--hostname-override=192.168.135.132"

#

## location of the api-server

## COMMENT THIS ON KUBERNETES 1.8+

#KUBELET_API_SERVER="--api-servers="

#

## pod infrastructure container

--pod-infra-container-image=

#

## Add your own!

KUBELET_ARGS="--cgroup-driver=systemd --cluster-dns=10.254.0.2 --experimental-bootstrap-kubeconfig=/kbs/kubernetes/bootstrap.kubeconfig --kubeconfig=/kbs/kubernetes/kubelet.kubeconfig --cert-dir=/kbs/tls/ssl --cluster-domain=cluster.local --hairpin-mode promiscuous-bridge --serialize-image-pulls=false"
```
4 启动
[root@kbs2 kubernetes]# swapoff -a

[root@kbs3 kubectl]# systemctl start kubelet



5 通过kublet的TLS证书请求
```shell
[root@kbs1 ~]# kubectl get csr

NAME AGE REQUESTOR CONDITION

node-csr-e4aRv8JCrgRUPlmp8dj4AcawCvbmxv2m2tK48wJACq8 61m kubelet-bootstrap Approved,Issued

node-csr-hASvMT6CGwYS1Q7mzauWpoko-SL5x7DActhXPgW3vYw 20m kubelet-bootstrap Pending

[root@kbs1 ~]# kubectl certificate approve node-csr-hASvMT6CGwYS1Q7mzauWpoko-SL5x7DActhXPgW3vYw

certificatesigningrequest.certificates.k8s.io/node-csr-hASvMT6CGwYS1Q7mzauWpoko-SL5x7DActhXPgW3vYw approved

[root@kbs1 ~]# kubectl get nodes

NAME STATUS ROLES AGE VERSION

192.168.135.131 Ready <none> 34m v1.16.6

192.168.135.132 Ready <none> 10s v1.16.6
```
6 kube-proxy.service
```shell
[root@kbs2 ~]# cat /usr/lib/systemd/system/kube-proxy.service

[Unit]

Description=Kubernetes Kube-Proxy Server

Documentation=

After=network.target

[Service]

EnvironmentFile=-/kbs/kubernetes/config

EnvironmentFile=-/kbs/kubernetes/proxy

ExecStart=/kbs/kubernetes/server/bin/kube-proxy \

$KUBE_LOGTOSTDERR \

$KUBE_LOG_LEVEL \

$KUBE_MASTER \

$KUBE_PROXY_ARGS

Restart=on-failure

LimitNOFILE=65536

[Install]

WantedBy=multi-user.target
```
7 /kbs/kubernetes/proxy
```shell
[root@kbs2 ~]# cat /kbs/kubernetes/proxy

###

# kubernetes proxy config

# default config should be adequate

# Add your own!

KUBE_PROXY_ARGS="--bind-address=192.168.135.131 --hostname-override=192.168.135.131 --kubeconfig=/kbs/kubernetes/kube-proxy.kubeconfig --cluster-cidr=10.254.0.0/16"
```
8 启动
[root@kbs2 kubernetes]# systemctl start kube-proxy
