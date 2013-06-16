---
layout: post
title: "Postgresql HA"
description: ""
category: 
tags: [postgresql]
---
{% include JB/setup %}

# 提纲
* Postgresql主从配置
* PG集群的写入高可用配置
* 如何监控PG集群健康状态

# 图

* 构件

![PGHASTRUCTURE](http://farm4.staticflickr.com/3834/9037909153_63e40cc6b5.jpg)

* 数据流
![PGHA](http://lxxstc.github.io/images/PGHA.png)

* 主从结构拓补

![PGHATopology](http://farm8.staticflickr.com/7395/9040263240_d620583b25.jpg)

# 概念

## Write Ahead Log
* Why? [Write Ahead Log]
    减少磁盘IO
    做PITR
* Location: $PGDATA/pg_xlog
* Log Segment Size: 16MB determinated at compile time
* rotation
* [Write Ahead Log Configuration]

## check_point
* [Defination][Write Ahead Log Configuration]
Checkpoints are points in the sequence of transactions at which it is guaranteed that the heap and index data files
have been updated with all information written before the checkpoint.
At checkpoint time, all dirty data pages are flushed to disk and a special checkpoint record is written to the log file.
* checkpoint 和 wal log segment的对应关系。1 checkpoint ：n wal log segment 或者 n checkpoint : 1 wal log segment

```
checkpoint_segments = 3 # default
checkpoint_timeout = 5 # minute default
checkpoint_completion_target = 
```


## PITR & Logging Ship (warming standby)
* 参数 master postgresql.conf

```
archive_mode = on # ArchPG
archive_command = '/usr/bin/omnipitr-archive -l /var/log/qfare_omnipitr/omnipitr-^Y^m^d.log -s /var/run/qfare_omnipitr
-dr gzip=rsync://l-interdb3.f.cn1.qunar.com/slave_PITR/ -dr gzip=rsync://l-interdb7.f.cn1.qunar.com/slave_PITR/ -dr gzip=rsync://l-interdb4.f.cn1.qunar.com/slave_PITR/ -db /var/run/qfare_omnipitr/dstbackup --pid-file /var/run/qfare_omnipitr/omnipitr.pid -v "%p"'
archive_timeout
```
* 参数 standby recovery.conf

```
```

* 如果一个从挂了，会是什么状况

## Streaming ([Hot Standby])
* [Hot Standby Configuration]

```
hot_standby = on
wal_level  = 'hot_standby'      # determines how much information is written to the WAL
max_wal_senders = 5             # Set the maximum number of concurrent connections from the stand                                           by servers
wal_keep_segments = 150         # This sets only the minimum number of segments retained in pg_xlog,  the system might need to retain more segments for WAL archival or to recover from a checkpoint
```

## Master and Standby
* Standby：If hot_standby is turned on in postgresql.conf and there is a recovery.conf
* Standby -> Master: 
  * touch the trigger file;
  * pg_ctl promote \[-s\] \[-D datadir\]

## Time Line
* 什么是[Time Line]

```
# config in recovery.conf
recovery_target_timeline = 'latest' #This feature isn't documented in 9.0 (documentation bug?) but it still
has an effect

```

* 通过 pg_controldata 查看 time line

```
$ pg_controldata $PG_DATA
pg_control version number:            922
Catalog version number:               201204301
Database system identifier:           5810270594475932634
Database cluster state:               in production
pg_control last modified:             Fri 14 Jun 2013 09:14:20 PM CST
Latest checkpoint location:           61/F34CE340
Prior checkpoint location:            61/F250F690
Latest checkpoint's REDO location:    61/F2EBD858
Latest checkpoint's TimeLineID:       4
Latest checkpoint's full_page_writes: off
Latest checkpoint's NextXID:          0/5951991
Latest checkpoint's NextOID:          167847
Latest checkpoint's NextMultiXactId:  2198
Latest checkpoint's NextMultiOffset:  4405
Latest checkpoint's oldestXID:        1672
Latest checkpoint's oldestXID's DB:   1
Latest checkpoint's oldestActiveXID:  5951991
Time of latest checkpoint:            Fri 14 Jun 2013 09:11:50 PM CST
Minimum recovery ending location:     0/0
Backup start location:                0/0
Backup end location:                  0/0
End-of-backup record required:        no
Current wal_level setting:            hot_standby
Current max_connections setting:      220
Current max_prepared_xacts setting:   0
Current max_locks_per_xact setting:   64
Maximum data alignment:               8
Database block size:                  8192
Blocks per segment of large relation: 131072
WAL block size:                       8192
Bytes per WAL segment:                16777216
Maximum length of identifiers:        64
Maximum columns in an index:          32
Maximum size of a TOAST chunk:        1996
Date/time type storage:               64-bit integers
Float4 argument passing:              by value
Float8 argument passing:              by value
```

# 配置文件
## postgresql.conf
## recovery.conf
```
standby_mode = 'on'
primary_conninfo = 'host=avdbm.f.qunar.com port=5492 user=postgres'
restore_command = '/usr/bin/omnipitr-restore -l /var/log/avdb_omnipitr/omnipitr-^Y^m^d.log -s gzip=/export/slave_PITR/ -f /var/run/avdb_omnipitr/recovery.stop -r -p /var/run/avdb_omnipitr/removal.stop -sr -v %f %p'
trigger_file = '/export/slave_PITR//failover'
recovery_target_timeline = 'latest'
```

# 方案

## Streaming vs Logging ship
* Why This?
  * streaming 为 hotstandby，为读服务提供负载均衡
  * 在streaming的主从链接断掉（主要是因为wal_keep_segments设置的比较小），Logging ship会接过主从同步的工作。


## 工具

###  [OmniPITR]
* Archiving Tool For Logging Ship
* [Binary Replication Tools]

### Rsync
* Rsync Deamon on Slave Server


### [Check_postgres]
* Monitor Tool
* 提供了主从集群的监控项，我没有使用，这里抛砖引玉

### 集群配置工具 LXX的
* 配置文件生成工具


# 监控

* 监控平台 nagios + nrpe plugin + rpm

* Streaming Process & Reciving Process

```
# on master
$ ps auxw | grep sender
postgres 26829  0.0  0.0 8732256 1700 ?        Ss   May28   1:58 postgres: wal sender process postgres 192.168.24.96(36517) streaming 62/E27C5D80
postgres 26833  0.0  0.0 8732256 1704 ?        Ss   May28   2:00 postgres: wal sender process postgres 192.168.24.50(57856) streaming 62/E27C5D80
postgres 26834  0.0  0.0 8732256 1696 ?        Ss   May28   1:46 postgres: wal sender process postgres 192.168.24.153(57102) streaming 62/E27C5D80

# on slave
$ ps aux | grep rec
postgres  3970  0.0 13.0 8740512 8580476 ?     Ss   May15  21:15 postgres: startup process   recovering 0000000400000062000000E2
40004    10918  0.0  0.0 103280   796 pts/0    S+   17:34   0:00 grep rec
postgres 11243  0.0  0.0 8753228 3416 ?        Ss   May28   8:22 postgres: wal receiver process   streaming 62/E2A43310
```

* RealTime Data Write
  * 建立一张监控表，定时更新表中的数据，查看主从数据是否保持一致

* OmniPITR Log
  * 监控logging ship 是否正常
  * omnipitr创建log的权限是600，所以用非postgres用户没法读，我是改了一下代码，改成644了

* [Check_postgres]


# pg_ha部署流程和操作方法(pg已经生产，但未配置主从或者写高可用)
* 在集群各机器上部署依赖
* 打包
* 部署从(因为会随时切换主从，所有认为所有的机器都是从库，除了当前主库不要出现recovery.conf文件外，其他配置相同)
  * rsyncd
  * postgresql.conf
  * recoery.conf
* 部署主
  * pg_hba
  * postgresql.conf
* 验证
  * 测试各服务是否正常，并查看各个机器的主从角色
* 从库开始备份主库数据
* 主从切换
  * 注意depromoting pg 的timeline是否追上promoting pg的timeline
  * 切换完成，用监控项的 主从实时数据写入脚本 测试一下

* 部署监控

# 可能遇到的问题

## streaming链接未建立

## omnipitr出错

## psql无法链接从库
* 从库还没追上主，查看log

## 锁

# 其他



[Binary Replication Tools]: http://wiki.postgresql.org/wiki/Binary_Replication_Tools
[Binary Replication Tutorial]: http://wiki.postgresql.org/wiki/Binary_Replication_Tutorial
[Time Line]: http://www.postgresql.org/docs/9.2/static/continuous-archiving.html
[Write Ahead Log]: http://www.postgresql.org/docs/9.2/static/wal-intro.html
[Write Ahead Log Configuration]: http://www.postgresql.org/docs/9.2/static/wal-configuration.html
[Hot Standby Configuration]: http://www.postgresql.org/docs/9.2/static/runtime-config-replication.html#GUC-HOT-STANDBY
[Hot Standby]: www.postgresql.org/docs/9.2/static/hot-standby.html
[Check_postgres]: http://bucardo.org/wiki/Check_postgres
[OmniPITR]: https://github.com/omniti-labs/omnipitr
