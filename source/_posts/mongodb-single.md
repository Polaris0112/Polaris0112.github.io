---
title: MongoDB单实例部署
date: 2018-02-24
categories: MongoDB单实例部署
tags: 
- NoSql
- MongoDB

---
本帖子记录的是MongoDB单实例部署全过程

## MongoDB简介

MongoDB是用C++语言编写的非关系型数据库。特点是高性能、易部署、易使用，存储数据十分方便，主要特性有： 

- 面向集合存储，易于存储对象类型的数据 

- 模式自由 

- 支持动态查询 

- 支持完全索引，包含内部对象 

- 支持复制和故障恢复 

- 使用高效的二进制数据存储，包括大型对象 

- 文件存储格式为BSON(一种JSON的扩展)


[官网下载地址](https://www.mongodb.com/download-center#atlas)


[官方文档](https://docs.mongodb.com/manual/introduction/)
 

## 安装

- 系统：CentOS 7/RHEL 7


### yum安装方式

``` bash
## 以3.4.x为例

## 创建yum源文件
$ vim /etc/yum.repos.d/mongodb-org-3.4.repo

## 把下面的内容复制到上面的文件中
[mongodb-org-3.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.4/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.4.asc123456

## 启动yum命令开始安装
$ yum install -y mongodb-org
```

各个安装包说明：

![mongodb-org](/images/mongodb-org.png)



## 编译安装方式

``` bash
## 以3.4.13版本为例

## 使用wget命令下载安装包
$ wget http://downloads.mongodb.org/linux/mongodb-linux-x86_64-rhel70-3.4.13.tgz

## 解压压缩包，并配置相关文件
$ tar zxf mongodb-linux-x86_64-rhel70-3.4.13.tgz
$ mv mongodb-linux-x86_64-rhel70-3.4.13 mongodb
$ mv mongodb /usr/local/
## 那么/usr/local/mongodb/bin/下的命令就是mongodb相关的命令

## 若系统不存在mongodb用户则进行创建
$ groupadd mongod
$ useradd -r -g mongod -s /sbin/nologin -M mongod

## 现在创建mongodb数据库所需要的文件路径
$ mkdir -p /data/db/mongodb/single/data
$ mkdir -p /data/db/mongodb/single/etc
$ mkdir -p /data/db/mongodb/single/log
$ chown -R mongod:mongod /data/db/mongodb

```



## 配置文件参数说明

``` bash
storage:  
   dbPath: <string>                       #存储数据目录  
   indexBuildRetry: <boolean>         #当构建索引时mongod意外关闭，那么再次启动是否重新构建索引；索引构建失败，mongod重启后将会删除尚未完成的索引，但是否重建由此参数决定。默认值为true。  
   repairPath: <string>                   #配合--repair启动命令参数，在repair期间使用此目录存储临时数据，repair结束后此目录下数据将被删除，此配置仅对mongod进程有效。不建议在配置文件中配置，而是使用mongod启动命令指定。  
   journal:  
      enabled: <boolean>              #是否开启journal日志持久存储，journal日志用来数据恢复，是mongod最基础的特性，通常用于故障恢复。  
      commitIntervalMs: <num>         #New in version 3.2.  日志提交间隔  
   directoryPerDB: <boolean>          #将不同DB的数据分子目录存储，基于dbPath，默认为 false  
   syncPeriodSecs: <int>              #使用fsync操作将数据flush到磁盘的时间间隔，默认值为60（单位：秒），建议不修改  
   engine: <string>                       #存储引擎，3.2后默认wiredTiger 可选 mmapv1  
   mmapv1:  
      preallocDataFiles: <boolean>        #Deprecated since version 2.6. 已废弃  
      nsSize: <int>  
      quota:  
         enforced: <boolean>  
         maxFilesPerDB: <int>  
      smallFiles: <boolean>  
      journal:  
         debugFlags: <int>  
         commitIntervalMs: <num>  
   wiredTiger:  
      engineConfig:  
         cacheSizeGB: <number>                #GB，此值决定了wiredTiger与mmapv1的内存模型不同，它可以限制mongod对内存的使用量  
         statisticsLogDelaySecs: <number>  
         journalCompressor: <string>      #journal日志的压缩算法，可选值为“none”、“snappy”、“zlib”。  
         directoryForIndexes: <boolean>       #是否将索引和collections数据分别存储在dbPath单独的目录中。即index数据保存“index”子目录，collections数据保存在“collection”子目录。默认值为false，仅对mongod有效。  
      collectionConfig:  
         blockCompressor: <string>            #collection数据压缩算法，可选值“none”、“snappy”、“zlib”  
      indexConfig:    
         prefixCompression: <boolean>     #是否对索引数据使用“前缀压缩”（prefix compression，一种算法）。前缀压缩，对那些经过排序的值存储，有很大帮助，可以有效的减少索引数据的内存使用量。默认值为true。  

operationProfiling:  
   slowOpThresholdMs: <int>    #数据库profiler判定一个操作是“慢查询”的时间阀值，单位毫秒；  
   mode: <string>          #off：关闭profiling  slowOp：on，只包含慢操作日志； all  On. Includes all operations.  
                #Database profiling can impact database performance. Enable this option only after careful consideration.  
                #对性能有影响，建议只在调试期间开启  
systemLog:          #系统日志配置  
   verbosity: <int> #日志级别，0：默认值，包含“info”信息，1~5，即大于0的值均会包含debug信息  
   quiet: <boolean>  #"安静"，此时mongod/mongos将会尝试减少日志的输出量。不建议在production环境下开启，否则将会导致跟踪错误比较困难。  
   traceAllExceptions: <boolean> # 打印异常详细信息  
   syslogFacility: <string> #The facility level used when logging messages to syslog.  default:user  
   path: <string>         #日志文件路径  
   logAppend: <boolean>       #启动或重启后是否追加写入  
   logRotate: <string>        #防止一个日志文件特别大，可选项：rename(重命名日志文件，默认值);reopen(使用linux日志rotate特性，关闭并重新打开此日志文件，可以避免日志丢失，但是logAppend必须为true)  
   destination: <string>  #日志输出目的地，可为 file 或 syslog; if file, you must also specify systemLog.path.  
   timeStampFormat: <string>#时间格式 默认为 iso8601-local     
   component:  
      accessControl:  
         verbosity: <int>  
      command:  
         verbosity: <int>  

net:                    #网络相关配置  
   port: <int>            #监听端口  
   bindIp: <string>       #监听ip，不指定则监听所有网卡，  
   maxIncomingConnections: <int> #进程允许的最大连接数，默认:65536  
   wireObjectCheck: <boolean> #当客户端写入数据时，mongos/mongod是否检测数据的有效性(BSON)，如果数据格式不良，此insert、update操作将会被拒绝；默认值为true  
   ipv6: <boolean>    #是否支持mongos/mongod多个实例之间使用IPV6网络，默认值为false。此值需要在整个cluster中保持一致。  
   unixDomainSocket:  
      enabled: <boolean>  
      pathPrefix: <string>  
      filePermissions: <int>  
   http:  
      enabled: <boolean>  #Enable or disable the HTTP interface.   
      JSONPEnabled: <boolean>  
      RESTInterfaceEnabled: <boolean>  
   ssl:  
      sslOnNormalPorts: <boolean>  # deprecated since 2.6  
      mode: <string>  
      PEMKeyFile: <string>  
      PEMKeyPassword: <string>  
      clusterFile: <string>  
      clusterPassword: <string>  
      CAFile: <string>  
      CRLFile: <string>  
      allowConnectionsWithoutCertificates: <boolean>  
      allowInvalidCertificates: <boolean>  
      allowInvalidHostnames: <boolean>  
      disabledProtocols: <string>  
      FIPSMode: <boolean>  

```

 



**mongodb配置文件例子**(/data/db/mongodb/single/etc/config.yml)


``` bash
systemLog:
   destination: file
   path: "/data/db/mongodb/single/log/mongodb.log"
   quiet: true
   logAppend: true
storage:
   dbPath: "/data/db/mongodb/single/data"
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
   pidFilePath: "/data/db/mongodb/single/mongodb.pid"
net:
   bindIp: 192.168.0.123
   port: 37017
#security:
   ##keyFile: /data/mongodb/config/data/keyfile
   ##authorization: enabled

```



## 启动命令 

因为安全问题，不建议直接使用`root`来启动数据库

- yum安装方式

``` bash
$ sudo -u mongod mongod -f /data/db/mongodb/single/config.yml
```

- 编译安装方式

``` bash
$ /usr/local/mongodb/bin/mongod -f /data/db/mongodb/single/config.yml
```



