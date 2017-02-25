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







######### $1 ###############

<html>
<head><title>Index of /wars_beta/201702241/</title></head>
<body bgcolor="white">
<h1>Index of /wars_beta/201702241/</h1><hr><pre><a href="../">../</a>
<a href="mo_agent.war">mo_agent.war</a>                                       24-Feb-2017 07:35            97110689
<a href="mo_agent_core.war">mo_agent_core.war</a>                                  24-Feb-2017 07:35            85160737
<a href="mo_boss.war">mo_boss.war</a>                                        24-Feb-2017 09:07           111437764
<a href="mo_page.war">mo_page.war</a>                                        24-Feb-2017 07:45            87313275
<a href="mo_payment.war">mo_payment.war</a>                                     24-Feb-2017 07:45            93228435
<a href="mo_settle.war">mo_settle.war</a>                                      24-Feb-2017 07:35            90557505
</pre><hr></body>
</html>


```
