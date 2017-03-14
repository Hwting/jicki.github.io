---
layout: post
title: kargo kubernetes 1.5.4
categories: docker
description: kargo kubernetes 1.5.4
keywords: docker
---

# 1、初始化环境

## 1.1、环境：

| 节点 |    IP   |  角色  |
|------|---------|--------|
|node-1|10.6.0.52| Master |
|node-2|10.6.0.53| Master |
|node-3|10.6.0.55| Node   |
|node-4|10.6.0.56| Node   |

## 1.2、配置SSH Key 登陆

```
# 确保本机也可以 ssh 连接，否则下面部署失败

ssh-keygen -t rsa -N ""

ssh-copy-id -i /root/.ssh/id_rsa.pub 10.6.0.52

ssh-copy-id -i /root/.ssh/id_rsa.pub 10.6.0.53

ssh-copy-id -i /root/.ssh/id_rsa.pub 10.6.0.55

ssh-copy-id -i /root/.ssh/id_rsa.pub 10.6.0.56

```



# 2、获取 Kargo

> Kargo 官方github
> https://github.com/kubernetes-incubator/kargo


## 2.1、安装基础软件

> Kargo 是基于 ansible 统一部署，所以必须安装 ansible

```
# 安装 centos 额外的yum源
yum install -y epel-release

# 安装 软件
yum install -y python-pip python34 python-netaddr python34-pip ansible

# 升级 Jinja2 否则报错 no test named 'equalto'
pip install --upgrade Jinja2
```

## 2.2、获取源码

```
git clone https://github.com/kubernetes-incubator/kargo
```

## 2.3、编辑配置文件

```
cd kargo

vim inventory/group_vars/k8s-cluster.yml

# 配置文件如下:

# 启动集群的基础系统
bootstrap_os: centos

# etcd 数据存放位置
etcd_data_dir: /var/lib/etcd

# 二进制文件将要安装的位置
bin_dir: /usr/local/bin

# Kubernetes 配置文件存放目录以及命名空间
kube_config_dir: /etc/kubernetes
kube_script_dir: "{{ bin_dir }}/kubernetes-scripts"
kube_manifest_dir: "{{ kube_config_dir }}/manifests"
system_namespace: kube-system

# 日志存放位置
kube_log_dir: "/var/log/kubernetes"

# kubernetes 证书存放位置
kube_cert_dir: "{{ kube_config_dir }}/ssl"

# token存放位置
kube_token_dir: "{{ kube_config_dir }}/tokens"

# basic auth 认证文件存放位置
kube_users_dir: "{{ kube_config_dir }}/users"

# 关闭匿名授权
kube_api_anonymous_auth: false

## 使用的 kubernetes 版本
kube_version: v1.5.4

# 安装过程中缓存文件下载位置(最少 1G)
local_release_dir: "/tmp/releases"

# 重试次数，比如下载失败等情况
retry_stagger: 5

# 证书组
kube_cert_group: kube-cert

# 集群日志等级
kube_log_level: 2

# HTTP 下 api server 密码及用户
kube_api_pwd: "test123"
kube_users:
  kube:
    pass: "{{kube_api_pwd}}"
    role: admin
  root:
    pass: "{{kube_api_pwd}}"
    role: admin

# 网络 CNI 组件 (calico, weave or flannel)
kube_network_plugin: calico

# 服务地址分配
kube_service_addresses: 10.233.0.0/18

# pod 地址分配
kube_pods_subnet: 10.233.64.0/18

# 网络节点大小分配
kube_network_node_prefix: 24

# api server 监听地址及端口
kube_apiserver_ip: "{{ kube_service_addresses|ipaddr('net')|ipaddr(1)|ipaddr('address') }}"
kube_apiserver_port: 6443 # (https)
kube_apiserver_insecure_port: 8080 # (http)

# 默认 dns 后缀
cluster_name: cluster.local

# Subdomains of DNS domain to be resolved via /etc/resolv.conf for hostnet pods
ndots: 2

# DNS 组件 dnsmasq_kubedns/kubedns
dns_mode: dnsmasq_kubedns

# Can be docker_dns, host_resolvconf or none
resolvconf_mode: docker_dns

# 部署 netchecker 来检测 DNS 和 HTTP 状态
deploy_netchecker: true 

# skydns service IP 配置
skydns_server: "{{ kube_service_addresses|ipaddr('net')|ipaddr(3)|ipaddr('address') }}"
dns_server: "{{ kube_service_addresses|ipaddr('net')|ipaddr(2)|ipaddr('address') }}"
dns_domain: "{{ cluster_name }}"

# docker 存储目录
docker_daemon_graph: "/var/lib/docker"

# docker 的额外配置参数，默认会在 /etc/systemd/system/docker.service.d/ 创建相关配置，如果节点已经安装了 docker，并且做了自己的配置，比如启用的 device mapper ，那么要删除/更改这里，防止冲突导致 docker 无法启动

docker_options: "--insecure-registry={{ kube_service_addresses }} --graph={{ docker_daemon_graph }} --iptables=false"
docker_bin_dir: "/usr/bin"

# 组件部署方式
# Settings for containerized control plane (etcd/kubelet/secrets)
etcd_deployment_type: docker
kubelet_deployment_type: docker
cert_management: script
vault_deployment_type: docker

# 这里修改为 读取本地的,否则会自动去下载
k8s_image_pull_policy: IfNotPresent

# Monitoring apps for k8s
efk_enabled: true 

```

