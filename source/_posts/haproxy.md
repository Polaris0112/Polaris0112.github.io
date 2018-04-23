---
title: HAProxy日常使用说明
date: 2018-02-23
categories: HAProxy使用
tags: 
- haproxy

---
本帖子记录的是HAProxy安装和配置说明

## 简介

HAProxy提供高可用性、负载均衡以及基于TCP和HTTP应用的代理，支持虚拟主机，它是免费、快速并且可靠的一种解决方案。

HAProxy特别适用于那些负载特大的web站点，这些站点通常又需要会话保持或七层处理。

HAProxy运行在当前的硬件上，完全可以支持数以万计的并发连接。并且它的运行模式使得它可以很简单安全的整合进您当前的架构中， 同时可以保护你的web服务器不被暴露到网络上。

HAProxy实现了一种事件驱动, 单一进程模型，此模型支持非常大的并发连接数。多进程或多线程模型受内存限制 、系统调度器限制以及无处不在的锁限制，很少能处理数千并发连接。事件驱动模型因为在有更好的资源和时间管理的用户空间(User-Space) 实现所有这些任务，所以没有这些问题。此模型的弊端是，在多核系统上，这些程序通常扩展性较差。这就是为什么他们必须进行优化以 使每个CPU时间片(Cycle)做更多的工作。

 

## 安装

- 系统：CentOS 7/RHEL 7

``` bash
## 以1.6.9为例
#下载
wget http://fossies.org/linux/misc/haproxy-1.6.9.tar.gz
#解压
tar -zxvf haproxy-1.6.9.tar.gz
cd haproxy-1.6.9
#安装
make TARGET=linux2628 ARCH=x86_64 PREFIX=/usr/local/haproxy
make install PREFIX=/usr/local/haproxy

#参数说明
TARGET=linux26 #内核版本，使用uname -r查看内核，如：2.6.18-371.el5，此时该参数就为linux26；kernel 大于2.6.28的用：TARGET=linux2628
ARCH=x86_64 #系统位数
PREFIX=/usr/local/haprpxy #/usr/local/haprpxy为haprpxy安装路径
```


也可以直接使用`yum`命令进行下载，如：`yum install -y haproxy`

