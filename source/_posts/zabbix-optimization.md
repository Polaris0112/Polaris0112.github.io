---
title: Zabbix性能优化
date: 2018-02-26
categories: Zabbix性能优化
tags: 
- Zabbix
- Optimization

---
本帖子记录的是从多个方面优化Zabbix性能，使其能监控大量服务器。

## 背景

由于计划日益增加的服务器数量，所以需要监控系统的性能可以跟上处理数量庞大的服务器数据。因此需要对Zabbix服务端进行性能优化。

主要优化有几方面：

一、从软件设置层面来看，Zabbix的C-S模式收集数据主要分为被动模式和主动模式，当选择被动模式的时候，是服务端根据主机列表来逐个请求数据，这样的话当客户端数量越来越大，那么服务端接受对应数据的间隔也会越来越长从而导致数据延迟。另一个模式是主动模式，在这个模式下，服务端设置好监控项之后，客户端（配置文件中设置好开启主动模式）就会请求服务端的监控列表，然后客户端会对应这个监控列表上的监控项进行数据收集并主动发送给服务端。这样，服务端只需要提供一个监控列表，就可能让客户端自行提取，服务端不需要等待返回数据，而是客户端直接把数据发回服务端，这样可以大大减少服务端的网络请求的压力。


二、从软件架构层面来看，对于大量的服务器，我们可以使用Zabbix-Proxy这个服务，这个服务本意是监控客户端与服务端在不同内网的情况使用的，不过用上Zabbix-Proxy之后，它也算是一个服务端，只不过它不会有前端界面，它只是负责统一接收客户端数据并发送给服务端，这样的话可以大大减少服务端处理数据的压力。


三、从数据版本和引擎层面来看，本人会建议使用Percona+TokuDB的组合。MySQL 4和5使用默认的MyISAM存储引擎安装每个表。从5.5开始，MySQL已将默认存储引擎从MyISAM更改为InnoDB。MyISAM没有提供事务支持，而InnoDB提供了事务支持。与MyISAM相比，InnoDB提供了许多细微的性能改进，并且在处理潜在的数据丢失时提供了更高的可靠性和安全性。Percona XtraDB 是 InnoDB 存储引擎的增强版，被设计用来更好的使用更新计算机硬件系统的性能，同时还包含有一些在高性能环境下的新特性。XtraDB 存储引擎是完全的向下兼容，在 MariaDB 中，XtraDB 存储引擎被标识为”ENGINE=InnoDB”，这个与 InnoDB 是一样的，所以你可以直接用XtraDB 替换掉 InnoDB 而不会产生任何问题。Percona XtraDB 包含有所有 InnoDB 健壮性，可依赖的 ACID 兼容设计和高级 MVCC 架构。XtraDB 在 InnoDB 的坚实基础上构建，使 XtraDB 具有更多的特性，更好调用，更多的参数指标和更多的扩展。从实践的角度来看，XtraDB 被设计用来在多核心的条件下更有效的使用内存和更加方便，更加可用。新的特性被用来降低 InnoDB 的局限性。性能层面，XtraDB与内置的MySQL 5.1 InnoDB 引擎相比，它每分钟可处理2.7倍的事务。同时，按照官方的介绍，TokuDB 引擎是可扩展的，支持事务 ACID 特性，支持多版本控制(MVCC)，这几点等同 InnoDB 的特性，不过对基于索引的查询做了很好的改进，还提供了支持在线表更改的支持(不是所有字段都支持， 后面再说明)，在磁盘和缓存方面也做了很好的改进。TokuDB 结合 松散树索引(Fractal Tree indexing)，可以应用于高负载的大量写(write-intensive)的场景里。对于这个组合，本人觉得很适合运用在Zabbix这种若有大量数据刷新写入的场景里面使用。


四、从数据库表层面来看，可以把Zabbix中的几个大表进行表分区。Zabbix的大表有：

- history
- history_log
- history_str
- history_text
- history_uint
- trends
- trends_uint



第一个方案在客户端的配置文件找到`ServerActive`并填入服务器IP作为值，重启客户端则开启了主动模式，然后在Web端对应的监控项修改属性，改成主动模式即可

