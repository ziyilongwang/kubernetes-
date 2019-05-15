---
master节点部署
---
# 简介
kubernetes master节点仅仅需要运行三个进程，这三个进程为kube-apiserver、kube-scheduler、kube-controller-manager。

master节点上的三个组件功能紧密相连，同时又有制约：
master节点同时只能有一个 kube-scheduler、kube-controller-manager 进程处于工作状态，如果运行多个，则需要通过选举产生一个 leader。
因此要实现kuber-master的高可用其实只需要处理apiserver的高可用就行，而其他两个组件可用使用etcd进行选举产生。请在进行如下操作之前，请
确认已经执行过如下操作：haproxy和keepalived的正确安装，ssl.install文件的顺利下发。ssl.install文件包含了kubeconfig文件的下发。

# 节点规划
## ip地址规划
物理机ip        |虚拟ip     | 名字  |  realserver | 软件版本|
 :--------:   | :-----:   | :----: | :-----:   |:-----:  |
 10.240.29.226     | 10.240.29.249      |  k8s-haproxy2-gpu-m6 |10.240.29.221 10.240.29.233  10.240.29.220 | 1.9.0 |
 10.249.29.239        | 10.240.29.249    |   k8s-haproxy1-gpu-m6  |10.240.29.221 10.240.29.233  10.240.29.220 | 1.9.0 |

# 安装部署
注意：在kubernetes-server包中默认包含了客户端相关的包
## 软件包下载
```
wget https://dl.k8s.io/v1.9.0/kubernetes-server-linux-amd64.tar.gz
```
## master安装步骤说明
这是整理好的目录结构：
```
[root@JXQ-23-58-39 k8s-master]# tree
.
├── files
│   ├── config
│   └── kubernetes-server-linux-amd64.tar.gz
├── install.sls
├── kube-api
│   ├── files
│   │   ├── apiserver
│   │   ├── config
│   │   └── kube-apiserver.service
│   └── install.sls
├── kube-contorller-manager
│   ├── files
│   │   ├── controller-manager
│   │   └── kube-controller-manager.service
│   └── install.sls
└── kube-scheduler
    ├── files
    │   ├── kube-scheduler.service
    │   └── scheduler
    └── install.sls

7 directories, 13 files
```
如下是k8s-master的安装文件：

```
k8s-source-install:
  file.managed:
    - source: salt://k8s-master/files/kubernetes-server-linux-amd64.tar.gz
    - name: /usr/local/src/kubernetes-server-linux-amd64.tar.gz
    - user: root
    - mode: 644
  cmd.run:
    - name: cd /usr/local/src  && tar -xvf kubernetes-server-linux-amd64.tar.gz && cd kubernetes && cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl} /bin
k8s-config-manged:
  file.managed:
    - source: salt://k8s-master/files/config
    - name: /export/kubernetes/config
    - user: root
    - mode: 644
include:
  - k8s-master.kube-api.install
  - k8s-master.kube-contorller-manager.install
  - k8s-master.kube-scheduler.instal
```
## 配置kube-apiserver



```
将token进行下发，目前安装的过程中，token的安装放在了ssl下发那个步骤里面。即ssl.install里面。


[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
User=root
ExecStart=/usr/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --advertise-address={{ fqdn_ip }} \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/export/Log/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address={{ fqdn_ip }} \
  --client-ca-file=/export/kubernetes/ssl/ca.pem \
  --enable-swagger-ui=true \
  --etcd-cafile=/export/kubernetes/ssl/ca.pem \
  --etcd-certfile=/export/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/export/kubernetes/ssl/kubernetes-key.pem  \
  --etcd-servers=https://{{ ETCD1 }}:2379,https://{{ ETCD2 }}:2379,https://{{ ETCD3 }}:2379 \
  --event-ttl=1h \
  --kubelet-https=true \
  --insecure-bind-address=0.0.0.0 \
  --runtime-config=rbac.authorization.k8s.io/v1alpha1 \
  --service-account-key-file=/export/kubernetes/ssl/ca-key.pem \
  --service-cluster-ip-range=10.254.0.0/16 \
  --service-node-port-range=30000-32000 \
  --tls-cert-file=/export/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/export/kubernetes/ssl/kubernetes-key.pem \
  --enable-bootstrap-token-auth \
  --token-auth-file=/export/kubernetes/token.csv \
  --v=2
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

## kube-controller-manager安装
```
[root@JXQ-23-58-39 k8s-master]# cat kube-controller-manager/install.sls 
kube-controller-manager:
  file.managed:
    - source: salt://k8s-master/kube-controller-manager/files/controller-manager
    - name: /export/kubernetes/controller-manager
    - user: root
    - mode: 644