## 2.4、生成集群配置文件

```
cd kargo

CONFIG_FILE=inventory/inventory.cfg python3 contrib/inventory_builder/inventory.py 10.6.0.52 10.6.0.53 10.6.0.55 10.6.0.56

# 输入如下：

DEBUG: Adding group all
DEBUG: Adding group kube-master
DEBUG: Adding group kube-node
DEBUG: Adding group etcd
DEBUG: Adding group k8s-cluster:children
DEBUG: Adding group calico-rr
DEBUG: adding host node1 to group all
DEBUG: adding host node2 to group all
DEBUG: adding host node3 to group all
DEBUG: adding host node4 to group all
DEBUG: adding host kube-node to group k8s-cluster:children
DEBUG: adding host kube-master to group k8s-cluster:children
DEBUG: adding host node1 to group etcd
DEBUG: adding host node2 to group etcd
DEBUG: adding host node3 to group etcd
DEBUG: adding host node1 to group kube-master
DEBUG: adding host node2 to group kube-master
DEBUG: adding host node1 to group kube-node
DEBUG: adding host node2 to group kube-node
DEBUG: adding host node3 to group kube-node
DEBUG: adding host node4 to group kube-node


# 生成的配置文件在当前目录，既 kargo/inventory 目录下 inventory.cfg

# 配置文件如下(默认配置双master，可自行修改)：
# SSH 非 22 端口 添加 ansible_port=xxx

[all]
node1    ansible_host=10.6.0.52 ansible_port=33 ip=10.6.0.52
node2    ansible_host=10.6.0.53 ansible_port=33 ip=10.6.0.53
node3    ansible_host=10.6.0.55 ansible_port=33 ip=10.6.0.55
node4    ansible_host=10.6.0.56 ansible_port=33 ip=10.6.0.56

[kube-master]
node1    
node2    

[kube-node]
node1    
node2    
node3    
node4    

[etcd]
node1    
node2    
node3    

[k8s-cluster:children]
kube-node        
kube-master      

[calico-rr]

```

