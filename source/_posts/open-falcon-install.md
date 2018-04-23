---
title: open-falcon安装部署教程
date: 2018-03-09
categories: open-falcon安装部署教程
tags: 
- open-falcon
- centos

---
本帖子记录的是open-falcon安装部署教程

## 环境准备

- 系统：CentOS 7 最小安装版

## 安装redis

```bash
$ yum install -y redis
```


## 安装mysql

```bash
$ yum install -y mysql-server
```

**注意，请确保redis和MySQL已启动。**



## 初始化MySQL表结构

```bash
$ cd /tmp/ && git clone https://github.com/open-falcon/falcon-plus.git 
$ cd /tmp/falcon-plus/scripts/mysql/db_schema/
$ mysql -h 127.0.0.1 -u root -p < 1_uic-db-schema.sql
$ mysql -h 127.0.0.1 -u root -p < 2_portal-db-schema.sql
$ mysql -h 127.0.0.1 -u root -p < 3_dashboard-db-schema.sql
$ mysql -h 127.0.0.1 -u root -p < 4_graph-db-schema.sql
$ mysql -h 127.0.0.1 -u root -p < 5_alarms-db-schema.sql
$ rm -rf /tmp/falcon-plus/
```

**如果你是从v0.1.0升级到当前版本v0.2.0，那么只需要执行如下命令：**

```bash
$ mysql -h 127.0.0.1 -u root -p < 5_alarms-db-schema.sql
```



## 安装go环境

