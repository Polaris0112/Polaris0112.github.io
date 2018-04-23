---
title: Zabbix YUM方式部署
date: 2018-02-25
categories: Zabbix YUM安装
tags: 
- Zabbix
- Yum
- Installation

---
本帖子记录的是使用YUM方式安装Zabbix3.4.x

## Zabbix简介

Zabbix 是由Alexei Vladishev创建，目前由Zabbix SIA在持续开发和支持。

Zabbix 是一个企业级的分布式开源监控方案。

Zabbix是一款能够监控各种网络参数以及服务器健康性和完整性的软件。Zabbix使用灵活的通知机制，允许用户为几乎任何事件配置基于邮件的告警。这样可以快速反馈服务器的问题。基于已存储的数据，Zabbix提供了出色的报告和数据可视化功能。这些功能使得Zabbix成为容量规划的理想方案。

Zabbix支持主动轮询和被动捕获。Zabbix所有的报告、统计信息和配置参数都可以通过基于Web的前端页面进行访问。基于Web的前端页面可以确保您从任何方面评估您的网络状态和服务器的健康性。适当的配置后，Zabbix可以在IT基础架构监控方面扮演重要的角色。对于只有少量服务器的小型组织和拥有大量服务器的大型公司也同样如此。

Zabbix是免费的。Zabbix是根据GPL通用公共许可证第2版编写和发行的。这意味着它的源代码都是免费发行的，可供公众任意使用。