第二个方案可以参考[《Zabbix YUM方式部署》](https://polaris0112.github.io/2018/02/25/zabbix-yum-installation/)最后有提及到Zabbix-Proxy的部署方式



## 安装部署Percona+TokuDB引擎

关于Percona的安装教程可以参考[《Zabbix 编译安装部署》](https://polaris0112.github.io/2018/02/25/zabbix-source-installation/)中的`安装Percona Mysql数据库(比原生Mysql性能更优)`部分，对应安装即可。

不过除了安装主要的Percona数据库以外，还要下载TokuDB插件以及其依赖jemalloc 3.3.0以上版本
``` bash
$ yum install -y jemalloc-devel.x86_64
$ yum install -y Percona-Server-tokudb-57
```

（弃用）关闭Transparent Huge Pages(THP)
Percona安装TokuDB已经默认配置，不需要单独配置
```bash
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
    echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi
```


假设你的Percona已经正常运行（并且设置好root密码），那么可以执行以下命令安装TokuDB插件
```bash
$ ps_tokudb_admin --enable -uroot -p
#password

#然后会出现一些日志信息，有可能会出现以下信息
#PLEASE RESTART MYSQL SERVICE AND RUN THIS SCRIPT AGAIN TO FINISH INSTALLATION!
#按照提示，重启mysql，然后再跑多一次这个命令

$ systemctl restart mysql
$ ps_tokudb_admin --enable -uroot -p
#password

#出现信息
#Installing TokuDB engine...
#INFO: Successfully installed TokuDB engine plugin.
#即为安装成功
```


查看是否已经安装成功
```bash
$ mysql -u root -p
#password

mysql > show engines;
+--------------------+---------+----------------------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                                    | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------------------+--------------+------+------------+
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables                  | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                                         | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                                      | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears)             | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                                         | NO           | NO   | NO         |
| TokuDB             | YES     | Percona TokuDB Storage Engine with Fractal Tree(tm) Technology             | YES          | YES  | YES        |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                                      | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                                     | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Percona-XtraDB, Supports transactions, row-level locking, and foreign keys | YES          | YES  | YES        |
| FEDERATED          | NO      | Federated MySQL storage engine                                             | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------------------+--------------+------+------------+

#见到Engine一列中有TokuDB则是安装成功
```


## 卸载TokuDB方法

```bash
$ ps_tokudb_admin --disable -uroot -p
#password

# 手动卸载插件
$ mysql -u root -p
#password

mysql > UNINSTALL PLUGIN tokudb;
mysql > UNINSTALL PLUGIN tokudb_file_map;
mysql > UNINSTALL PLUGIN tokudb_fractal_tree_info;
mysql > UNINSTALL PLUGIN tokudb_fractal_tree_block_map;
mysql > UNINSTALL PLUGIN tokudb_trx;
mysql > UNINSTALL PLUGIN tokudb_locks;
mysql > UNINSTALL PLUGIN tokudb_lock_waits;
mysql > UNINSTALL PLUGIN tokudb_background_job_status;
```

以上命令跑完即可卸载tokudb插件。


## 修改Zabbix部分表的引擎

这步操作的前提是已经做好Zabbix数据库初始化的步骤，详细可以参考[《Zabbix YUM方式部署》](https://polaris0112.github.io/2018/02/25/zabbix-yum-installation/)或者[《Zabbix 编译安装部署》](https://polaris0112.github.io/2018/02/25/zabbix-source-installation/)

```bash
$ mysql -u zabbix -p zabbix
#zabbix mysql password

mysql > alter table history engine = 'TOKUDB';
mysql > alter table history_log engine = 'TOKUDB';
mysql > alter table history_str engine = 'TOKUDB';
mysql > alter table history_text engine = 'TOKUDB';
mysql > alter table history_uint engine = 'TOKUDB';
mysql > alter table trends engine = 'TOKUDB';
mysql > alter table trends_uint engine = 'TOKUDB';
```

操作完成后，zabbix的数据库大表的引擎将会改用tokudb，性能更佳。


## 把Zabbix数据库中需要经常操作的大表进行表分区

**分表前提**

- 按时间范围分表（字段clock，字段无索引）

- MySQL分区表要求范围字段是唯一索引或主键索引，或者是其中一部分，需要修改前核实clock是否在索引中



### 创建4个存储过程

存储过程1

```bash
DELIMITER $$
CREATE PROCEDURE `partition_create`(SCHEMANAME varchar(64), TABLENAME varchar(64), PARTITIONNAME varchar(64), CLOCK int)
BEGIN
        /*
           SCHEMANAME = The DB schema in which to make changes
           TABLENAME = The table with partitions to potentially delete
           PARTITIONNAME = The name of the partition to create
        */
        /*
           Verify that the partition does not already exist
        */

        DECLARE RETROWS INT;
        SELECT COUNT(1) INTO RETROWS
        FROM information_schema.partitions
        WHERE table_schema = SCHEMANAME AND table_name = TABLENAME AND partition_description >= CLOCK;

        IF RETROWS = 0 THEN
        /*
           1. Print a message indicating that a partition was created.
           2. Create the SQL to create the partition.
           3. Execute the SQL from #2.
        */
        SELECT CONCAT( "partition_create(", SCHEMANAME, ",", TABLENAME, ",", PARTITIONNAME, ",", CLOCK, ")" ) AS msg;
        SET @sql = CONCAT( 'ALTER TABLE ', SCHEMANAME, '.', TABLENAME, ' ADD PARTITION (PARTITION ', PARTITIONNAME, ' VALUES LESS THAN (', CLOCK, '));' );
        PREPARE STMT FROM @sql;
        EXECUTE STMT;
        DEALLOCATE PREPARE STMT;
        END IF;
END$$
DELIMITER ;
```


存储过程2

```bash
DELIMITER $$
CREATE PROCEDURE `partition_drop`(SCHEMANAME VARCHAR(64), TABLENAME VARCHAR(64), DELETE_BELOW_PARTITION_DATE BIGINT)
BEGIN
        /*
           SCHEMANAME = The DB schema in which to make changes
           TABLENAME = The table with partitions to potentially delete
           DELETE_BELOW_PARTITION_DATE = Delete any partitions with names that are dates older than this one (yyyy-mm-dd)
        */
        DECLARE done INT DEFAULT FALSE;
        DECLARE drop_part_name VARCHAR(16);

        /*
           Get a list of all the partitions that are older than the date
           in DELETE_BELOW_PARTITION_DATE.  All partitions are prefixed with
           a "p", so use SUBSTRING TO get rid of that character.
        */
        DECLARE myCursor CURSOR FOR
        SELECT partition_name
        FROM information_schema.partitions
        WHERE table_schema = SCHEMANAME AND table_name = TABLENAME AND CAST(SUBSTRING(partition_name FROM 2) AS UNSIGNED) < DELETE_BELOW_PARTITION_DATE;
        DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

        /*
           Create the basics for when we need to drop the partition.  Also, create
           @drop_partitions to hold a comma-delimited list of all partitions that
           should be deleted.
        */
        SET @alter_header = CONCAT("ALTER TABLE ", SCHEMANAME, ".", TABLENAME, " DROP PARTITION ");
        SET @drop_partitions = "";

        /*
           Start looping through all the partitions that are too old.
        */
        OPEN myCursor;
        read_loop: LOOP
        FETCH myCursor INTO drop_part_name;
        IF done THEN
    LEAVE read_loop;
        END IF;
        SET @drop_partitions = IF(@drop_partitions = "", drop_part_name, CONCAT(@drop_partitions, ",", drop_part_name));
        END LOOP;
        IF @drop_partitions != "" THEN
        /*
           1. Build the SQL to drop all the necessary partitions.
           2. Run the SQL to drop the partitions.
           3. Print out the table partitions that were deleted.
        */
        SET @full_sql = CONCAT(@alter_header, @drop_partitions, ";");
        PREPARE STMT FROM @full_sql;
        EXECUTE STMT;
        DEALLOCATE PREPARE STMT;

        SELECT CONCAT(SCHEMANAME, ".", TABLENAME) AS `table`, @drop_partitions AS `partitions_deleted`;
        ELSE
        /*
           No partitions are being deleted, so print out "N/A" (Not applicable) to indicate
           that no changes were made.
        */
        SELECT CONCAT(SCHEMANAME, ".", TABLENAME) AS `table`, "N/A" AS `partitions_deleted`;
        END IF;
END$$
DELIMITER ;
```


存储过程3

```bash
DELIMITER $$
CREATE PROCEDURE `partition_maintenance`(SCHEMA_NAME VARCHAR(32), TABLE_NAME VARCHAR(32), KEEP_DATA_DAYS INT, HOURLY_INTERVAL INT, CREATE_NEXT_INTERVALS INT)
BEGIN
        DECLARE OLDER_THAN_PARTITION_DATE VARCHAR(16);
        DECLARE PARTITION_NAME VARCHAR(16);
        DECLARE OLD_PARTITION_NAME VARCHAR(16);
        DECLARE LESS_THAN_TIMESTAMP INT;
        DECLARE CUR_TIME INT;

        CALL partition_verify(SCHEMA_NAME, TABLE_NAME, HOURLY_INTERVAL);
        SET CUR_TIME = UNIX_TIMESTAMP(DATE_FORMAT(NOW(), '%Y-%m-%d 00:00:00'));

        SET @__interval = 1;
        create_loop: LOOP
        IF @__interval > CREATE_NEXT_INTERVALS THEN
    LEAVE create_loop;
        END IF;

        SET LESS_THAN_TIMESTAMP = CUR_TIME + (HOURLY_INTERVAL * @__interval * 3600);
        SET PARTITION_NAME = FROM_UNIXTIME(CUR_TIME + HOURLY_INTERVAL * (@__interval - 1) * 3600, 'p%Y%m%d%H00');
        IF(PARTITION_NAME != OLD_PARTITION_NAME) THEN
    CALL partition_create(SCHEMA_NAME, TABLE_NAME, PARTITION_NAME, LESS_THAN_TIMESTAMP);
        END IF;
        SET @__interval=@__interval+1;
        SET OLD_PARTITION_NAME = PARTITION_NAME;
        END LOOP;

        SET OLDER_THAN_PARTITION_DATE=DATE_FORMAT(DATE_SUB(NOW(), INTERVAL KEEP_DATA_DAYS DAY), '%Y%m%d0000');
        CALL partition_drop(SCHEMA_NAME, TABLE_NAME, OLDER_THAN_PARTITION_DATE);

END$$
DELIMITER ;
```


存储过程4

```bash
DELIMITER $$
CREATE PROCEDURE `partition_verify`(SCHEMANAME VARCHAR(64), TABLENAME VARCHAR(64), HOURLYINTERVAL INT(11))
BEGIN
        DECLARE PARTITION_NAME VARCHAR(16);
        DECLARE RETROWS INT(11);
        DECLARE FUTURE_TIMESTAMP TIMESTAMP;

        /*
         * Check if any partitions exist for the given SCHEMANAME.TABLENAME.
         */
        SELECT COUNT(1) INTO RETROWS
        FROM information_schema.partitions
        WHERE table_schema = SCHEMANAME AND table_name = TABLENAME AND partition_name IS NULL;

        /*
         * If partitions do not exist, go ahead and partition the table
         */
        IF RETROWS = 1 THEN
        /*
         * Take the current date at 00:00:00 and add HOURLYINTERVAL to it.  This is the timestamp below which we will store values.
         * We begin partitioning based on the beginning of a day.  This is because we don't want to generate a random partition
         * that won't necessarily fall in line with the desired partition naming (ie: if the hour interval is 24 hours, we could
         * end up creating a partition now named "p201403270600" when all other partitions will be like "p201403280000").
         */
        SET FUTURE_TIMESTAMP = TIMESTAMPADD(HOUR, HOURLYINTERVAL, CONCAT(CURDATE(), " ", '00:00:00'));
        SET PARTITION_NAME = DATE_FORMAT(CURDATE(), 'p%Y%m%d%H00');

        -- Create the partitioning query
        SET @__PARTITION_SQL = CONCAT("ALTER TABLE ", SCHEMANAME, ".", TABLENAME, " PARTITION BY RANGE(`clock`)");
        SET @__PARTITION_SQL = CONCAT(@__PARTITION_SQL, "(PARTITION ", PARTITION_NAME, " VALUES LESS THAN (", UNIX_TIMESTAMP(FUTURE_TIMESTAMP), "));");

        -- Run the partitioning query
        PREPARE STMT FROM @__PARTITION_SQL;
        EXECUTE STMT;
        DEALLOCATE PREPARE STMT;
        END IF;
END$$
DELIMITER ;
```


### 存储过程中有四个功能：

- partition_create - 这将在给定模式中的给定表上创建一个分区。

- partition_drop - 这将删除给定模式中给定表上给定时间戳的分区。

- partition_maintenance - 此功能是用户调用的。它负责解析给定的参数，然后根据需要创建/删除分区。

- partition_verify - 检查给定模式中给定表上是否启用了分区。如果没有启用，它将创建一个单独的分区。


**partition_create**
```bash
程序定义：partition_create（SCHEMANAME varchar（64），TABLENAME varchar（64），PARTITIONNAME varchar（64），CLOCK int）
示例：CALL partition_create（“zabbix”，“history”，“p20131216”，1387267200）;
SCHEMANAME =要在其中进行更改的DB模式
TABLENAME =要在其上创建PARTITIONNAME的表
PARTITIONNAME =要创建的分区的名称
将创建CLOCK = PARTITIONNAME以保存“clock”列值小于此值的值
```


**partition_drop**
```bash
过程定义：partition_drop（SCHEMANAME VARCHAR（64），TABLENAME VARCHAR（64），DELETE_BELOW_PARTITION_DATE VARCHAR（64））
示例：CALL partition_drop（“zabbix”，“history”，“20131216”）;
SCHEMANAME =要在其中进行更改的DB模式
TABLENAME =要删除PARTITIONNAME的表
DELETE_BELOW_PARTITION_DATE =允许的最旧分区日期。所有旧版本的分区将被删除。格式为yyyymmdd。
```


**partition_maintenance**
```bash
过程定义：partition_maintenance（SCHEMA_NAME VARCHAR（32），TABLE_NAME VARCHAR（32），KEEP_DATA_DAYS INT，HOURLY_INTERVAL INT，CREATE_NEXT_INTERVALS INT）
示例：CALL partition_maintenance（'zabbix'，'history'，28，24，14 ）;
SCHEMA_NAME =要在其中进行更改的DB模式
TABLE_NAME =要进行更改的表
KEEP_DATA_DAYS =要保留的分区的最大天数。所有超过此天数的分区将被删除。
HOURLY_INTERVAL =分区之间的小时间隔。例如，每日分区的值为24，小时分区的值为1。
CREATE_NEXT_INTERVALS =提前创建的值的分区数。
```


**partition_verify**
```bash
过程定义：partition_verify（SCHEMANAME VARCHAR（64），TABLENAME VARCHAR（64），HOURLYINTERVAL INT（11））
示例：CALL partition_verify（“zabbix”，“history”）;
SCHEMANAME =要在其中进行更改的DB模式
TABLENAME =要检查分区的表
HOURLY_INTERVAL =分区之间的小时间隔。例如，每日分区的值为24，小时分区的值为1。
```


### 分区表需求

- 每月一个分区（24小时*31约等于720小时）

- 历史保存1年数据（12个月）

- 趋势保存2年数据（24个月）

- 未来周期12个（未来12个月）



单独语句：
```bash
mysql > CALL partition_maintenance('zabbix', 'history', 24, 720, 12);
mysql > CALL partition_maintenance(zabbix, 'trends', 48, 720, 12);
```
解释：创建24个分区，其中未来月份12个，每个周期存储720小时数据



### 添加以下存储过程，用来增加新的分区和删除旧的分区
```bash
DELIMITER $$
CREATE PROCEDURE `partition_maintenance_all`(SCHEMA_NAME VARCHAR(32))
BEGIN
    CALL partition_maintenance(SCHEMA_NAME, 'history', 24, 720, 12);
    CALL partition_maintenance(SCHEMA_NAME, 'history_log', 24, 720, 12);
    CALL partition_maintenance(SCHEMA_NAME, 'history_str', 24, 720, 12);
    CALL partition_maintenance(SCHEMA_NAME, 'history_text', 24, 720, 12);
    CALL partition_maintenance(SCHEMA_NAME, 'history_uint', 24, 720, 12);
    CALL partition_maintenance(SCHEMA_NAME, 'trends', 48, 720, 12);
    CALL partition_maintenance(SCHEMA_NAME, 'trends_uint', 48, 720, 12);
END$$
DELIMITER ;
```


存储过程文件下载：
[partition_call.sql](https://raw.githubusercontent.com/Polaris0112/Ops-Tools/master/zabbix_installation/partition_call.sql)

[partition_all.sql](https://raw.githubusercontent.com/Polaris0112/Ops-Tools/master/zabbix_installation/partition_all.sql)


导入mysql数据库中
```bash
$ mysql -u zabbix -p zabbix < partition_call.sql
#zabbix mysql password

$ mysql -u zabbix -p zabbix < partition_all.sql
#zabbix mysql password
```

直接执行
```bash
$ mysql -u zabbix -p zabbix -e "call partition_maintenance_all('zabbix');"
#zabbix mysql password
```

执行后，观察分区表状态是否建立


### 定时调用此存储过程

用来增加新的分区和删除旧的分区，注意定时调用的间隔不能小于每次创建的未来的分区周期，如上情况，**最少12个月调用一次**

创建调用脚本：
```bash
$ vim /root/shell/create_partition.sh
/opt/tokudb/mysql/bin/mysql -uzabbix -pzabbix zabbix -e "call partition_maintenance_all('zabbix');"
```

crontab -e 加入以下：
```bash
0 3 * */10 * sh /root/shell/create_partition.sh
```


### 关闭housekeeper功能

在网页端的**管理** --> **一般** ，页面右上下拉菜单选择**管家**，然后把开启内部管家的勾选去掉，再点击**更新**。