go-v1.10下载地址：
- [linux64位下载地址](https://dl.google.com/go/go1.10.linux-amd64.tar.gz)

- [linux32位下载地址](https://dl.google.com/go/go1.10.linux-386.tar.gz)

- [windows64位编译包下载地址](https://dl.google.com/go/go1.10.windows-amd64.zip)

- [windows64位安装包下载地址](https://dl.google.com/go/go1.10.windows-amd64.msi)

- [windows32位编译包下载地址](https://dl.google.com/go/go1.10.windows-386.zip)

- [windows32位安装包下载地址](https://dl.google.com/go/go1.10.windows-386.msi)


```bash
## 下载linux64位压缩包
$ wget https://dl.google.com/go/go1.10.linux-amd64.tar.gz
$ tar -C /usr/local -xzf go1.10.linux-amd64.tar.gz
$ echo "export PATH=$PATH:/usr/local/go/bin" >> /etc/profile
$ source /etc/profile
$ go version
go version go1.10 linux/amd64
```
go环境部署完毕。



## 下载编译好的二进制版本

[官方编译好的v0.2.1版本](https://github.com/open-falcon/falcon-plus/releases/download/v0.2.1/open-falcon-v0.2.1.tar.gz)

```bash
$ mkdir -p ~/open-falcon
$ cd ~/open-falcon
$ wget https://github.com/open-falcon/falcon-plus/releases/download/v0.2.1/open-falcon-v0.2.1.tar.gz
$ tar -xzvf open-falcon-v0.2.1.tar.gz
```


## 首先确认配置文件中数据库账号密码与实际相同，否则需要修改配置文件

### 修改配置文件cfg.json

猜测部分模块依赖连接数据库，因为如果不修改配置文件，aggregator模块会出现无法启动，graph、hbs、nodata、api、alarm模块会出现开启不报错但是状态为开启失败的情况。

如果需要每个模块都能正常启动，需要将上面模块的cfg.json的数据库信息进行修改。根据本教程的配置，需要修改配置文件所在的目录： 

| 模块        |         配置文件所在路径                    |
| --------    | :----:                                      |
| aggregator  | ~/open-falcon/aggregator/config/cfg.json    |
| graph       | ~/open-falcon/graph/config/cfg.json         |
| hbs         | ~/open-falcon/hbs/config/cfg.json           |
| nodata      | ~/open-falcon/nodata/config/cfg.json        |
| api         | ~/open-falcon/api/config/cfg.json           |
| alarm       | ~/open-falcon/alarm/config/cfg.json         |


**1）修改aggregator的配置文件**

```bash
$ vim ~/open-falcon/aggregator/config/cfg.json
```
![open-falcon-aggregator-cfg](/images/open-falcon-aggregator-cfg.png)

mysql的root密码为空，则去掉“password”，若不为空，则用root密码替换“password”。



**2）修改graph的配置文件**

```bash
$ vim ~/open-falcon/graph/config/cfg.json
```
![open-falcon-graph-cfg](/images/open-falcon-graph-cfg.png)

mysql的root密码为空，则去掉“password”，若不为空，则用root密码替换“password”。



**3）修改hbs的配置文件**

```bash
$ vim ~/open-falcon/hbs/config/cfg.json
```
![open-falcon-hbs-cfg](/images/open-falcon-hbs-cfg.png)

mysql的root密码为空，则去掉“password”，若不为空，则用root密码替换“password”。



**4）修改nodata的配置文件**

```bash
$ vim ~/open-falcon/nodata/config/cfg.json
```
![open-falcon-nodata-cfg](/images/open-falcon-nodata-cfg.png)

mysql的root密码为空，则去掉“password”，若不为空，则用root密码替换“password”。



**5）修改api的配置文件**

```bash
$ vim ~/open-falcon/api/config/cfg.json
```
![open-falcon-api-cfg](/images/open-falcon-api-cfg.png)

mysql的root密码为空，则去掉“password”，若不为空，则用root密码替换“password”。



**6）修改alarm的配置文件**

```bash
$ vim ~/open-falcon/alarm/config/cfg.json
```
![open-falcon-alarm-cfg](/images/open-falcon-alarm-cfg.png)

mysql的root密码为空，则去掉“password”，若不为空，则用root密码替换“password”。 



## 启动后端

```bash
$ cd ~/open-falcon
$ ./open-falcon start
[falcon-graph] 25103
[falcon-hbs] 25113
[falcon-judge] 25122
[falcon-transfer] 25129
[falcon-nodata] 25136
[falcon-aggregator] 25142
[falcon-agent] 25151
[falcon-gateway] 25158
[falcon-api] 25165
[falcon-alarm] 25174

## 检查任务情况
$ ./open-falcon check
        falcon-graph         UP           25103 
          falcon-hbs         UP           25113 
        falcon-judge         UP           25122 
     falcon-transfer         UP           25129 
       falcon-nodata         UP           25136 
   falcon-aggregator         UP           25142 
        falcon-agent         UP           25151 
      falcon-gateway         UP           25158 
          falcon-api         UP           25165 
        falcon-alarm         UP           25174 

## 更多工具用法
# ./open-falcon [start|stop|restart|check|monitor|reload] module
# For debugging , You can check ~/open-falcon/$moduleName/log/logs/xxx.log
```




## 部署前端

### 获取前端代码

```bash
## 先安装依赖
$ yum install -y python-virtualenv
$ yum install -y python-devel
$ yum install -y openldap-devel
$ yum install -y mysql-devel
$ yum install -y git
$ yum groupinstall "Development tools" -y

## 下载前端源码
$ cd ~/open-falcon
$ git clone https://github.com/open-falcon/dashboard.git

## 进入前端文件夹用pip安装对应依赖
$ cd ./dashboard
$ virtualenv ./env
$ ./env/bin/pip install -r pip_requirements.txt
```


### 修改配置

根据本次记录的配置，dashboard的配置文件在`~/open-falcon/dashboard/rrd/config.py`，需要根据实际情况对内部配置进行修改。

按照上述的数据库账号密码进行修改

```bash
## API_ADDR 表示后端api组件的地址
API_ADDR = "http://127.0.0.1:8080/api/v1" 

## 根据实际情况，修改PORTAL_DB_*, 默认用户名为root，默认密码为""
## 根据实际情况，修改ALARM_DB_*, 默认用户名为root，默认密码为""
```
![open-falcon-dashboard-cfg](/images/open-falcon-dashboard-cfg.png)

设置对应的IP地址，提供访问入口，并配置数据库对应账号密码



## 开启8081端口

```bash
# CentOS 7
$ firewall-cmd --add-port=8081/tcp --permanent
$ firewall-cmd --reload

# CentOS 6
$ iptables -I INPUT -p tcp --dport 8081 -j ACCEPT
```


### 以开发者模式启动

```bash
$ ./env/bin/python wsgi.py

## open http://127.0.0.1:8081 in your browser.
```


### 在生产环境启动

```bash
$ bash control start

## open http://127.0.0.1:8081 in your browser.
```


### 停止dashboard运行

```bash
$ bash control stop
```


### 查看日志

```bash
$ bash control tail
```


## dashbord用户管理

dashbord没有默认创建任何账号包括管理账号，需要你通过页面进行注册账号。
想拥有管理全局的超级管理员账号，需要手动注册用户名为root的账号（第一个帐号名称为root的用户会被自动设置为超级管理员）。
超级管理员可以给普通用户分配权限管理。

**小提示：**
注册账号能够被任何打开dashboard页面的人注册，所以当给相关的人注册完账号后，需要去关闭注册账号功能。只需要去修改api组件的配置文件cfg.json，将signup_disable配置项修改为true，重启api即可。当需要给人开账号的时候，再将配置选项改回去，用完再关掉即可。
