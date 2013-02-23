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

## 使用pg_reorg进行数据库空间整理


[pg_reorg_src]: https://github.com/reorg/pg_repack
