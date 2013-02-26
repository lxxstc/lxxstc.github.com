---
layout: post
title: "使用Pg_reorg进行数据库空间整理"
description: ""
category: tech
tags: [postgresql]
---
{% include JB/setup %}

## pg_reorg是什么
-----
- pg_repack is a PostgreSQL extension which lets you remove bloat from tables and indexes, and optionally restore the physical order of clustered indexes. Unlike CLUSTER and VACUUM FULL it works online, without holding an exclusive lock on the processed tables during processing. pg_repack is efficient to boot, with performance comparable to using CLUSTER directly. [详情见][pg_reorg_src]
- pg_repack 是 pg_reorg 在github上的 Repository.改了一个名字
- [使用文档][reorg_tutorial]
- pg_reorg换了个名字，在github上叫pg_repack

## 效果如何
-----
- Instagram说，他们用得很爽
- 我#TODO

## pg_reorg 原理
-----
 pg_reorg 做到碎片整理的三部曲:
1. 先在表上获取一个[exclusive lock][exclusive_lock]
2. 创建一个临时表，收集原始表的改变. 在原始表加一个trigger，将所有变化复制到临时表上
3. `CREATE TABLE`用`SELECT FROM…ORDER BY`, 这会在磁盘上创建一个根据索引顺序存储的新表
4. 从临时表中将在`SELECT FROM`开始后发生的所有改变，同步到新表上
5. 切换到新表上FROM

- [详情见instagram原文][pg_reorg_from_instagram]

## 遇到的问题
-----
> 当存在没有提交的transaction时，pg_reorg会等在那里，直到所有的事务全部提交.下面是pg_reorg中查询事务的SQL语句

    # SELECT * FROM pg_locks WHERE locktype = 'virtualxid' AND pid <> pg_backend_pid() and virtualtransaction = ANY($1);
> 而LOG中将输出`NOTICE: Waiting for 4 transactions to finish. First PID: 11987`.
> 只有等所有的transactions都退出，pg_reorg才会进行后面的操作，并退出。否则将会陷入死循环.
> 我们可以根据`First PID`查看到底是哪个进程，为什么没有完成事务.并处理掉相应的进程.
> 下面是一些查看方法:
    $ ps auxxwwww | grep 11987
    $ host xxx.xxx.xxx.xxx

* 注：什么是[virtualxid]

## 编译和安装
-----
### Ubuntu下安装依赖

    $ sudo apt-get install libedit-dev
    $ sudo apt-get install libpam0g-dev



### 编译和安装
    $ git clone https://github.com/reorg/pg_repack.git
    $ cd pg_repack
    $ make && sudo make install

### 在Postgresql库中安装pg_repack扩展
    # CREATE EXTENSION pg_repack;

### pg_reorg在CentOS上构建RPM;和安装
[pg91_reorg.spec]

    $ wget http://pgfoundry.org/frs/download.php/3395/pg_reorg-1.1.8.tar.gz -O /tmp/pg_reorg-1.1.8.tar.gz
    $ cp /tmp/pg91_reorg-1.1.8.tar.gz /home/q/xiaoxu.lv/rpmbuild/SOURCES/
    $ rpm -bb pg91_reorg.spec
    $ sudo rpm -ivh pg91_reorg-1.1.8-1.x86_64.rpm

### 在Postgresql库中安装pg_reorg
    $ psql -f $PGSHARE/contrib/pg_reorg.sql -d your_database


## 使用pg_reorg进行数据库空间整理
-----
### Do online VACUUM FULL
    $ pg_repack -U postgres --no-order -t countries -d postgres -E debug
    #或者
    $ pg_reorg -U postgres --no-order -t countries -d postgres -E debug

