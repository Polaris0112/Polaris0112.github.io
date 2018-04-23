---
title: 通过packstack快速安装openstack
date: 2018-03-10
categories: 通过packstack快速安装openstack
tags: 
- OpenStack
- PackStack

---
本帖子记录的是通过packstack快速安装openstack

## 环境准备

- 系统：CentOS 7 最小安装版

- CPU：2+

- 内存：6GB+

- 硬盘：50GB+

- 网卡：2个



## 配置pike版本openstack源（此处采用阿里源）

```bash
$ cat <<END >/etc/yum.repos.d/pike.repo
[pike]
name=pike
baseurl=http://mirrors.aliyun.com/centos/7.4.1708/cloud/x86_64/openstack-pike/
enabled=1
gpgcheck=0
END
```



## 安装epel源

```bash
$ yum install epel-release -y
```


## 安装packstack安装工具和依赖

```bash
$ yum install -y centos-release-openstack-ocata  centos-release-openstack-mitaka  
$ yum update -y
$ yum -y install openstack-packstack
```


## 安装allinone的的环境

```bash
$ packstack --allinone
```

**注意：**

- 安装大概一个小时左右，由于网络问题可能会报错，报错后重复执行即可
```bash
$ packstack --answer-file packstack-answers-20171129-031548.txt
```

- 自定义安装
如果需要自定义参数，需要先生存应答文件
```bash
$ packstack --gen-answer-file=packstack-answers
```

通过应答文件执行安装
```bash
$ packstack --answer-file=packstack-answers
```

**注意：用户名和密码在执行命令的当前目录下的keystonerc_admin中**


```bash
$ source keystonerc_admin

## 查看用户
$ openstack user list 
+----------------------------------+------------+
| ID                               | Name       |
+----------------------------------+------------+
| 15b05802627f4f42b4ae3253f5ce6730 | cinder     |
| 2ac16759b27f480da9adff72188c79f3 | glance     |
| 2aced06e7c984399a7cc799de9199244 | aodh       |
| 3fdc02269da74e67be50c1d2f8711f3b | gnocchi    |
| 70a112fdec0842979e8e9e5e9f171791 | nova       |
| 866962fe0ee14c379dc6245a8ee5a707 | placement  |
| a1c35ba084554dd1bca310667a82bdd2 | ceilometer |
| bce952e6b65641aa93aab872d04e7c0c | admin      |
| c2082398028442a7bff898fb6745af57 | swift      |
| e43f38301fed4a44ae070b6e7b0a0026 | neutron    |
| fbbd1aff07fd4b48a0020bddd2c8deea | demo       |
+----------------------------------+------------+

## 查看项目
$ openstack project list 
+----------------------------------+----------+
| ID                               | Name     |
+----------------------------------+----------+
| 2bbfb410124f4454a45f012a6274eff7 | demo     |
| 400487fe18a541f1a3c4c462b4ff9fa6 | admin    |
| 8b051a62445f4106a17f5521cd713841 | services |
+----------------------------------+----------+

## 查看服务
$ openstack service list 
+----------------------------------+------------+--------------+
| ID                               | Name       | Type         |
+----------------------------------+------------+--------------+
| 128e1c41dae648fd84266be576ab6b8e | aodh       | alarming     |
| 1549860212bd444eb96ebe3399e1af05 | cinderv2   | volumev2     |
| 5144ecf07843499aa76a1c5bdeafcfa2 | nova       | compute      |
| 5670a37cdd7e48c4b77b744082dfd5cd | cinder     | volume       |
| 7963a27f90de40ed900bcff7f0771ab9 | swift      | object-store |
| 9aa244907a4e428b8a8c996e086a518f | placement  | placement    |
| a967ff55c55143439e899cbd0530af6b | neutron    | network      |
| c06cc37f779a4163b147fca99ec3cc33 | keystone   | identity     |
| c8a39eec1cac44719bbb15b883d62735 | glance     | image        |
| d8871026f6cc4c00b25bcfcf6ae05064 | ceilometer | metering     |
| f2b5cd4ee2344f389e6eb1e9a8fbccfc | gnocchi    | metric       |
| fccbbd2dbf3141acbe8a2291df1b67a1 | cinderv3   | volumev3     |
+----------------------------------+------------+--------------+

## 查看catalog
$ openstack catalog list 
+------------+--------------+----------------------------------------------------------------------------+
| Name       | Type         | Endpoints                                                                  |
+------------+--------------+----------------------------------------------------------------------------+
| aodh       | alarming     | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:8042                                       |
|            |              | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:8042                                         |
|            |              | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:8042                                          |
|            |              |                                                                            |
| cinderv2   | volumev2     | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:8776/v2/400487fe18a541f1a3c4c462b4ff9fa6   |
|            |              | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:8776/v2/400487fe18a541f1a3c4c462b4ff9fa6      |
|            |              | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:8776/v2/400487fe18a541f1a3c4c462b4ff9fa6     |
|            |              |                                                                            |
| nova       | compute      | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:8774/v2.1/400487fe18a541f1a3c4c462b4ff9fa6    |
|            |              | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:8774/v2.1/400487fe18a541f1a3c4c462b4ff9fa6 |
|            |              | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:8774/v2.1/400487fe18a541f1a3c4c462b4ff9fa6   |
|            |              |                                                                            |
| cinder     | volume       | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:8776/v1/400487fe18a541f1a3c4c462b4ff9fa6   |
|            |              | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:8776/v1/400487fe18a541f1a3c4c462b4ff9fa6     |
|            |              | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:8776/v1/400487fe18a541f1a3c4c462b4ff9fa6      |
|            |              |                                                                            |
| swift      | object-store | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:8080/v1/AUTH_400487fe18a541f1a3c4c462b4ff9fa6|
|            |              | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:8080/v1/AUTH_400487fe18a541f1a3c4c462b4ff9fa6 |
|            |              | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:8080/v1/AUTH_400487fe18a541f1a3c462b4ff9fa6|
|            |              |                                                                            |
| placement  | placement    | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:8778/placement                               |
|            |              | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:8778/placement                                |
|            |              | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:8778/placement                             |
|            |              |                                                                            |
| neutron    | network      | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:9696                                         |
|            |              | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:9696                                       |
|            |              | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:9696                                          |
|            |              |                                                                            |
| keystone   | identity     | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:35357/v3                                      |
|            |              | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:5000/v3                                      |
|            |              | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:5000/v3                                    |
|            |              |                                                                            |
| glance     | image        | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:9292                                         |
|            |              | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:9292                                       |
|            |              | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:9292                                          |
|            |              |                                                                            |
| ceilometer | metering     | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:8777                                          |
|            |              | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:8777                                       |
|            |              | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:8777                                         |
|            |              |                                                                            |
| gnocchi    | metric       | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:8041                                         |
|            |              | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:8041                                       |
|            |              | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:8041                                          |
|            |              |                                                                            |
| cinderv3   | volumev3     | RegionOne                                                                  |
|            |              |   public: http://172.16.8.100:8776/v3/400487fe18a541f1a3c4c462b4ff9fa6     |
|            |              | RegionOne                                                                  |
|            |              |   internal: http://172.16.8.100:8776/v3/400487fe18a541f1a3c4c462b4ff9fa6   |
|            |              | RegionOne                                                                  |
|            |              |   admin: http://172.16.8.100:8776/v3/400487fe18a541f1a3c4c462b4ff9fa6      |
|            |              |                                                                            |
+------------+--------------+----------------------------------------------------------------------------+ 
```








