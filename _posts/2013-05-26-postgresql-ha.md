---
layout: post
title: "Postgresql HA"
description: ""
category: 
tags: [postgresql]
---
{% include JB/setup %}

# 概念

## Write Ahead Log
* Why? [Write Ahead Log]
    减少磁盘IO
    做PITR
* Location: $PGDATA/pg_xlog
* Size: 16MB determinated at compile time
* rotation

## check_point
'''
checkpoint_segments = 3 # default
checkpoint_timeout = 5 # minute default
checkpoint_completion_target = 
'''

## PITR & Logging Ship (warming standby)
* 参数 master postgresql.conf
```ini
#------------------------------------------------------------------------------
# LOG SHIPPING REPLICATION OPTIONS
#------------------------------------------------------------------------------
archive_mode = on # ArchPG
archive_command = '/usr/bin/omnipitr-archive -l /var/log/qfare_omnipitr/omnipitr-^Y^m^d.log -s /var/run/qfare_omnipitr
-dr gzip=rsync://l-interdb3.f.cn1.qunar.com/slave_PITR/ -dr gzip=rsync://l-interdb7.f.cn1.qunar.com/slave_PITR/ -dr gzip=rsync://l-interdb4.f.cn1.qunar.com/slave_PITR/ -db /var/run/qfare_omnipitr/dstbackup --pid-file /var/run/qfare_omnipitr/omnipitr.pid -v "%p"'
archive_timeout
```
* 如果一个从挂了，会是什么状况

## Streaming ([Hot Standby])
* [Hot Standby Configuration]
```
wal_keep_segments =   # This sets only the minimum number of segments retained in pg_xlog,  the system might need to retain more segments for WAL archival or to recover from a checkpoint
```

## Time Line
* 什么是[Time Line]
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


## 关键参数

* full_page
* 



# 监控
* Streaming Process & Reciving Process
* RealTime Data Write



# 流程
* 打包

# 可能遇到的问题

## streaming链接未建立

## omnipitr出错

## 无法链接从库

## 锁

# 其他



[Binary Replication Tools]: http://wiki.postgresql.org/wiki/Binary_Replication_Tools
[Binary Replication Tutorial]: http://wiki.postgresql.org/wiki/Binary_Replication_Tutorial
[Time Line]: http://www.postgresql.org/docs/9.2/static/continuous-archiving.html
[Write Ahead Log]: http://www.postgresql.org/docs/9.2/static/wal-intro.html
[Hot Standby Configuration]: http://www.postgresql.org/docs/9.2/static/runtime-config-replication.html#GUC-HOT-STANDBY
[Hot Standby]: www.postgresql.org/docs/9.2/static/hot-standby.html