kube-controller-init:
  file.managed:
    - source: salt://k8s-master/kube-controller-manager/files/kube-controller-manager.service 
    - name: /usr/lib/systemd/system/kube-controller-manager.service
    - user: root
    - mode: 644
  service.running:
    - name: kube-controller-manager.service
    - enable: True
    - reload: True
    - require: 
      - file: kube-controller-init

[root@JXQ-23-58-39 k8s-master]# cat kube-controller-manager/files/controller-manager 
###
# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# controller manager默认会为所有的网络提供服务，这里指定服务地址为本机
# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--address=127.0.0.1 \
            --service-cluster-ip-range=10.254.0.0/16 \
            --cluster-name=kubernetes \
            --cluster-signing-cert-file=/export/kubernetes/ssl/ca.pem \
            --cluster-signing-key-file=/export/kubernetes/ssl/ca-key.pem \
             --service-account-private-key-file=/export/kubernetes/ssl/ca-key.pem \
            --root-ca-file=/export/kubernetes/ssl/ca.pem \
            --leader-elect=true \
            --node-monitor-grace-period=40s \
            --node-monitor-period=5s \
            --pod-eviction-timeout=50ms"

```

##  kube-schedule安装配置
```
[root@JXQ-23-58-39 k8s-master]# cat kube-scheduler/install.sls 
kube-scheduler-conf-managed:
  file.managed:
    - source: salt://k8s-master/kube-scheduler/files/scheduler
    - name: /export/kubernetes/scheduler
    - user: root
    - mode: 644
kube-scheduler-init:
  file.managed:
    - source: salt://k8s-master/kube-scheduler/files/kube-scheduler.service
    - name: /usr/lib/systemd/system/kube-scheduler.service
    - user: root
    - mode: 644
  service.running:
    - name: kube-scheduler.service
    - enable: True
    - reload: True
    - require: 
      - file: kube-scheduler-init
kube-scheduler.service  scheduler               
[root@JXQ-23-58-39 k8s-master]# cat kube-scheduler/files/kube-scheduler.service 
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/export/kubernetes/config
EnvironmentFile=-/export/kubernetes/scheduler
ExecStart=/usr/bin/kube-scheduler \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
[root@JXQ-23-58-39 k8s-master]# cat kube-scheduler/files/
kube-scheduler.service  scheduler               
[root@JXQ-23-58-39 k8s-master]# cat kube-scheduler/files/scheduler 
###
# kubernetes scheduler config

# default config should be adequate

# Add your own!
KUBE_SCHEDULER_ARGS="--leader-elect=true --address=127.0.0.1"
[root@JXQ-23-58-39 k8s-master]# 
```

# 部署完成之后进行测试
```
[root@JXQ-23-58-39 k8s-master]#  curl http://10.240.29.249:8080
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1beta1",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1beta1",
    "/apis/apps",
    "/apis/apps/v1",
    "/apis/apps/v1beta1",
 ```   
  ```   
 [root@JXQ-23-58-39 k8s-node]# salt '*master*'  cmd.run 'kubectl get cs'
k8s-master1-m6-gpu:
    NAME                 STATUS    MESSAGE              ERROR
    scheduler            Healthy   ok                   
    controller-manager   Healthy   ok                   
    etcd-0               Healthy   {"health": "true"}   
    etcd-2               Healthy   {"health": "true"}   
    etcd-1               Healthy   {"health": "true"}
k8s-master3-m6-gpu:
    NAME                 STATUS    MESSAGE              ERROR
    controller-manager   Healthy   ok                   
    scheduler            Healthy   ok                   
    etcd-2               Healthy   {"health": "true"}   
    etcd-1               Healthy   {"health": "true"}   
    etcd-0               Healthy   {"health": "true"}
k8s-master2-m6-gpu:
    NAME                 STATUS    MESSAGE              ERROR
    scheduler            Healthy   ok                   
    controller-manager   Healthy   ok                   
    etcd-0               Healthy   {"health": "true"}   
    etcd-2               Healthy   {"health": "true"}   
    etcd-1               Healthy   {"health": "true"}
[root@JXQ-23-58-39 k8s-node]# 
 ```   
