---
title: 分布式存储系统Ceph部署过程
date: 2018-03-14
categories: 分布式存储系统Ceph部署过程记录
tags: 
- Ceph

---
本帖子记录的是分布式存储系统Ceph部署过程记录。

## 环境准备

这里我只是测试环境，所以使用一个监控节点，两个存储节点，具体如下：
 
|ip地址(双网卡)              |        主机名    |       ceph磁盘      |    系统                               |      备注           |
| :----:                     |      :----:      |      :----:         |  :-----:                              |   :-----:           |
|192.168.0.200/172.16.0.100  |      ceph-node1  |     100G            |  CentOS Linux release 7.4.1708 (Core) | 作为mds、mon、osd0  |
|192.168.0.201/172.16.0.101  |      ceph-node2  |     100G            |  CentOS Linux release 7.4.1708 (Core) | 作为osd1            |
|192.168.0.202/172.16.0.102  |      ceph-node3  |     100G            |  CentOS Linux release 7.4.1708 (Core) | 作为osd2            |
|192.168.0.203/172.16.0.103  |      ceph-client |     挂载点：/cephfs |  CentOS Linux release 7.4.1708 (Core) | ceph客户端          |

Ceph的文件系统作为一个目录挂载到客户端cephclient的/cephfs目录下，可以像操作普通目录一样对此目录进行操作。


## 安装前准备(每个节点都需要做以下操作)

分别在ceph的三个节点机（ceph-node1、ceph-node2、ceph-node3）上添加hosts
```bash
$ vi /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.0.200 ceph-node1
192.168.0.201 ceph-node2
192.168.0.202 ceph-node3
192.168.0.203 ceph-client
```


### 关闭防火墙和`selinux`

```bash
$ sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
$ setenforce 0
$ systemctl stop firewalld
$ systemctl disable firewalld
```


### 创建用户`ceph`

分别在ceph的三个节点机（ceph-node1、ceph-node2、ceph-node3）上创建用户ceph，密码统一设置为ceph

```bash
$ adduser ceph
$ echo "ceph"|passwd --stdin ceph
$ echo "ceph ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ceph
$ chmod 0440 /etc/sudoers.d/ceph
```


### 配置`NTP`服务

由于Ceph对时间比较敏感，所以建议使用ntp服务保持三台服务器的时间保持同步

```bash
$ yum install -y ntp
$ vi /etc/ntp.conf
# For more information about this file, see the man pages
# ntp.conf(5), ntp_acc(5), ntp_auth(5), ntp_clock(5), ntp_misc(5), ntp_mon(5).
 
driftfile /var/lib/ntp/drift
 
# Permit time synchronization with our time source, but do not
# permit the source to query or modify the service on this system.
restrict default nomodify notrap nopeer noquery
 
# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 127.0.0.1
restrict ::1
 
# 允许1-4网段的服务器来校时，不允许客户端来修改，登录ntp服务器 
restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap
restrict 192.168.2.0 mask 255.255.255.0 nomodify notrap
restrict 192.168.3.0 mask 255.255.255.0 nomodify notrap
restrict 192.168.4.0 mask 255.255.255.0 nomodify notrap
 
 
# Hosts on local network are less restricted.
#restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap
 
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
 
server time1.aliyun.com
server time2.aliyun.com
server time3.aliyun.com
server time4.aliyun.com
server time5.aliyun.com
server time6.aliyun.com
server time7.aliyun.com
 
#broadcast 192.168.1.255 autokey        # broadcast server
#broadcastclient                        # broadcast client
#broadcast 224.0.1.1 autokey            # multicast server
#multicastclient 224.0.1.1              # multicast client
#manycastserver 239.255.254.254         # manycast server
#manycastclient 239.255.254.254 autokey # manycast client
 
# Enable public key cryptography.
#crypto
 
includefile /etc/ntp/crypto/pw
 
# Key file containing the keys and key identifiers used when operating
# with symmetric key cryptography. 
keys /etc/ntp/keys
 
# Specify the key identifiers which are trusted.
#trustedkey 4 8 42
 
# Specify the key identifier to use with the ntpdc utility.
#requestkey 8
 
# Specify the key identifier to use with the ntpq utility.
#controlkey 8
 
# Enable writing of statistics records.
#statistics clockstats cryptostats loopstats peerstats
 
# Disable the monitoring facility to prevent amplification attacks using ntpdc
# monlist command when default restrict does not include the noquery flag. See
# CVE-2013-5211 for more details.
# Note: Monitoring will not be disabled with the limited restriction flag.
disable monitor

# Enable Logfile
logfile /var/log/ntp.log



$ systemctl start ntpd
$ systemctl enable ntpd 
```



