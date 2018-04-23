---
title: Ceph日常命令使用
date: 2018-03-16
categories: Ceph日常命令使用
tags: 
- Ceph

---
本帖子记录的是分布式存储系统Ceph的日常命令使用方法。

## Ceph日常操作命令

查看状态命令：
查看ceph集群状态：ceph -s
查看mon状态：ceph mon stat
查看msd状态：ceph msd stat
查看osd状态：ceph osd stat
查看osd目录树（可以查看每个osd挂在哪台机，是否已启动）：ceph osd tree
 
启动ceph进程命令：
需要在对应的节点进行启动（如果对应节点没有该服务，会进行提示）
启动mon进程：service ceph start mon.ceph-node1
启动msd进程：service ceph start msd.ceoh-node1
启动osd进程：service ceph start osd.0（在ceph-node1上）
启动osd进程：service ceph start osd.1（在ceph-node2上）
启动osd进程：service ceph start osd.2（在ceph-node3上）
 


```bash
## 查看机器的监控状态
$ ceph health
 
## 查看ceph的实时运行状态
$ ceph -w
 
## 检查信息状态信息
$ ceph -s
 
## 查看ceph存储空间
$ ceph df
 
## 删除一个节点的所有的ceph数据包
$ ceph-deploy purge ceph-node1
$ ceph-deploy purgedata ceph-node1
 
## 为ceph创建一个admin用户并为admin用户创建一个密钥，把密钥保存到/etc/ceph目录下：
$ ceph auth get-or-create client.admin mds 'allow' osd 'allow ' mon 'allow ' > /etc/ceph/ceph.client.admin.keyring
## 或
$ ceph auth get-or-create client.admin mds 'allow' osd 'allow ' mon 'allow ' -o /etc/ceph/ceph.client.admin.keyring
 
## 为osd.0创建一个用户并创建一个key
$ ceph auth get-or-create osd.0 mon 'allow rwx' osd 'allow *' -o /var/lib/ceph/osd/ceph-0/keyring
 
## 为mds.node1创建一个用户并创建一个key（ceph-node1是节点名称）
$ ceph auth get-or-create mds.node1 mon 'allow rwx' osd 'allow ' mds 'allow ' -o /var/lib/ceph/mds/ceph-ceph-node1/keyring
 
## 查看ceph集群中的认证用户及相关的key
$ ceph auth list
 
## 删除集群中的一个认证用户
$ ceph auth del osd.0
 
## 查看集群的详细配置（ceph-node1是节点名称）
$ ceph daemon mon.ceph-node1 config show | more
 
## 查看集群健康状态细节
$ ceph health detail
 
## 查看ceph log日志所在的目录
$ ceph-conf –name mon.node1 –show-config-value log_file
```


## 关于mon节点的相关操作命令

```bash
## 查看mon的状态信息
$ ceph mon stat
 
## 查看mon的选举状态
$ ceph quorum_status
 
## 查看mon的映射信息
$ ceph mon dump
 
## 删除一个mon节点
$ ceph mon remove node1
 
## 获得一个正在运行的mon map，并保存在1.txt文件中
$ ceph mon getmap -o 1.txt
 
## 查看上面获得的map
$ monmaptool --print 1.txt
 
## map注入新加入的节点（如新节点主机名为ceph-node4）
$ ceph-mon -i ceph-node4 --inject-monmap 1.txt
 
## 查看mon的amin socket
$ ceph-conf --name mon.ceph-node1 --show-config-value admin_socket
 
## 查看mon的详细状态（ceph-node1为mon节点主机名）
$ ceph daemon mon.ceph-node1 mon_status
 
## 删除一个mon节点（ceph-node1为mon节点主机名）
$ ceph mon remove ceph-node1
```


## msd节点相关操作命令

```bash
## 查看msd状态
$ ceph mds stat
 
## 查看msd的映射信息
$ ceph mds dump
 
## 删除一个mds节点
$ ceph mds rm 0 mds.ceph-node1
```



## osd节点相关操作命令

