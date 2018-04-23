---
title: Ceph Client挂载方法
date: 2018-03-14
categories: Ceph Client挂载方法
tags: 
- Ceph

---
本帖子记录的是分布式存储系统Ceph的客户端挂载记录。

## Client挂载

client挂载方式有两种：内核kernal方式和fuse方式。

特别需要注意的是：
client挂载前需要在client节点安装ceph，详细请参考[abc](abc)

- 系统环境：CentOS 7


### 安装`ceph-fuse`

```bash
$ yum install -y epel-release
$ yum install -y ceph-fuse
```


### 创建挂载目录

```bash
$ mkdir /cephfs
```


### 复制配置文件

```bash
## 将ceph配置文件ceph.conf从管理节点copy到client节点
$ rsync -e "ssh -p22" -avp root@192.168.0.200:/etc/ceph/ceph.conf /etc/ceph/

## 将ceph的ceph.client.admin.keyring从管理节点copy到client节点
$ rsync -e "ssh -p22" -avp root@192.168.0.200:/etc/ceph/ceph.client.admin.keyring /etc/ceph/

## 查看ceph授权
$ ceph auth list
installed auth entries:
 
mds.ceph-node1
   key: AQDV/3lasqfVARAAa/eOiCbWQ/ccFLkVD/9UCQ==
   caps: [mds] allow
   caps: [mon] allow profile mds
   caps: [osd] allow rwx
osd.0
   key: AQAB9XlapzMfOxAAfKLo3Z6FZMyqWUW9F5FU4Q==
   caps: [mon] allow profile osd
   caps: [osd] allow *
osd.1
   key: AQAL9XlaT06zLRAABQr9kweLPiHL2Icdgj8YsA==
   caps: [mon] allow profile osd
   caps: [osd] allow *
osd.2
   key: AQAV9XladRLjORAATbxZHpD34ztqJJ3vFGkVOg==
   caps: [mon] allow profile osd
   caps: [osd] allow *
client.admin
   key: AQCnH3lalDbeHhAAC6y7YERqDDGL+f8S+sP1bw==
   caps: [mds] allow
   caps: [mon] allow *
   caps: [osd] allow *
client.bootstrap-mds
   key: AQCoH3laUqT5JRAAiVMRF8ueWQXfgQjIPalNMQ==
   caps: [mon] allow profile bootstrap-mds
client.bootstrap-osd
   key: AQCnH3laBzU7MBAAnreE1I/0Egh8gvMseoy93w==
   caps: [mon] allow profile bootstrap-osd
client.bootstrap-rgw
   key: AQCoH3laziwXDxAAenU6cqw0gRcQ3Y0JtygtPQ==
   caps: [mon] allow profile bootstrap-rgw
 
```

若出现
```bash
Traceback (most recent call last):
  File "/usr/bin/ceph", line 118, in <module>
    import rados
ImportError: No module named rados
```
类似缺少包的报错，请用`root`用户修改`/usr/bin/ceph`的文件头`#!/usr/bin/env python`改为`#!/usr/bin/python2.7` (CentOS 6请改为`#!/usr/bin/python2.6`)


挂载到指定目录
将ceph挂载到/cephfs

```bash
$ ceph-fuse -m 192.168.0.200:6789 /cephfs

$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        48G  1.5G   45G   4% /
tmpfs           1.9G     0  1.9G   0% /dev/shm
/dev/vda1       190M   36M  145M   20% /boot
/dev/vda5        47G   52M   44G   1% /home
ceph-fuse       300G   16G  285G   6% /cephfs
```

如上可以看到，clinet节点已经将三个osd存储节点（每个20G，一共30G）的ceph存储挂载上了，总共60G！
可以在/cephfs下存放数据，当ceph的部分osd节点挂掉后，不会影响client对ceph存储数据的读取操作！
 
取消挂载
```bash
$ umount /cephfs
```



## 直接挂载

首先也是要安装好ceph，详细参考上述过程

从管理节点中查看ceph client keyring信息
```bash
$ sudo -iu ceph
$ cd /home/ceph/ceph-cluster
$ cat ceph.client.admin.keyring
[client.admin]
	key = AQBETrlakTy4LBAA4yMZc1q3nQHbUBD+X2KTmg==
	caps mds = "allow *"
	caps mon = "allow *"
	caps osd = "allow *"
```

获取`ceph.client.admin.keyring`的`key`的值`AQBETrlakTy4LBAA4yMZc1q3nQHbUBD+X2KTmg==`

复制到客户端中
```bash
$ cat /root/admin.secret
AQBETrlakTy4LBAA4yMZc1q3nQHbUBD+X2KTmg==

## 开始挂载
$ mount -t ceph 192.168.0.200:6789:/ /cephfs -o name=admin,secretfile=/root/admin.secret  
$ df -h
Filesystem                  Size  Used  Avail Use% Mounted on
/dev/vda2                   48G   1.5G  45G   4%   / 
tmpfs                       1.9G  0     1.9G  0%   /dev/shm
/dev/vda1                   190M  36M   145M  20%  /boot
/dev/vda5                   47G   52M   44G   1%   /home
192.168.0.200:6789:/        300G   16G   285G 6%  /cephfs

## 取消挂载
$ umount /cephfs
```