## 安装并配置

### 更换系统源

把系统默认源更换为阿里云源

```bash
$ rm -rf /etc/yum.repos.d/*.repo
$ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
$ wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
$ sed -i '/aliyuncs/d' /etc/yum.repos.d/CentOS-Base.repo
$ sed -i 's/$releasever/7/g' /etc/yum.repos.d/CentOS-Base.repo
$ sed -i '/aliyuncs/d' /etc/yum.repos.d/epel.repo
$ yum clean all
$ yum makecache fast
```



## 安装Ceph-Deploy(主节点/管理节点)

以下操作仅在主节点/管理节点中进行，**不需要所有节点进行以下操作**

```bash
$ yum install http://mirrors.163.com/ceph/rpm-jewel/el7/noarch/ceph-deploy-1.5.38-0.noarch.rpm
$ ceph-deploy --version
1.5.39
```


### 创建`CEPH`集群

```bash
$ sudo -iu ceph
$ mkdir -p ceph-cluster
$ cd /home/ceph/ceph-cluster
$ ceph-deploy new ceph-node-1 ceph-node-2 ceph-node-3
[ceph_deploy.conf][DEBUG ] found configuration file at: /home/ceph/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.37): /usr/bin/ceph-deploy new ceph-node1
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  func                          : <function new at 0x112b500>
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0xf26e18>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  ssh_copykey                   : True
[ceph_deploy.cli][INFO  ]  mon                           : ['ceph-node1','ceph-node2','ceph-node3']
[ceph_deploy.cli][INFO  ]  public_network                : None
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  cluster_network               : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  fsid                          : None
[ceph_deploy.new][DEBUG ] Creating new cluster named ceph
···
···
```

### 编辑`CEPH`配置文件

在`global`下添加以下配置：

```bash
$ vi ceph.conf
[global]
mon_clock_drift_allowed = 5
osd_journal_size = 20480
public_network=172.16.0.0/24
```



### 安装`CEPH`(所有节点都需要安装)

```bash
$ ceph-deploy install --release jewel --repo-url http://mirrors.163.com/ceph/rpm-jewel/el7 --gpg-url http://mirrors.163.com/ceph/keys/release.asc ceph-client ceph-node-1 ceph-node-2 ceph-node-3

## 检查CEPH版本
$ ceph -v
ceph version 10.2.10 (5dc1e4c05cb68dbf62ae6fce3f0700e4654fdbbe)
```


### 初始化MOM节点(主节点/管理节点)

```bash
$ ceph-deploy mon create-initial
···
···
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.client.admin.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mds.keyring
[ceph_deploy.gatherkeys][INFO  ] keyring 'ceph.mon.keyring' already exists
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-osd.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-rgw.keyring
[ceph_deploy.gatherkeys][INFO  ] Destroy temp directory /tmp/tmp8sI0aU
```


### 查看集群状态（在OSD节点上查看）

```bash
$ ceph -s
     cluster 60597e53-ad29-44bd-8dcd-db6aeae6f580
     health HEALTH_ERR
            no osds
     monmap e2: 3 mons at {ceph-node-1=172.16.0.100:6789/0,ceph-node-2=172.16.0.101:6789/0,ceph-node-3=172.16.0.102:6789/0}
            election epoch 6, quorum 0,1,2 ceph-node-1,ceph-node-2,ceph-node-3
     osdmap e1: 0 osds: 0 up, 0 in
            flags sortbitwise,require_jewel_osds
      pgmap v2: 64 pgs, 1 pools, 0 bytes data, 0 objects
            0 kB used, 0 kB / 0 kB avail
                  64 creating
```