```bash
## 查看ceph osd运行状态
$ ceph osd stat
 
## 查看osd映射信息
$ ceph osd dump
 
## 查看osd的目录树
$ ceph osd tree
 
## down掉一个osd硬盘(比如down掉osd.0节点磁盘）
$ ceph osd down 0    
 
## 在集群中删除一个osd硬盘
$ ceph osd rm 0
 
## 在集群中删除一个osd 硬盘 crush map
$ ceph osd crush rm osd.0
 
## 在集群中删除一个osd的host节点
$ ceph osd crush rm node1
 
## 查看最大osd的个数（默认最大是4个osd节点）
$ ceph osd getmaxosd
 
## 设置最大的osd的个数（当扩大osd节点的时候必须扩大这个值）
$ ceph osd setmaxosd 10

## 设置osd crush的权重为1.0
$ ceph osd crush set {id} {weight} [{loc1} [{loc2} …]]

## 例如：
$ ceph osd crush set 3 3.0 host=ceph-node4
set item id 3 name 'osd.3' weight 3 at location {host=node4} to crush map
$ ceph osd tree
# id weight type name up/down reweight
-1 6 root default
-2 1 host ceph-node1
0 1 osd.0 up 1
-3 1 host ceph-node2
1 1 osd.1 up 1
-4 1 host ceph-node3
2 1 osd.2 up 1
-5 3 host ceph-node4
3 3 osd.3 up 0.5
 
## 或者用下面的方式
$ ceph osd crush reweight osd.3 1.0
reweighted item id 3 name 'osd.3' to 1 in crush map
$ ceph osd tree
# id weight type name up/down reweight
-1 4 root default
-2 1 host node1
0 1 osd.0 up 1
-3 1 host node2
1 1 osd.1 up 1
-4 1 host node3
2 1 osd.2 up 1
-5 1 host node4
3 1 osd.3 up 0.5
 
 
## 设置osd的权重
 
$ ceph osd reweight 3 0.5
reweighted osd.3 to 0.5 (8327682)
$ ceph osd tree
# id weight type name up/down reweight
-1 4 root default
-2 1 host node1
0 1 osd.0 up 1
-3 1 host node2
1 1 osd.1 up 1
-4 1 host node3
2 1 osd.2 up 1
-5 1 host node4
3 1 osd.3 up 0.5
 
## 把一个osd节点逐出集群
 
$ ceph osd out osd.3
marked out osd.3.
$ ceph osd tree
# id weight type name up/down reweight
-1 4 root default
-2 1 host node1
0 1 osd.0 up 1
-3 1 host node2
1 1 osd.1 up 1
-4 1 host node3
2 1 osd.2 up 1
-5 1 host node4
3 1 osd.3 up 0
## osd.3的reweight变为0了就不再分配数据，但是设备还是存活的
 
## 把逐出的osd加入集群
 
$ ceph osd in osd.3
marked in osd.3.
$ ceph osd tree
# id weight type name up/down reweight
-1 4 root default
-2 1 host node1
0 1 osd.0 up 1
-3 1 host node2
1 1 osd.1 up 1
-4 1 host node3
2 1 osd.2 up 1
-5 1 host node4
3 1 osd.3 up 1
 
## 暂停osd （暂停后整个集群不再接收数据）
$ ceph osd pause
# set pauserd,pausewr
 
## 再次开启osd （开启后再次接收数据）
$ ceph osd unpause
# unset pauserd,pausewr
 
## 查看一个集群osd.2参数的配置
$ ceph –admin-daemon /var/run/ceph/ceph-osd.2.asok config show | less
```



## PG组相关操作命令

```bash
## 查看pg组的映射信息
$ ceph pg dump
 
## 查看一个PG的map
$ ceph pg map 0.3f
osdmap e88 pg 0.3f (0.3f) -> up [0,2] acting [0,2]
##其中的[0,2]代表存储在osd.0、osd.2节点，osd.0代表主副本的存储位置
 
## 查看PG状态
$ ceph pg stat
v1164: 448 pgs: 448 active+clean; 10003 MB data, 23617 MB used, 37792 MB / 61410 MB avail
 
## 查询一个pg的详细信息
$ ceph pg 0.26 query
 
## 查看pg中stuck的状态
$ ceph pg dump_stuck unclean
ok
$ ceph pg dump_stuck inactive
ok
$ ceph pg dump_stuck stale
ok
 
## 显示一个集群中的所有的pg统计
$ ceph pg dump –format plain
 
## 恢复一个丢失的pg
$ ceph pg {pg-id} mark_unfound_lost revert
 
## 显示非正常状态的pg
$ ceph pg dump_stuck inactive|unclean|stale
```


## pool相关操作命令

```bash

ph集群中的pool数量
$ ceph osd lspools
0 data,1 metadata,2 rbd,
 
## 在ceph集群中创建一个pool
$ ceph osd pool create kevin 100
## 这里的100指的是PG组，kevin是集群名称
 
## 为一个ceph pool配置配额
$ ceph osd pool set-quota data max_objects 10000
 
## 在集群中删除一个pool（集群名字需要重复两次）
$ ceph osd pool delete kevin kevin --yes-i-really-really-mean-it
 
 
## 显示集群中pool的详细信息
$ rados df
pool name category KB objects clones degraded unfound rd rd KB wr wr KB
data - 475764704 116155 0 0 0 0 0 116379 475764704
metadata - 5606 21 0 0 0 0 0 314 5833
rbd - 0 0 0 0 0 0 0 0 0
total used 955852448 116176
total avail 639497596
total space 1595350044


## 给一个pool创建一个快照
$ ceph osd pool mksnap data date-snap
created pool data snap date-snap
 
## 删除pool的快照
$  ceph osd pool rmsnap data date-snap
removed pool data snap date-snap
 
## 查看data池的pg数量
$  ceph osd pool get data pg_num
pg_num: 64
 
## 设置data池的最大存储空间为100T（默认是1T)
$  ceph osd pool set data target_max_bytes 100000000000000
set pool 0 target_max_bytes to 100000000000000
 
## 设置data池的副本数是3
$  ceph osd pool set data size 3
set pool 0 size to 3
 
## 设置data池能接受写操作的最小副本为2
$  ceph osd pool set data min_size 2
set pool 0 min_size to 2
 
## 查看集群中所有pool的副本尺寸
$ ceph osd dump | grep 'replicated size'
pool 0 'data' replicated size 3 min_size 2 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 26 owner 0 flags hashpspool crash_replay_interval 45 target_bytes 100000000000000 stripe_width 0
pool 1 'metadata' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 1 owner 0 flags hashpspool stripe_width 0
pool 2 'rbd' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 1 owner 0 flags hashpspool stripe_width 0
 
## 设置一个pool的pg数量
$ ceph osd pool set data pg_num 100
set pool 0 pg_num to 100
 
## 设置一个pool的pgp数量
$ ceph osd pool set data pgp_num 100
set pool 0 pgp_num to 100

```



