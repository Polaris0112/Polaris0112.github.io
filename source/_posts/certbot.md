---
title: Certbot安装/部分问题解决方案
date: 2018-02-22
categories: Certbot安装/部分问题解决方案
tags: 
- Certbot
- https

---

本帖子记录的是Nginx + Certbot的安装过程以及在阿里云CentOS7上部署Certbot遇到的问题以及解决方法


## Using CentOS 7/RHEL 7

### 安装

Certbot安装包已经包含在EPEL(Extra Packages for Enterprise Linux)源中，所以如果安装Certbot需要先安装[EPEL源](https://fedoraproject.org/wiki/EPEL#How_can_I_use_these_extra_packages.3F)。

```bash
## 在CentOS/RHEL 7中使用

$ yum -y install yum-utils
$ yum-config-manager --enable rhui-REGION-rhel-server-extras rhui-REGION-rhel-server-optional
$ yum install certbot-nginx
```


### 使用

由于Certbot已经有nginx插件，所以安装证书只需要运行以下命令

```bash
$ sudo certbot --nginx
```

运行后按照提示输入指定的参数值后就会自动配置nginx并启用，若需要自定义nginx配置文件则可以先使用以下命令

```bash
$ sudo certbot --nginx certonly
```

更多请参考[文档](https://certbot.eff.org/docs)


### 更新证书

根据官方文档上的说明，Certbot的证书有效期为90天，有效期少于30天才能更新一次，大于30天不用更新(不能更新)

测试更新命令如下

```bash
$ sudo certbot renew --dry-run
```

若上述命令测试通过(无报错)，则可以把以下命令写入到[cron job](http://www.unixgeeks.org/security/newbie/unix/cron-1.html)或者[systemd timer](https://wiki.archlinux.org/index.php/Systemd/Timers)中，就可以实现长期自动更新(官方推荐是一天执行两次确保证书能及时更新)

```bash
$ certbot renew
```

官方crontab例子参考(命令会在正午和午夜进行)

``` bash
0 0,12 * * * python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew
```


## 其他安装方法

### 使用certbot-auto脚本安装Certbot

certbot-auto脚本会安装Certbot，并且能够自己解决RPM包和Python包依赖问题，同样非常方便。同时certbot-auto是对certbot的封装，即certbot-auto提供certbot的所有功能。在使用此方法安装Certbot后，运行certbot-auto命令获取证书。

``` bash
## 1) 获取certbot-auto脚本
$ wget https://dl.eff.org/certbot-auto

## 2) 使certbot-auto脚本可执行
$ sudo chmod a+x ./certbot-auto

## 3) 运行certbot-auto，安装Certbot
$ ./certbot-auto

## 4) 获取Lets Encrypt证书，certbot-auto是对脚本certbot的封装，并且已经更新到了最新版本，所以使用此脚本获取证书
$ ./certbot-auto
```

### 使用pip安装Certbot

此种方法比较简单，最好先创建一个Python虚拟环境，然后再安装Certbot

```bash
1) 安装虚拟环境软件包(针对于Python2.7)
$ sudo yum install python-virtualenv

2) 创建虚拟环境
$ sudo virtualenv /usr/local/python-certbot

3) 激活虚拟环境
$ source /usr/local/python-certbot/bin/activate

4) 首先更新pip
$ pip install --upgrade pip

5) 安装Certbot
$ pip install certbot

6) 安装Certbot Apache插件
$ pip install certbot-apache

或者安装Certbot Nginx插件：
$ pip install certbot-nginx

7) 在安装Certbot Apache插件时，如提示如下错误：
OSError: ctypes.util.find_library() did not manage to locate a library called 'augeas'

那是因为缺少augeas库文件，安装augeas库文件：
$ sudo yum install augeas-libs

然后就能成功安装Certbot Apache插件。

8) 获取Lets Encrypt证书，同时配置Apache服务器
$ sudo certbot --apache

或者，同时配置Nginx服务器
$ sudo certbot --nginx
```



## 在阿里云CentOS7上部署Certbot遇到的坑以及解决方法

**报错1：**ImportError: No module named ‘requests.packages.urllib3’ (感觉是阿里云的centos7镜像本身python包的问题，因为直接import requests都会报错)

解决方法：更改/usr/lib/python2.7/site-packages/requests/exceptions.py模块导入部分

```bash
#把/usr/lib/python2.7/site-packages/requests/exceptions.py 传入包的部分
from .packages.urllib3.exceptions import HTTPError as BaseHTTPError
#改为
from urllib3.exceptions import HTTPError as BaseHTTPError
```


**报错2：**ImportError: cannot import name UnrewindableBodyError

解决方法： 重新安装对应版本的urllib3

```bash
#报错
pkg_resources.DistributionNotFound: The 'urllib3<1.22,>=1.21.1' distribution was not found and is required by requests

#解决方法：
$ pip install urllib3==1.21.1
```


**报错3：**pkg_resources.ContextualVersionConflict: (idna 2.6 (/usr/lib/python2.7/site-packages), Requirement.parse(‘idna<2.6,>=2.5’), set([‘requests’]))

解决方法：先检查pip freeze|grep pyOpenSSL版本(需要2.5~2.6之间)，理论上都要根据报错的版本进行安装，不符合则安装

```bash
$ pip install idna==2.5
```


**报错4：**ImportError: ‘pyOpenSSL’ module missing required functionality. Try upgrading to v0.14 or newer.

解决方法：这个是yum下的openssl版本过低

```bash
$ wget ftp://mirror.switch.ch/pool/4/mirror/centos/7.4.1708/cloud/x86_64/openstack-ocata/common/pyOpenSSL-0.15.1-1.el7.noarch.rpm
$ rpm -Uvh pyOpenSSL-0.15.1-1.el7.noarch.rpm
#rpm -qa|grep pyOpenSSL
#pyOpenSSL-0.15.1-1.el7.noarch
```


**报错5：**AttributeError: ‘module’ object has no attribute ‘SSL_ST_INIT’

解决方法：

```bash
$ pip install pyOpenSSL==16.2.0
```



## 迁移https证书过程备注：

- 复制/etc/letsencrypt/archive对应两个域名的文件夹到新机器上

- 复制/etc/letsencrypt/live对应两个域名的文件夹到新机器上

- 在新机器上的/etc/letsencrypt/renewal/目录下拷贝测试conf作为旧域名的新配置文件，[renewalparams]以下的参数不变，修改其余路径相关对应域名即可

- 添加nginx/conf.d/对应下的配置文件，注意配置ssl_certificate和ssl_certificate_key


## nginx配置

以下是Certbot自动生成的nginx配置文件，把http访问的域名都会转到https


```bash
server {
    listen       80;
    server_name  test-certbot.xxx.com;


    if ($scheme != "https") {
        return 301 https://$host$request_uri;
    } # managed by Certbot

}
```




