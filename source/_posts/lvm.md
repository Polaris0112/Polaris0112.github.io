---
title: LVM创建/扩容/常用操作
date: 2018-02-20
categories: LVM使用
tags: 
- centos
- lvm

---
本帖子记录的是LVM初始化和扩容等操作

## 相关概念

- LVM是逻辑盘卷管理（Logical Volume Manager）的简称，它是Linux环境下对磁盘分区进行管理的一种机制，LVM是建立在硬盘和分区之上的一个逻辑层，来提高磁盘分区管理的灵活性。LVM是在磁盘分区和文件系统之间添加的一个逻辑层，来为文件系统屏蔽下层磁盘分区布局，提供一个抽象的盘卷，在盘卷上建立文件系统。物理卷（physical volume）物理卷就是指硬盘分区或从逻辑上与磁盘分区具有同样功能的设备（如RAID），是LVM的基本存储逻辑块，但和基本的物理存储介质（如分区、磁盘等）比较，却包含有与LVM相关的管理参数。

- 物理卷：Physical Volume，简称PV，一个物理卷只不过是一个有LVM管理数据添加在里面的物理存储介质。要使用LVM系统，首先对要用于LVM的磁盘进行初始化，初始化的目的就是将磁盘或分区标识为LVM 的物理卷。使用pvcreate 命令可以将一个磁盘标记为 LVM 物理卷。

- 物理分区：Physical Extents，简称PE，LVM将每个物理卷分别叫做物理分区的可寻址存储单元，存储单元的大小通常为几MB。磁盘的开头部分为LVM元数据，之后从索引为零开始，每个物理分区的索引依次递增一，按顺序进行分配。

- 卷组：Volume Group，简称VG，物理卷可以组织为卷组。卷组可以由一个或多个物理卷组成，同时系统中可以有多个卷组。创建了卷组之后，该卷组（而不是磁盘）便是表示数据存储的实体。因此，尽管以前是将磁盘从一个系统移动到另一个系统，使用了 LVM 之后，会将卷组从一个系统移动到另一个系统。出于这种原因，通常在一个系统上创建多个卷组会比较方便。

- 逻辑分区：Logical Extents，简称LE，逻辑卷的基本分配单元称为逻辑分区。逻辑分区映射到物理分区，因此，如果物理分区的尺寸小为4MB，那么逻辑分区的尺寸也将为4MB。逻辑卷的大小取决于所分配的逻辑分区数量。

![lvm-concept](/images/lvm-concept.png)


## 运行环境

- CentOS 7/RHEL 7

建议在安装系统的时候就选择好LVM分区，不然已经有数据的话再选择LVM需要格式化。


## 对新增加的硬盘进行分区、格式化 

``` bash
## 在CentOS/RHEL 7中使用

$ fdisk /dev/sda　　　 
## 继续输入命令： 
## p　　　　　　　查看已分区数量
## n　　　　　　　新增加一个分区 
## p　　　　　　　分区类型我们选择为主分区 
##　　　　　　分区号按照默认选择（已经存在的会顺着按下一个分区号生成）
## 回车　　　　　　默认（起始扇区） 
## 回车　　　　　　默认（结束扇区） 
## t　　　　　　　修改分区类型 
##　　　　　　选分区3 
## 8e　　　　　　修改为LVM（8e就是LVM） 
## w　　　　　　写分区表 
## q　　　　　　完成，退出fdisk命令 
##　　系统提示重启。

## 开机后，格式化，等待一会儿： 
$ mkfs.ext3 /dev/sda3

```

## 添加新LVM到已有的LVM组，实现扩容