## rados指令

```bash
$ rados lspools
rbd
cephfs_data
cephfs_metadata
 
## 查看ceph集群中有多少个pool,并且每个pool容量及利用情况
$ rados df
pool name                 KB      objects       clones     degraded      unfound           rd        rd KB           wr        wr KB
cephfs_data                0            0            0            0           0            0            0            0            0
cephfs_metadata           21           20            0            0           0            0            0           45           36
rbd                        0            0            0            0           0            0            0            0            0
  total used        15833064           20
  total avail       47044632
  total space       62877696
 
## 创建一个pool,名称为kevin
$ rados mkpool kevin
successfully created pool kevin
 
## 查看ceph pool中的ceph object （这里的object是以块形式存储的）
$ rados ls -p kevin|more
 
## 创建一个对象object（下面的kevin是pool名称）
$ rados create kevin-object -p kevin
$ rados -p kevin ls
kevin-object
 
## 删除一个对象
$ rados rm kevin-object -p kevin
$ rados -p kevin ls
```



## rbd命令的用法

```bash
## 查看ceph中一个pool里的所有镜像
$ rbd ls kevin     ##kevin是一个pool名
## 或者
$ rbd list kevin
 
## 查看ceph pool中一个镜像的信息（kevin是pool名，wangshibo是镜像名）
$ rbd info -p kevin --image wangshibo
 
## 在kevin池中创建一个命名为wangshibo的10000M的镜像
$ rbd create -p kevin --size 10000 wangshibo
$ rbd -p kevin info wangshibo                 ## 查看新建的镜像的信息
rbd image 'wangshibo':
   size 10000 MB in 2500 objects
   order 22 (4096 kB objects)
   block_name_prefix: rb.0.1079.2ae8944a
   format: 1
 
## 删除一个镜像
$ rbd rm -p kevin wangshibo
Removing image: 100% complete...done.
 
## 调整一个镜像的尺寸（前提是wangshibo镜像已经创建并没有被删除）
$ rbd resize -p kevin --size 20000 wangshibo
Resizing image: 100% complete...done.
 
## 查看调整后的wangshibo镜像大小
$ rbd -p kevin info wangshibo
rbd image 'wangshibo':
   size 20000 MB in 5000 objects
   order 22 (4096 kB objects)
   block_name_prefix: rb.0.107d.2ae8944a
   format: 1
 
## 给一个镜像创建一个快照（如下，池/镜像@快照 ）
$ rbd snap create kevin/wangshibo@wangshibo123
 
## 查看快照
$ rbd info kevin/wangshibo@wangshibo123
rbd image 'wangshibo':
   size 20000 MB in 5000 objects
   order 22 (4096 kB objects)
   block_name_prefix: rb.0.107d.2ae8944a
   format: 1
   protected: False
 
## 查看一个镜像文件的快照
$ rbd snap ls -p kevin wangshibo
SNAPID NAME             SIZE
     4 wangshibo123 20000 MB
 
## 删除一个镜像文件的一个快照
$ rbd snap rm kevin/wangshibo@wangshibo123
$ rbd snap ls -p kevin wangshibo        //wangshibo123快照已经被删除
 
## 如果发现不能删除显示的报错信息是此快照备写保护了，下面命令是删除写保护后再进行删除。
$ rbd snap unprotect kevin/wangshibo@wangshibo123
$ rbd snap rm kevin/wangshibo@wangshibo123
 
## 删除一个镜像文件的所有快照
$ rbd snap purge -p kevin wangshibo
Removing all snapshots: 100% complete...done.
 
## 把ceph pool中的一个镜像导出
$ rbd export -p kevin --image wangshibo
Exporting image: 100% complete...done.
 
## 把一个镜像导入ceph中 （但是直接导入是不能用的，因为没有经过openstack,openstack是看不到的）
$ rbd import /root/ceph_test.img -p kevin --image wangshibo
```





