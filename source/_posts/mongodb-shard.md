---
title: MongoDB Sharding集群部署
date: 2018-02-24
categories: MongoDB分片集群部署
tags: 
- NoSql
- MongoDB
- Sharding
- replSet

---
本帖子记录的是MongoDB分片+副本集集群部署

## MongoDB Sharding简介
  分片（sharding）是MongoDB用来将大型集合分割到不同服务器（或者说一个集群）上所采用的方法。尽管分片起源于关系型数据库分区，但MongoDB分片完全又是另一回事。和MySQL分区方案相比，MongoDB的最大区别在于它几乎能自动完成所有事情，只要告诉MongoDB要分配数据，它就能自动维护数据在不同服务器之间的均衡。 


### 分片的目的
  高数据量和吞吐量的数据库应用会对单机的性能造成较大压力,大的查询量会将单机的CPU耗尽,大的数据量对单机的存储压力较大,最终会耗尽系统的内存而将压力转移到磁盘IO上。

  为了解决这些问题,有两个基本的方法: 垂直扩展和水平扩展。

  垂直扩展：增加更多的CPU和存储资源来扩展容量。

  水平扩展：将数据集分布在多个服务器上。水平扩展即分片。


### 分片设计思想
  分片为应对高吞吐量与大数据量提供了方法。使用分片减少了每个分片需要处理的请求数，因此，通过水平扩展，集群可以提高自己的存储容量和吞吐量。举例来说，当插入一条数据时，应用只需要访问存储这条数据的分片.

  使用分片减少了每个分片存储的数据。

  例如，如果数据库1tb的数据集，并有4个分片，然后每个分片可能仅持有256 GB的数据。如果有40个分片，那么每个切分可能只有25GB的数据。

![mongodb-sharding-concept](/images/mongodb-sharding-concept.png)


### 分片设计架构

![mongodb-sharding-struct](/images/mongodb-sharding-struct.png)


![mongodb-sharding-pic](/images/mongodb-sharding-pic.png)


## 分片部署

- 系统：CentOS 7/RHEL 7


### 前期准备

