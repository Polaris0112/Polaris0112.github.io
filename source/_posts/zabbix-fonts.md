---
title: Zabbix汉化字体修改
date: 2018-02-28
categories: Zabbix汉化字体修改
tags: 
- Zabbix
- Font

---
本帖子记录的是Zabbix3.4安装后，设置为中文环境，但是查看"图形"监控的时候图形下方的图例中文也会出现乱码（方块），所以需要修改字体文件来显示正常中文。

## 原因

Zabbix的图形显示用的字体是dejavu，不支持中文。



## 解决方法


### Zabbix3.2.x以及之前的解决方法

这个版本之前的都是放在Web服务的文件夹中，比如

- nginx：使用这个Web服务的路径在(yum安装)`/usr/share/nginx/html/`目录下，(编译安装)自定义目录路径下。

- apache：使用这个Web服务的路径在(默认路径)`/var/www/html/`目录下。

这两个Web服务的文件下一般都会有一个zabbix的文件(或者叫其他名字)，然后里面的`/path/zabbix/fonts/`目录下会有一个叫`DejaVuSans.ttf`的文件。

因为前端的php已经默认了使用`DejaVuSans.ttf`这个文件作为字体文件，所以我们可以不修改php，把我们想要的字体改成对应的名字就行。

[微软雅黑](https://github.com/Polaris0112/Ops-Tools/raw/master/zabbix_installation/MSYH.TTF)  <--这个是我从Windows7中导出来的微软雅黑字体，亲测可用

```bash
## 然后把这个字体先复制`/path/zabbix/fonts`下，下面以nginx为例
$ cp MSYH.TTF /usr/share/nginx/html/zabbix/fonts/

## 更改原本字体文件名作为备份
$ mv /usr/share/nginx/html/zabbix/fonts/DejaVuSans.ttf /usr/share/nginx/html/zabbix/fonts/DejaVuSans.ttf.bak

## 把新的字体改到对应的字体文件名
$ mv /usr/share/nginx/html/zabbix/fonts/MSYH.TTF /usr/share/nginx/html/zabbix/fonts/DejaVuSans.ttf 
```

无需重启服务，直接刷新网页端即可。




### Zabbix3.4.x的解决方法

以下方法使用的是用yum方法安装Zabbix3.4.7之后修改字体，测试通过。

```bash
## 先下载字体，如上
$ wget https://github.com/Polaris0112/Ops-Tools/raw/master/zabbix_installation/MSYH.TTF
$ mv MSYH.TTF /usr/share/fonts/MSYH.TTF

## zabbix配置文件(/usr/share/zabbix/include/defines.inc.php)里，定义的字体叫做graphfont.ttf
## 然后一路软链接到DejaVuSans.ttf，如下：
## /usr/share/zabbix/graphfont.ttf -> /etc/alternatives/zabbix-web-font -> /usr/share/fonts/dejavu/DejaVuSans.ttf
## 那么，修改掉最后一层软连接的目标字体就可以了。执行类似下面的命令就可以了。

## 先备份原有字体
$ mv /etc/alternatives/zabbix-web-font /etc/alternatives/zabbix-web-font_bak 

## 创建新增软连接
$ ln -s /usr/share/fonts/MSYH.TTF /etc/alternatives/zabbix-web-font
```

无需重启服务，直接刷新网页端即可。