想了解更多Zabbix相关信息，可以参考[Zabbix官方文档](https://www.zabbix.com/documentation/3.4/zh/start)


## 安装环境


- CentOS 7/RHEL 7

整套安装流程是在CentOS7上的zabbix-server和zabbix-agent，使用前端网页服务器是apahce


### yum安装Zabbix Server、mariadb
``` bash
$ rpm -i http://repo.zabbix.com/zabbix/3.4/rhel/7/x86_64/zabbix-release-3.4-2.el7.noarch.rpm
$ yum install -y zabbix-server-mysql zabbix-web-mysql zabbix-agent mariadb*

```


### 启动、配置mariadb
``` bash
## 启动并设置开机自启动数据库
$ systemctl start mariadb
$ systemctl enable mariadb


## 配置mariadb
$ mysql_secure_installation


## 首先是设置密码，会提示先输入密码
$ Enter current password for root (enter for none):<–初次运行直接回车


## 设置密码
$ Set root password? [Y/n] <– 是否设置root用户密码，输入y并回车或直接回车
$ New password: <– 设置root用户的密码
$ Re-enter new password: <– 再输入一次你设置的密码


## 其他配置
$ Remove anonymous users? [Y/n] <– 是否删除匿名用户，回车

$ Disallow root login remotely? [Y/n] <–是否禁止root远程登录,回车,

$ Remove test database and access to it? [Y/n] <– 是否删除test数据库，回车

$ Reload privilege tables now? [Y/n] <– 是否重新加载权限表，回车


## 初始化MariaDB完成，接下来测试登录
$ mysql -uroot -ppassword
## 能进入数据库则完成安装
```


### 初始化zabbix mariadb
``` bash
$ mysql -uroot -p
#password

mysql> create database zabbix character set utf8 collate utf8_bin;
mysql> grant all privileges on zabbix.* to zabbix@localhost identified by 'password';
mysql> quit;


## 导入zabbix初始化sql
$ zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix
```


### 配置zabbix_server.conf
``` bash
$ vi /etc/zabbix/zabbix_server.conf
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=password
```


### 启动zabbix相关服务和http服务
``` bash
$ systemctl start zabbix-server zabbix-agent httpd
$ systemctl enable zabbix-server zabbix-agent httpd
```


### 配置Zabbix前端的PHP文件
``` bash
$ vi /etc/httpd/conf.d/zabbix.conf

## 对应位置默认已经设置好，剩下的就是需要修改date.timezone，把默认注释的去掉，然后后面的值(Europe/Riga)改成当前时区(Asia/Shanghai)
php_value max_execution_time 300
php_value memory_limit 128M
php_value post_max_size 16M
php_value upload_max_filesize 2M
php_value max_input_time 300
php_value always_populate_raw_post_data -1
php_value date.timezone Aisa/Shanghai

```


### 配置Zabbix Web

登录到`http://server_ip_or_name/zabbix`，然后根据指示来安装[前端界面](https://www.zabbix.com/documentation/3.4/manual/installation/install#installing_frontend)


### Zabbix使用手册

Zabbix日常使用和新版本特征可以参考[官方使用手册](https://www.zabbix.com/documentation/3.4/manual/quickstart/login)


需要安装其他版本的zabbix或者不同的系统或者不同的数据库可以参考[官方安装手册](https://www.zabbix.com/download?zabbix=3.4&os_distribution=centos&os_version=7&db=MySQL)进行选择对应的安装教程。




## 其他


### 部署分布式监控Zabbix Proxy

Zabbix Proxy适用于大规模分布式监控场景，采用的一种分担Server端压力的分层结构，Proxy只负责一定区域内的数据采集工作，然后定期将数据一次性发送给Server，极大的减轻了Server的负载压力，使得可以支持更大规模的监控需求。

![zabbix-proxy](/images/zabbix-proxy.png)



### 用yum安装部署zabbix-proxy


``` bash
## 安装zabbix-release rpm包，若已经进行则可以无视以下这一条命令
$ rpm -i http://repo.zabbix.com/zabbix/3.4/rhel/7/x86_64/zabbix-release-3.4-2.el7.noarch.rpm

## 安装zabbix-proxy和zabbix-java-gateway
$ yum install -y  zabbix-proxy zabbix-java-gateway

## 默认已经存在mysql，没有则自行安装，安装配置命令上述教程提及到
$ mysql -u root -p
> create database zabbix_proxy character set utf8;       #数据名可以跟server端名称不同
> GRANT ALL PRIVILEGES ON zabbix_proxy.*  TO 'zabbix'@'localhost'  IDENTIFIED BY 'zabbix-proxy';
> GRANT ALL PRIVILEGES ON zabbix_proxy.* TO 'zabbix'@'%'  IDENTIFIED BY 'zabbix-proxy';
> flush PRIVILEGES;
> set GLOBAL max_connections=10000;
> flush privileges;

## 初始化zabbix-proxy数据库
$ cd /usr/share/doc/zabbix-proxy-mysql-3.0.1/
$ zcat schema.sql.gz | mysql -uroot -p zabbix_proxy

## 修改zabbix-proxy.conf配置文件
$ vi /etc/zabbix/zabbix-proxy.conf

Server=10.40.150.150    #同步指向的server端的IP，非本地IP。可以是server端的主机域名，但要确保proxy端解析server的域名，并且网络可达
Hostname=zabbix-proxy-test  #proxy本地的名称，此名称需要与将来在server端的WEB页面上的代理程序名称一致，名称自定义
DBHost=localhost            #与上面配置对应
DBName=zabbix_proxy   #与上面配置对应
DBUser=zabbix                #与上面配置对应
DBPassword=zabbix-proxy   #与上面配置对应
DBPort=3306                       #与上面配置对应
 
StartDiscoverers=4              #与server端配置的功能说明一致。
 
JavaGateway=127.0.0.1       #与server端配置的功能说明一致。
JavaGatewayPort=10052     #与server端配置的功能说明一致。
StartJavaPollers=4               #与server端配置的功能说明一致。
 
StartSNMPTrapper=1


## 启动服务并设置开机启动
$ systemctl start zabbix-java-gateway zabbix-proxy
$ systemctl enable zabbix-java-gateway zabbix-proxy
```

致此，系统级proxy安装设置完成。回到server 端WEB页面(管理-->agent代理程序-->创建代理)添加代理机(主动式)，被动式的话需要填写proxy的公网IP。

如果之后，我们可以在主机页面上做监控添加，选择指向agent上报至proxy端。




