---
layout: default
title: Yj
---

## 多节点kvm通信网络隔离


### 添加网卡

[rocky](#heading-ids)	
```shell
[root@localhost system-connections]# systemctl restart NetworkManager
[root@localhost system-connections]# ip a   查看新增的网卡名
[root@localhost ~]# cd /etc/NetworkManager/system-connections/
[root@localhost system-connections]# cp ens33.nmconnection ens37.nmconnection   修改配置文件
sudo nmcli connection reload
[root@localhost system-connections]# nmcli connection up ens37
```

### 创建网桥
[centos](#heading-ids)	

```shell
sudo brctl addbr virbr1
sudo brctl addif virbr1 ens37

[root@localhost network-scripts]# cat ifcfg-virbr1 
TYPE=Bridge
NAME=virbr1
DEVICE=virbr1
ONBOOT=yes
BOOTPROTO=static
IPADDR=10.0.0.11
NETMASK=255.255.255.0

[root@localhost network-scripts]# vi ifcfg-ens37 
[root@localhost network-scripts]# cat ifcfg-ens37 
TYPE=Ethernet
NAME=ens37
DEVICE=ens37
ONBOOT=yes
BRIDGE=virbr1
[root@localhost network-scripts]# nmcli connection reload
[root@localhost network-scripts]# nmcli connection up ens37
[root@localhost network-scripts]# nmcli connection up virbr1
```

[rocky](#heading-ids)	
```shell
[root@localhost system-connections]# sudo nmcli connection add type bridge con-name virbr1 ifname virbr1     
[root@localhost system-connections]# sudo nmcli connection add type ethernet con-name virbr1-slave ifname ens37 master virbr1
[root@localhost system-connections]# nmcli connection down ens37
[root@localhost system-connections]# nmcli connection reload
[root@localhost system-connections]# nmcli systemctl restart NetworkManager

[root@localhost NetworkManager]# cd system-connections/
[root@localhost system-connections]# pwd
/etc/NetworkManager/system-connections
[root@localhost system-connections]# ls
ens33.nmconnection  virbr1.nmconnection  virbr1-slave.nmconnection
[root@localhost system-connections]# cat virbr1.nmconnection 
[connection]
id=virbr1
uuid=b8910127-0939-4146-b034-b02012567ea7
type=bridge
interface-name=virbr1

[ethernet]

[bridge]

[ipv4]
method=manual
address=10.0.0.12


[ipv6]
addr-gen-mode=default
method=auto

[proxy]
[root@localhost system-connections]# cat virbr1-slave.nmconnection 
[connection]
id=virbr1-slave
uuid=19661e97-b88e-4f45-be46-90ec202dd1b5
type=ethernet
interface-name=ens37
master=virbr1
slave-type=bridge

[ethernet]

[bridge-port]

```

### 创建虚拟机

[rocky 宿主机](#heading-ids)	
```shell
4: virbr1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:0c:29:64:d9:26 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.12/24 brd 10.0.0.255 scope global noprefixroute virbr1
       valid_lft forever preferred_lft forever
    inet6 fe80::391a:ad3d:726f:a496/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
5: virbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:54:00:7e:fd:09 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
```
```shell
qemu-img create -f qcow2 /opt/vmdata/rocky1.qcow2 20G
virt-install --virt-type kvm --name rocky1 --ram 1024 --vcpus 1 --cdrom=/tmp/Rocky-9.2-x86_64-dvd.iso --disk /opt/vmdata/rocky1.qcow2 --network network=default,model=virtio  --network bridge=virbr1,model=virtio --os-variant=rocky9.0

```
```shell
kvm rocky1 创建两张网卡
    virbr0 使用default 获取与主机ens33相同网段ip
	virbr1 使用私网地址10.0.0.0网段
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:27:d9:ae brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.44/24 brd 192.168.122.255 scope global dynamic noprefixroute enp1s0
       valid_lft 3543sec preferred_lft 3543sec
    inet6 fe80::5054:ff:fe27:d9ae/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: enp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:e7:cf:99 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.21/24 brd 10.0.0.255 scope global noprefixroute enp2s0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fee7:cf99/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```
***设置通往10.0.0.0网段的gw为本地virbr1的地址***
```shell
[root@localhost ~]# route add -net 10.0.0.0/24 gw 10.0.0.12
设置后可以ping centos宿主机的 virbr1 也可以ping centos宿主机的kvm
kvm的10.0.0.0网段也可以互相ping
```



[centos 宿主机](#heading-ids)	
```shell
4: virbr1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:0c:29:df:a3:fd brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.11/24 brd 10.0.0.255 scope global noprefixroute virbr1
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fedf:a3fd/64 scope link 
       valid_lft forever preferred_lft forever
5: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:8c:4c:9c brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever

```

```shell
qemu-img create -f qcow2 /opt/vmdata/centos1.qcow2 20G 
virt-install --virt-type kvm --name centos1 --ram 1024 --vcpus 1 --cdrom=/var/lib/libvirt/images/CentOS-7-x86_64-DVD-2009.iso --disk /opt/vmdata/centos1.qcow2  --network bridge=virbr1,model=virtio --os-variant=centos7.0
```
```shell
kvm centos1 创建一张网卡
	virbr1 使用私网地址10.0.0.0网段
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:54:e6:5c brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.22/24 brd 10.0.0.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::e8f0:192a:a2c0:5cd6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

```
***设置通往10.0.0.0网段的gw为本地virbr1的地址***
```shell
[root@localhost ~]# route add -net 10.0.0.0/24 gw 10.0.0.12
设置后可以ping rocky宿主机的 virbr1 也可以ping rocky宿主机的kvm
kvm的10.0.0.0网段也可以互相ping

```



可以有三台那么第一台的网卡不设置网关，其余两台网关设置为第一台的ip地址
也可以将每个虚拟机的网关配置为其各自宿主机的网桥地址