```
# 这里修改一下  下载 镜像 的版本
# 我在使用的时候 官方已经更新到 1.5.4 了， kargo 里的还是 1.5.3 版本的

cd kargo
vi roles/download/defaults/main.yml



local_release_dir: /tmp

# if this is set to true will only download files once. Doesn't work
# on Container Linux by CoreOS unless the download_localhost is true and localhost
# is running another OS type. Default compress level is 1 (fastest).
download_run_once: False
download_compress: 1

# if this is set to true, uses the localhost for download_run_once mode
# (requires docker and sudo to access docker). You may want this option for
# local caching of docker images or for Container Linux by CoreOS cluster nodes.
# Otherwise, uses the first node in the kube-master group to store images
# in the download_run_once mode.
download_localhost: False

# Always pull images if set to True. Otherwise check by the repo's tag/digest.
download_always_pull: False

# Versions
kube_version: v1.5.4
etcd_version: v3.0.6
#TODO(mattymo): Move calico versions to roles/network_plugins/calico/defaults
# after migration to container download
calico_version: "v1.0.2"
calico_cni_version: "v1.5.6"
calico_policy_version: "v0.5.2"
weave_version: 1.8.2
flannel_version: v0.6.2
pod_infra_version: 3.0

# Download URL's
etcd_download_url: "https://storage.googleapis.com/kargo/{{etcd_version}}_etcd"

# Checksums
etcd_checksum: "385afd518f93e3005510b7aaa04d38ee4a39f06f5152cd33bb86d4f0c94c7485"

# Containers
# Possible values: host, docker
etcd_deployment_type: "docker"
etcd_image_repo: "quay.io/coreos/etcd"
etcd_image_tag: "{{ etcd_version }}"
flannel_image_repo: "quay.io/coreos/flannel"
flannel_image_tag: "{{ flannel_version }}"
calicoctl_image_repo: "calico/ctl"
calicoctl_image_tag: "{{ calico_version }}"
calico_node_image_repo: "calico/node"
calico_node_image_tag: "{{ calico_version }}"
calico_cni_image_repo: "calico/cni"
calico_cni_image_tag: "{{ calico_cni_version }}"
calico_policy_image_repo: "calico/kube-policy-controller"
calico_policy_image_tag: "{{ calico_policy_version }}"
# TODO(adidenko): switch to "calico/routereflector" when
# https://github.com/projectcalico/calico-bird/pull/27 is merged
calico_rr_image_repo: "quay.io/l23network/routereflector"
calico_rr_image_tag: "v0.1"
exechealthz_version: 1.2
exechealthz_image_repo: "gcr.io/google_containers/exechealthz-amd64"
exechealthz_image_tag: "{{ exechealthz_version }}"
hyperkube_image_repo: "quay.io/coreos/hyperkube"
hyperkube_image_tag: "{{ kube_version }}_coreos.0"
pod_infra_image_repo: "gcr.io/google_containers/pause-amd64"
pod_infra_image_tag: "{{ pod_infra_version }}"
netcheck_tag: "v1.0"
netcheck_agent_img_repo: "quay.io/l23network/k8s-netchecker-agent"
netcheck_server_img_repo: "quay.io/l23network/k8s-netchecker-server"
weave_kube_image_repo: "weaveworks/weave-kube"
weave_kube_image_tag: "{{ weave_version }}"
weave_npc_image_repo: "weaveworks/weave-npc"
weave_npc_image_tag: "{{ weave_version }}"

nginx_image_repo: nginx
nginx_image_tag: 1.11.4-alpine
dnsmasq_version: 2.72
dnsmasq_image_repo: "andyshinn/dnsmasq"
dnsmasq_image_tag: "{{ dnsmasq_version }}"
kubednsmasq_version: 1.4
kubednsmasq_image_repo: "gcr.io/google_containers/kube-dnsmasq-amd64"
kubednsmasq_image_tag: "{{ kubednsmasq_version }}"
kubedns_version: 1.9
kubedns_image_repo: "gcr.io/google_containers/kubedns-amd64"
kubedns_image_tag: "{{ kubedns_version }}"
test_image_repo: busybox
test_image_tag: latest
elasticsearch_version: "v2.4.1"
elasticsearch_image_repo: "gcr.io/google_containers/elasticsearch"
elasticsearch_image_tag: "{{ elasticsearch_version }}"
fluentd_version: "1.22"
fluentd_image_repo: "gcr.io/google_containers/fluentd-elasticsearch"
fluentd_image_tag: "{{ fluentd_version }}"
kibana_version: "v4.6.1"
kibana_image_repo: "gcr.io/google_containers/kibana"
kibana_image_tag: "{{ kibana_version }}"

downloads:
  netcheck_server:
    container: true
    repo: "{{ netcheck_server_img_repo }}"
    tag: "{{ netcheck_tag }}"
    sha256: "{{ netcheck_server_digest_checksum|default(None) }}"
    enabled: "{{ deploy_netchecker|bool }}"
  netcheck_agent:
    container: true
    repo: "{{ netcheck_agent_img_repo }}"
    tag: "{{ netcheck_tag }}"
    sha256: "{{ netcheck_agent_digest_checksum|default(None) }}"
    enabled: "{{ deploy_netchecker|bool }}"
  etcd:
    version: "{{etcd_version}}"
    dest: "etcd/etcd-{{ etcd_version }}-linux-amd64.tar.gz"
    sha256: >-
      {%- if etcd_deployment_type in [ 'docker', 'rkt' ] -%}{{etcd_digest_checksum|default(None)}}{%- else -%}{{etcd_checksum}}{%- endif -%}
    source_url: "{{ etcd_download_url }}"
    url: "{{ etcd_download_url }}"
    unarchive: true
    owner: "etcd"
    mode: "0755"
    container: "{{ etcd_deployment_type in [ 'docker', 'rkt' ] }}"
    repo: "{{ etcd_image_repo }}"
    tag: "{{ etcd_image_tag }}"
  hyperkube:
    container: true
    repo: "{{ hyperkube_image_repo }}"
    tag: "{{ hyperkube_image_tag }}"
    sha256: "{{ hyperkube_digest_checksum|default(None) }}"
  flannel:
    container: true
    repo: "{{ flannel_image_repo }}"
    tag: "{{ flannel_image_tag }}"
    sha256: "{{ flannel_digest_checksum|default(None) }}"
    enabled: "{{ kube_network_plugin == 'flannel' or kube_network_plugin == 'canal' }}"
  calicoctl:
    container: true
    repo: "{{ calicoctl_image_repo }}"
    tag: "{{ calicoctl_image_tag }}"
    sha256: "{{ calicoctl_digest_checksum|default(None) }}"
    enabled: "{{ kube_network_plugin == 'calico' or kube_network_plugin == 'canal' }}"
  calico_node:
    container: true
    repo: "{{ calico_node_image_repo }}"
    tag: "{{ calico_node_image_tag }}"
    sha256: "{{ calico_node_digest_checksum|default(None) }}"
    enabled: "{{ kube_network_plugin == 'calico' or kube_network_plugin == 'canal' }}"
  calico_cni:
    container: true
    repo: "{{ calico_cni_image_repo }}"
    tag: "{{ calico_cni_image_tag }}"
    sha256: "{{ calico_cni_digest_checksum|default(None) }}"
    enabled: "{{ kube_network_plugin == 'calico' or kube_network_plugin == 'canal' }}"
  calico_policy:
    container: true
    repo: "{{ calico_policy_image_repo }}"
    tag: "{{ calico_policy_image_tag }}"
    sha256: "{{ calico_policy_digest_checksum|default(None) }}"
    enabled: "{{ kube_network_plugin == 'canal' }}"
  calico_rr:
    container: true
    repo: "{{ calico_rr_image_repo }}"
    tag: "{{ calico_rr_image_tag }}"
    sha256: "{{ calico_rr_digest_checksum|default(None) }}"
    enabled: "{{ peer_with_calico_rr is defined and peer_with_calico_rr}} and kube_network_plugin == 'calico'"
  weave_kube:
    container: true
    repo: "{{ weave_kube_image_repo }}"
    tag: "{{ weave_kube_image_tag }}"
    sha256: "{{ weave_kube_digest_checksum|default(None) }}"
    enabled: "{{ kube_network_plugin == 'weave' }}"
  weave_npc:
    container: true
    repo: "{{ weave_npc_image_repo }}"
    tag: "{{ weave_npc_image_tag }}"
    sha256: "{{ weave_npc_digest_checksum|default(None) }}"
    enabled: "{{ kube_network_plugin == 'weave' }}"
  pod_infra:
    container: true
    repo: "{{ pod_infra_image_repo }}"
    tag: "{{ pod_infra_image_tag }}"
    sha256: "{{ pod_infra_digest_checksum|default(None) }}"
  nginx:
    container: true
    repo: "{{ nginx_image_repo }}"
    tag: "{{ nginx_image_tag }}"
    sha256: "{{ nginx_digest_checksum|default(None) }}"
  dnsmasq:
    container: true
    repo: "{{ dnsmasq_image_repo }}"
    tag: "{{ dnsmasq_image_tag }}"
    sha256: "{{ dnsmasq_digest_checksum|default(None) }}"
  kubednsmasq:
    container: true
    repo: "{{ kubednsmasq_image_repo }}"
    tag: "{{ kubednsmasq_image_tag }}"
    sha256: "{{ kubednsmasq_digest_checksum|default(None) }}"
  kubedns:
    container: true
    repo: "{{ kubedns_image_repo }}"
    tag: "{{ kubedns_image_tag }}"
    sha256: "{{ kubedns_digest_checksum|default(None) }}"
  testbox:
    container: true
    repo: "{{ test_image_repo }}"
    tag: "{{ test_image_tag }}"
    sha256: "{{ testbox_digest_checksum|default(None) }}"
  exechealthz:
    container: true
    repo: "{{ exechealthz_image_repo }}"
    tag: "{{ exechealthz_image_tag }}"
    sha256: "{{ exechealthz_digest_checksum|default(None) }}"
  elasticsearch:
    container: true
    repo: "{{ elasticsearch_image_repo }}"
    tag: "{{ elasticsearch_image_tag }}"
    sha256: "{{ elasticsearch_digest_checksum|default(None) }}"
  fluentd:
    container: true
    repo: "{{ fluentd_image_repo }}"
    tag: "{{ fluentd_image_tag }}"
    sha256: "{{ fluentd_digest_checksum|default(None) }}"
  kibana:
    container: true
    repo: "{{ kibana_image_repo }}"
    tag: "{{ kibana_image_tag }}"
    sha256: "{{ kibana_digest_checksum|default(None) }}"

download:
  container: "{{ file.container|default('false') }}"
  repo: "{{ file.repo|default(None) }}"
  tag: "{{ file.tag|default(None) }}"
  enabled: "{{ file.enabled|default('true') }}"
  dest: "{{ file.dest|default(None) }}"
  version: "{{ file.version|default(None) }}"
  sha256: "{{ file.sha256|default(None) }}"
  source_url: "{{ file.source_url|default(None) }}"
  url: "{{ file.url|default(None) }}"
  unarchive: "{{ file.unarchive|default('false') }}"
  owner: "{{ file.owner|default('kube') }}"
  mode: "{{ file.mode|default(None) }}"

```







