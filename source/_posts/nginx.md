---
title: Nginx正反向代理/负载均衡
date: 2018-02-21
categories: Nginx正反向代理/负载均衡
tags: 
- nginx
- proxy
- load-balance

---
本帖子记录的是Nginx配置正向代理、反向代理和负载均衡相关参数。

## 相关概念

- 正向代理是一个位于客户端和目标服务器之间的代理服务器(中间服务器)。为了从原始服务器取得内容，客户端向代理服务器发送一个请求，并且指定目标服务器，之后代理向目标服务器转交并且将获得的内容返回给客户端。正向代理的情况下客户端必须要进行一些特别的设置才能使用。

![nginx-forward-proxy](/images/nginx-forward-proxy.png)

- 反向代理正好相反。对于客户端来说，反向代理就好像目标服务器。并且客户端不需要进行任何设置。客户端向反向代理发送请求，接着反向代理判断请求走向何处，并将请求转交给客户端，使得这些内容就好似他自己一样，一次客户端并不会感知到反向代理后面的服务，也因此不需要客户端做任何设置，只需要把反向代理服务器当成真正的服务器就好了。

![nginx-reverse-proxy](/images/nginx-reverse-proxy.png)

- 负载均衡：当反向代理服务器不止一个的时候，我们甚至可以把它们做成集群，当更多的用户访问资源服务器B的时候，让不同的代理服务器Z（x）去应答不同的用户，然后发送不同用户需要的资源。

![nginx-balance.png](/images/nginx-balance.png)

### 正向代理和反向代理区别

正向代理需要你主动设置代理服务器ip或者域名进行访问，由设置的服务器ip或者域名去获取访问内容并返回；而反向代理不需要你做任何设置，直接访问服务器真实ip或者域名，但是服务器内部会自动根据访问内容进行跳转及内容返回，你不知道它最终访问的是哪些机器。

正向代理是代理客户端，为客户端收发请求，使真实客户端对服务器不可见；而反向代理是代理服务器端，为服务器收发请求，使真实服务器对客户端不可见。

从上面的描述也能看得出来正向代理和反向代理最关键的两点区别：

 - 是否指定目标服务器

 - 客户端是否要做设置

下面用一张图来表示两者的差异：

![nginx-proxy](/images/nginx-proxy.png)


正向代理中，proxy和client同属一个LAN，对server透明； 反向代理中，proxy和server同属一个LAN，对client透明。 实际上proxy在两种代理中做的事都是代为收发请求和响应，不过从结构上来看正好左右互换了下，所以把前者那种代理方式叫做正向代理，后者叫做反向代理。



**从用途上来区分：**

正向代理：正向代理用途是为了在防火墙内的局域网提供访问internet的途径。另外还可以使用缓冲特性减少网络使用率

反向代理：反向代理的用途是将防火墙后面的服务器提供给internet用户访问。同时还可以完成诸如负载均衡等功能




**从安全性来讲：**

正向代理：正向代理允许客户端通过它访问任意网站并且隐蔽客户端自身，因此你必须采取安全措施来确保仅为经过授权的客户端提供服务

反向代理：对外是透明的，访问者并不知道自己访问的是代理。对访问者而言，他以为访问的就是原始服务器



## Nginx 配置 


## 正向代理

nginx.conf 文件：

``` bash
server {
    # 配置DNS解析IP地址，比如 Google Public DNS，以及超时时间（5秒）
    resolver 8.8.8.8;    # 必需
    resolver_timeout 5s;

    # 监听端口
    listen 8080;

    access_log  /home/reistlin/logs/proxy.access.log;
    error_log   /home/reistlin/logs/proxy.error.log;

    location / {
        # 配置正向代理参数
        proxy_pass $scheme://$host$request_uri;
        # 解决如果URL中带"."后Nginx 503错误
        proxy_set_header Host $http_host;

        # 配置缓存大小
        proxy_buffers 256 4k;
        # 关闭磁盘缓存读写减少I/O
        proxy_max_temp_file_size 0;
         # 代理连接超时时间
        proxy_connect_timeout 30;

        # 配置代理服务器HTTP状态缓存时间
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 301 1h;
        proxy_cache_valid any 1m;
    }
}
```

