---
title: 浅谈Docker基础之数据管理
date: 2018-03-20
categories: Docker数据卷
tags: 
- Docker

---
本帖子记录的是浅谈Docker基础之数据管理

用户在使用 Docker 的过程中，往往需要能查看容器内应用产生的数据，或者需要把容器内的数据进行备份，甚至多个容器之间进行数据的共享，这必然涉及容器的数据管理操作。容器中管理数据主要有两种方式：数据卷(Data Volumes)，数据卷容器(Data Volume Containers)。


## 数据卷

数据卷是一个可供容器使用的特殊目录，它绕过文件系统，可以提供很多有用的特性：

1.数据卷可以在容器之间共享和重用。

2.对数据卷的更改会立即生效。

3.对数据卷的更新不会影响镜像。

4.数据卷会一直存在，直到没有容器使用。

数据卷的使用，类似于 linux 下对目录或文件进行 mount 操作。



## 在容器内创建一个数据卷

在用 docker run 命令的时候，使用 -v 标记可以在容器内创建一个数据卷。多次使用 -v 标记可以创建多个数据卷。

下面的例子中我们使用 myimg/webapp 镜像创建一个 web 容器，并创建一个数据卷挂载到容器的 /webdata 目录。

```bash
$ sudo docker run -d -P --name web -v /webdata myimg/webapp python app.py
```


## 挂载一个主机目录作为数据卷

使用 -v 标记也可以指定挂载一个本地的已有目录到容器中去作为数据卷：

```bash
$ sudo docker run -d -P --name web -v /var/data:/opt/webdata myimg/webapp python app.py
```

上面的命令挂载主机的 /var/data 目录到容器的 /opt/webdata 目录。

这个功能在接下测试的时候特别方便，比如用户可以放置一些程序或数据到本地目录中，然后在容器中使用。另外，本地目录的路径必须是绝对路径，如果目录不存在，Docker 会自动创建。
Docker 挂载数据卷的默认权限是可读写(rw)，用户也可以通过 ro 标记指定为只读：

```bash
$ sudo docker run -d -P --name web -v /var/data:/opt/webdata:ro myimg/webapp python app.py
```

加了 :ro 之后，容器内挂载的数据卷内的数据就变成只读的了。



## 挂载一个本地主机文件作为数据卷

-v 标记也可以挂载一个主机中的文件到容器中作为数据卷，但是这样做会带来一些问题。建议还是挂载文件所在的目录。



## 数据卷容器

如果用户需要在容器之间共享一些持续更新的数据，最简单的方式是使用数据卷容器。数据卷容器其实就是一个普通的容器，专门用它提供数据卷供其他容器挂载。下面简单介绍其使用方法。

首先要创建一个数据卷容器 mydata，并在其中创建一个数据卷挂载到 /data 目录。

```bash
$ sudo docker run -it -v /data --name mydata ubuntu
```

然后在其他容器中使用 --volumes-from 来挂载 mydata 容器中的数据卷。例如创建两个容器 mycon1 和 mycon2，并从 mydata 容器挂载数据卷：

```bash
$ sudo docker run -it --volumes-from mydata --name mycon1 ubuntu
$ sudo docker run -it --volumes-from mydata --name mycon2 ubuntu
```

(注意，命令中没有指定数据卷的信息，也就是说新容器中挂载数据卷的目录和源容器中是一样的。)

此时容器 mycon1 和 mycon2 都挂载同一个数据卷到相同的目录 /data。三个容器任何一个在该目录下写入数据其他容器都能看到。

可以多次使用 --volumes-from 参数来从多个容器挂载多个数据卷。还可以从其他已经挂载了容器的容器来挂载数据卷。并且使用 --volumes-from 参数所挂载数据卷的容器自身并不需要保持在运行状态。
但删除挂载了数据卷的容器时，数据卷并不会被自动删除。如果要删除一个数据卷，必须在删除最后一个还挂载着它的容器时显式的使用 docker rm -v 命令来指定同时删除关联的容器。

使用数据卷容器可以让用户在容器之间自由的升级和移动数据卷，下面会进行详细的介绍。



## 利用数据卷容器迁移数据

可以利用数据卷容器对其中的数据卷进行备份、恢复，以实现数据的迁移。

### 备份

使用下面的命令来备份 mydata 数据卷容器内的数据卷：

```bash
$ sudo docker run --volumes-from mydata -v $(pwd):/backup --name worker ubuntu tar cvf /backup/backup.tar /data
```

这个命令首先利用 Ubuntu 镜像创建了一个容器 worker。又使用 --volumes-from mydata 参数来让 worker 容器挂载 mydata 容器的数据卷。接下来使用 -v $(pwd):/backup 参数来挂载本地的当前目录到 worker 容器的 /backup 目录。

在 worker 容器启动后，使用了 tar cvf /backup/backup.tar /data 命令来将 /data 下内容备份为容器内的 /backup/backup.tar，即宿主主机的当前目录下的backup.tar。



### 恢复

如果要恢复数据到一个容器，可以按照下面的操作。首先创建一个带有数据卷的容器 mydata2:

```bash
$ sudo docker run -v /data –name mydata2 ubuntu /bin/bash
```

然后创建另一个新的容器，挂载 mydata2 的数据卷，并使用 tar 解压缩备份文件到所挂载的容器卷中：

```bash
$ sudo docker run --volumes-from mydata2 -v $(pwd):/backup busybox tar xvf /backup/backup.tar
```




