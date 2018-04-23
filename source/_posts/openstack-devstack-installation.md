---
title: 通过devstack安装部署OpenStack
date: 2018-03-10
categories: 通过devstack安装部署OpenStack
tags: 
- OpenStack
- DevStack

---
本帖子记录的是通过devstack安装部署OpenStack

## 环境准备

- 系统：CentOS 7 最小安装版

```bash
$ vi /etc/selinux/config  
#SELINUX=enforcing       //注释掉  
SELINUX=disabled         //增加  

$ cd /home  
$ git clone http://git.trystack.cn/openstack-dev/devstack.git -b stable/ocata  
```
注意一定要使用 -b stable/ocata 指定devstack版本（与要安装的openstack版本一致），否则基本上不会安装成功！



## 创建stack用户

```bash
$ cd /home/devstack/tools  
$ ./create-stack-user.sh  
$ chown -R stack:stack /home/devstack  
$ chmod 777 /opt/stack -R  
$ chmod 777 /var/run/openvswitch/db.sock  
$ su stack  
```



## 创建local.conf文件

```bash
[[local|localrc]]  
GIT_BASE=http://git.trystack.cn  
NOVNC_REPO=http://git.trystack.cn/kanaka/noVNC.git  
SPICE_REPO=http://git.trystack.cn/git/spice/spice-html5.git  
RECLONE=True  
DOWNLOAD_DEFAULT_IMAGES=False  
IMAGE_URLS=http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img  
ADMIN_PASSWORD=openstack  
DATABASE_PASSWORD=$ADMIN_PASSWORD  
RABBIT_PASSWORD=$ADMIN_PASSWORD  
SERVICE_PASSWORD=$ADMIN_PASSWORD  
HOST_IP=192.168.0.120  
LOGFILE=/opt/stack/logs/stack.sh.log  
LOGDAYS=2  
LOG_COLOR=True  
HORIZON_BRANCH=stable/ocata  
KEYSTONE_BRANCH=stable/ocata  
NOVA_BRANCH=stable/ocata  
NEUTRON_BRANCH=stable/ocata  
GLANCE_BRANCH=stable/ocata  
CINDER_BRANCH=stable/ocata  
KEYSTONE_TOKEN_FORMAT=UUID  
# Enabling Neutron (network) Service  
disable_service n-net  
enable_service q-svc  
enable_service q-agt  
enable_service q-dhcp  
enable_service q-l3  
enable_service q-meta  
enable_service q-metering  
enable_service neutron  
# VLAN configuration  
Q_PLUGIN=ml2  
ENABLE_TENANT_VLANS=True  
TENANT_VLAN_RANGE=1100:2999  
SWIFT_HASH=66a3d6b56c1f479c8b4e70ab5c2000f5  
SWIFT_REPLICAS=1  
SWIFT_DATA_DIR=$DEST/data  
```


## stackrc文件配置修改
```bash
# 只需修改一处，改成如下即可，使用https而不是用git，因为git被阻塞了，不能访问下载编译中需要的文件  
# Base GIT Repo URL  
# Another option is https://git.openstack.org  
#GIT_BASE=${GIT_BASE:-git://git.openstack.org}  
GIT_BASE=${GIT_BASE:-https://git.openstack.org}  
```



## 开始安装
```bash
[stack@localhost devstack]$ ./stack.sh  
（注意：除首次安装外，每次执行./stack.sh前都要执行./unstack.sh,否则很容易报错，安装失败）  
```


## 遇到错误
```bash
2018-03-04 06:40:36.101 | Collecting libvirt-python===2.5.0 (from -c /opt/stack/requirements/upper-constraints.txt (line 188))  
2018-03-04 06:40:42.129 |   Downloading http://pypi.doubanio.com/packages/42/e2/40956b304fc69ba4947929398bdaf116c5e4f8a4db1dd57060a45ca84ff1/libvirt-python-2.5.0.tar.gz (172kB)  
2018-03-04 06:40:43.172 | Building wheels for collected packages: libvirt-python  
2018-03-04 06:40:43.175 |   Running setup.py bdist_wheel for libvirt-python: started  
2018-03-04 06:40:43.841 |   Running setup.py bdist_wheel for libvirt-python: finished with status 'error'  
2018-03-04 06:40:43.841 |   Complete output from command /usr/bin/python2 -u -c "import setuptools, tokenize;__file__='/tmp/pip-build-KkW7rf/libvirt-python/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" bdist_wheel -d /tmp/tmpoNOPZYpip-wheel- --python-tag cp27:  
2018-03-04 06:40:43.841 |   running bdist_wheel  
2018-03-04 06:40:43.841 |   running build  
```


参考:

- [地址1](https://stackoverflow.com/questions/42098592/libvirt-python-install-on-mac-error)
- [地址2](http://blog.csdn.net/lyjshen/article/details/69467444)


解决:

```bash
$ pip install  https://libvirt.org/sources/python/libvirt-python-3.0.0.tar.gz  
$ vi /opt/stack/requirements/global-requirements.txt  
$ vi /opt/stack/requirements/upper-constraints.txt  
```

上面两个文件将libvirt-python修改为3.2.0
然后执行

```bash
$ ./unstack.sh  
$ ./stack.sh  
```


## 安装成功界面

```bash
This is your host IP address: 192.168.0.120
This is your host IPv6 address: ::1
Horizon is now available at http://192.168.0.120/dashboard
Keystone is serving at http://192.168.0.120/identity/
The default users are: admin and demo
The password: openstack
DevStack Version: ocata
Change: 66827dda40737f482b00580b52bc2865c605dcae Bump NOVA_READY_TIMEOUT to 120s 2018-02-23 16:58:30 -0500
OS Version: CentOS 7.4.1708 Core
```
![openstack-devstack](/images/openstack-devstack.png)


### 关闭防火墙

```bash
$ systemctl stop firewalld.service  
$ systemctl disable firewalld.service  
$ cat /etc/sysconfig/iptables  
# sample configuration for iptables service  
# you can edit this manually or use system-config-firewall  
# please do not ask us to add additional ports/services to this default configuration  
*filter  
:INPUT ACCEPT [0:0]  
:FORWARD ACCEPT [0:0]  
:OUTPUT ACCEPT [0:0]  
#-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT  
#-A INPUT -p icmp -j ACCEPT  
#-A INPUT -i lo -j ACCEPT  
#-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT  
#-A INPUT -j REJECT --reject-with icmp-host-prohibited  
#-A FORWARD -j REJECT --reject-with icmp-host-prohibited  
COMMIT  
$ systemctl restart iptables.service  
$ systemctl enable iptables.service  
```

http://192.168.0.120/dashboard登录，用户名为admin，密码为openstack

![openstack-devstack-1](/images/openstack-devstack-1.png)