这里也记录下各个系统各haproxy版本的rpm包[下载地址](https://pkgs.org/download/haproxy)


## 配置文件说明

- **启动配置文件`/usr/local/haproxy/haproxy.cfg`**


``` bash
###########全局配置#########
global
　　log 127.0.0.1 local0 #[日志输出配置，所有日志都记录在本机，通过local0输出]
　　log 127.0.0.1 local1 notice #定义haproxy 日志级别[error warringinfo debug]
　　daemon #以后台形式运行harpoxy
　　nbproc 1 #设置进程数量
　　maxconn 4096 #默认最大连接数,需考虑ulimit-n限制
　　#user haproxy #运行haproxy的用户
　　#group haproxy #运行haproxy的用户所在的组
　　#pidfile /var/run/haproxy.pid #haproxy 进程PID文件
　　#ulimit-n 819200 #ulimit 的数量限制
　　#chroot /usr/share/haproxy #chroot运行路径
　　#debug #haproxy 调试级别，建议只在开启单进程的时候调试
　　#quiet

########默认配置############
defaults
　　log global
　　mode http #默认的模式mode { tcp|http|health }，tcp是4层，http是7层，health只会返回OK
　　option httplog #日志类别,采用httplog
　　option dontlognull #不记录健康检查日志信息
　　retries 2 #两次连接失败就认为是服务器不可用，也可以通过后面设置
　　#option forwardfor #如果后端服务器需要获得客户端真实ip需要配置的参数，可以从Http Header中获得客户端ip
　　option httpclose #每次请求完毕后主动关闭http通道,haproxy不支持keep-alive,只能模拟这种模式的实现
　　#option redispatch #当serverId对应的服务器挂掉后，强制定向到其他健康的服务器，以后将不支持
　　option abortonclose #当服务器负载很高的时候，自动结束掉当前队列处理比较久的链接
　　maxconn 4096 #默认的最大连接数
　　timeout connect 5000ms #连接超时
　　timeout client 30000ms #客户端超时
　　timeout server 30000ms #服务器超时
　　#timeout check 2000 #心跳检测超时
　　#timeout http-keep-alive10s #默认持久连接超时时间
　　#timeout http-request 10s #默认http请求超时时间
　　#timeout queue 1m #默认队列超时时间
　　balance roundrobin #设置默认负载均衡方式，轮询方式
　　#balance source #设置默认负载均衡方式，类似于nginx的ip_hash
　　#balnace leastconn #设置默认负载均衡方式，最小连接数

########统计页面配置########
listen stats
　　bind 0.0.0.0:1080 #设置Frontend和Backend的组合体，监控组的名称，按需要自定义名称
　　mode http #http的7层模式
　　option httplog #采用http日志格式
　　#log 127.0.0.1 local0 err #错误日志记录
　　maxconn 10 #默认的最大连接数
　　stats refresh 30s #统计页面自动刷新时间
　　stats uri /stats #统计页面url
　　stats realm XingCloud\ Haproxy #统计页面密码框上提示文本
　　stats auth admin:admin #设置监控页面的用户和密码:admin,可以设置多个用户名
　　stats auth Frank:Frank #设置监控页面的用户和密码：Frank
　　stats hide-version #隐藏统计页面上HAProxy的版本信息
　　stats admin if TRUE #设置手工启动/禁用，后端服务器(haproxy-1.4.9以后版本)

########设置haproxy 错误页面#####
#errorfile 403 /home/haproxy/haproxy/errorfiles/403.http
#errorfile 500 /home/haproxy/haproxy/errorfiles/500.http
#errorfile 502 /home/haproxy/haproxy/errorfiles/502.http
#errorfile 503 /home/haproxy/haproxy/errorfiles/503.http
#errorfile 504 /home/haproxy/haproxy/errorfiles/504.http

########frontend前端配置##############
frontend main
　　bind *:80 #这里建议使用bind *:80的方式，要不然做集群高可用的时候有问题，vip切换到其他机器就不能访问了。
　　acl web hdr(host) -i www.abc.com  #acl后面是规则名称，-i为忽略大小写，后面跟的是要访问的域名，如果访问www.abc.com这个域名，就触发web规则，。
　　acl img hdr(host) -i img.abc.com  #如果访问img.abc.com这个域名，就触发img规则。
　　use_backend webserver if web   #如果上面定义的web规则被触发，即访问www.abc.com，就将请求分发到webserver这个作用域。
　　use_backend imgserver if img   #如果上面定义的img规则被触发，即访问img.abc.com，就将请求分发到imgserver这个作用域。
　　default_backend dynamic #不满足则响应backend的默认页面

########backend后端配置##############
backend webserver #webserver作用域
　　mode http
　　balance roundrobin #balance roundrobin 负载轮询，balance source 保存session值，支持static-rr，leastconn，first，uri等参数
　　option httpchk /index.html HTTP/1.0 #健康检查, 检测文件，如果分发到后台index.html访问不到就不再分发给它
　　server web1 10.16.0.9:8085 cookie 1 weight 5 check inter 2000 rise 2 fall 3
　　server web2 10.16.0.10:8085 cookie 2 weight 3 check inter 2000 rise 2 fall 3
　　#cookie 1表示serverid为1，check inter 1500 是检测心跳频率 
　　#rise 2是2次正确认为服务器可用，fall 3是3次失败认为服务器不可用，weight代表权重

backend imgserver
　　mode http
　　option httpchk /index.php
　　balance roundrobin 
　　server img01 192.168.137.101:80 check inter 2000 fall 3
　　server img02 192.168.137.102:80 check inter 2000 fall 3

backend dynamic 
　　balance roundrobin 
　　server test1 192.168.1.23:80 check maxconn 2000 
　　server test2 192.168.1.24:80 check maxconn 2000


listen tcptest 
　　bind 0.0.0.0:5222 
　　mode tcp 
　　option tcplog #采用tcp日志格式 
　　balance source 
　　#log 127.0.0.1 local0 debug 
　　server s1 192.168.100.204:7222 weight 1 
　　server s2 192.168.100.208:7222 weight 1
```



- **负载均衡算法**

``` bash
一、roundrobin，表示简单的轮询，每个服务器根据权重轮流使用，在服务器的处理时间平均分配的情况下这是最流畅和公平的算法。该算法是动态的，对于实例启动慢的服务器权重会在运行中调整。

二、static-rr，表示根据权重，建议关注；每个服务器根据权重轮流使用，类似roundrobin，但它是静态的，意味着运行时修改权限是无效的。另外，它对服务器的数量没有限制。

三、leastconn，表示最少连接者先处理，建议关注；leastconn建议用于长会话服务，例如LDAP、SQL、TSE等，而不适合短会话协议。如HTTP.该算法是动态的，对于实例启动慢的服务器权重会在运行中调整。

四、source，表示根据请求源IP，建议关注；对请求源IP地址进行哈希，用可用服务器的权重总数除以哈希值，根据结果进行分配。
           只要服务器正常，同一个客户端IP地址总是访问同一个服务器。如果哈希的结果随可用服务器数量而变化，那么客户端会定向到不同的服务器；
           该算法一般用于不能插入cookie的Tcp模式。它还可以用于广域网上为拒绝使用会话cookie的客户端提供最有效的粘连；
           该算法默认是静态的，所以运行时修改服务器的权重是无效的，但是算法会根据“hash-type”的变化做调整。
五、uri，表示根据请求的URI；表示根据请求的URI左端（问号之前）进行哈希，用可用服务器的权重总数除以哈希值，根据结果进行分配。
        只要服务器正常，同一个URI地址总是访问同一个服务器。
        一般用于代理缓存和反病毒代理，以最大限度的提高缓存的命中率。该算法只能用于HTTP后端；
        该算法一般用于后端是缓存服务器；
        该算法默认是静态的，所以运行时修改服务器的权重是无效的，但是算法会根据“hash-type”的变化做调整。
六、url_param，表示根据请求的URl参数'balance url_param' requires an URL parameter name
              在HTTP GET请求的查询串中查找<param>中指定的URL参数，基本上可以锁定使用特制的URL到特定的负载均衡器节点的要求；
              该算法一般用于将同一个用户的信息发送到同一个后端服务器；
              该算法默认是静态的，所以运行时修改服务器的权重是无效的，但是算法会根据“hash-type”的变化做调整。
七、hdr(name)，表示根据HTTP请求头来锁定每一次HTTP请求；
              在每个HTTP请求中查找HTTP头<name>，HTTP头<name>将被看作在每个HTTP请求，并针对特定的节点；
              如果缺少头或者头没有任何值，则用roundrobin代替；
              该算法默认是静态的，所以运行时修改服务器的权重是无效的，但是算法会根据“hash-type”的变化做调整。
八、rdp-cookie(name)，表示根据据cookie(name)来锁定并哈希每一次TCP请求。
                     为每个进来的TCP请求查询并哈希RDP cookie<name>；
                     该机制用于退化的持久模式，可以使同一个用户或者同一个会话ID总是发送给同一台服务器。
                     如果没有cookie，则使用roundrobin算法代替；
                     该算法默认是静态的，所以运行时修改服务器的权重是无效的，但是算法会根据“hash-type”的变化做调整。

#其实这些算法各有各的用法，我们平时应用得比较多的应该是roundrobin、source和lestconn。
```


- **ACL规则定义**


``` bash
########ACL策略定义#########################
1、#如果请求的域名满足正则表达式返回true -i是忽略大小写
acl denali_policy hdr_reg(host) -i ^(www.inbank.com|image.inbank.com)$

2、#如果请求域名满足www.inbank.com 返回 true -i是忽略大小写
acl tm_policy hdr_dom(host) -i www.inbank.com

3、#在请求url中包含sip_apiname=，则此控制策略返回true,否则为false
acl invalid_req url_sub -i sip_apiname=#定义一个名为invalid_req的策略

4、#在请求url中存在timetask作为部分地址路径，则此控制策略返回true,否则返回false
acl timetask_req url_dir -i timetask

5、#当请求的header中Content-length等于0时返回 true
acl missing_cl hdr_cnt(Content-length) eq 0

#########acl策略匹配相应###################
1、#当请求中header中Content-length等于0 阻止请求返回403
block if missing_cl

2、#block表示阻止请求，返回403错误，当前表示如果不满足策略invalid_req，或者满足策略timetask_req，则阻止请求。
block if !invalid_req || timetask_req

3、#当满足denali_policy的策略时使用denali_server的backend
use_backend denali_server if denali_policy

4、#当满足tm_policy的策略时使用tm_server的backend
use_backend tm_server if tm_policy

5、#reqisetbe关键字定义，根据定义的关键字选择backend
reqisetbe ^Host:\ img dynamic
reqisetbe ^[^\ ]*\ /(img|css)/ dynamic
reqisetbe ^[^\ ]*\ /admin/stats stats

6、#以上都不满足的时候使用默认mms_server的backend
default_backend mms

haproxy acl定义
```




## 启动

- 编译安装

```bash
$ /usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/haproxy.cfg 
```

- yum安装

```bash
$ service haproxy start      #CentOS6
$ systemctl start haproxy    #CentOS7
```