## 2.5、部署集群

```
# 执行如下命令，请确保SSH KEY 登陆, 端口一致

ansible-playbook -i inventory/inventory.cfg cluster.yml -b -v --private-key=~/.ssh/id_rsa

```

## 2.6、测试

```
# 两个 master 中使用 kubectl get nodes

[root@k8s-node-1 ~]# kubectl get nodes
NAME         STATUS    AGE
k8s-node-1   Ready     16m
k8s-node-2   Ready     20m
k8s-node-3   Ready     16m
k8s-node-4   Ready     16m



[root@k8s-node-2 ~]# kubectl get nodes
NAME         STATUS    AGE
k8s-node-1   Ready     11m
k8s-node-2   Ready     16m
k8s-node-3   Ready     11m
k8s-node-4   Ready     11m




[root@k8s-node-1 ~]# kubectl get pods --namespace=kube-system
NAME                                  READY     STATUS    RESTARTS   AGE
dnsmasq-411420702-z0gkx               1/1       Running   0          16m
dnsmasq-autoscaler-1155841093-1hxdl   1/1       Running   0          16m
elasticsearch-logging-v1-kgt1t        1/1       Running   0          15m
elasticsearch-logging-v1-vm4bd        1/1       Running   0          15m
fluentd-es-v1.22-6gql6                1/1       Running   0          15m
fluentd-es-v1.22-8zkjh                1/1       Running   0          15m
fluentd-es-v1.22-cjskv                1/1       Running   0          15m
fluentd-es-v1.22-j4857                1/1       Running   0          15m
kibana-logging-2924323056-x3vjk       1/1       Running   0          15m
kube-apiserver-k8s-node-1             1/1       Running   0          15m
kube-apiserver-k8s-node-2             1/1       Running   0          20m
kube-controller-manager-k8s-node-1    1/1       Running   0          16m
kube-controller-manager-k8s-node-2    1/1       Running   0          21m
kube-proxy-k8s-node-1                 1/1       Running   0          16m
kube-proxy-k8s-node-2                 1/1       Running   0          21m
kube-proxy-k8s-node-3                 1/1       Running   0          16m
kube-proxy-k8s-node-4                 1/1       Running   0          16m
kube-scheduler-k8s-node-1             1/1       Running   0          16m
kube-scheduler-k8s-node-2             1/1       Running   0          21m
kubedns-3830354952-pfl7n              3/3       Running   4          16m
kubedns-autoscaler-54374881-64x6d     1/1       Running   0          16m
nginx-proxy-k8s-node-3                1/1       Running   0          16m
nginx-proxy-k8s-node-4                1/1       Running   0          16m



[root@k8s-node-1 ~]# kubectl get pods
NAME                             READY     STATUS    RESTARTS   AGE
netchecker-agent-3x3sj           1/1       Running   0          16m
netchecker-agent-ggxs2           1/1       Running   0          16m
netchecker-agent-hostnet-45k84   1/1       Running   0          16m
netchecker-agent-hostnet-kwvc8   1/1       Running   0          16m
netchecker-agent-hostnet-pwm77   1/1       Running   0          16m
netchecker-agent-hostnet-z4gmq   1/1       Running   0          16m
netchecker-agent-q3291           1/1       Running   0          16m
netchecker-agent-qtml6           1/1       Running   0          16m
netchecker-server                1/1       Running   0          16m


```


