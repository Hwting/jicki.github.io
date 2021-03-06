---
layout: post
title: kubernetes Grafana2
categories: kubernetes
description: kubernetes Grafana2
keywords: kubernetes
feature-img: "assets/img/pexels/desk-top.jpeg"
---

> kubernetes 基于 Grafana2 监控

## 下载 yaml 文件

```
mkdir grafana

curl -O https://github.com/jicki/kuberneres/blob/master/grafana/grafana-deployment.yaml

curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/grafana/grafana-service.yaml

curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/grafana/heapster-deployment.yaml

curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/grafana/heapster-service.yaml

curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/grafana/influxdb-deployment.yaml

curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/grafana/influxdb-service.yaml

curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/grafana/monitoring2-namespace.yaml

```

## 编辑 yaml 文件

```
cd grafana

# 这里只需要编辑 influxdb-deployment.yaml

# 这里面的 volume 需要修改为自己的配置

```

## 下载 镜像

```
# 提前下载镜像，因为被墙，你懂得

http://pan.baidu.com/s/1kViIjaR

```



## 导入 yaml 文件

```
kubectl apply -f grafana

```


## 查看导入

```
kubectl get all --namespace=monitoring

NAME                           READY     STATUS    RESTARTS   AGE
po/grafana-434744905-ffzd6     1/1       Running   0          2h
po/heapster-1110581374-93kwc   1/1       Running   0          2h
po/influxdb-149151442-f890s    1/1       Running   0          2h

NAME               CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
svc/grafana2       10.233.17.138   <nodes>       3002:30002/TCP   2h
svc/heapster       10.233.43.80    <none>        80/TCP           2h
svc/influxdb-svc   10.233.39.147   <none>        8086/TCP         2h

NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/grafana    1         1         1            1           2h
deploy/heapster   1         1         1            1           2h
deploy/influxdb   1         1         1            1           2h

NAME                     DESIRED   CURRENT   READY     AGE
rs/grafana-434744905     1         1         1         2h
rs/heapster-1110581374   1         1         1         2h
rs/influxdb-149151442    1         1         1         2h
```


## 测试

```
http://node-ip:30002/

```

![Grafana Dashboard][1]


  [1]: https://jicki.me/images/posts/grafana/1.png

