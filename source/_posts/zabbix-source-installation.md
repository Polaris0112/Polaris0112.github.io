---
title: Zabbix 编译安装部署
date: 2018-02-25
categories: Zabbix 编译安装
tags: 
- Zabbix
- Source
- Installation

---
本帖子记录的是使用源码编译方式安装Zabbix3.4.x

## Zabbix简介

Zabbix 是由Alexei Vladishev创建，目前由Zabbix SIA在持续开发和支持。

Zabbix 是一个企业级的分布式开源监控方案。

Zabbix是一款能够监控各种网络参数以及服务器健康性和完整性的软件。Zabbix使用灵活的通知机制，允许用户为几乎任何事件配置基于邮件的告警。这样可以快速反馈服务器的问题。基于已存储的数据，Zabbix提供了出色的报告和数据可视化功能。这些功能使得Zabbix成为容量规划的理想方案。

Zabbix支持主动轮询和被动捕获。Zabbix所有的报告、统计信息和配置参数都可以通过基于Web的前端页面进行访问。基于Web的前端页面可以确保您从任何方面评估您的网络状态和服务器的健康性。适当的配置后，Zabbix可以在IT基础架构监控方面扮演重要的角色。对于只有少量服务器的小型组织和拥有大量服务器的大型公司也同样如此。

Zabbix是免费的。Zabbix是根据GPL通用公共许可证第2版编写和发行的。这意味着它的源代码都是免费发行的，可供公众任意使用。