### 配置管理节点`ceph-master`

用`ceph-deploy`把配置文件和`admin`密钥拷贝到`Master`节点

```bash
$ ceph-deploy admin ceph-master

## 确保你对ceph.client.admin.keyring有正确的操作权限
$ chmod +r /etc/ceph/ceph.client.admin.keyring
```



### 添加数据节点(在节点上用`root`进行操作)

```bash
[root@ceph-node1 ~]# mkdir /data
[root@ceph-node1 ~]# mkdir /data/osd0
[root@ceph-node1 ~]# chmod -R 777 /data/
 
[root@ceph-node2 ~]# mkdir /data
[root@ceph-node2 ~]# mkdir /data/osd1
[root@ceph-node2 ~]# chmod -R 777 /data/
 
[root@ceph-node3 ~]# mkdir /data
[root@ceph-node3 ~]# mkdir /data/osd2
[root@ceph-node3 ~]# chmod -R 777 /data/
```

分别挂载磁盘

```bash
[root@ceph-node1 ~]# mount /dev/vdb1 /data/osd0
 
[root@ceph-node2 ~]# mount /dev/vdb1 /data/osd1
 
[root@ceph-node3 ~]# mount /dev/vdb1 /data/osd2
```


### 在主节点/管理节点准备OSD

```bash
$ cd /home/ceph/ceph-cluster/
$ ceph-deploy osd prepare ceph-node1:/data/osd0 ceph-node2:/data/osd1 ceph-node3:/data/osd2
······
······
[ceph-node3][INFO  ] Running command: sudo /usr/sbin/ceph-disk -v prepare --cluster ceph --fs-type xfs -- /data/osd2
[ceph-node3][WARNIN] command: Running command: /usr/bin/ceph-osd --cluster=ceph --show-config-value=fsid
[ceph-node3][WARNIN] command: Running command: /usr/bin/ceph-osd --check-allows-journal -i 0 --log-file $run_dir/$cluster-osd-check.log --cluster ceph --setuser ceph --setgroup ceph
[ceph-node3][WARNIN] command: Running command: /usr/bin/ceph-osd --check-wants-journal -i 0 --log-file $run_dir/$cluster-osd-check.log --cluster ceph --setuser ceph --setgroup ceph
[ceph-node3][WARNIN] command: Running command: /usr/bin/ceph-osd --check-needs-journal -i 0 --log-file $run_dir/$cluster-osd-check.log --cluster ceph --setuser ceph --setgroup ceph
[ceph-node3][WARNIN] command: Running command: /usr/bin/ceph-osd --cluster=ceph --show-config-value=osd_journal_size
[ceph-node3][WARNIN] populate_data_path: Preparing osd data dir /data/osd2
[ceph-node3][WARNIN] command: Running command: /sbin/restorecon -R /data/osd2/ceph_fsid.5285.tmp
[ceph-node3][WARNIN] command: Running command: /usr/bin/chown -R ceph:ceph /data/osd2/ceph_fsid.5285.tmp
[ceph-node3][WARNIN] command: Running command: /sbin/restorecon -R /data/osd2/fsid.5285.tmp
[ceph-node3][WARNIN] command: Running command: /usr/bin/chown -R ceph:ceph /data/osd2/fsid.5285.tmp
[ceph-node3][WARNIN] command: Running command: /sbin/restorecon -R /data/osd2/magic.5285.tmp
[ceph-node3][WARNIN] command: Running command: /usr/bin/chown -R ceph:ceph /data/osd2/magic.5285.tmp
[ceph-node3][INFO  ] checking OSD status...
[ceph-node3][DEBUG ] find the location of an executable
[ceph-node3][INFO  ] Running command: sudo /bin/ceph --cluster=ceph osd stat --format=json
[ceph-node3][WARNIN] there are 2 OSDs down
[ceph-node3][WARNIN] there are 2 OSDs out
[ceph_deploy.osd][DEBUG ] Host ceph-node3 is now ready for osd use.
```


