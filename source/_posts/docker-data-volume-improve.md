---
title: Docker数据卷之进阶篇
date: 2018-03-21
categories: Docker数据卷
tags: 
- Docker

---
本帖子记录的是Docker数据卷之进阶篇

在《浅谈Docker基础之数据管理》[https://polaris0112.github.io/2018/03/20/docker-data-volume/]一文中介绍了 `Docker` 数据卷(volume) 的基本用法。随着使用的深入，笔者对 `Docker` 数据卷的理解与认识也在不断的增强。本文将在前文的基础上介绍 `Docker` 数据卷的原理及一些高级用法。

## 为什么需要数据卷？

这得从 docker 容器的文件系统说起。出于效率等一系列原因，docker 容器的文件系统在宿主机上存在的方式很复杂，这会带来下面几个问题：

- 不能在宿主机上很方便地访问容器中的文件。
- 无法在多个容器之间共享数据。
- 当容器删除时，容器中产生的数据将丢失。

为了解决这些问题，docker 引入了数据卷(volume) 机制。数据卷是存在于一个或多个容器中的特定文件或文件夹，这个文件或文件夹以独立于 docker 文件系统的形式存在于宿主机中。数据卷的最大特定是：其生存周期独立于容器的生存周期。

使用数据卷的最佳场景

- 在多个容器之间共享数据，多个容器可以同时以只读或者读写的方式挂载同一个数据卷，从而共享数据卷中的数据。
- 当宿主机不能保证一定存在某个目录或一些固定路径的文件时，使用数据卷可以规避这种限制带来的问题。
- 当你想把容器中的数据存储在宿主机之外的地方时，比如远程主机上或云存储上。
- 当你需要把容器数据在不同的宿主机之间备份、恢复或迁移时，数据卷是很好的选择。




## docker volume 子命令

`docker` 专门提供了 volume 子命令来操作数据卷：
`create` 创建数据卷
`inspect` 显示数据卷的详细信息 
`ls` 列出所有的数据卷
`prune` 删除所有未使用的 volumes，并且有 -f 选项
`rm` 删除一个或多个未使用的 volumes，并且有 -f 选项

先创建一个名称为 hello 的数据卷并通过 ls 命令进行查看：
![docker-data-volume-1](/images/docker-data-volume-1.png)

然后可以使用 inspect 命令看看数据卷 hello 的详细信息：
![docker-data-volume-2](/images/docker-data-volume-2.png)

在这里我们可以看到创建数据卷的时间；该数据卷使用的驱动程序为默认的 "local"，表示数据卷使用宿主机的本地存储；数据卷的挂载点，默认是本机 /var/lib/docker/volumes 下的一个目录。
最后我们可以使用 rm 或 prune 命令删除数据卷，后面笔者会介绍一些实际使用中与数据卷的删除有关的一些实践。




## 使用 mount 语法挂载数据卷

之前我们使用 --volume(-v) 选项来挂载数据卷，现在 docker 提供了更强大的 --mount 选项来管理数据卷。mount 选项可以通过逗号分隔的多个键值对一次提供多个配置项，因此 mount 选项可以提供比 volume 选项更详细的配置。使用 mount 选项的常用配置如下：

`type` 指定挂载方式，我们这里用到的是 volume，其实还可以有 bind 和 tmpfs。
`volume-driver` 指定挂载数据卷的驱动程序，默认值是 local。
`source` 指定挂载的源，对于一个命名的数据卷，这里应该指定这个数据卷的名称。在使用时可以写 source，也可以简写为 src。
`destination` 指定挂载的数据在容器中的路径。在使用时可以写 destination，也可以简写为 dst 或 target。
`readonly` 指定挂载的数据为只读。
`volume-opt` 可以指定多次，用来提高更多的 mount 相关的配置。

下面我们看个具体的例子：

```bash
$ docker volume create hello
$ docker run -id --mount type=volume,source=hello,target=/world ubuntu /bin/bash
```

我们创建了名称为 hello 的数据卷，然后把它挂在到容器中的 /world 目录。通过 inspect 命令查看容器的详情中的 "Mounts" 信息可以验证实际的数据卷挂载结果 ：
![docker-data-volume-3](/images/docker-data-volume-3.png)




## 使用 volume driver 把数据存储到其它地方

除了默认的把数据卷中的数据存储在宿主机，docker 还允许我们通过指定 volume driver 的方式把数据卷中的数据存储在其它的地方，比如 Azrue Storge 或 AWS 的 S3。
简单起见，我们接下来的 demo 演示如何通过 vieux/sshfs 驱动把数据卷的存储在其它的主机上。
docker 默认是不安装 vieux/sshfs 插件的，我们可以通过下面的命令进行安装：

```bash
$ docker plugin install --grant-all-permissions vieux/sshfs
```

然后通过 vieux/sshfs 驱动创建数据卷，并指定远程主机的登录用户名、密码和数据存放目录：

```bash
$ docker volume create --driver vieux/sshfs \
    -o sshcmd=nick@10.32.2.134:/home/nick/sshvolume \
    -o password=yourpassword \
    mysshvolume
```

注意，请确保你指定的远程主机上的挂载点目录是存在的(demo 中是 /home/nick/sshvolume 目录)，否则在启动容器时会报错。
最后在启动容器时指定挂载这个数据卷：

```bash
$ docker run -id \
    --name testcon \
    --mount type=volume,volume-driver=vieux/sshfs,source=mysshvolume,target=/world \
    ubuntu /bin/bash
```

这就搞定了，你在容器中 /world 目录下操作的文件都存储在远程主机的 /home/nick/sshvolume 目录中。进入容器 testcon 然后在 /world 目录中创建一个文件，然后打开远程主机的  /home/nick/sshvolume 目录进行查看，你新建的文件是不是已经出现在那里了！




## 数据卷原理

下图描述了 docker 容器挂载数据的三种方式：
![docker-data-volume-4](/images/docker-data-volume-4.png)

数据卷是完全被 docker 管理的，就像上图中的黄色区域描述的一样，docker 在宿主机的文件系统中找了个文件管理数据卷相关的数据。因此你可能根本不需要知道数据卷文件在宿主机上的存储位置(事实上抱着刨根问底的精神我们还是很想搞清楚它背后的工作原理！)。

docker 数据卷的本质是容器中的一个特殊目录。在容器创建的过程中，docker 会将宿主机上的指定目录(一个以数据卷 ID 为名称的目录)挂载到容器中指定的目录上。这里使用的挂载方式为绑定挂载(bind mount)，所以挂载完成后的宿主机目录和容器内的目标目录表现一致。
比如我们执行下面的命令创建数据卷 hello，并挂载到容器 testcon 的 /world 目录：

```bash
$ docker volume create hello
$ docker run -id --name testcon --mount type=volume,source=hello,target=/world ubuntu /bin/bash
```

实际上在容器的创建过程中，类似于在容器中执行了下面的代码：

```bash
// 将数据卷 hello 在宿主机上的目录绑定挂载到 rootfs 中指定的挂载点 /world 上
mount("/var/lib/docker/volumes/hello/_data", "rootfs/world", "none", MS_BIND, NULL)
```

在处理完所有的 mount 操作之后(真正需要 docker 容器挂载的除了数据卷目录还包括 rootfs，init-layer 里的内容，/proc 设备等)，docker 只需要通过 chdir 和 pivot_root 切换进程的根目录到 rootfs 中，这样容器内部进程就只能看见以 rootfs 为根的文件系统以及被 mount 到 rootfs 之下的各项目录了。例如我们启动的 testcon 中的文件系统为：

![docker-data-volume-5](/images/docker-data-volume-5.png)

下面我们介绍几个数据卷在使用中比较常见的问题。




## 数据的覆盖问题

- 如果挂载一个空的数据卷到容器中的一个非空目录中，那么这个目录下的文件会被复制到数据卷中。

- 如果挂载一个非空的数据卷到容器中的一个目录中，那么容器中的目录中会显示数据卷中的数据。如果原来容器中的目录中有数据，那么这些原始数据会被隐藏掉。

这两个规则都非常重要，灵活利用第一个规则可以帮助我们初始化数据卷中的内容。掌握第二个规则可以保证挂载数据卷后的数据总是你期望的结果。




## 在 Dockerfile 中添加数据卷

在 Dockerfile 中我们可以使用 VOLUME 指令向容器添加数据卷：
```bash
VOLUME /data
```
在使用 docker build 命令生成镜像并且以该镜像启动容器时会挂载一个数据卷到 /data 目录。根据我们已知的数据覆盖规则，如果镜像中存在 /data 目录，这个目录中的内容将全部被复制到宿主机中对应的目录中，并且根据容器中的文件设置合适的权限和所有者。
**注意，VOLUME 指令不能挂载主机中指定的目录。这是为了保证 Dockerfile 的可一致性，因为不能保证所有的宿主机都有对应的目录。**
在实际的使用中，这里还有一个陷阱需要大家注意：在 Dockerfile 中使用 VOLUME 指令之后的代码，如果尝试对这个数据卷进行修改，这些修改都**不会生效**！下面是一个这样的例子：

```bash
FROM ubuntu
RUN useradd nick
VOLUME /data
RUN touch /data/test.txt
RUN chown -R nick:nick /data
```

通过这个 Dockerfile 创建镜像并启动容器后，该容器中存在用户 nick，并且能够看到 /data 目录挂载的数据卷。但是 /data 目录内并没有文件 test.txt，更别说 test.txt 文件的所有者属性了。要解释这个现象需要我们了解通过 Dockerfile 创建镜像的过程：
Dockerfile 中除了 FROM 指令的每一行都是基于上一行生成的临时镜像运行一个容器，执行一条指令并执行类似 docker commit 的命令得到一个新的镜像。这条类似 docker commit 的命令不会对挂载的数据卷进行保存。
所以上面的 Dockerfile 最后两行执行时，都会在一个临时的容器上挂载 /data，并对这个临时的数据卷进行操作，但是这一行指令执行并提交后，这个临时的数据卷并没有被保存。因而我们最终通过镜像创建的容器所挂载的数据卷是没有被最后两条指令操作过的。我们姑且叫它 "Dockerfile 中数据卷的初始化问题"。

下面的写法可以解决 Dockerfile 中数据卷的初始化问题：

```bash
FROM ubuntu
RUN useradd nick
RUN mkdir /data && touch /data/test.txt
RUN chown -R nick:nick /data
VOLUME /data
```

通过这个 Dockerfile 创建镜像并启动容器后，数据卷的初始化是符合预期的。这是由于在挂载数据卷时，/data 已经存在，/data 中的文件以及它们的权限和所有者设置会被复制到数据卷中。
还有另外一种方法可以解决 Dockerfile 中数据卷的初始化问题。就是利用 CMD 指令和 ENTRYPOINT 指令的执行特点：与 RUN 指令在镜像构建过程中执行不同，CMD 指令和 ENTRYPOINT 指令是在容器启动时执行。因此使用下面的 Dockerfile 也可以达到对数据卷的初始化目的：

```bash
FROM ubuntu
RUN useradd nick
VOLUME /data
CMD touch /data/test.txt && chown -R nick:nick /data && /bin/bash
```



## 总结

数据卷解决了用户数据的持久化问题，能够让用户在容器中产生的数据超出容器自身的生命周期。因此对于容器技术来说掌握数据卷的使用非常必要。希望本文能够帮助您理解数据卷相关的内容。