想了解更多Zabbix相关信息，可以参考[Zabbix官方文档](https://www.zabbix.com/documentation/3.4/zh/start)


## 安装环境


- CentOS 7/RHEL 7

整套安装流程是在CentOS7上的zabbix-server和zabbix-agent(版本为3.4.6)，使用前端网页服务器是nginx


### 添加zabbix用户
``` bash
$ useradd -r -s /sbin/nologin zabbix
```


### 安装Percona Mysql数据库(比原生Mysql性能更优)
``` bash
## 添加到yum repo
$ vi /etc/yum.repos.d/percona-release.repo
########################################
# Percona releases and sources, stable #
########################################
[percona-release-$basearch]
name = Percona-Release YUM repository - $basearch
baseurl = http://repo.percona.com/release/$releasever/RPMS/$basearch
enabled = 1
gpgcheck = 1
gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-percona
[percona-release-noarch]
name = Percona-Release YUM repository - noarch
baseurl = http://repo.percona.com/release/$releasever/RPMS/noarch
enabled = 1
gpgcheck = 1
gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-percona
[percona-release-source]
name = Percona-Release YUM repository - Source packages
baseurl = http://repo.percona.com/release/$releasever/SRPMS
enabled = 0
gpgcheck = 1
gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-percona


## 下载percona key并安装
$ wget http://www.percona.com/downloads/RPM-GPG-KEY-percona -O /etc/pki/rpm-gpg/RPM-GPG-KEY-percona
$ yum install -y Percona-Server-devel-57 Percona-Server-shared-57 Percona-Server-client-57 Percona-Server-server-57 percona-zabbix-templates Percona-Server-shared


## 开启mysql服务并设置开机启动
$ systemctl start mysql      ## 注意启动服务的时候会进行一连串初始化日志输出，其中会包括一个随机生成的密码作为初次启动设置的root密码(留意有一行有"A temporary password"相关字眼)，下一步需要用到
$ systemctl enable mysql


## 使用上述获得的密码对mysql的root密码进行修改
$ mysql -u root -p"<temporary password>" -e 'SET PASSWORD = PASSWORD("'<your root password>'");ALTER USER "root"@localhost PASSWORD EXPIRE NEVER;flush privileges;' --connect-expired-password
## 以上需要修改的是`<temporary password>`，`<your root password>`
## 若命令报错，有可能是您设置的root密码安全性不足，需要8位其中包括至少一个大写字母、小写字母数字和特殊符号。


## 以下是目前我正在使用的mysql配置文件，仅供参考，不一定符合每个场景使用
$ vi /etc/my.cnf
# *** default location during install, and will be replaced if you
# *** upgrade to a newer version of MySQL.
[mysqld]
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin
# These are commonly set, remove the # and set as required.
# basedir = .....
# datadir = .....
# port = .....
# server_id = .....
# socket = .....
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M 
sync_binlog=0
show_compatibility_56=ON
character-set-server = utf8
# network
connect_timeout = 60
wait_timeout = 2880
max_connections = 2048
max_allowed_packet = 32M
max_connect_errors = 1000
# limits
tmp_table_size = 512M
max_heap_table_size = 256M
# logs
log_error = /var/log/mysql/mysql-error.log
slow_query_log_file = /var/1og/mysql/mysql-slow.log
slow_query_log = 1
long_query_time = 20
log_error_verbosity=2
# innodb
#innodb_data_home_dir = /var/lib/mysql/data
innodb_file_per_table = 1
innodb_status_file = 1
innodb_buffer_pool_size = 2G
innodb_flush_method = O_DIRECT
innodb_io_capacity = 2000
innodb_flush_log_at_trx_commit = 2
innodb_support_xa = 0
innodb_log_file_size = 512M
# other stuff
event_scheduler = 1
query_cache_type = 0
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid


## 若修改过配置文件，记得要重启mysql服务
$ systemctl restart mysql
```


## 安装nginx
``` bash
$ yum install epel-release
$ yum install nginx
$ vi /etc/nginx/conf.d/zabbix.conf
server {
    listen      80;
    #server_name zabbix.test;
    charset     utf-8;

    #root        /var/www/zabbix;
    root        /usr/share/nginx/html/zabbix;
    index       index.php;

    location ~* /\.ht {
        deny  all;
    }

    location ~* /(api|conf|include)/ {
        rewrite ^/(.*)$ http://localhost/zabbix/index.php permanent;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param   SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param   QUERY_STRING    $query_string;
        fastcgi_pass    127.0.0.1:9000;
        fastcgi_intercept_errors on;
        error_page      403 404 502 503 504 http://localhost/zabbix/index.php;
    }

    location ~* \.(css|gif|jpeg|jpg|js|txt|png|tif|tiff|ico|jng|bmp|doc|pdf|rtf|xls|xpi|zip|tgz|gz|bz2|tar|mid|midi|mp3)$ {
        root /usr/share/nginx/html/zabbix/;
    }
}
```



### 安装PHP5.6提供前端支持
``` bash
## 安装rpm
$ rpm -ivh http://mirror.webtatic.com/yum/el7/webtatic-release.rpm


## yum安装php5.6
$ yum install -y php56w-mcrypt php56w-mbstring php56w-common php56w-xmlrpc php56w-mysql php56w-pdo php56w-gd php56w-ldap php56w-xml php56w-cli php56w-bcmath php56w php56w-fpm
$ service php-fpm start
```



### 下载Zabbix-3.4.6源码包并安装
``` bash
## 下载源码包
$ wget -c https://nchc.dl.sourceforge.net/project/zabbix/ZABBIX%20Latest%20Stable/3.4.6/zabbix-3.4.6.tar.gz
$ tar zxf zabbix-3.4.6.tar.gz
$ cd zabbix-3.4.6
$ ./configure --prefix=/usr/local/zabbix --enable-server --enable-agent --with-mysql --with-net-snmp --with-libcurl
$ make
$ make install 


## 初始化mysql的zabbix数据库(需要用到上述安装mysql的root密码)
$ mysql -uroot -p"<your root password>" -e 'CREATE USER "zabbix"@"localhost" IDENTIFIED BY "'<zabbix db password>'";'
$ mysql -uroot -p"<your root password>" -e 'create database zabbix character set utf8;'
$ mysql -uroot -p"<your root password>" -e 'grant all privileges on zabbix.* to zabbix@localhost identified by "'<zabbix db password>'";flush privileges;'

## 导入初始化的sql文件
$ mysql -uzabbix -p"<zabbix db password>" -e "use zabbix;source database/mysql/schema.sql;"
$ mysql -uzabbix -p"<zabbix db password>" -e "use zabbix;source database/mysql/images.sql;"
$ mysql -uzabbix -p"<zabbix db password>" -e "use zabbix;source database/mysql/data.sql;"
```


### 配置zabbix_server.conf
``` bash
sed -i 's/^DBName=.*$/DBName=zabbix/g' /usr/local/zabbix/etc/zabbix_server.conf
sed -i 's/^DBUser=.*$/DBUser=zabbix/g' /usr/local/zabbix/etc/zabbix_server.conf
sed -i 's/^DBPassword=.*$/DBPassword='<zabbix db password>'/g' /usr/local/zabbix/etc/zabbix_server.conf
sed -i 's/^ServerActive=.*$/ServerActive=127.0.0.1/g' /usr/local/zabbix/etc/zabbix_agentd.conf

## 复制数据库监控模板
cp /var/lib/zabbix/percona/templates/userparameter_percona_mysql.conf /usr/local/zabbix/etc/zabbix_agentd.conf.d/

## 复制启动脚本并修改为目前路径
cp misc/init.d/fedora/core/zabbix_* /etc/init.d/
sed -i 's#BASEDIR=/usr/local#BASEDIR=/usr/local/zabbix#g' /etc/init.d/zabbix_server
sed -i 's#BASEDIR=/usr/local#BASEDIR=/usr/local/zabbix#g' /etc/init.d/zabbix_agentd

```


### 配置服务文件
``` bash
$ vi /etc/services     # 在文件最后添加
zabbix-agent 10050/tcp Zabbix Agent
zabbix-agent 10050/udp Zabbix Agent
zabbix-trapper 10051/tcp Zabbix Trapper
zabbix-trapper 10051/udp Zabbix Trapper

```


### 配置php.ini
``` bash
sed -i 's/^.*post_max_size.*$/post_max_size = 16M/g' /etc/php.ini
sed -i 's/^.*max_execution_time.*$/max_execution_time = 300/g' /etc/php.ini
sed -i 's/^.*max_input_time.*$/max_input_time = 300/g' /etc/php.ini
sed -i 's/^.*always_populate_raw_post_data.*$/always_populate_raw_post_data = -1/g' /etc/php.ini
sed -i 's/^.*date.timezone.*$/date.timezone = Asia\/Shanghai/g' /etc/php.ini
sed -i 's/mbstring.func_overload.*$/mbstring.func_overload = 2/g' /etc/php.ini
```


### 复制Zabbix前端文件
``` bash
mkdir -p /usr/share/nginx/html/zabbix
cp -a frontends/php/* /usr/share/nginx/html/zabbix/
cp /var/lib/zabbix/percona/templates/userparameter_percona_mysql.conf /usr/local/zabbix/etc/zabbix_agentd.conf.d/
chmod 777 -R  /var/lib/php/session/
```


### 关闭系统相关安全设置并开机自启动相关
``` bash
$ systemctl stop firewalld
$ setenforce 0
$ sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux 
$ echo "/etc/init.d/zabbix_server start" >> /etc/rc.local
$ echo "/etc/init.d/zabbix_agentd start" >> /etc/rc.local
$ systemctl enable mysql
$ systemctl enable nginx
```


### 重启服务
``` bash
$ systemctl restart nginx zabbix_server zabbix_agentd
```


### Zabbix Web页面

请参考[官方手册](https://www.zabbix.com/documentation/3.4/manual/installation/install#installing_frontend)