### 激活OSD（仍然在主节点/管理节点上操作）

```bash
$ ceph-deploy osd activate ceph-node1:/data/osd0 ceph-node2:/data/osd1 ceph-node3:/data/osd2
······
······
[ceph-node3][WARNIN] start_daemon: Starting ceph osd.4...
[ceph-node3][WARNIN] command_check_call: Running command: /usr/bin/systemctl disable ceph-osd@4
[ceph-node3][WARNIN] command_check_call: Running command: /usr/bin/systemctl disable ceph-osd@4 --runtime
[ceph-node3][WARNIN] command_check_call: Running command: /usr/bin/systemctl enable ceph-osd@4
[ceph-node3][WARNIN] Created symlink from /etc/systemd/system/ceph-osd.target.wants/ceph-osd@4.service to /usr/lib/systemd/system/ceph-osd@.service.
[ceph-node3][WARNIN] command_check_call: Running command: /usr/bin/systemctl start ceph-osd@4
[ceph-node3][INFO  ] checking OSD status...
[ceph-node3][DEBUG ] find the location of an executable
[ceph-node3][INFO  ] Running command: sudo /bin/ceph --cluster=ceph osd stat --format=json
[ceph-node3][WARNIN] there are 2 OSDs down
[ceph-node3][WARNIN] there are 2 OSDs out
[ceph-node3][INFO  ] Running command: sudo systemctl enable ceph.target
```


### 开机挂载磁盘（所有`OSD`节点都要操作）

挂载好的新磁盘会在下一次重启或开机的时候失效，因此需要在每台机上对新磁盘进行挂载，并编辑/etc/fstab文件

```bash
[root@ceph-node1 ~]# cat /etc/fstab
........
/dev/vdb1               /data/osd0                   xfs     defaults        0 0
 
[root@ceph-node2 ~]# cat /etc/fstab
........
/dev/vdb1               /data/osd1                   xfs     defaults        0 0
 
[root@ceph-node3 ~]# cat /etc/fstab
........
/dev/vdb1               /data/osd2                   xfs     defaults        0 0
```


### 分发配置和密钥（主节点/管理节点上执行）

```bash
$ sudo -iu ceph
$ cd /home/ceph/ceph-cluster
$ ceph-deploy admin ceph-node1 ceph-node2 ceph-node3
······
······
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to ceph-node1
[ceph-node1][DEBUG ] connection detected need for sudo
[ceph-node1][DEBUG ] connected to host: ceph-node1 
[ceph-node1][DEBUG ] detect platform information from remote host
[ceph-node1][DEBUG ] detect machine type
[ceph-node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to ceph-node2
ceph@ceph-node2's password: 
[ceph-node2][DEBUG ] connection detected need for sudo
ceph@ceph-node2's password: 
[ceph-node2][DEBUG ] connected to host: ceph-node2 
[ceph-node2][DEBUG ] detect platform information from remote host
[ceph-node2][DEBUG ] detect machine type
[ceph-node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to ceph-node3
ceph@ceph-node3's password: 
[ceph-node3][DEBUG ] connection detected need for sudo
ceph@ceph-node3's password: 
[ceph-node3][DEBUG ] connected to host: ceph-node3 
[ceph-node3][DEBUG ] detect platform information from remote host
[ceph-node3][DEBUG ] detect machine type
[ceph-node3][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
```


### 查看集群状态

```bash
$ ceph health
HEALTH_OK

$ ceph -s
    cluster 12d4f209-69ea-4ad9-9507-b00557b42815
     health HEALTH_OK
     monmap e1: 3 mons at {ceph-node-1=172.16.0.100:6789/0,ceph-node-2=172.16.0.101:6789/0,ceph-node-3=172.16.0.102:6789/0}
            election epoch 2, quorum 0 ceph-node1
     osdmap e14: 3 osds: 3 up, 3 in
      pgmap v26: 64 pgs, 1 pools, 0 bytes data, 0 objects
            15459 MB used, 45944 MB / 61404 MB avail
                  64 active+clean
```



