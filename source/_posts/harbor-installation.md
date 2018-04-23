---
title: Docker镜像仓库Harbor搭建
date: 2018-03-12
categories: Docker镜像仓库Harbor搭建
tags: 
- Docker
- Registry
- Harbor

---
本帖子记录的是Docker镜像仓库Harbor在CentOS 7环境下搭建全过程


## 简介
![harbor-logo](/images/harbor-logo.png)
Harbor 是一个企业级的 Docker Registry，可以实现 images 的私有存储和日志统计权限控制等功能，并支持创建多项目(Harbor 提出的概念)，基于官方 Registry V2 实现。官方提供了两种版本：[在线版v1.4.0](https://storage.googleapis.com/harbor-releases/release-1.4.0/harbor-online-installer-v1.4.0.tgz)和[离线版v1.4.0](https://storage.googleapis.com/harbor-releases/release-1.4.0/harbor-offline-installer-v1.4.0.tgz)。
更多版本请看，[官方release](https://storage.googleapis.com/harbor-releases)

## 环境准备

- CentOS 7

- Docker Version: 17.12.1

- Docker-Compose Version: 1.18.0


Docker环境请参考[《Docker安装部署教程》](https://polaris0112.github.io/2018/03/11/docker-install/)

`Docker-Compose`可以使用以下命令进行安装

```bash
$ curl -L https://github.com/docker/compose/releases/download/1.18.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```


## Harbor服务搭建

### 从官网下载安装包(推荐离线安装包)

因为国内连国外网经常出现慢或者断线的情况，导致安装某些依赖镜像的时候卡住，镜像又不支持断点续传，所以最好使用离线安装包。

```bash
## 下载harbor-v1.4.0离线安装包
$ wget https://storage.googleapis.com/harbor-releases/release-1.4.0/harbor-offline-installer-v1.4.0.tgz
$ tar xf harbor-offline-installer-v1.4.0.tgz
```


### 配置Harbor

解压缩之后，会有一个`Harbor`的文件夹，里面会有一个`harbor.cfg`配置文件，我们需要对里面的参数进行更改。

```bash
$ vim harbor.cfg
## 配置域名或者直接IP地址
hostname = harbor-test.com                              ##需要修改

## 访问协议，默认是http，也可以设置https，如果设置https，则nginx ssl需要设置on
ui_url_protocol = https

## 设置进程数
max_job_workers = 3 

## 设置自定义ssl认证
customize_crt = on
ssl_cert = /data/cert/harbor-test.com.crt               ##需要修改
ssl_cert_key = /data/cert/harbor-test.com.key           ##需要修改
secretkey_path = /data
admiral_url = NA
log_rotate_count = 50
log_rotate_size = 200M

## 邮件设置，发送重置密码邮件时使用
email_identity = 
email_server = smtp.exmail.qq.com                       ##需要修改
email_server_port = 645                                 ##需要修改
email_username =                                        ##需要修改
email_password =                                        ##需要修改
email_from = Harbor <xxxx@xxx.com>                      ##需要修改
email_ssl = false
email_insecure = false

## Harbor界面登录密码
harbor_admin_password = Harbor123                       ##需要修改
auth_mode = db_auth

## LDAP认证时配置项，如果上方auth_mode选择db_auth的话，下方其实没用到，但是注释的话会有报错。
ldap_url = ldaps://ldap.mydomain.com
ldap_searchdn = uid=searchuser,ou=people,dc=mydomain,dc=com
ldap_search_pwd = password
ldap_basedn = ou=people,dc=mydomain,dc=com
ldap_uid = uid 
ldap_scope = 2 
ldap_timeout = 5
ldap_verify_cert = true

## 是否开启自注册
self_registration = on

## Token有效时间，默认30分钟
token_expiration = 30

## 用户创建项目权限控制，默认是everyone（所有人），也可以设置为adminonly（只能管理员）
project_creation_restriction = everyone

## 数据库配置相关
db_host = mysql
db_password = 
db_port = 3306
db_user = root

## redis配置，如果有需要，没有的话可以留空
redis_url =

## clair镜像扫描功能，可以检查镜像操作系统以及上面安装包是否与已知不安全的包版本相匹配，从而提高镜像安全性。
clair_db_host = postgres
clair_db_password = password
clair_db_port = 5432
clair_db_username = postgres
clair_db = postgres

## 同理，以下设置的是uaa_auth登录方式
uaa_endpoint = uaa.mydomain.org
uaa_clientid = id
uaa_clientsecret = secret
uaa_verify_cert = true
uaa_ca_cert = /path/to/ca.pem

## 此处的registry后端使用filesystem
registry_storage_provider_name = filesystem
registry_storage_provider_config =
```

由于上述配置文件中涉及到ssl证书，生成在`/data/cert`目录下，所以需要执行以下生成命令：
```bash
$ mkdir -p /data/cert
$ cd /data/cert
$ localdomain=harbor-test.com   ##修改为自己的域名
$ openssl req -nodes -subj "/C=CN/ST=GuangDong/L=GuangZhou/CN=$localdomain" -newkey rsa:2048 -keyout $localdomain.key -out $localdomain.csr
$ openssl x509 -req -days 3650 -in $localdomain.csr -signkey $localdomain.key -out $localdomain.crt
$ openssl x509 -req -in $localdomain.csr -CA $localdomain.crt -CAkey $localdomain.key -CAcreateserial -out $localdomain.crt -days 10000
```

命令正常完成后应该会有四个文件，后缀名分别是.crt，.csr，.key，.srl，文件名就是对应的域名

修改完配置文件后，在的当前目录执行

```bash
$ ./install.sh
## 运行结果
[Step 0]: checking installation environment ...

Note: docker version: 17.12.1

Note: docker-compose version: 1.18.0

[Step 1]: loading Harbor images ...
651f69aef02c: Loading layer [==================================================>]  135.8MB/135.8MB
40a1aad64343: Loading layer [==================================================>]  23.24MB/23.24MB
3fe2713e4072: Loading layer [==================================================>]  12.16MB/12.16MB
ba3a1eb0e375: Loading layer [==================================================>]   17.3MB/17.3MB
447427ec5e1a: Loading layer [==================================================>]  15.87kB/15.87kB
4ccb4026663c: Loading layer [==================================================>]  3.072kB/3.072kB
16faa95946a1: Loading layer [==================================================>]  29.46MB/29.46MB
Loaded image: vmware/notary-server-photon:v0.5.1-v1.4.0
fa7ba9fd42c9: Loading layer [==================================================>]  10.95MB/10.95MB
4e400f9ae23e: Loading layer [==================================================>]   17.3MB/17.3MB
2802fb27c88b: Loading layer [==================================================>]  15.87kB/15.87kB
e6367a4e1e1e: Loading layer [==================================================>]  3.072kB/3.072kB
8ece8dfcdd98: Loading layer [==================================================>]  28.24MB/28.24MB
Loaded image: vmware/notary-signer-photon:v0.5.1-v1.4.0
a7dd1a8afcaf: Loading layer [==================================================>]  396.7MB/396.7MB
05adebbe496f: Loading layer [==================================================>]  9.216kB/9.216kB
86eb534949fa: Loading layer [==================================================>]  9.216kB/9.216kB
d7f127c69380: Loading layer [==================================================>]   7.68kB/7.68kB
5ac1c4dc5ee9: Loading layer [==================================================>]  1.536kB/1.536kB
d0bec56b5b1a: Loading layer [==================================================>]  9.728kB/9.728kB
4bbe83860556: Loading layer [==================================================>]   2.56kB/2.56kB
e526f9e6769f: Loading layer [==================================================>]  3.072kB/3.072kB
Loaded image: vmware/harbor-db:v1.4.0
1cff102bbda2: Loading layer [==================================================>]  154.1MB/154.1MB
04c9f3e07de1: Loading layer [==================================================>]  10.75MB/10.75MB
7b6c7bf54f5c: Loading layer [==================================================>]  2.048kB/2.048kB
42f8acdb7fe3: Loading layer [==================================================>]  48.13kB/48.13kB
5b6299d0a1df: Loading layer [==================================================>]   10.8MB/10.8MB
Loaded image: vmware/clair-photon:v2.0.1-v1.4.0
6534131f457c: Loading layer [==================================================>]  94.76MB/94.76MB
73f582101e4b: Loading layer [==================================================>]  6.656kB/6.656kB
86d847823c48: Loading layer [==================================================>]  6.656kB/6.656kB
Loaded image: vmware/postgresql-photon:v1.4.0
5cd250d5a352: Loading layer [==================================================>]  23.24MB/23.24MB
ad3fd52b54f3: Loading layer [==================================================>]  14.99MB/14.99MB
13b1e24cc368: Loading layer [==================================================>]  14.99MB/14.99MB
Loaded image: vmware/harbor-adminserver:v1.4.0
c26c69706710: Loading layer [==================================================>]  23.24MB/23.24MB
223f6fe02cc8: Loading layer [==================================================>]  23.45MB/23.45MB
1fc843c8698a: Loading layer [==================================================>]  7.168kB/7.168kB
e09293610ee7: Loading layer [==================================================>]  10.39MB/10.39MB
d59f9780b1d8: Loading layer [==================================================>]  23.44MB/23.44MB
Loaded image: vmware/harbor-ui:v1.4.0
dd4753242e59: Loading layer [==================================================>]  73.07MB/73.07MB
95aed61ca251: Loading layer [==================================================>]  3.584kB/3.584kB
1864f9818562: Loading layer [==================================================>]  3.072kB/3.072kB
da2a19f80b81: Loading layer [==================================================>]  4.096kB/4.096kB
058531639e75: Loading layer [==================================================>]  3.584kB/3.584kB
a84e69fb619b: Loading layer [==================================================>]  10.24kB/10.24kB
Loaded image: vmware/harbor-log:v1.4.0
b1056051f246: Loading layer [==================================================>]  23.24MB/23.24MB
07678065e08b: Loading layer [==================================================>]  19.19MB/19.19MB
a2d9bdb8f5fb: Loading layer [==================================================>]  19.19MB/19.19MB
Loaded image: vmware/harbor-jobservice:v1.4.0
7f58ce57cd5e: Loading layer [==================================================>]  4.805MB/4.805MB
Loaded image: vmware/nginx-photon:v1.4.0
4c8965978b77: Loading layer [==================================================>]  23.24MB/23.24MB
1466c942edde: Loading layer [==================================================>]  2.048kB/2.048kB
ac5c17331735: Loading layer [==================================================>]  2.048kB/2.048kB
86824c7c466a: Loading layer [==================================================>]  2.048kB/2.048kB
fd3bd0e70d67: Loading layer [==================================================>]   22.8MB/22.8MB
b02195d77636: Loading layer [==================================================>]   22.8MB/22.8MB
Loaded image: vmware/registry-photon:v2.6.2-v1.4.0
Loaded image: vmware/photon:1.0
Loaded image: vmware/mariadb-photon:v1.4.0
454c81edbd3b: Loading layer [==================================================>]  135.2MB/135.2MB
e99db1275091: Loading layer [==================================================>]  395.4MB/395.4MB
051e4ee23882: Loading layer [==================================================>]  9.216kB/9.216kB
6cca4437b6f6: Loading layer [==================================================>]  9.216kB/9.216kB
1d48fc08c8bc: Loading layer [==================================================>]   7.68kB/7.68kB
0419724fd942: Loading layer [==================================================>]  1.536kB/1.536kB
526b2156bd7a: Loading layer [==================================================>]  637.8MB/637.8MB
9ebf6900ecbd: Loading layer [==================================================>]  78.34kB/78.34kB
Loaded image: vmware/harbor-db-migrator:1.4

[Step 2]: preparing environment ...
Clearing the configuration file: ./common/config/adminserver/env
Clearing the configuration file: ./common/config/ui/env
Clearing the configuration file: ./common/config/ui/app.conf
Clearing the configuration file: ./common/config/ui/private_key.pem
Clearing the configuration file: ./common/config/db/env
Clearing the configuration file: ./common/config/jobservice/env
Clearing the configuration file: ./common/config/jobservice/app.conf
Clearing the configuration file: ./common/config/registry/config.yml
Clearing the configuration file: ./common/config/registry/root.crt
Clearing the configuration file: ./common/config/nginx/cert/harbor.qtz.crt
Clearing the configuration file: ./common/config/nginx/cert/harbor.qtz.key
Clearing the configuration file: ./common/config/nginx/nginx.conf
Clearing the configuration file: ./common/config/log/logrotate.conf
loaded secret from file: /data/secretkey
Generated configuration file: ./common/config/nginx/nginx.conf
Generated configuration file: ./common/config/adminserver/env
Generated configuration file: ./common/config/ui/env
Generated configuration file: ./common/config/registry/config.yml
Generated configuration file: ./common/config/db/env
Generated configuration file: ./common/config/jobservice/env
Generated configuration file: ./common/config/log/logrotate.conf
Generated configuration file: ./common/config/jobservice/app.conf
Generated configuration file: ./common/config/ui/app.conf
Generated certificate, key file: ./common/config/ui/private_key.pem, cert file: ./common/config/registry/root.crt
The configuration files are ready, please use docker-compose to start the service.

Creating harbor-log ... done
[Step 3]: checking existing instance of Harbor ...

Creating registry ... done
Creating harbor-ui ... done
Creating network "harbor_harbor" with the default driver
Creating nginx ... done
Creating harbor-db ... 
Creating harbor-adminserver ... 
Creating registry ... 
Creating harbor-ui ... 
Creating harbor-jobservice ... 
Creating nginx ... 

✔ ----Harbor has been installed and started successfully.----

Now you should be able to visit the admin portal at https://harbor.qtz. 
For more details, please visit https://github.com/vmware/harbor .

```


## 查看启动后状态

- Docker

```bash
$ docker images
REPOSITORY                                TAG                 IMAGE ID            CREATED             SIZE
vmware/clair-photon                       v2.0.1-v1.4.0       a1df3526fe43        6 weeks ago         300MB
vmware/notary-server-photon               v0.5.1-v1.4.0       3edfddb8ece2        6 weeks ago         211MB
vmware/notary-signer-photon               v0.5.1-v1.4.0       cc70a05cdb6a        6 weeks ago         209MB
vmware/registry-photon                    v2.6.2-v1.4.0       8920f621ddd1        6 weeks ago         198MB
vmware/nginx-photon                       v1.4.0              20c8a01ac6ab        6 weeks ago         135MB
vmware/harbor-log                         v1.4.0              9e818c7a27ab        6 weeks ago         200MB
vmware/harbor-jobservice                  v1.4.0              29c14d91b043        6 weeks ago         191MB
vmware/harbor-ui                          v1.4.0              6cb4318eda6a        6 weeks ago         209MB
vmware/harbor-adminserver                 v1.4.0              8145970fa013        6 weeks ago         182MB
vmware/harbor-db                          v1.4.0              c38da34727f0        6 weeks ago         521MB
vmware/mariadb-photon                     v1.4.0              8457013cf6e3        6 weeks ago         521MB
vmware/postgresql-photon                  v1.4.0              59aa61520094        6 weeks ago         221MB
vmware/harbor-db-migrator                 1.4                 7a4d871b612e        2 months ago        1.15GB
vmware/photon                             1.0                 9b411d78ad9e        2 months ago        130MB
```

- Docker-Compose

```bash
$ docker-compose ps
       Name                     Command               State                                Ports                              
------------------------------------------------------------------------------------------------------------------------------
harbor-adminserver   /harbor/start.sh                 Up                                                                      
harbor-db            /usr/local/bin/docker-entr ...   Up      3306/tcp                                                        
harbor-jobservice    /harbor/start.sh                 Up                                                                      
harbor-log           /bin/sh -c /usr/local/bin/ ...   Up      127.0.0.1:1514->10514/tcp                                       
harbor-ui            /harbor/start.sh                 Up                                                                      
nginx                nginx -g daemon off;             Up      0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp, 0.0.0.0:80->80/tcp
registry             /entrypoint.sh serve /etc/ ...   Up      5000/tcp                                                        
```


## 启动后

启动完成后，我们访问刚设置的hostname即可，`https://harbor-test.com`

如果端口占用，我们可以去修改docker-compose.yml文件中，对应服务的端口映射。

登录`https://harbor-test.com`

用户名和密码在配置文件中可以查看，这里我设置的是admin/Harbor123

![harbor-login](/images/harbor-login.png)


创建自己的项目

![harbor-create-project](/images/harbor-create-project.png)


创建完之后点击对应项目，里面会有教程如何推送/拉取镜像

![harbor-push](/images/harbor-push.png)




## Docker对应相关配置

以CentOS 7 为例，安装Harbor镜像仓库之后，使用`docker login`登录会出现certificate signed by unknown authority类似这样的错误，原因是Harbor如果是部署在内网的话没有HTTPS证书。

对于这种情况，一般有两个解决方法：

- 部署到外网，然后添加HTTPS证书，如Certbot

- Docker配置文件中加--insecure-registry参数


这里就说一下怎么添加--insecure-registry参数

找到Docker的配置文件`/usr/lib/systemd/system/docker.service`

```bash
$ vim /usr/lib/systemd/system/docker.service

## 找到对应行ExecStart= 并修改
ExecStart=/usr/bin/dockerd --insecure-registry harbor-test.com

## 先reload再重启docker服务
$ systemctl daemon-reload
$ systemctl restart docker
```

再尝试使用`docker login harbor-test.com`就可以登录成功并使用仓库进行推送/拉取镜像。



