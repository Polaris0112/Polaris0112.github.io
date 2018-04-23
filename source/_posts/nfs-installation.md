---
title: NFS安装部署
date: 2018-02-27
categories: NFS安装部署
tags: 
- NFS
- CentOS

---

本帖子记录的是在CentOS7系统中安装NFS网络文件存储系统


## 环境准备

- CentOS7

### 1.关闭系统防火墙和selinux

```bash
$ systemctl status firewalld
firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: active (running) since Sat 2017-06-03 09:38:00 CST; 8s ago
     Docs: man:firewalld(1)
 Main PID: 24067 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─24067 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid

Jun 03 09:37:58 linuxidc systemd[1]: Starting firewalld - dynamic firewall daemon...
Jun 03 09:38:00 linuxidc systemd[1]: Started firewalld - dynamic firewall daemon.

## 关闭防火墙
$ systemctl stop firewalld


## 把selinux设为许可模式
$ setenforce 0

## 要想禁用selinux,则需要编辑selinux的配置文件,把SELINUX设置成disabled,然后重启生效
$ vi /etc/sysconfig/selinux
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted 

```


### 2.安装所需的软件包

```bash
$ yum install -y rpc-bind nfs-utils
```


### 3.服务端配置

NFS服务的主要配置文件为 /etc/exports.
/etc/exports文件内容格式：
- <输出目录> 客户端（选项:访问权限,用户映射,其他]
 - 输出目录是指NFS系统中所定义的共享给客户端使用的文件系统
 - 客户端是定义网络中可以访问这个NFS共享目录的IP地址或网段或域名等
 - 客户端常用的指定方式
   - 指定ip地址的主机：192.168.100.1
   - 指定一个子网：192.168.100.0/24 也可以写成:192.168.100.0/255.255.255.0
   - 指定域名的主机：david.bsmart.cn
   - 指定域中的所有主机：*.bsmart.cn
   - 所有主机：*
- 选项用来设置输出目录的访问权限、用户映射等。
- NFS主要有3类选项：
  - 设置输出目录只读：ro
  - 设置输出目录读写：rw
- 用户映射选项
  - all_squash：将远程访问的所有普通用户及所属组都映射为匿名用户或用户组（nfsnobody）；
  - no_all_squash：与all_squash取反（默认设置）；
  - root_squash：将root用户及所属组都映射为匿名用户或用户组（默认设置）；
  - no_root_squash：与rootsquash取反；
  - anonuid=xxx：将远程访问的所有用户都映射为匿名用户，并指定该用户为本地用户（UID=xxx）；
  - anongid=xxx：将远程访问的所有用户组都映射为匿名用户组账户，并指定该匿名用户组账户为本地用户组账户（GID=xxx）；
- 其它选项
 - secure：限制客户端只能从小于1024的tcp/ip端口连接nfs服务器（默认设置）；
 - insecure：允许客户端从大于1024的tcp/ip端口连接服务器；
 - sync：将数据同步写入内存缓冲区与磁盘中，效率低，但可以保证数据的一致性；
 - async：将数据先保存在内存缓冲区中，必要时才写入磁盘；
 - wdelay：检查是否有相关的写操作，如果有则将这些写操作一起执行，这样可以提高效率（默认设置）；
 - no_wdelay：若有写操作则立即执行，应与sync配合使用；
 - subtree：若输出目录是一个子目录，则nfs服务器将检查其父目录的权限(默认设置)；
 - no_subtree：即使输出目录是一个子目录，nfs服务器也不检查其父目录的权限，这样可以提高效率；
-修改/etc/exports文件,定义NFS共享

```bash
## 先创建本地目录并把共享目录的权限设定为所有用户都可读可写权限
$ mkdir -p /data
$ chmod 766 /data

## 修改NFS配置文件,定义共享
$ vi /etc/exports
/data *(rw,sync)

## 把RPCbind加入开机处启动选项中
$ systemctl enable rpcbind 

## 启动RPCbind
$ systemctl start rpcbind

## 把NFS加入到开机自启动选项中
$ systemctl enable nfs

## 开启NFS服务
$ systemctl start nfs

## 查看本机共享的文件系统
$ exportfs
```




## 客户端配置（以下操作均在客户端上）

### 1.客户端关闭防火墙和selinux,方法同上.


### 2.客户端安装NFS软件包,并把NFS服务设为开机自启动,方法同上.


### 3.挂载共享的NFS文件系统

```bash
## 把服务端(192.168.0.100)的共享目录挂载到本机的/share上
$ mkdir -p /upload
$ mount 192.168.0.100:/data /share

## 查看是否已经挂载成功
$ mount | grep media
192.168.0.100:/data on /share type nfs4 (rw,relatime,vers=4.0,rsize=131072,wsize=131072,namlen=255,hard,prot

## 把共享目录写入系统挂载文件系统
$ vi /etc/fstab
192.168.16.29:/data        /share      nfs4    defaults    0 0

```


## 测试

### 1.在服务端共享目录中新建一个10M大小的文件

```bash
## 在NFS共享目录上新建一个10M大小的文件,提示成功
$ cd /data
$ dd if=/dev/zero of=/data/f1 bs=1M count=10
$ ll -h
total 10M
-rw-r--r--. 1 root root 10M Mar  6 10:14 f1
```


### 2.在客户端新建另一个文件f2,同时尝试删除另一个文件f1

```bash
$ cd /share
$ ls
f1

## 在共享目录上新建一个文件f2,未报错
$ touch f2
## 删除存在的文件f1,未报错
$ rm -f f1
```


### 3.在服务端查看共享目录中的文件

```bash
$ ls
f2
```

部署完毕。
