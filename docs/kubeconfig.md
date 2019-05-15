k8s安装之kubeconfig配置(三)
---
# 简介

服务器规划：

|物理机ip        |虚拟ip     | 用途  |  realserver | 软件版本|
:--------:   | :-----:   | :----: | :-----:   |:-----:  |
10.240.29.226     | 10.240.29.249      |  k8s-haproxy2-gpu-m6 |10.240.29.221 10.240.29.233  10.240.29.220 | 1.9.0 |
10.249.29.239        | 10.240.29.249    |   k8s-haproxy1-gpu-m6  |10.240.29.221 10.240.29.233  10.240.29.220 | 1.9.0 |
首先将执行下面的命令，将kubectl安装到master机器上，具体步骤可以参照下篇文章，kubectl的安装。

```
查看到客户端版本目前为1.9.0
# kubectl version
    Client Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.0", GitCommit:"925c127ec6b946659ad0fd596fa959be43f0cc05", GitTreeState:"clean", BuildDate:"2017-12-15T21:07:38Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
    The connection to the server localhost:8080 was refused - did you specify the right host or port?

```
# kubeocnfig文件
kubeconfig文件记录k8s集群的各种信息，对集群构建非常重要。
kubectl命令行工具从~/.kube/config，即kubectl的kubeconfig文件中获取访问kube-apiserver的地址，证书和用户名等信息
kubelet/kube-proxy等在Node上的程序进程同样通过bootstrap.kubeconfig和kube-proxy.kubeconfig上提供的认证与授权信息与Master进行通讯
Kubelet在首次启动时，会向kube-apiserver发送TLS Bootstrapping请求。如果kube-apiserver验证其与自己的token.csv一致，则为kubelet生成CA与key。
# token文件
先在`master`节点上安装 kubectl 然后再进行下面的操作。
```
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF

生成的文件：
[root@JXQ-23-58-39 files]# cat token.csv 
bd8cc63aacf12ba3d4cac2464b185740,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
```
将生成的token文件发送到集群的/export/kubernetes目录下。
## token文件怎么更新？
`BOOTSTRAP_TOKEN` 将被写入到 kube-apiserver 使用的 `token.csv` 文件和 kubelet 使用的 `bootstrap.kubeconfig` 文件，如果后续重新生成了 BOOTSTRAP_TOKEN，则需要：

- 更新 token.csv 文件，分发到所有机器 (master 和 node）的 /export/kubernetes/ 目录下，分发到node节点上非必需；
- 重新生成 bootstrap.kubeconfig 文件，分发到所有 node 机器的 /export/kubernetes/ 目录下；
- 重启 kube-apiserver 和 kubelet 进程；
- 重新 approve kubelet 的 csr 请求；

分发token到节点上：

```
salt '*master1*' state.sls  ssl.install saltenv=gpu 
```

## 生成kubectl kubelet的kubeconfig文件bootstrap.kubeconfig和kubelet.kubeconfig

创建 `kubelet bootstrapping kubeconfig `配置(需要提前安装 kubectl 命令)，对于 node 节点，api server 地址为master-vip:6443(为多个apiserver做负载均衡后的负载IP)，如果想把 master
也当做 node 使用，那么 master 上 api server 地址应该为 masterIP:6443。所以以下配置只适合 node 节点，如果想把 master 也当做 node，那么需要重新生成下面的 kubeconfig 配置，
并把 api server 地址修改为当前 master的虚ip地址。

```
export KUBE_APISERVER="https://10.240.29.249:6443"
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/export/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```
- --embed-certs 为 true 时表示将 certificate-authority 证书写入到生成的 `bootstrap.kubeconfig `文件中；
- 设置客户端认证参数时没有指定秘钥和证书，后续由 kube-apiserver 自动生成；
 



**创建 kube-proxy kubeconfig 配置，同上面一样，如果想要把 master 当 node 使用，需要修改 api server**

```
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/export/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kube-proxy \
  --client-certificate=/export/kubernetes/ssl/kube-proxy.pem \
  --client-key=/export/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

- 设置集群参数和客户端认证参数时 --embed-certs 都为 true，这会将 certificate-authority、client-certificate 和 client-key 指向的证书文件内容写入到生成的 kube-proxy.kubeconfig 文件中；
- kube-proxy.pem 证书中 CN 为 system:kube-proxy，kube-apiserver 预定义的 RoleBinding cluster-admin 将User system:kube-proxy 与 Role system:node-proxier 绑定，该 Role 授予了调用 kube-apiserver Proxy 相关 API 的权限；

**分发 kubeconfig 文件**

将两个 kubeconfig 文件分发到所有 Node 机器的 /export/kubernetes/ 目录;我这里是将其放到k8s集群的每个机器上了。以备后用。

```
salt '*master*'  state.sls  ssl.install 
```