```
# 配置一个 nginx deplyment 与 nginx  service

apiVersion: extensions/v1beta1 
kind: Deployment 
metadata: 
  name: nginx-dm
spec: 
  replicas: 2
  template: 
    metadata: 
      labels: 
        name: nginx 
    spec: 
      containers: 
        - name: nginx 
          image: nginx:alpine 
          imagePullPolicy: IfNotPresent
          ports: 
            - containerPort: 80

```

```
apiVersion: v1 
kind: Service
metadata: 
  name: nginx-dm 
spec: 
  ports: 
    - port: 80
      targetPort: 80
      protocol: TCP 
  selector: 
    name: nginx
```


```
# 导入 yaml 文件


[root@k8s-node-1 ~]# kubectl apply -f nginx.yaml 
deployment "nginx-dm" created
service "nginx-dm" created



[root@k8s-node-1 ~]# kubectl get pods -o wide
NAME                             READY     STATUS    RESTARTS   AGE       IP              NODE
nginx-dm-4194680597-0h071        1/1       Running   0          9m        10.233.75.8     k8s-node-4
nginx-dm-4194680597-dzcf3        1/1       Running   0          9m        10.233.76.124   k8s-node-3


[root@k8s-node-1 ~]# kubectl get svc -o wide    
NAME                 CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE       SELECTOR
kubernetes           10.233.0.1      <none>        443/TCP          39m       <none>
netchecker-service   10.233.0.126    <nodes>       8081:31081/TCP   33m       app=netchecker-server
nginx-dm             10.233.56.138   <none>        80/TCP           10m       name=nginx

```