备注：

 - 配置 nginx 正向代理服务，一般是配置到一个 server 块中。注意，在该 server 块中，不要出现 server_name 指令，即不要设置虚拟主机的名称和 IP；

 - resolver 是必需的，如果没有该指令， nginx 无法处理接收到的域名；

 - nginx 代理服务不支持正向代理 HTTPS 站点；



## 反向代理

nginx.conf 文件：

``` bash
upstream webServer01 {
    server 127.0.0.1:3001;
    keepalive 64;
}

upstream webServer02 {
    server 127.0.0.1:3002;
    keepalive 64;
}

server {
    listen 80;
    server_name www.test01.com;
    #access_log /var/log/nginx/test01.log;
    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host  $http_host;
        proxy_set_header X-Nginx-Proxy true;
        proxy_set_header Connection "";
        proxy_pass       http://webServer01;

    }

}

server {
    listen 80;
    server_name www.test02.com;
    #access_log /var/log/nginx/test02.log;
    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host  $http_host;
        proxy_set_header X-Nginx-Proxy true;
        proxy_set_header Connection "";
        proxy_pass       http://webServer02;
    }
}

```


## 负载均衡

Nginx负载均衡是通过upstream模块来实现的，内置实现了三种负载策略，配置还是比较简单的。[官网负载均衡配置说明](http://nginx.org/en/docs/http/load_balancing.html)

- 轮循（默认） 
  Nginx根据请求次数，将每个请求均匀分配到每台服务器


- 最少连接 
  将请求分配给连接数最少的服务器。Nginx会统计哪些服务器的连接数最少。


- IP Hash 
  绑定处理请求的服务器。第一次请求时，根据该客户端的IP算出一个HASH值，将请求分配到集群中的某一台服务器上。后面该客户端的所有请求，都将通过HASH算法，找到之前处理这台客户端请求的服务器，然后将请求交给它来处理。



### 轮循

``` bash
http {

    # ... 省略其它配置

    upstream tomcats {
        server 192.168.0.100:8080;
        server 192.168.0.101:8080;
        server example.com:8080;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://tomcats;
        }
    }

    # ... 省略其它配置
}
```

- proxy_pass http://tomcats：表示将所有请求转发到tomcats服务器组中配置的某一台服务器上。

- upstream模块：配置反向代理服务器组，Nginx会根据配置，将请求分发给组里的某一台服务器。tomcats是服务器组的名称。

- upstream模块下的server指令：配置处理请求的服务器IP或域名，端口可选，不配置默认使用80端口。通过上面的配置，Nginx默认将请求依次分配给100，101，102来处理，可以通过修改下面这些参数来改变默认的分配策略：

 - weight 

   默认为1，将请求平均分配给每台server

   ``` bash
upstream tomcats {
    server 192.168.0.100:8080 weight=2;  # 2/6次
    server 192.168.0.101:8080 weight=3;  # 3/6次
    server 192.168.0.102:8080 weight=1;  # 1/6次
}
   ```
   上例配置，表示6次请求中，100分配2次，101分配3次，102分配1次

 - max_fails 

   默认为1。某台Server允许请求失败的次数，超过最大次数后，在fail_timeout时间内，新的请求将不会分配给这台机器。如果设置为0，Nginx会将这台Server置为永久无效状态，然后将请求发给定义了proxy_next_upstream, fastcgi_next_upstream, uwsgi_next_upstream, scgi_next_upstream, and memcached_next_upstream指令来处理这次错误的请求。

 - fail_timeout 

   默认为10秒。某台Server达到max_fails次失败请求后，在fail_timeout期间内，nginx会认为这台Server暂时不可用，不会将请求分配给它

   ``` bash
upstream tomcats {
    server 192.168.0.100:8080 weight=2 max_fails=3 fail_timeout=15;
    server 192.168.0.101:8080 weight=3;
    server 192.168.0.102:8080 weight=1;
}
   ```
   
   192.168.0.100这台机器，如果有3次请求失败，nginx在15秒内，不会将新的请求分配给它。

 - backup 

   备份机，所有服务器挂了之后才会生效

   ``` bash
upstream tomcats {
    server 192.168.0.100:8080 weight=2 max_fails=3 fail_timeout=15;
    server 192.168.0.101:8080 weight=3;

    server 192.168.0.102:8080 backup;
}
   ```

   在100和101都挂了之前，102为不可用状态，不会将请求分配给它。只有当100和101都挂了，102才会被启用。

 - down 

   标识某一台server不可用。可能能通过某些参数动态的激活它吧，要不真没啥用。

   ``` bash
upstream tomcats {
    server 192.168.0.100:8080 weight=2 max_fails=3 fail_timeout=15;

    server 192.168.0.101:8080 down;

    server 192.168.0.102:8080 backup;
}
   ```
  
   表示101这台Server为无效状态，不会将请求分配给它。

 - max_conns 

   限制分配给某台Server处理的最大连接数量，超过这个数量，将不会分配新的连接给它。默认为0，表示不限制。注意：1.5.9之后的版本才有这个配置

   ``` bash
upstream tomcats {
    server 192.168.0.100:8080 max_conns=1000;
}
   ```
  
   表示最多给100这台Server分配1000个请求，如果这台Server正在处理1000个请求，nginx将不会分配新的请求给到它。假如有一个请求处理完了，还剩下999个请求在处理，这时nginx也会将新的请求分配给它。

 - resolve 

   将server指令配置的域名，指定域名解析服务器。需要在http模块下配置resolver指令，指定域名解析服务

   ``` bash
http {
    resolver 10.0.0.1;

    upstream u {
        zone ...;
        ...
        server example.com resolve;
    }
}
   ```

   表示example.com域名，由10.0.0.1服务器来负责解析。 
   
   upstream模块server指令的其它参数和详细配置说明，请参考[官方文档](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server)。



## 第三方负载策略

- fair

  根据服务器的响应时间来分配请求，响应时间短的优先分配，即负载压力小的优先会分配。

  由于fair模块是第三方提供的，所以在编译nginx源码的时候，需要将fair添加到nginx模块中。

 - 下载fair模块源码 
   
   [下载地址](https://github.com/xyang0917/nginx-upstream-fair)

   ``` bash
$ cd /opt
$ wget https://github.com/xyang0917/nginx-upstream-fair/archive/master.zip
$ unzip master.zip
   ```
   解压后的目录名为：nginx-upstream-fair-master

 - 重新编译nginx，将fair模块添加到编译参数 

   ``` bash
$ cd /opt/nginx-nginx-1.10.0
$ ./configure --prefix=/opt/nginx --add-module=/opt/nginx-upstream-fair-master
$ make
   ```
   **注意：**不要执行make install，这样会覆盖之前nginx的配置 

 - 将新编译的nginx可执行程序拷贝到/opt/nginx/sbin/目录下，覆盖之前安装的nginx编译后的nginx执行程序，放在nginx源码的objs目录下
 
   ``` bash
$ ps -aux | grep nginx
$ kill -9 nginx进程ID  # 停止nginx服务
$ cp /opt/nginx-1.10.0/objs/nginx /opt/nginx/sbin/  # 覆盖旧的nginx
$ nginx # 启动服务
   ```

   配置使用fair负载策略模块：

   ``` bash
upstream tomcats {
    fair;
    server 192.168.0.100:8080;
    server 192.168.0.101:8080;
    server 192.168.0.102:8080;
}
   ```
   由于采用fair负载策略，配置weigth参数改变负载权重将无效。




- url_hash

  按请求url的hash结果来分配请求，使每个url定向到同一个后端服务器，服务器做缓存时比较有效。


  1.7.2版本以后，url_hash模块已经集成到了nginx源码当中，不需要单独安装。之前的版本仍需要单独安装，[下载地址](https://github.com/evanmiller/nginx_upstream_hash)

  安装方法和fair模块一样，先下载url_hash源码，然后重新编译nginx源码，将url_hash模块添加到编译配置参数当中，最后将编译后生成的nginx二进制文件替换之前安装的nginx二进制文件即可。

``` bash
upstream tomcats {
    server 192.168.0.100:8080;
    server 192.168.0.101:8080;
    server 192.168.0.102:8080;
    hash $request_uri;
}
```