> `-E debug`为打开debug level的log，这个可以随时观察当前运行的状态.下面是一次完整VACUMM FULL的log

    $ pg_reorg --no-order -U postgres -p 5491 -d test_lxx  -t all_trip_2012_02 -E debug
    LOG: SET statement_timeout = 0
    LOG: SET search_path = pg_catalog, pg_temp, public
    LOG: SET client_min_messages = warning
    LOG: SELECT * FROM reorg.tables WHERE relid = $1::regclass
    DEBUG: ---- reorg_one_table ----
    DEBUG: target_name    : all_trip_2012_02
    DEBUG: target_oid     : 1196120
    DEBUG: target_toast   : 1196131
    DEBUG: target_tidx    : 1196133
    DEBUG: pkid           : 1196137
    DEBUG: ckid           : 0
    DEBUG: create_pktype  : CREATE TYPE reorg.pk_1196120 AS (id integer)
    DEBUG: create_log     : CREATE TABLE reorg.log_1196120 (id bigserial PRIMARY KEY, pk reorg.pk_1196120, row all_trip_2012_02)
    DEBUG: create_trigger : CREATE TRIGGER z_reorg_trigger BEFORE INSERT OR DELETE OR UPDATE ON all_trip_2012_02 FOR EACH ROW EXECUTE PROCEDURE reorg.reorg_trigger('INSERT INTO reorg.log_1196120(pk, row) VALUES( CASE WHEN $1 IS NULL THEN NULL ELSE (ROW($1.id)::reorg.pk_1196120) END, $2)')
    DEBUG: create_table   : CREATE TABLE reorg.table_1196120 WITH (oids=false) TABLESPACE pg_default AS SELECT id,dep,arr,transfer,dep_country,arr_country,transfer_country,airline_designator,source,status FROM ONLY all_trip_2012_02
    DEBUG: drop_columns   : (skipped)
    DEBUG: delete_log     : DELETE FROM reorg.log_1196120
    DEBUG: lock_table     : LOCK TABLE all_trip_2012_02 IN ACCESS EXCLUSIVE MODE
    DEBUG: sql_peek       : SELECT * FROM reorg.log_1196120 ORDER BY id LIMIT $1
    DEBUG: sql_insert     : INSERT INTO reorg.table_1196120 VALUES ($1.*)
    DEBUG: sql_delete     : DELETE FROM reorg.table_1196120 WHERE (id) = ($1.id)
    DEBUG: sql_update     : UPDATE reorg.table_1196120 SET (id, dep, arr, transfer, dep_country, arr_country, transfer_country, airline_designator, source, status) = ($2.id, $2.dep, $2.arr, $2.transfer, $2.dep_country, $2.arr_country, $2.transfer_country, $2.airline_designator, $2.source, $2.status) WHERE (id) = ($1.id)
    DEBUG: sql_pop        : DELETE FROM reorg.log_1196120 WHERE id <= $1
    DEBUG: ---- setup ----
    LOG: BEGIN ISOLATION LEVEL READ COMMITTED
    LOG: SET LOCAL statement_timeout = 100
    LOG: LOCK TABLE all_trip_2012_02 IN ACCESS EXCLUSIVE MODE
    LOG: RESET statement_timeout
    LOG: SELECT reorg.conflicted_triggers($1)
    LOG: CREATE TYPE reorg.pk_1196120 AS (id integer)
    LOG: CREATE TABLE reorg.log_1196120 (id bigserial PRIMARY KEY, pk reorg.pk_1196120, row all_trip_2012_02)
    LOG: CREATE TRIGGER z_reorg_trigger BEFORE INSERT OR DELETE OR UPDATE ON all_trip_2012_02 FOR EACH ROW EXECUTE PROCEDURE reorg.reorg_trigger('INSERT INTO reorg.log_1196120(pk, row) VALUES( CASE WHEN $1 IS NULL THEN NULL ELSE (ROW($1.id)::reorg.pk_1196120) END, $2)')
    LOG: SELECT reorg.disable_autovacuum('reorg.log_1196120')
    LOG: COMMIT
    DEBUG: ---- copy tuples ----
    LOG: BEGIN ISOLATION LEVEL SERIALIZABLE
    LOG: SELECT set_config('work_mem', current_setting('maintenance_work_mem'), true)
    LOG: SET LOCAL synchronize_seqscans = off
    LOG: SELECT reorg.array_accum(virtualtransaction) FROM pg_locks WHERE locktype = 'virtualxid' AND pid <> pg_backend_pid() AND (virtualxid, virtualtransaction) <> ('1/1', '-1/0')
    LOG: DELETE FROM reorg.log_1196120
    LOG: CREATE TABLE reorg.table_1196120 WITH (oids=false) TABLESPACE pg_default AS SELECT id,dep,arr,transfer,dep_country,arr_country,transfer_country,airline_designator,source,status FROM ONLY all_trip_2012_02
    LOG: SELECT reorg.disable_autovacuum('reorg.table_1196120')
    LOG: COMMIT
    DEBUG: ---- create indexes ----
    LOG: SELECT indexrelid, reorg.reorg_indexdef(indexrelid, indrelid), indisvalid, pg_get_indexdef(indexrelid) FROM pg_index WHERE indrelid = $1
    DEBUG: [0]
    DEBUG: target_oid   : 1196137
    DEBUG: create_index : CREATE UNIQUE INDEX index_1196137 ON reorg.table_1196120 USING btree (id)
    LOG: CREATE UNIQUE INDEX index_1196137 ON reorg.table_1196120 USING btree (id)
    DEBUG: [1]
    DEBUG: target_oid   : 1196139
    DEBUG: create_index : CREATE INDEX index_1196139 ON reorg.table_1196120 USING btree (airline_designator)
    LOG: CREATE INDEX index_1196139 ON reorg.table_1196120 USING btree (airline_designator)
    DEBUG: [2]
    DEBUG: target_oid   : 1196140
    DEBUG: create_index : CREATE INDEX index_1196140 ON reorg.table_1196120 USING btree (arr_country)
    LOG: CREATE INDEX index_1196140 ON reorg.table_1196120 USING btree (arr_country)
    DEBUG: [3]
    DEBUG: target_oid   : 1196141
    DEBUG: create_index : CREATE INDEX index_1196141 ON reorg.table_1196120 USING btree (arr)
    LOG: CREATE INDEX index_1196141 ON reorg.table_1196120 USING btree (arr)
    DEBUG: [4]
    DEBUG: target_oid   : 1196142
    DEBUG: create_index : CREATE INDEX index_1196142 ON reorg.table_1196120 USING btree (dep_country)
    LOG: CREATE INDEX index_1196142 ON reorg.table_1196120 USING btree (dep_country)
    DEBUG: [5]
    DEBUG: target_oid   : 1196143
    DEBUG: create_index : CREATE INDEX index_1196143 ON reorg.table_1196120 USING btree (dep)
    LOG: CREATE INDEX index_1196143 ON reorg.table_1196120 USING btree (dep)
    DEBUG: [6]
    DEBUG: target_oid   : 1196144
    DEBUG: create_index : CREATE INDEX index_1196144 ON reorg.table_1196120 USING btree (transfer_country)
    LOG: CREATE INDEX index_1196144 ON reorg.table_1196120 USING btree (transfer_country)
    DEBUG: [7]
    DEBUG: target_oid   : 1196145
    DEBUG: create_index : CREATE INDEX index_1196145 ON reorg.table_1196120 USING btree (transfer)
    LOG: CREATE INDEX index_1196145 ON reorg.table_1196120 USING btree (transfer)
    LOG: SELECT reorg.reorg_apply($1, $2, $3, $4, $5, $6)
    LOG: SELECT pid FROM pg_locks WHERE locktype = 'virtualxid' AND pid <> pg_backend_pid() AND virtualtransaction = ANY($1)
    NOTICE: 0=====
    NOTICE: SELECT pid FROM pg_locks WHERE locktype = 'virtualxid' AND pid <> pg_backend_pid() AND virtualtransaction = ANY($1)
    NOTICE: 423783840
    DEBUG: ---- swap ----
    LOG: BEGIN ISOLATION LEVEL READ COMMITTED
    LOG: SET LOCAL statement_timeout = 100
    LOG: LOCK TABLE all_trip_2012_02 IN ACCESS EXCLUSIVE MODE
    LOG: RESET statement_timeout
    LOG: SELECT reorg.reorg_apply($1, $2, $3, $4, $5, $6)
    LOG: SELECT reorg.reorg_swap($1)
    LOG: COMMIT
    DEBUG: ---- drop ----
    LOG: BEGIN ISOLATION LEVEL READ COMMITTED
    LOG: SELECT reorg.reorg_drop($1)
    LOG: COMMIT
    DEBUG: ---- analyze ----
    LOG: BEGIN ISOLATION LEVEL READ COMMITTED
    LOG: ANALYZE all_trip_2012_02
    LOG: COMMIT


[pg_reorg_src]: https://github.com/reorg/pg_repack
[reorg_tutorial]: http://reorg.projects.pgfoundry.org/pg_reorg.html
[pg91_reorg.spec]: https://gist.github.com/lxxstc/bc3798b410e869b5e8ff
[pg_reorg_from_instagram]: http://instagram-engineering.tumblr.com/post/40781627982/handling-growth-with-postgres-5-tips-from-instagram
[exclusive_lock]: http://www.postgresql.org/docs/current/static/explicit-locking.html
[virtualxid]: http://www.postgresql.org/docs/9.2/static/view-pg-locks.html
