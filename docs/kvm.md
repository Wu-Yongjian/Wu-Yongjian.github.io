---
layout: default
title: Yj
---



## 虚拟机迁移

### 创建虚拟机
```shell
[root@localhost vmdata]# qemu-img create -f qcow2 /opt/vmdata/rocky1.qcow2 20G 
Formatting '/opt/vmdata/rocky1.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=21474836480 lazy_refcounts=off refcount_bits=16
[root@localhost ~]# virsh net-list
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes

[root@localhost ~]# virt-install --virt-type kvm --name rocky1 --ram 1024 --vcpus 1 --cdrom=/tmp/Rocky-9.2-x86_64-dvd.iso --disk /opt/vmdata/rocky1.qcow2 --network network=default,model=virtio --os-variant=rocky9.0

[root@localhost ~]# virsh list
 Id   Name     State
------------------------
 5    rocky1   running
 
```

### 本地复制虚拟机
```shell
[root@localhost ~]# ls /etc/libvirt/qemu/rocky1.xml 
/etc/libvirt/qemu/rocky1.xml
[root@localhost ~]# ls /opt/vmdata/rocky1.qcow2 
/opt/vmdata/rocky1.qcow2

[root@localhost qemu]# cp rocky1.xml rocky2.xml 
[root@localhost vmdata]# cp rocky1.qcow2 rocky2.qcow2 

[root@localhost qemu]# systemctl restart libvirtd 或者 virsh define /etc/libvirt/qemu/rocky2.xml 

[root@localhost qemu]# diff rocky1.xml rocky2.xml 
4c4
<   virsh edit rocky1
---
>   virsh edit rocky2
9,10c9,10
<   <name>rocky1</name>
<   <uuid>e0ab2127-3b4f-4edf-b5c9-1562be82fe89</uuid>
---
>   <name>rocky2</name>
>   <uuid>e0ab2127-3b4f-4edf-b5c9-1562be82fe99</uuid>
44c44
<       <source file='/opt/vmdata/rocky1.qcow2'/>
---
>       <source file='/opt/vmdata/rocky2.qcow2'/>
135c135
<       <mac address='52:54:00:fd:74:61'/>
---
>       <mac address='52:54:00:fd:74:62'/>
[root@localhost qemu]#virsh start rocky1
[root@localhost qemu]#virsh start rocky2

[root@localhost qemu]# virsh list
 Id   Name     State
------------------------
 1    rocky2   running
 2    rocky1   running
 
```



### 迁移虚拟机

```shell

[root@localhost qemu]# scp /opt/vmdata/rocky1.qcow2 192.168.198.140:/opt/vmdata
root@192.168.198.140's password: 
rocky1.qcow2    
[root@localhost vmdata]# virt-install --virt-type kvm --name rocky1 --ram 1024 --vcpus 1  --disk /opt/vmdata/rocky1.qcow2 --network network=default,model=virtio --boot hd
[root@localhost ~]# virsh list
 Id    Name                           State
----------------------------------------------------
 6     rocky1                        running
````
