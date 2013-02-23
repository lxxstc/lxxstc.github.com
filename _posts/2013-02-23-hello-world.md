---
layout: post
title: "Hello World"
description: "Just Test Post"
category: test
tags: [other, jekyll]
---
{% include JB/setup %}

## 装主题
    $ rake theme:install git='https://github.com/jekyllbootstrap/theme-tom.git'

## 切换主题
    $ rake theme:switch name="the-program"

## 加一个POST
    $ rake post title="Hello World"