```
# 部署一个  curl 的 pods 用来测试 内部通信


apiVersion: v1
kind: Pod
metadata:
  name: curl
spec:
  containers:
  - name: curl
    image: radial/busyboxplus:curl
    command:
    - sh
    - -c
    - while true; do sleep 1; done

```    

```    
# 导入 yaml 文件

[root@k8s-node-1 ~]# kubectl apply -f curl.yaml 
pod "curl" created

    
[root@k8s-node-1 ~]# kubectl get pods -o wide
NAME                             READY     STATUS    RESTARTS   AGE       IP              NODE
curl                             1/1       Running   0          2m        10.233.75.22    k8s-node-4


```

```
# 测试 curl --> nginx-svc


[root@k8s-node-1 ~]# kubectl exec -it curl curl nginx-dm


<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```







```
# 创建一个 zk  集群 zk-deplyment  和 service


apiVersion: extensions/v1beta1
kind: Deployment 
metadata: 
  name: zookeeper-1
spec: 
  replicas: 1
  template: 
    metadata: 
      labels: 
        name: zookeeper-1 
    spec: 
      containers: 
        - name: zookeeper-1
          image: jicki/zk:alpine 
          imagePullPolicy: IfNotPresent
          env:
          - name: NODE_ID
            value: "1"
          - name: NODES
            value: "0.0.0.0,zookeeper-2,zookeeper-3"
          ports:
          - containerPort: 2181

```