``` bash
$ pvcreate /dev/sda3　　　                              #创建pv

$ pvdisplay                                             #查看创建好的pv信息

$ vgdisplay                                             #查看vg信息，记录需要扩容的vg名称

$ vgcreate vg_test /dev/sdb3                            #没有则创建

$ vgextend vg_test /dev/sda3                            #将初始化过的分区加入到虚拟卷组vg_test

$ lvextend -l +20G /dev/mapper/vg_test-root             #扩展已有卷的容量

$ lvextend -l +100%FREE /dev/mapper/vg_test-root        #把所有空余的空间全部加入到卷组

$ lvcreate -L 15G -n lv_test vg_test                    #若初始化，则创建LV

## CentOS7使用以下命令,进行文件系统的真正扩容
$ xfs_growfs /dev/mapper/vg_test-root

## CentOS6使用以下命令,进行文件系统的真正扩容
$ resize2fs /dev/mapper/vg_test-root

## 查看新的磁盘空间
$ df -h

```



## LVM常用命令

- lvremove : 移除逻辑卷
```bash
$ lvremove /dev/vgname/lvname     
```

如果该lv已经mount了，lvremove不能使用，必须unmount后才可以使用。

- lvcreate ：创建逻辑卷
```bash
$ lvcreate --name vol_01  -L  10G vgname
```

-L ：指定逻辑卷大小，单位可以是"kKmMgGtT"

- lvextend -L +10G  /dev/vgname/lvname

给lvname添加10G的容量

另外一种写法：
```bash
$ lvcreate --size 40GB --name vol vgname
```

- vgcreate：创建卷组vg
```bash
$ vgcreate vg-test /dev/sdb1 /dev/sdb2
```

- vgremove：强制删除vg，及时vg下有lv，也会被删除
```bash
$ vgremove vgname -f 
```

- vgreduce ：移除物理卷
```bash
$ my_volume_group /dev/hda1

$ vgreduce --removemissing  vgname 
```

- vgextend ：扩容物理卷
```bash
$ vgextend vgname   /dev/sdb1(物理卷位置)
```



## 其他

为了系统启动是自动加载文件系统，需要在 /etc/fstab 添加

```bash
/dev/vg_test/lv_test   /opt/test ext3 defaults 1 2
```

有时可能因为逻辑卷为打开而无法启动系统

以下是系统启动LVM的过程图：

![lvm-run](/images/lvm-run.png)




## 自动化脚本例子