准备三台服务器做副本集，具体操作可以参考[《MongoDB复制集部署》](https://polaris0112.github.io/2018/02/24/mongodb-replset/)

三个节点都用相同端口配置
Sharding port: 27100  27101 27102 27103
Config Server port: 27000
Mongos port: 30000

**请注意，配置文件会区别，不能完全照搬**

假设我们都已经安装好MongoDB，参考[《MongoDB单实例部署》](https://polaris0112.github.io/2018/02/24/mongodb-single/)

这个例子我们是分4片，4个Sharding


先在192.168.0.100创建目录路径
``` bash
## Sharding 0
$ mkdir -p /data/db/mongodb/shard/s0/data
$ mkdir -p /data/db/mongodb/shard/s0/log
$ mkdir -p /data/db/mongodb/shard/s0/etc

## Sharding 1
$ mkdir -p /data/db/mongodb/shard/s1/data
$ mkdir -p /data/db/mongodb/shard/s1/log
$ mkdir -p /data/db/mongodb/shard/s1/etc

## Sharding 2
$ mkdir -p /data/db/mongodb/shard/s2/data
$ mkdir -p /data/db/mongodb/shard/s2/log
$ mkdir -p /data/db/mongodb/shard/s2/etc

## Sharding 3
$ mkdir -p /data/db/mongodb/shard/s3/data
$ mkdir -p /data/db/mongodb/shard/s3/log
$ mkdir -p /data/db/mongodb/shard/s3/etc
```

## 编辑Sharding 0的配置文件(/data/db/mongodb/shard/s0/etc/config.yml)
``` bash
systemLog:
   destination: "file"
   path: "/data/db/mongodb/shard/s0/log/s0.log"
   quiet: true
   logAppend: true
storage:
   dbPath: "/data/db/mongodb/shard/s0/data"
   journal:
      enabled: true
   engine: "wiredTiger"
   wiredTiger:
      engineConfig:
          cacheSizeGB: 1
processManagement:
   fork: true
   pidFilePath: "/data/db/mongodb/shard/s0/shard.pid"
net:
   bindIp: "192.168.0.100"
   port: 27100
sharding:
   clusterRole: "shardsvr"
replication:
  replSetName: "s0"
##security:
##  authorization: enabled
##  keyFile: "/data/db/mongodb/mongodb-keyfile"
```

参照以上的配置文件，修改`path`、`dbPath`、`pidFilePath`、`bindIp`、`port`、`replSetName`为对应的shard配置(sharding在同一节点上)

同理，副本集也要对应放置相同的sharding配置文件，然后启动该实例。

启动命令：
``` bash
$ sudo -u mongod mongod -f /data/db/mongodb/shard/s0/etc/config.yml
```
其他实例可以仿照这个启动命令进行启动

启动后，先配置每个分片的副本集，对应一个sharding一个副本集，副本集名为s0,s1,s2,s3

副本集详细配置请参考[《MongoDB复制集部署》](https://polaris0112.github.io/2018/02/24/mongodb-replset/)


然后我们来创建分片中的config server。

先创建文件目录

``` bash
$ mkdir -p /data/db/mongodb/config/data
$ mkdir -p /data/db/mongodb/config/log
$ mkdir -p /data/db/mongodb/config/etc
```

创建配置文件(/data/db/mongodb/config/etc/config.yml)

``` bash
systemLog:
   destination: "file"
   path: "/data/db/mongodb/config/log/config.log"
   quiet: true
   logAppend: true
storage:
   dbPath: "/data/db/mongodb/config/data"
   journal:
      enabled: true
   engine: "wiredTiger"
   wiredTiger:
      engineConfig:
          cacheSizeGB: 1
processManagement:
   fork: true
   pidFilePath: "/data/db/mongodb/config/config.pid"
net:
   bindIp: "192.168.0.100"
   port: 27000
sharding:
   clusterRole: "configsvr"
replication:
  replSetName: "cs"
##security:
##  authorization: enabled
##  keyFile: "/data/db/mongodb/mongodb-keyfile"

```

同样地，在三个节点都创建对应的配置文件(IP地址需要修改为本机的IP)，并创建副本集。

启动实例并配置副本集
``` bash
$ mongod -f /data/db/mongodb/config/etc/config.yml
```


创建mongos文件路径

``` bash
$ mkdir -p /data/db/mongodb/mongos/data
$ mkdir -p /data/db/mongodb/mongos/log
$ mkdir -p /data/db/mongodb/mongos/etc
```


创建mongos配置文件(/data/db/mongodb/mongos/etc/config.yml)

``` bash
systemLog:
   destination: "file"
   path: "/data/db/mongodb/mongos/log/mongos.log"
   quiet: true
   logAppend: true
   logRotate: reopen
processManagement:
   fork: true
   pidFilePath: "/data/db/mongodb/mongos/mongos.pid"
net:
   bindIp: "192.168.0.100"
   port: 30000
sharding:
   configDB: 192.168.0.100:27100,192.168.0.101:27100,192.168.0.102:27100
   chunkSize: 64
##security:
##  authorization: enabled
##   keyFile: "/data/db/mongodb/mongodb-keyfile"

```

启动命令：

``` bash
$ mongod -f /data/db/mongodb/mongos/etc/config.yml
```

mongos作为整个分片集群的入口，不需要配置副本集，不过可以配置多个节点的mongos，留意的是`configDB`需要指定所有节点的config server，若配置了多个节点的mongos，可以使用mongodb uri达到高可用的效果。


mongos成功启动后，可以开始进行sharding的配置。

进入其中一个mongos

``` bash
$ mongo 192.168.0.100:30000/admin

## 添加分片
mongos> sh.addShard("shard1/192.168.0.100:27100,192.168.0.101:27100,192.168.0.102:27100")
mongos> sh.addShard("shard2/192.168.0.100:27101,192.168.0.101:27101,192.168.0.102:27101")
mongos> sh.addShard("shard3/192.168.0.100:27102,192.168.0.101:27102,192.168.0.102:27102")
mongos> sh.addShard("shard4/192.168.0.100:27103,192.168.0.101:27103,192.168.0.102:27103")

## 查看集群状态
mongos> sh.status()

## 指定特定数据库分片生效
mongos> db.runCommand( { enablesharding :"test"});

## 指定数据库里需要分片的集合和片键
db.runCommand( { shardcollection : "testdb.table1",key : {id: 1} } )
```

到此，分片+副本集部署完毕。



### 记得修改文件路径权限，最好使用mongod用户来启动
``` bash
$ chown -R mongod:mongod /data/db/mongodb

```




