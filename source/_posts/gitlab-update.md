---
title: Gitlab踩坑升级过程
date: 2018-03-01
categories: Gitlab升级 
tags: 
- Gitlab
- Update
- Migration

---
本帖子记录的是Gitlab跨大版本更新，数据迁移备份过程记录

## 背景简述

由于各个项目迭代次数较多，Gitlab的服务器硬件相对较低，而且也出现过遇到当前Gitlab版本Bug的情况，所以最近就打算一次性把Gitlab迁移到配置较好的企业服务器并升级到最新版本。

目前原来版本是gitlab-ce-8.8.0.ce.0，现在最新的版本是gitlab-ce-10.5.2.ce.0，这次过程就是从8.8.0升级到10.5.2。


## Gitlab部署

- 系统以CentOS7为例


下载RPM包地址：[Gitlab-CE清华大学镜像站](https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/)


## 先安装依赖

参考[官方说明](https://about.gitlab.com/downloads/#centos7)中的Install and configure the necessary dependencies一节。

``` bash
$ yum install curl policycoreutils openssh-server openssh-clients
$ systemctl enable sshd
$ systemctl start sshd
$ yum install postfix
$ systemctl enable postfix
$ systemctl start postfix
$ firewall-cmd --permanent --add-service=http
$ systemctl reload firewalld
```

## 下载与原来版本一致的RPM包

这里我先记录如何进行数据迁移，**数据迁移必须使用版本一致的gitlab**，跨版本的数据恢复可能会报错。


### 先下载gitlab-ce-8.8.0.ce.0
``` bash
## 下载
$ wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-8.8.0-ce.0.el7.x86_64.rpm

## 安装
$ rpm -i gitlab-ce-8.8.0-ce.0.el7.x86_64.rpm
```


## 修改配置文件(/etc/gitlab/gitlab.rb)

修改配置项：

- 域名：external_url 'http://gitlab.rd.exmaple.com'

- 时区：gitlab_rails['time_zone'] = 'Asia/Shanghai'

- 通知发送的邮箱: gitlab_rails['gitlab_email_from'] = 'gitlab-notifications@example.com'

- 修改SMTP服务器配置：gitlab_rails['smtp*']

- 仓库路径：git_data_dir "/home/gitlab/git-data"

 - 注意，不支持符号链接


参见：[配置文件](https://github.com/Polaris0112/Ops-Tools/blob/master/gitlab/gitlab.rb)

**另外留意配置文件的权限问题**

正确的是：

``` bash
# 正确的是
# -rw-------. 1 root root 20227 Feb 28 16:59 gitlab.rb

# 修改权限命令
$ chmod 600 /etc/gitlab/gitlab.rb
```


## 编译部署

``` bash
# 编译
$ gitlab-ctl reconfigure

# 重启服务
$ gitlab-ctl restart
```


## 备份

### 配置文件中相关的选项

- 备份目录：`gitlab_rails['backup_path'] = "/home/gitlab/backups"`

- 备份文件保存时长: `gitlab_rails['backup_keep_time'] = 604800`


### 手动命令操作备份

``` bash
$ gitlab-rake gitlab:backup:create
```


## 恢复

**大致来说：**

- 需保证gitlab版本和备份时一致

- 需要启动服务，但停掉前台和队列，以防止用户操作干扰

- 从备份机或者原本服务器上提取备份压缩包，传输至新服务器

- 把备份文件放到/var/opt/gitlab/backups/路径下

- 跑恢复脚本gitlab-rake gitlab:backup:restore，指定时间戳参数BACKUP

- 把服务都启动

- 检查确认


### 操作步骤

详细说明可参考官方文档中的[Backup restore - Restore a previously created backup](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/raketasks/backup_restore.md#restore-a-previously-created-backup)一节。下面是原文，源代码部署部分省略掉了：

停前台服务：

``` bash
$ gitlab-ctl stop unicorn
$ gitlab-ctl stop sidekiq
$ gitlab-ctl stop nginx
```

恢复对应时间戳备份的数据(**注意：以下命令的压缩文件、服务器IP、时间戳均需要替换指定的环境中具体的值**)：

``` bash
# 从旧服务器找到备份文件并scp到新服务器
$ scp 1519857115_gitlab_backup.tar 192.168.0.85:./

# 到新服务器把压缩文件移动到指定路径下并修改所属用户
$ mv 1519857115_gitlab_backup.tar /var/opt/gitlab/backups/
$ chown -R git:git /var/opt/gitlab/backups

# 恢复备份
$ gitlab-rake gitlab:backup:restore BACKUP=1519857115

## 恢复过程中需要交互两次，均输入yes继续即可。
```

启动服务、检查：

``` bash
$ gitlab-ctl restart
$ gitlab-rake gitlab:check SANITIZE=true
```


## 远程备份

远程备份

- 开源的auto-gitlab-backup项目可以完成一般备份和/etc/gitlab/*备份，以及同步到远端的工作。

- 用crontab触发每日定时执行备份

- 参见: [配置文件](https://github.com/Polaris0112/Ops-Tools/blob/master/gitlab/auto-gitlab-backup.conf)



## 更新Gitlab

### 先关闭前端服务，防止用户人为操作

``` bash
$ gitlab-ctl stop unicorn
$ gitlab-ctl stop sidekiq
$ gitlab-ctl stop nginx
```

执行完毕后，原本gitlab网页应该就不能访问，用户也不能拉取或者提交代码。

### 下载新版Gitlab并更新

这里要注意的是，跨版本更新不能跨两个大版本，即8.8.0不能直接升级到10.5.2，升级后会因为PostgreSQL数据库会启动失败导致页面出现502，部分服务不能启动。

所以我们要一级一级升，目前确认的是可以先从8.8.0升级到9.5.9然后再升级到10.5.2
``` bash
# 如果怕担心升级失败的话，可以先升级一个小版本进行测试
# $ wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-8.8.1-ce.0.el7.x86_64.rpm
# $ rpm -Uvh gitlab-ce-8.8.1-ce.0.el7.x86_64.rpm
# 等待一段时间
# 执行完成后，重新配置gitlab
# $ gitlab-ctl reconfigure
# 完成后重启gitlab
# $ gitlab-ctl restart
# 启动完需要等服务启动一分钟左右后，才登陆网页查看

## 直接升级到9.5.9
$ wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-9.5.9-ce.0.el7.x86_64.rpm
$ rpm- Uvh gitlab-ce-9.5.9-ce.0.el7.x86_64.rpm
# 等待一段时间
# 执行完成后，重新配置gitlab
$ gitlab-ctl reconfigure
# 完成后重启gitlab
$ gitlab-ctl restart
## 然后查看页面有没有更新成功


## 若成功，则重复第一步
$ gitlab-ctl stop unicorn
$ gitlab-ctl stop sidekiq
$ gitlab-ctl stop nginx

$ wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-10.5.2-ce.0.el7.x86_64.rpm
$ rpm -Uvh gitlab-ce-10.5.2-ce.0.el7.x86_64.rpm
$ gitlab-ctl reconfigure
$ gitlab-ctl restart

```

所有执行完成后，等待服务启动一分钟以后就可以开启网页查看gitlab是否更新成功。




## 踩过的坑

### 编译GitLab出现ruby_block[supervise_redis_sleep] action run并卡住

解决方案：

- 按住CTRL+C强制结束

- 运行`systemctl restart gitlab-runsvdir`

- 再次执行`gitlab-ctl reconfigure`


[解决方案来源](https://gitlab.com/gitlab-org/omnibus-gitlab/issues/160)




## 完整卸载gitlab

- 停止gitlab

``` bash
$ gitlab-ctl stop
```

- 卸载gitlab

``` bash
$ rpm -e gitlab-ce
```

- 查看gitlab进程

kill掉所有的gitlab进程

确认所有gitlab进程已经被删除

- 删除gitlab文件

``` bash
$ find / -name gitlab|xargs rm -rf    # 删除所有包含gitlab的文件及目录

```