```
apiVersion: extensions/v1beta1 
kind: Deployment
metadata:
  name: zookeeper-2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: zookeeper-2
    spec:
      containers:
        - name: zookeeper-2
          image: jicki/zk:alpine
          imagePullPolicy: IfNotPresent
          env:
          - name: NODE_ID
            value: "2"
          - name: NODES
            value: "zookeeper-1,0.0.0.0,zookeeper-3"
          ports:
          - containerPort: 2181

```


```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: zookeeper-3
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: zookeeper-3
    spec:
      containers:
        - name: zookeeper-3
          image: jicki/zk:alpine
          imagePullPolicy: IfNotPresent
          env:
          - name: NODE_ID
            value: "3"
          - name: NODES
            value: "zookeeper-1,zookeeper-2,0.0.0.0"
          ports:
          - containerPort: 2181
```

```
apiVersion: v1 
kind: Service 
metadata: 
  name: zookeeper-1 
  labels:
    name: zookeeper-1
spec: 
  ports: 
    - name: client
      port: 2181
      protocol: TCP
    - name: followers
      port: 2888
      protocol: TCP
    - name: election
      port: 3888
      protocol: TCP
  selector: 
    name: zookeeper-1

```

```

apiVersion: v1 
kind: Service 
metadata: 
  name: zookeeper-2
  labels:
    name: zookeeper-2
spec: 
  ports: 
    - name: client
      port: 2181
      protocol: TCP
    - name: followers
      port: 2888
      protocol: TCP
    - name: election
      port: 3888
      protocol: TCP
  selector: 
    name: zookeeper-2

```

```

apiVersion: v1 
kind: Service 
metadata: 
  name: zookeeper-3
  labels:
    name: zookeeper-3
spec: 
  ports: 
    - name: client
      port: 2181
      protocol: TCP
    - name: followers
      port: 2888
      protocol: TCP
    - name: election
      port: 3888
      protocol: TCP
  selector: 
    name: zookeeper-3

```

```
# 查看状态  pods 与 service

[root@k8s-node-1 ~]# kubectl get pods -o wide
NAME                             READY     STATUS    RESTARTS   AGE       IP              NODE
zookeeper-1-3762028479-gd5rm     1/1       Running   0          1m        10.233.76.125   k8s-node-3
zookeeper-2-4266983361-cz80w     1/1       Running   0          1m        10.233.75.23    k8s-node-4
zookeeper-3-479264707-hlv3x      1/1       Running   0          1m        10.233.75.24    k8s-node-4



[root@k8s-node-1 ~]# kubectl get svc -o wide    
NAME                 CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE       SELECTOR
zookeeper-1          10.233.25.46    <none>        2181/TCP,2888/TCP,3888/TCP   1m        name=zookeeper-1
zookeeper-2          10.233.49.4     <none>        2181/TCP,2888/TCP,3888/TCP   1m        name=zookeeper-2
zookeeper-3          10.233.50.206   <none>        2181/TCP,2888/TCP,3888/TCP   1m        name=zookeeper-3

```


## 2.7、后续..

```
# 等待更新ing.......
```



