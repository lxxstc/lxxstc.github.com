---
layout: post
title: "使用Pg_reorg进行数据库空间整理"
description: ""
category: tech
tags: [postgresql]
---
{% include JB/setup %}

## pg_reorg是什么
- pg_repack is a PostgreSQL extension which lets you remove bloat from tables and indexes, and optionally restore the physical order of clustered indexes. Unlike CLUSTER and VACUUM FULL it works online, without holding an exclusive lock on the processed tables during processing. pg_repack is efficient to boot, with performance comparable to using CLUSTER directly. [详情见][pg_reorg_src]
- [使用文档][reorg_tutorial]
- pg_reorg换了个名字，在github上叫pg_repack

## 效果如何
- Instagram说，他们用得很爽
- 我#TODO

## pg_reorg 原理
- pg_reorg 做到碎片整理的三部曲:
1. 先在表上获取一个[exclusive lock][exclusive_lock]
2. 创建一个临时表，收集原始表的改变. 在原始表加一个trigger，将所有变化复制到临时表上
3. CREATE TABLE用SELECT FROM…ORDER BY, 这会在磁盘上创建一个根据索引顺序存储的新表
4. 从临时表中将在SELECT FROM开始后发生的所有改变,同步到新表上
5. 切换到新表上

- [详情见instagram原文][pg_reorg_from_instagram]


## 编译和安装
### Ubuntu下安装依赖

    $ sudo apt-get install libedit-dev
    $ sudo apt-get install libpam0g-dev

### CentOS下安装依赖
    $ 
    $ 

### 编译和安装
    $ git clone https://github.com/reorg/pg_repack.git
    $ cd pg_repack
    $ make && sudo make install

### 在Postgresql库中安装扩展
    # CREATE EXTENSION pg_repack;

### 构建RPM
[pg_reorg.spec]


    $


## 使用pg_reorg进行数据库空间整理
### Do online VACUUM FULL
    $ pg_repack -U postgres --no-order -t countries -d postgres


[pg_reorg_src]: https://github.com/reorg/pg_repack
[reorg_tutorial]: http://reorg.projects.pgfoundry.org/pg_reorg.html
[pg_reorg.spec]: https://gist.github.com/lxxstc/bc3798b410e869b5e8ff
[pg_reorg_from_instagram]: http://instagram-engineering.tumblr.com/post/40781627982/handling-growth-with-postgres-5-tips-from-instagram
[exclusive_lock]: http://www.postgresql.org/docs/current/static/explicit-locking.html
