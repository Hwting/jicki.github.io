---
layout: post
title: kubernetes jenkins
categories: kubernetes
description: kubernetes jenkins
keywords: kubernetes
---


# 基于 Kubernetes Jenkins slave 动态资源


## 环境说明

> 请自动配置 kubernetes 集群, kubernetes 版本为 1.9.1

```
172.16.1.64 - kubernetes master
172.16.1.65 - kubernetes master or node
172.16.1.66 - kubernetes node
```


## 先决条件


> 后端存储, 用于存放 jenkins home 目录，以及 slave 构建依赖包目录


```
# 配置jenkins 后端存储, 如 nfs, gfs, cephfs 等， 请自行配置，或者参考 之前的文章。

# 基于单独的 nfs 最为后端 文档在这里 https://jicki.me/2017/01/01/kubernetes-nfs/ 

```


## 创建 jenkins master



```
# 首先创建一个 namespace

vi jenkins-namespace.yaml


apiVersion: v1
kind: Namespace
metadata:
  name: jenkins

```

```
# 导入文件

kubectl apply -f jenkins-namespace.yaml 
namespace "jenkins" created


# 查看

[root@kubernetes-64 jenkins]# kubectl get namespace
NAME            STATUS    AGE
default         Active    26d
ingress-nginx   Active    15d
jenkins         Active    23s

```




```
# 创建 jenkins 的 rbac , jenkins 与 api-service 通讯必须要 rabc 认证


vi jenkins-rbac.yaml

---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: jenkins
  name: jenkins-admin
  namespace: jenkins

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
  namespace: jenkins
  labels:
    k8s-app: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: jenkins


```


```
# 导入文件

[root@kubernetes-64 jenkins]# kubectl apply -f jenkins-rbac.yaml 
serviceaccount "jenkins-admin" created
clusterrolebinding "jenkins-admin" created

# 查看
[root@kubernetes-64 jenkins]# kubectl get serviceaccount -n jenkins
NAME            SECRETS   AGE
default         1         3h
jenkins-admin   1         1m


```





> 这里需要注意 nfs 目录的权限, 官方镜像提示 jenkins user - uid 1000 ，我们需要在 nfs 源目录授权 1000 这个uid的权限，否则创建失败


```
vi jenkins-deployment.yaml


apiVersion: apps/v1beta1
kind: Deployment  
metadata:  
  name: jenkins
  namespace: jenkins
spec:  
  replicas: 1  
  strategy:  
    type: RollingUpdate  
    rollingUpdate:  
      maxSurge: 2  
      maxUnavailable: 0  
  template:  
    metadata:  
      labels:  
        app: jenkins  
    spec:
      serviceAccount: "jenkins-admin"
      containers:  
      - name: jenkins
        image: jenkins/jenkins
        imagePullPolicy: IfNotPresent  
        ports:  
        - containerPort: 8080  
          name: web  
          protocol: TCP  
        - containerPort: 50000  
          name: agent  
          protocol: TCP  
        volumeMounts:  
        - name: jenkinshome
          mountPath: /var/jenkins_home
        env:  
        - name: JAVA_OPTS  
          value: "-Xms512m -Xmx1024m -XX:PermSize=512m -XX:MaxPermSize=1024m -Duser.timezone=Asia/Shanghai"
        - name: TRY_UPGRADE_IF_NO_MARKER
          value: "true"
      volumes:  
      - name: jenkinshome
        nfs:  
          server: 172.16.1.66
          path: "/opt/data/jenkins"


---
kind: Service  
apiVersion: v1  
metadata:  
  labels:  
      app: jenkins  
  name: jenkins
  namespace: jenkins
spec:  
  ports:  
  - port: 8080  
    targetPort: 8080  
    name: web  
  - port: 50000  
    targetPort: 50000  
    name: agent  
  selector:
    app: jenkins
    
    
---
apiVersion: extensions/v1beta1  
kind: Ingress  
metadata:  
  name: jenkins
  namespace: jenkins
spec:  
  rules:  
  - host: jenkins.jicki.me
    http:  
      paths:  
      - path: /  
        backend:  
          serviceName: jenkins  
          servicePort: 8080


```


```
# 导入 文件

[root@kubernetes-64 jenkins]# kubectl apply -f jenkins-deployment.yaml 
deployment "jenkins" created
service "jenkins" created
ingress "jenkins" created



# 查看 服务

[root@kubernetes-64 jenkins]# kubectl get pods -n jenkins
NAME                     READY     STATUS    RESTARTS   AGE
jenkins-c5f45c84-8vlcp   1/1       Running   0          6m
[root@kubernetes-64 jenkins]# 
[root@kubernetes-64 jenkins]# kubectl get svc -n jenkins
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
jenkins   ClusterIP   10.254.47.35   <none>        8080/TCP,50000/TCP   6m
[root@kubernetes-64 jenkins]# 
[root@kubernetes-64 jenkins]# kubectl get ingress -n jenkins   
NAME      HOSTS              ADDRESS   PORTS     AGE
jenkins   jenkins.jicki.me             80        6m



# 查看 jenkins 初始密码

[root@kubernetes-64 jenkins]# kubectl logs pods/jenkins-c5f45c84-8vlcp -n jenkins


*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

b2458cfb46724268a4db025c54f172ab

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
```


## 配置 jenkins


```
# 登录 web ui

http://jenkins.jicki.me

```

![jenkins1][1]
![jenkins2][2]



```
# Master 需要安装的插件

1. Pipeline
2. kubernetes plugin 
3. Build WIth Parameters    (构建 输入参数的插件)
4. Git Parameter Plug-In  ( git 分支 构建选择)


```


![jenkins3][3]


## 配置 kubernetes 云


```
# 点击【系统管理】-【系统设置】-【新增一个云】-【Kubernetes】
```

![jenkins4][4]

![jenkins5][5]



```
# 创建一个基于 pipeline 的配置  cloud 字段配置为 kubernetes 云的 name .


podTemplate(label: 'jenkins-pod', cloud: 'Jenkins-Cloud') {
    node('jenkins-pod') {
        stage('Run shell') {
            sh 'echo hello world'
        }
    }
}

```


```
# 点击立即构建, 控制台输出如下:


Started by user 小炒肉
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] podTemplate
[Pipeline] {
[Pipeline] node
Still waiting to schedule task
jenkins-slave-bkcfk-vpzz0 is offline
Running on jenkins-slave-bkcfk-vpzz0 in /home/jenkins/workspace/test-k8s
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Run shell)
[Pipeline] sh
[test-k8s] Running shell script
+ echo hello world
hello world
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] }
[Pipeline] // podTemplate
[Pipeline] End of Pipeline
Finished: SUCCESS

```


## 制作 slave 镜像

> 官方默认的 jenkins-slave 里面不存在任何的工具，依赖，项目环境等，我们需要制作一个属于自己项目的 jenkins-slave 镜像。

> 官方 jenkins-slave dockerfile 地址 https://github.com/jenkinsci/docker-jnlp-slave 


  [1]: http://jicki.me/images/posts/jenkins/jenkins1.png
  [2]: http://jicki.me/images/posts/jenkins/jenkins2.png
  [3]: http://jicki.me/images/posts/jenkins/jenkins3.png
  [4]: http://jicki.me/images/posts/jenkins/jenkins4.png
  [5]: http://jicki.me/images/posts/jenkins/jenkins5.png
