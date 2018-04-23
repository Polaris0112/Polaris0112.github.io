---
title: supervisor 安装、配置、常用命令
date: 2018-03-08
categories: supervisor 安装、配置、常用命令
tags: 
- supervisor

---
本帖子记录的是supervisor 安装、配置、常用命令

## 简述

Supervisor是用Python写的一款应用程监控管理工具，能够启动，停止，重启死进程，提供web管理界面，XML-RPC接口及事件监听。通常我们写了一些脚本都不会带有daemon功能，而是加&或者nohub，screen什么的丢到后台去运行，同时使用corntab定时检测脚本是否存活，以便重新运行脚本。使用Supervisor可以将这些脚本，程序转为守护进程，自动重启它们；还可以监控机器的进程运行状况，输出警报等。

Supervisor只能运行于Python 2.x的环境，但子进程可以为其他任意程序，比如Python 3，PHP等。这里使用pip来安装


## 安装

- CentOS系统下：`$ yum install supervisor`，通过这种方式安装后，自动设置为开机启动

- 也可以通过`pip install supervisor`进行安装，但是需要手动启动，然后手动设置为开机启动


## Supervisor 配置

Supervisor 是一个 C/S 模型的程序，`supervisord`是`server`端，`supervisorctl`是`client`端。


### supervisord

下面介绍 supervisord 配置方法。supervisord 的配置文件默认位于`/etc/supervisord.conf`，内容如下（;后面为注释）：

```bash
; supervisor config file

[unix_http_server]
file=/var/run/supervisor.sock   ; (the path to the socket file) UNIX socket 文件，supervisorctl 会使用
chmod=0700                       ; sockef file mode (default 0700) socket 文件的 mode，默认是 0700

[supervisord]
logfile=/var/log/supervisor/supervisord.log ; (main log file;default $CWD/supervisord.log) 日志文件，默认是 $CWD/supervisord.log
pidfile=/var/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid) pid 文件
childlogdir=/var/log/supervisor            ; ('AUTO' child log dir, default $TEMP)

; the below section must remain in the config file for RPC
; (supervisorctl/web interface) to work, additional interfaces may be
; added by defining them in separate rpcinterface: sections
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket 通过 UNIX socket 连接 supervisord，路径与 unix_http_server 部分的 file 一致

; 在增添需要管理的进程的配置文件时，推荐写到 `/etc/supervisor/conf.d/` 目录下，所以 `include` 项，就需要像如下配置。
; 包含其他的配置文件
[include]
files = /etc/supervisor/conf.d/*.conf ; 引入 `/etc/supervisor/conf.d/` 下的 `.conf` 文件
```


### program 配置

program 的配置文件就写在，`supervisord` 配置中`include`项的路径下：`/etc/supervisor/conf.d/`，然后`program`的配置文件命名规则推荐：`app_name.conf`

```bash
[program:app] ; 程序名称，在 supervisorctl 中通过这个值来对程序进行一系列的操作
autorestart=True      ; 程序异常退出后自动重启
autostart=True        ; 在 supervisord 启动的时候也自动启动
redirect_stderr=True  ; 把 stderr 重定向到 stdout，默认 false
environment=PATH="/home/app_env/bin"  ; 可以通过 environment 来添加需要的环境变量，一种常见的用法是使用指定的 virtualenv 环境
command=python server.py  ; 启动命令，与手动在命令行启动的命令是一样的
user=ubuntu           ; 用哪个用户启动
directory=/home/app/  ; 程序的启动目录
```

**需要注意：**

- 用 supervisord 管理时，gunicorn 的 daemon 选项需要设置为 False

- 如果启动命令需要包含`workon`，修改environment参数：
```shell
environment=PATH="/home/username/.virtualenvs/myproject/bin"
```
   

## supervisorctl 操作

supervisorctl 是 supervisord 的命令行客户端工具，使用的配置和 supervisord 一样，这里就不再说了。下面，主要介绍 supervisorctl 操作的常用命令：

输入命令 `supervisorctl` 进入 supervisorctl 的 shell 交互界面（还是纯命令行），就可以在下面输入命令了：

- help # 查看帮助
- status # 查看程序状态
- stop program_name # 关闭 指定的程序
- start program_name # 启动 指定的程序
- restart program_name # 重启 指定的程序
- tail -f program_name # 查看 该程序的日志
- update # 重启配置文件修改过的程序（修改了配置，通过这个命令加载新的配置)


也可以直接通过 shell 命令操作：

- supervisorctl status
- supervisorctl update
...





