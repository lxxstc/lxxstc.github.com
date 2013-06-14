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
* PG集群的写入高可用

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
* Size: 16MB determinated at compile time
* rotation
* [Write Ahead Log Configuration]

## check_point

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
* Standby -> Master: touch the trigger file

## Time Line
* 什么是[Time Line]

```
recovery_target_timeline = 'latest' #This feature isn't documented in 9.0 (documentation bug?) but it still
has an effect

```

* 通过 pg_controldata 查看 time line



# 配置文件
## postgresql.conf
## recovery.conf

# 方案

## Streaming vs Logging ship
* Why This?

## 结构
* 
* 


## 工具

###  Omnipitr

* [Binary Replication Tools]

### 集群配置工具 LXX的



# 监控
* Streaming Process & Reciving Process
* RealTime Data Write
* OmniPITR Log



# 流程
* 打包

# 可能遇到的问题

## streaming链接未建立

## omnipitr出错

## psql无法链接从库

## 锁

# 其他



[Binary Replication Tools]: http://wiki.postgresql.org/wiki/Binary_Replication_Tools
[Binary Replication Tutorial]: http://wiki.postgresql.org/wiki/Binary_Replication_Tutorial
[Time Line]: http://www.postgresql.org/docs/9.2/static/continuous-archiving.html
[Write Ahead Log]: http://www.postgresql.org/docs/9.2/static/wal-intro.html
[Write Ahead Log Configuration]: http://www.postgresql.org/docs/9.2/static/wal-configuration.html
[Hot Standby Configuration]: http://www.postgresql.org/docs/9.2/static/runtime-config-replication.html#GUC-HOT-STANDBY
[Hot Standby]: www.postgresql.org/docs/9.2/static/hot-standby.html
