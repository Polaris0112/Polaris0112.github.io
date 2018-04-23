---
title: Ceph修改OSD和Monitor的网络
date: 2018-03-17
categories: Ceph修改OSD和Monitor的网络
tags: 
- Ceph

---
本帖子记录的是分布式存储系统Ceph修改OSD和Monitor的网络过程记录。

## 背景

随着Ceph的应用越来越深入，我们也遇到了越来越多的需求，本次我们将讲一下如何修改OSD和Monitor的网络。

Ceph应用中，在部署完ceph集群并运行一段时间后，我们很可能会遇到机房网络变动，或集群网络升级的情况，这时我们都期望能在尽量减少对现有Ceph集群影响的情况下，修改Ceph的OSD和Monitor网络，而不是简单粗暴的重新部署Ceph集群。
本文基于Ceph的官方文档和作者的实践，给出了正确修改Ceph OSD和Monitor网络的方法。

Ceph版本：Hammer 0.94.5
测试系统环境：Ubuntu 14.04


## 修改OSD的网络

由于Ceph OSD的网络配置是启动时读取ceph.conf配置动态加载的，所以修改OSD的网络比较简单，步骤如下：

修改下`ceph.conf`中`pubulic/cluster network`信息
     例如：`public network = 172.16.1.0/24`
           `cluster network = 172.16.1.0/24`

把`ceph.conf`更新到所有OSD节点测配合
```bash
$ ceph-deploy --overwrite-conf config push <node1> <node2> <node3>
```

重启所有的`osd daemon`
```bash
$ restart ceph-all
```

检查`OSD`使用的网络
```bash
$ netstat -nap | grep ceph-osd
tcp        0      0 172.16.1.16:844         0.0.0.0:*               LISTEN      23412/ceph-osd
tcp        0      0 172.16.1.16:812         0.0.0.0:*               LISTEN      19423/ceph-osd
tcp        0      0 172.16.1.16:845         0.0.0.0:*               LISTEN      23412/ceph-osd
tcp        0      0 172.16.1.16:813         0.0.0.0:*               LISTEN      19529/ceph-osd
```



## 修改Monitor的网络

因为Ceph的Monitor是集群中非常重要的模块，它们需要维护一组合理有效的Monitor节点信息，这些节点之间彼此能发现，它们通过选举达成一致的状态，来保证整个Ceph系统处于一个可用的一致状态。
不同于别的Ceph Daemon通过ceph.conf文件中的配置来与Monitor通信，Monitors之间则通过独立的monitor map来彼此发现，在monitor map中有monitor的ip信息，所以单独通过修改ceph.conf文件的方法来修改Monitor的网络是行不通的。
下面给出了两种办法来达到修改monitor网络的目的：

- 通过依次添加新的monitor node(使用新的网络)，再删除旧的monitor node的方法

**注意：**新的monitor网络跟旧的monitor的网络必须是互通的，否则该方法失效。

添加新的monitor节点步骤如下：

```bash
1). 在新的monitor节点创建默认文件夹
$ ssh {new-mon-host}
$ sudo mkdir /var/lib/ceph/mon/ceph-{mon-id}

2). 获取monitors的keyring文件，若内部集群没配置认证，该步会出错，忽略即可
$ ceph auth get mon. -o {tmp}/{key-filename}

3). 获取monitor map文件
$ ceph mon getmap -o {tmp}/{map-filename}

4). 在新的monitor节点生产monitor所需数据信息
$ sudo ceph-mon -i {mon-id} --mkfs --monmap {tmp}/{map-filename} --keyring {tmp}/{key-filename}

5). 添加新的monitor到monitor集群，绑定新的ip地址和port
$ ceph-mon -i {mon-id} --public-addr {ip:port}
```

删除旧的monitor节点步骤如下：
```bash
1). 在旧的monitor节点停止该monitor
$ service ceph -a stop mon.{mon-id}

2). 把该monitor从集群中移除
$ ceph mon remove {mon-id}
```

修改monitor map里的信息，更新后重启所有monitors
因为该方法一般是通过整体修改monitor map里的网络信息，然后重启monitors，所以会短暂影响Ceph集群的服务
```bash
1). 获取现在的monitor map
$ ceph mon getmap -o {tmp}/{filename}

2). 查看当前的monitor map信息
$ monmaptool --print {tmp}/{filename}
$ monmaptool: monmap file {tmp}/{filename}
epoch 1
fsid 224e376d-c5fe-4504-96bb-ea6332a19e61
last_changed 2012-12-17 02:46:41.591248
created 2012-12-17 02:46:41.591248
0: 10.0.0.1:6789/0 mon.a
1: 10.0.0.2:6789/0 mon.b
2: 10.0.0.3:6789/0 mon.c

3). 删除现有的monitors信息
$ monmaptool --rm a --rm b --rm c {tmp}/{filename}
monmaptool: monmap file {tmp}/{filename}
monmaptool: removing a
monmaptool: removing b
monmaptool: removing c
monmaptool: writing epoch 1 to {tmp}/{filename} (0 monitors)

4). 添加新的monitors信息
$ monmaptool --add a 10.1.0.1:6789 --add b 10.1.0.2:6789 --add c 10.1.0.3:6789 {tmp}/{filename}

monmaptool: monmap file {tmp}/{filename}
monmaptool: writing epoch 1 to {tmp}/{filename} (3 monitors)

5). 检查新的monitors信息
$ monmaptool --print {tmp}/{filename}
monmaptool: monmap file {tmp}/{filename}
epoch 1
fsid 224e376d-c5fe-4504-96bb-ea6332a19e61
last_changed 2012-12-17 02:46:41.591248
created 2012-12-17 02:46:41.591248
0: 10.1.0.1:6789/0 mon.a
1: 10.1.0.2:6789/0 mon.b
2: 10.1.0.3:6789/0 mon.c

6). copy修改后的{tmp}/{filename}文件到所有monitor节点

7). 停止所有的monitor
$ stop ceph-mon-all

8). 注入新的monitor map信息
$ ceph-mon -i {mon-id} --inject-monmap {tmp}/{filename}

9). 启动所有的monitro
$ start ceph-mon-all
```

