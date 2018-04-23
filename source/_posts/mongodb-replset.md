---
title: MongoDB复制集部署
date: 2018-02-24
categories: MongoDB复制集部署
tags: 
- NoSql
- MongoDB
- ReplSet

---
本帖子记录的是MongoDB复制集部署全过程,有关MongoDB简介、安装以及配置文件参数说明,请看回上一个帖子[MongoDB单实例部署](https://polaris0112.github.io/2018/02/24/mongodb-single/)

以下就直接介绍复制集以及部署过程


## MongoDB复制集简介

   Mongodb复制集由一组Mongod实例（进程）组成，包含一个Primary节点和多个Secondary节点，Mongodb Driver（客户端）的所有数据都写入Primary，Secondary从Primary同步写入的数据，以保持复制集内所有成员存储相同的数据集，提供数据的高可用。更多关于副本集的介绍请见[官网](https://docs.mongodb.com/manual/replication/)。

   下图（图片源于Mongodb官方文档）是一个典型的Mongdb复制集，包含一个Primary节点和2个Secondary节点。


![mongodb-replication](/images/mongodb-replication.png)


## 复制集(副本集)处理数据过程

副本集中数据同步过程：Primary节点写入数据，Secondary通过读取Primary的oplog得到复制信息，开始复制数据并且将复制信息写入到自己的oplog。如果某个操作失败，则备份节点停止从当前数据源复制数据。如果某个备份节点由于某些原因挂掉了，当重新启动后，就会自动从oplog的最后一个操作开始同步，同步完成后，将信息写入自己的oplog，由于复制操作是先复制数据，复制完成后再写入oplog，有可能相同的操作会同步两份，不过MongoDB在设计之初就考虑到这个问题，将oplog的同一个操作执行多次，与执行一次的效果是一样的。简单的说就是：

当Primary节点完成数据操作后，Secondary会做出一系列的动作保证数据的同步：
1：检查自己local库的oplog.rs集合找出最近的时间戳。
2：检查Primary节点local库oplog.rs集合，找出大于此时间戳的记录。
3：将找到的记录插入到自己的oplog.rs集合中，并执行这些操作。

副本集的同步和主从同步一样，都是异步同步的过程，不同的是副本集有个自动故障转移的功能。其原理是：slave端从primary端获取日志，然后在自己身上完全顺序的执行日志所记录的各种操作（该日志是不记录查询操作的），这个日志就是local数据 库中的oplog.rs表，默认在64位机器上这个表是比较大的，占磁盘大小的5%，oplog.rs的大小可以在启动参数中设 定：--oplogSize 1000,单位是M。

同期原理是，写操作发生在主库，从库同步主库的OpLog日志。

![mongodb-replication-run](/images/mongodb-replication-run.png)

集群中没有特定的主库，主库是选举产生，如果主库down了，会再选举出一台主库。

![mongodb-replication-fail](/images/mongodb-replication-fail.png)


注意：在副本集的环境中，要是所有的Secondary都宕机了，只剩下Primary。最后Primary会变成Secondary，不能提供服务。


 

## 安装部署

Mongodb的安装和配置文件的说明不再重复，详细请看帖子[《MongoDB单实例部署》](https://polaris0112.github.io/2018/02/24/mongodb-single/)


官方推荐最小的复制集是3节点集群，一主两从或者一主一从一仲裁(其中仲裁节点，即当参与选举的节点无法选出主节点时仲裁节点充当仲裁的作用，仲裁节点不存储数据。)

由于两从的配置除了ip设置不一样之外其他都一样的，所以这里例子示范的是一主一从一仲裁的搭建过程。

准备3台服务器：
```bash
192.168.0.100    #主节点
192.168.0.101    #从节点
192.168.0.102    #仲裁节点
```
均安装上MongoDB并配置好对应的目录(假设每个节点存放的路径都是在/data/db/mongodb/repl目录下的data、log和etc文件夹)


在主节点(192.168.0.100)编辑配置文件(/data/db/mongodb/repl/etc/config.yml)
```bash
systemLog:
   destination: "file"
   path: "/data/db/mongodb/repl/log/mongodb.log"
   quiet: true
   logAppend: true
storage:
   dbPath: "/data/db/mongodb/repl/data"
   journal:
      enabled: true
   engine: "wiredTiger"
   wiredTiger:
      engineConfig:
          cacheSizeGB: 1
          directoryForIndexes: true
      collectionConfig:
          blockCompressor: zlib
      indexConfig:
          prefixCompression: true
operationProfiling:
   slowOpThresholdMs: 100
   mode: "slowOp"
processManagement:
   fork: true
   pidFilePath: "/data/db/mongodb/repl/mongodb.pid"
net:
   bindIp: "192.168.0.100"
   port: 27100
replication:
  replSetName: "replSet"
##security:
##   authorization: enabled
##   keyFile: "/data/db/mongodb/repl/mongodb-keyfile"
```


从节点(192.168.0.101)的配置文件(/data/db/mongodb/repl/etc/config.yml)
```bash
systemLog:
   destination: "file"
   path: "/data/db/mongodb/repl/log/mongodb.log"
   quiet: true
   logAppend: true
storage:
   dbPath: "/data/db/mongodb/repl/data"
   journal:
      enabled: true
   engine: "wiredTiger"
   wiredTiger:
      engineConfig:
          cacheSizeGB: 1
          directoryForIndexes: true
      collectionConfig:
          blockCompressor: zlib
      indexConfig:
          prefixCompression: true
operationProfiling:
   slowOpThresholdMs: 100
   mode: "slowOp"
processManagement:
   fork: true
   pidFilePath: "/data/db/mongodb/repl/mongodb.pid"
net:
   bindIp: "192.168.0.101"
   port: 27100
replication:
  replSetName: "replSet"
##security:
##   authorization: enabled
##   keyFile: "/data/db/mongodb/repl/mongodb-keyfile"
```


仲裁节点(192.168.0.102)的配置文件(/data/db/mongodb/repl/etc/config.yml)
```bash
systemLog:
   destination: "file"
   path: "/data/db/mongodb/repl/log/mongodb.log"
   quiet: true
   logAppend: true
storage:
   dbPath: "/data/db/mongodb/repl/data"
   journal:
      enabled: true
   engine: "wiredTiger"
   wiredTiger:
      engineConfig:
          cacheSizeGB: 1
          directoryForIndexes: true
      collectionConfig:
          blockCompressor: zlib
      indexConfig:
          prefixCompression: true
operationProfiling:
   slowOpThresholdMs: 100
   mode: "slowOp"
processManagement:
   fork: true
   pidFilePath: "/data/db/mongodb/repl/mongodb.pid"
net:
   bindIp: "192.168.0.102"
   port: 27100
replication:
  replSetName: "replSet"
##security:
##   authorization: enabled
##   keyFile: "/data/db/mongodb/repl/mongodb-keyfile"
```


**注意：**

1、每个配置文件看上去好像是一样，不过最重要的是要确定好端口有没有在本机被占用和绑定的IP是否确定为本机IP

2、每个配置文件最后3行是开启用户认证的，如果是生产环境必须先在无用户认证下先创建好管理员账号，然后把用户认证的参数打开(取消注释)

3、增加安全认证机制KeyFile，配置文件最后的`keyfile`是MongoDB集群认证的方式之一，确保集群中的成员都是信任的，生成keyfile命令如下：
```bash
$ openssl rand -base64 745 > /data/db/mongodb/repl/mongodb-keyfile
$ chmod 600 /data/db/mongodb/repl/mongodb-keyfile
$ chown mongod:mongod /data/db/mongodb/repl/mongodb-keyfile
```


### 启动

在所有节点中执行命令

```bash
$ mongod -f /data/db/mongodb/repl/etc/config.yml
```

如果发现启动错误或者失败，可以先从配置文件检查，再看文件目录，查看文件目录是否建好，用户和组的设置有没有设成mongod，再不行请查看mongodb日志文件排错。

若使用MongoDB3.0.9，有可能是mongodb-keyfile最后两个`=`符号需要删去才能启动(之前遇到过的坑，应该是版本的bug)

成功启动后，登陆到主节点中

``` bash
$ mongo 192.168.0.100:27100/admin

> config = { _id:"replSet",
             members:[
                      {_id:0,host:"192.168.0.100:27100",priority:100},
                      {_id:1,host:"192.168.0.101:27100",priority:99},
                      {_id:2,host:"192.168.0.102:27100",arbiterOnly:true}
  ]}


> rs.initiate(config)

## 稍等一会之后，按几下回车，应该就会见到命令行发生改变

>
>
replSet:PRIMARY>

## 查看rs集群状态
replSet:PRIMARY> rs.status()

```

其中，刚才的配置里面`priority`的值越大，就会选取最大的那个做主节点。`arbiterOnly`则是定义这个节点是用于仲裁作用的。


副本集配置完成。










