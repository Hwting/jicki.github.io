---
layout: post
title: Shell Scrpit
categories: Linux
description: Shell Scrpit
keywords: Linux
---

> 一些自己折腾过的Shell脚本
> 很弱的脚本



## 截取文件内容

> 利用sed 截取文件中，匹配前后的中间内容


```
#!/bin/bash

curl=http://127.0.0.1/wars


if [ ! -n "$1" ] ;then
    echo "请输入目录名"
else
    echo "$curl/$1"
  wget $curl/$1
  cat $1 |grep war |sed -nr '/href/{s/.*="(.+)">.*/\1/;p}'> war.file
  rm -rf $1
  for i in $(cat war.file)
do
  wget $curl/$1/$i
done
fi
```