### 重新部署（温馨提示）

**部署过程中如果出现任何奇怪的问题无法解决，可以简单的删除所有节点，一切从头再来：**

```bash
$ ceph-deploy purge ceph-mon1 ceph-mon2 ceph-mon3 ceph-osd1 ceph-osd2
$ ceph-deploy purgedata ceph-mon1 ceph-mon2 ceph-mon3 ceph-osd1 ceph-osd2
$ ceph-deploy forgetkey
```



### 创建文件系统

创建文件系统的步骤参考[官网](http://docs.ceph.com/docs/master/cephfs/createfs/)
对于一个刚创建的MDS服务，虽然服务是运行的，但是它的状态直到创建 pools 以及文件系统的时候才会变为Active.
 
还没有创建时候的状态
```bash
$ ceph mds stat
e1: 0/0/0 up
```

创建管理节点（ceph-node1节点上）

```bash
$ sudo -iu ceph
$ cd /home/ceph/ceph-cluster
$ ceph-deploy mds create ceph-node1 ceph-node2 ceph-node3
······
······
[ceph-node2][INFO  ] Running command: service ceph start mds.ceph-node2
[ceph-node2][DEBUG ] === mds.ceph-node2 === 
[ceph-node2][DEBUG ] Starting Ceph mds.ceph-node2 on ceph-node2...
[ceph-node2][WARNIN] Running as unit ceph-mds.ceph-node2.1496648453.865988091.service.
[ceph-node2][INFO  ] Running command: systemctl enable ceph
[ceph-node2][WARNIN] ceph.service is not a native service, redirecting to /sbin/chkconfig.
[ceph-node2][WARNIN] Executing /sbin/chkconfig ceph on
[ceph_deploy][ERROR ] GenericError: Failed to create 1 MDSs

```

**注意：如果不创建mds管理节点，client客户端将不能正常挂载到ceph集群！！**



### 创建`pool`

```bash
$ ceph osd lspools
0 rbd,
```

新创建的ceph集群只有rdb一个pool。这时需要创建一个新的pool
新建pool，可参考[官网](http://docs.ceph.com/docs/master/rados/operations/pools/)

创建新的pool
```bash
$ ceph osd pool create cephfs_data 32                //用于数据的pool。后面的数字是PG的数量
pool 'cephfs_data' created
$ ceph osd pool create cephfs_metadata 32            //用于元数据的pool
pool 'cephfs_metadata' created

$ ceph fs new myceph  cephfs_metadata cephfs_data
new fs with metadata pool 2 and data pool 1

$ ceph osd lspools
0 rbd,1 cephfs_data,2 cephfs_metadata,

## 检验
$ ceph mds stat
e5: 1/1/1 up {0=ceph-node1=up:active}


$ ceph -s
    cluster 12d4f209-69ea-4ad9-9507-b00557b42815
     health HEALTH_OK
     monmap e1: 1 mons at {ceph-node-1=172.16.0.100:6789/0,ceph-node-2=172.16.0.101:6789/0,ceph-node-3=172.16.0.102:6789/0}
            election epoch 2, quorum 0 ceph-node1
     mdsmap e5: 1/1/1 up {0=ceph-node1=up:active}
     osdmap e19: 3 osds: 3 up, 3 in
      pgmap v37: 84 pgs, 3 pools, 1962 bytes data, 20 objects
            15460 MB used, 45943 MB / 61404 MB avail
                  84 active+clean

$ ceph osd tree
ID WEIGHT  TYPE NAME      UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 0.29279 root default                                     
-2 0.09760     host ceph6                                   
 2 0.09760         osd.2       up  1.00000          1.00000 
-3 0.09760     host ceph7                                   
 3 0.09760         osd.3       up  1.00000          1.00000 
-4 0.09760     host ceph8                                   
 4 0.09760         osd.4       up  1.00000          1.00000 

```


