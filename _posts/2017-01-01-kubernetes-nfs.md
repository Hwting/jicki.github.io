---
layout: post
title: kubernetes nfs
categories: [kubernetess]
description: kubernetes nfs
keywords: kubernetes
---

## 安装 NFS 服务

```
# 安装 NFS

yum -y install nfs-utils rpcbind
```


```
# k8s 所有节点 安装 NFS 客户端

yum -y install nfs-utils
```


## 配置 NFS

```
vi /etc/exports

增加

/opt/data/logs   172.16.1.0/24(rw,sync,no_root_squash)

```

## 启动 NFS

```
systemctl enable rpcbind.service    
systemctl enable nfs-server.service

systemctl start rpcbind.service    
systemctl start nfs-server.service

# 查看启动情况
rpcinfo -p


# 查看配置列表

showmount -e 172.16.1.20
```




## k8s deployment 挂载

```
配置
        volumeMounts:
        - name: logs
          mountPath:  /opt/local/nginx/logs
          readOnly: false
      volumes:
      - name: logs
        nfs:
          server: 172.16.1.20
          path: /opt/data/logs



```