``` shell
#!/bin/bash
# Usage: Automatic expand lv with LVM managed disk
#  Setp 1: Add Hard Disk or Storage to Computing unit
#   Setp 2: Execute this script with root privilege
#   Setp 3: Mind info of this script execution result
 
 
# Open the refrigerator door, get the shell script execution environment ready
# Put the elephant into the refrigerator, how the shell scripts works
# Close the refrigerator door, check out the result of execution
# Simetimes, we have to pull new elephant or elephant dung out here, unset variables of shell script
 
function check_execution_result(){
        if [[ ! -z $RETVAL ]]; then
                unset RETVAL
        fi
        RETVAL=$?
        if [[ $RETVAL -ne 0 ]]; then
                echo execution failed! 
                exit $RETVAL
        else
                echo execution successfully! 
        fi
        unset RETVAL
}
 
# lsblk --scsi
# lsblk --all
# NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# fd0               2:0    1    4K  0 disk 
# sda               8:0    0   40G  0 disk 
# ├─sda1            8:1    0  500M  0 part /boot
# └─sda2            8:2    0 39.5G  0 part 
#   ├─centos-swap 253:0    0  3.9G  0 lvm  [SWAP]
#   └─centos-root 253:1    0 35.6G  0 lvm  /
# sdb               8:16   0   16G  0 disk 
# sr0              11:0    1  6.6G  0 rom  
 
# Show present scsi disk online
# Q: Why use "xargs" here?
# A: Convert the text from multi-line single-column into single-line multi-column, for sed operation
ONLINE_SCSI_DISK_PRESENT=$(lsblk --all | grep disk | grep -v fd | awk '{print $1}' | xargs)
# TODO
# For execution this script beyond twice
ONLINE_SCSI_DISK_PRESENT=sda
# Find new scsi disk online
# TODO figure it out why there is host0? 
echo "- - -" >/sys/class/scsi_host/host0/scan
echo "- - -" >/sys/class/scsi_host/host1/scan
echo "- - -" >/sys/class/scsi_host/host2/scan
# Show new added scsi disk online
ONLINE_SCSI_DISK_NEWADD=$(lsblk --all | grep disk | grep -v fd | awk '{print $1}' | xargs echo | sed "s/$ONLINE_SCSI_DISK_PRESENT//g")
# Construct disk file with full path
echo New Added SCSI Disk: $ONLINE_SCSI_DISK_NEWADD
# Get VG Name
VG_Name=$(vgdisplay | grep 'VG Name' | awk '{print $NF}')
VG_PATH_TO_EXTEND=$(lvdisplay | grep 'LV Path' | awk '{print $NF}' | grep root)
for BLOCK in $ONLINE_SCSI_DISK_NEWADD; do
    ONLINE_SCSI_DISK_NEWADD_FILENAME="/dev/"$BLOCK
    # end-of-file contents and eof mark must start row1
    fdisk $ONLINE_SCSI_DISK_NEWADD_FILENAME >/dev/null 2>&1<<eof
n
p
1
 
 
 
t
8e
w
eof
    check_execution_result
    LVM_OPERATION_DISK_FILENAME=$ONLINE_SCSI_DISK_NEWADD_FILENAME"1"
    pvcreate $LVM_OPERATION_DISK_FILENAME >/dev/null 2>&1
    check_execution_result
    vgextend $VG_Name $LVM_OPERATION_DISK_FILENAME >/dev/null 2>&1
    check_execution_result
    lvresize -l +100%FREE $VG_PATH_TO_EXTEND >/dev/null 2>&1
    check_execution_result
 
# resize2fs - ext2/ext3/ext4 file system resizer
# xfs_growfs, xfs_info - expand an XFS filesystem
 
#[root@hlc7172009 ~]# resize2fs /dev/mapper/centos-root
#resize2fs 1.42.9 (28-Dec-2013)
#resize2fs: Bad magic number in super-block while trying to open /dev/mapper/centos-root
#Couldn't find valid filesystem superblock.
#[root@hlc7172009 ~]#
 
#[root@hlc7172009 ~]# xfs_growfs $VG_PATH_TO_EXTEND
#meta-data=/dev/mapper/centos-root isize=256    agcount=4, agsize=2334208 blks
#         =                       sectsz=512   attr=2, projid32bit=1
#         =                       crc=0
#data     =                       bsize=4096   blocks=9336832, imaxpct=25
#         =                       sunit=0      swidth=0 blks
#naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
#log      =internal               bsize=4096   blocks=4559, version=2
#         =                       sectsz=512   sunit=0 blks, lazy-count=1
#realtime =none                   extsz=4096   blocks=0, rtextents=0
#data blocks changed from 9336832 to 13530112
#[root@hlc7172009 ~]#
 
    # Check xfs_info if is installed 
    which xfs_info >/dev/null 2>&1
    if [[ $? -ne 0 ]]; then
        yum install xfsprogs -y >/dev/null 2>&1
    fi
    # end Check xfs_info if is installed
 
    # Check VG_PATH_TO_EXTEND if is xfs filesystem
    xfs_info $VG_PATH_TO_EXTEND >/dev/null 2>&1
    if [[ $? -ne 0 ]]; then
        # is not xfs
        VG_PATH_TO_EXTEND_IS_NOT_XFS=0
    else 
        # is xfs
        VG_PATH_TO_EXTEND_IS_NOT_XFS=1
    fi
    # end Check VG_PATH_TO_EXTEND if is xfs filesystem
 
    # TODO CentOS7 default filesystem is xfs, so we can check it out by OS if is CentOS7
 
    if [[ $VG_PATH_TO_EXTEND_IS_NOT_XFS ]]; then
        # is xfs
        xfs_growfs $VG_PATH_TO_EXTEND >/dev/null 2>&1
    else
        # is not xfs
        resize2fs $VG_PATH_TO_EXTEND >/dev/null 2>&1
    fi
    check_execution_result
    df -h
    lsblk --all
done
```


### 效果图

添加磁盘前：

![lvm-before](/images/lvm-before.jpg)


添加磁盘后运行脚本后：

![lvm-after](/images/lvm-after.jpg)








