---
title: Docker安装部署教程
date: 2018-03-11
categories: Docker安装部署教程
tags: 
- Docker
- CentOS

---
本帖子记录的是CentOS 7下安装部署Docker教程

## 环境准备

- 系统：CentOS 7 最小安装版


## 安装依赖

```bash
$ yum update
$ yum install -y yum-utils device-mapper-persistent-data lvm2
```


## 安装Docker-CE

```bash
## 官方方法添加yum源
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

## 国内访问以上官方会很慢，所以推荐使用阿里云源
$ vim /etc/yum.repo.d/docker.repo
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://download.docker.com/linux/centos/7/$basearch/stable
enabled=1
gpgcheck=0
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-stable-debuginfo]
name=Docker CE Stable - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/stable
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-stable-source]
name=Docker CE Stable - Sources
baseurl=https://download.docker.com/linux/centos/7/source/stable
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-edge]
name=Docker CE Edge - $basearch
baseurl=https://download.docker.com/linux/centos/7/$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-edge-debuginfo]
name=Docker CE Edge - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-edge-source]
name=Docker CE Edge - Sources
baseurl=https://download.docker.com/linux/centos/7/source/edge
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test]
name=Docker CE Test - $basearch
baseurl=https://download.docker.com/linux/centos/7/$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test-debuginfo]
name=Docker CE Test - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test-source]
name=Docker CE Test - Sources
baseurl=https://download.docker.com/linux/centos/7/source/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg


## 再安装docker-ce
$ yum install -y docker-ce
```

**可选：如果你想安装最新版本的Docker CE，你需要启用这些仓库（默认是关闭的）**

```bash
$ yum-config-manager --enable docker-ce-edge
$ yum-config-manager --enable docker-ce-test
```


## 启动服务

```bash
$ systemctl start docker

## 设置开机启动
$ systemctl enable docker
```


## 国内镜像市场

由于官方的镜像仓库在国外，所以很多时候拉了一半就会断开，每次也不会有断开重连机制，所以每一次的断开就是重新再下载的过程。

因此这里收集一些常用的国内镜像仓库地址：

- [DaoCloud](http://hub.daocloud.io/)

- [时速云](https://hub.tenxcloud.com/)

- [灵雀云](https://hub.alauda.cn/)

这几个国内比较常用的镜像市场。


## 镜像加速器

此外，除了使用国内镜像仓库的同时，也可配合镜像加速器来下载国外官方镜像。

比如阿里云、腾讯云、网易云也有做类似的功能，以下用阿里云为例子：

关于加速器的地址，你只需要登录容器[Hub服务](https://cr.console.aliyun.com/?spm=a2c4e.11153959.blogcont29941.9.520269d6odTEcW)的控制台，左侧的加速器帮助页面就会显示为你独立分配的加速地址。

```bash
#例如：
#公网Mirror：[系统分配前缀].mirror.aliyuncs.com
```

### 当你的docker版本较新时

当你下载安装的Docker Version不低于1.10时，建议直接通过daemon config进行配置。
使用配置文件 /etc/docker/daemon.json（没有时新建该文件）

```shell
{
    "registry-mirrors": ["<your accelerate address>"]
}
```

**以CentOS 7为例**

CentOS的配置方式略微复杂，需要先将默认的配置文件复制出来
/lib/systemd/system/docker.service -> /etc/systemd/system/docker.service
然后再将加速器地址添加到配置文件的启动命令
重启Docker就可以了。

```bash
$ cp -n /lib/systemd/system/docker.service /etc/systemd/system/docker.service
$ sed -i "s|ExecStart=/usr/bin/docker daemon|ExecStart=/usr/bin/docker daemon --registry-mirror=<your accelerate address>|g" /etc/systemd/system/docker.service
$ sed -i "s|ExecStart=/usr/bin/dockerd|ExecStart=/usr/bin/dockerd --registry-mirror=<your accelerate address>|g" /etc/systemd/system/docker.service
$ systemctl daemon-reload
$ service docker restart
```


### 阿里云加速器

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://sdu7uwdm.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```


### DaoCloud加速器

```bash
$ curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://942c18cb.m.daocloud.io
```


## 一键安装脚本（CentOS 7）

[下载地址](https://raw.githubusercontent.com/Polaris0112/Ops-Tools/master/docker/docker-install.sh)
