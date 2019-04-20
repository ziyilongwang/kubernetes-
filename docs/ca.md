k8s安装之证书生成及安装（二）
---
# 简介
由于 Etcd 和 Kubernetes 全部采用 TLS 通讯，所以先要生成 TLS 证书，证书生成工具采用 cfssl，具体使用方法这里不再详细阐述，生成证书时可在任一节点完成，这里在宿主机执行，证书列表
如下:

证书名称|	配置文件 |	用途|
 :--------:   | :-----:   | :-----: |
|ca.pem kubernetes.pem kubernetes-key.pem 		|ca-config.json、ca-csr.json kubernetes-csr.json  |	etcd 集群证书 etcd根证书
|ca.pem |ca-config.json	ca-csr.json |	kubelet k8s根证书
|ca.pem、kube-proxy-key.pem、kube-proxy.pem|k8s-gencert.json、kube-proxy-csr.json|	kube-proxy使用的证书
|ca.pem、admin-key.pem、admin.pem	|ca-config.json、admin-csr.json |	kubectl使用的证书
|ca.pem、kubernetes-key.pem、kubernetes.pem |	ca-config.json、kubernetes-csr.json	| kube-apiserver 使用的证书


# 创建TLS证书和秘钥 
kubernetes 系统各组件需要使用 TLS 证书对通信进行加密，本文档使用 CloudFlare 的 PKI 工具集 [cfssl](https://github.com/cloudflare/cfssl)来生成 Certificate Authority (CA) 证书和秘钥文件，CA 是自签名的证书，用来签名后续创建的其它 TLS 证书.


`注意：由于证书只生成一次，最终需要给k8s集群的各个节点去使用,后期将所有证书文件推送到各个节点`

**生成的 CA 证书和秘钥文件如下：**

```
ca-key.pem
ca.pem
kubernetes-key.pem
kubernetes.pem
kube-proxy.pem
kube-proxy-key.pem
admin.pem
admin-key.pem
```


##1. 安装cfssl工具

方式一：

```
$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ chmod +x cfssl_linux-amd64
$ sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl

$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ chmod +x cfssljson_linux-amd64
$ sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
$ chmod +x cfssl-certinfo_linux-amd64
$ sudo mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

$ export PATH=/root/local/bin:$PATH 
```


## 2. 创建 CA (Certificate Authority)

**创建CA配置文件**

```
$ mkdir /export/k8s/ssl/CA
$cfssl print-defaults config > config.json
$cfssl print-defaults csr > csr.json
# cat config.json 
{
    "signing": {
        "default": {
            "expiry": "87600"
        },
        "profiles": {
            "www": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            }
        }
    }
}
```
# 根据config.json文件的格式创建如下的ca-config.json文件
# 过期时间设置成了 87600h
手动添加CA配置文件：
```
# cat >ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}

EOF
```
- 字段说明：
- * ca-config.json：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；
- *signing：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE；
- *server auth：表示client可以用该 CA 对server提供的证书进行验证；
- *client auth：表示server可以用该CA对client提供的证书进行验证；

**创建 CA 证书签名请求**

```
手动创建CA证书签名请求：
#cat  >ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
- * "CN"：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；
- * "O"：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)；


**生成 CA 证书和私钥**

```
# cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2017/07/31 20:59:46 [INFO] generating a new CA key and certificate from CSR
2017/07/31 20:59:46 [INFO] generate received request
2017/07/31 20:59:46 [INFO] received CSR
2017/07/31 20:59:46 [INFO] generating key: rsa-2048
2017/07/31 20:59:46 [INFO] encoded CSR
2017/07/31 20:59:46 [INFO] signed certificate with serial number 691088125687719138673449869323212551920824346463

查看证书和私钥匙：
# ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem

```

#### 3. 创建 kubernetes 证书

**创建 kubernetes 证书签名请求**

```
# cat > kubernetes-csr.json <<EOF 
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "10.240.29.249",
      "10.240.29.220",
      "10.240.29.221",
      "10.240.29.222",
      "10.240.29.233",
      "10.240.29.234",
      "10.240.29.235",
      "10.254.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

`注意：如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表，由于该证书后续被 etcd 集群和 kubernetes master 集群使用，所以上面分别指定了 etcd 集群、kubernetes master 集群的主机 IP 和 kubernetes 服务的服务 IP（一般是 kue-apiserver 指定的 service-cluster-ip-range 网段的第一个IP，如 10.254.0.1`

**生成 kubernetes 证书和私钥**

```
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
2017/07/31 21:07:18 [INFO] generate received request
2017/07/31 21:07:18 [INFO] received CSR
2017/07/31 21:07:18 [INFO] generating key: rsa-2048
2017/07/31 21:07:19 [INFO] encoded CSR
2017/07/31 21:07:19 [INFO] signed certificate with serial number 107380041442520543246804407651135521354615931691

查看证书和私钥
# ls kubernetes*
kubernetes.csr  kubernetes-csr.json  kubernetes-key.pem  kubernetes.pem
```

#### 4. 创建 admin 证书

**创建 admin 证书签名请求**

```
# cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

```

- * 后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权；
- * kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 的所有 API的权限；
- * OU 指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限；

**生成 admin 证书和私钥**

```
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
2017/07/31 21:12:43 [INFO] generate received request
2017/07/31 21:12:43 [INFO] received CSR
2017/07/31 21:12:43 [INFO] generating key: rsa-2048
2017/07/31 21:12:44 [INFO] encoded CSR
2017/07/31 21:12:44 [INFO] signed certificate with serial number 551692651162919334285812157070169949251706186299
2017/07/31 21:12:44 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

查看证书和私钥：
# ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem

```
注意：这个admin 证书，是将来生成管理员用的kube config 配置文件用的，现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制， kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group（具体参考 Kubernetes中的用户与身份认证授权中 X509 Client Certs 一段）。

#### 5. 创建 kube-proxy 证书

**创建 kube-proxy 证书签名请求**

```
# cat > kube-proxy-csr.json <<EOF 
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

EOF
```

- * CN 指定该证书的 User 为 system:kube-proxy；
- * kube-apiserver 预定义的 RoleBinding cluster-admin 将User system:kube-proxy 与 Role system:node-proxier 绑定，该 Role 授予了调用 kube-apiserver Proxy 相关 API 的权限；

**生成 kube-proxy 客户端证书和私钥**

```
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
2017/07/31 21:17:27 [INFO] generate received request
2017/07/31 21:17:27 [INFO] received CSR
2017/07/31 21:17:27 [INFO] generating key: rsa-2048
2017/07/31 21:17:28 [INFO] encoded CSR
2017/07/31 21:17:28 [INFO] signed certificate with serial number 693071706048202812891639122320953216075635734090
2017/07/31 21:17:28 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

查看证书：
# ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem

```


#### 6. 校验证书

以 kubernetes 证书为例 

使用openssl工具：
```
# openssl x509  -noout -text -in  kubernetes.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            12:cf:16:6c:44:51:44:60:d1:90:c7:77:29:44:89:86:29:08:c3:2b
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=BeiJing, L=BeiJing, O=k8s, OU=System, CN=kubernetes
        Validity
            Not Before: Jul 31 13:02:00 2017 GMT
            Not After : Jul 29 13:02:00 2027 GMT
        Subject: C=CN, ST=BeiJing, L=BeiJing, O=k8s, OU=System, CN=kubernetes
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:cd:3d:d8:c9:34:68:1c:e3:ef:0d:4f:64:1d:c8:
                    02:67:39:fa:5a:1b:76:4e:7f:df:ea:57:9b:73:43:
                    a2:f6:ae:87:cc:78:7b:25:87:84:82:b3:9d:d8:27:
                    87:0d:6c:52:70:d2:d9:aa:a9:60:16:ed:15:db:b8:
                    6e:7d:f8:0c:8a:4c:b1:1c:ae:01:aa:1c:68:70:f7:
                    61:d8:b3:8c:c0:f6:31:04:94:fb:71:58:bc:82:08:
                    66:7b:67:a6:9e:5c:6f:b8:7c:94:86:3c:50:0d:cd:
                    1a:60:74:1d:60:7b:40:6a:ef:e9:be:e0:a1:f5:c5:
                    f5:0d:e9:8b:f1:72:bd:47:6d:ce:b0:60:7a:3f:ee:
                    75:4b:0b:91:8f:5b:f0:c1:32:e6:6b:22:03:d4:2a:
                    c3:ac:35:cd:dc:92:8e:51:23:0d:16:82:27:db:a5:
                    4f:38:ac:96:14:b5:50:b9:2a:b3:66:c6:33:d3:76:
                    25:0c:d4:0c:4c:15:7d:c6:33:01:e9:eb:29:23:99:
                    9b:81:e0:e8:ec:bf:e7:45:55:ca:a2:02:cd:4a:9f:
                    d1:43:0d:0b:62:12:08:ea:7a:4e:83:3c:10:4d:c8:
                    bc:67:d7:5f:58:f4:64:c2:d8:7e:c0:76:90:66:90:
                    28:58:dd:92:de:4e:c5:ba:52:33:ef:53:16:30:22:
                    a2:09
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier: 
                FF:3D:2E:6B:DD:5B:DA:66:EB:BF:F4:A4:DD:AA:0E:7F:F7:3D:CF:7C
            X509v3 Subject Alternative Name: 
                DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster, DNS:kubernetes.default.svc.cluster.local, IP Address:127.0.0.1, IP Address:172.25.51.115, IP Address:172.25.51.116, IP Address:172.25.51.117, IP Address:172.25.51.118, IP Address:172.25.51.119, IP Address:172.25.51.120, IP Address:172.25.51.121, IP Address:172.25.51.122, IP Address:172.25.51.123, IP Address:172.25.51.124, IP Address:172.25.47.77, IP Address:172.25.47.78, IP Address:172.25.44.6, IP Address:10.254.0.1
    Signature Algorithm: sha256WithRSAEncryption
         a6:b2:9a:1e:94:20:b4:71:1a:9e:ef:2c:e9:03:5b:23:f9:48:
         5e:3f:99:04:6f:8e:8d:a3:45:be:db:39:d9:4c:f4:1b:9d:84:
         28:ef:b8:e7:3a:83:4b:59:82:fb:33:da:35:91:b7:50:af:d9:
         8c:43:e0:a5:9a:91:d6:a6:59:fd:c7:a6:27:44:17:0b:ed:04:
         12:00:8e:f3:c6:ea:8f:0c:e7:86:75:b7:93:09:64:41:ea:d2:
         65:ce:c4:1d:72:68:d2:88:f6:8b:f6:bf:6d:6f:f7:d1:3f:e2:
         90:8d:e7:27:47:36:78:2b:e8:4f:21:43:ef:29:e7:0d:ed:88:
         33:84:42:4e:f0:84:c9:33:39:a9:26:96:d3:f4:39:df:89:e4:
         bf:1d:17:75:2c:d1:76:be:0e:67:cb:56:75:46:27:b4:a7:c8:
         26:a7:33:cb:8e:29:20:fc:6b:8e:aa:7f:89:f0:c7:e3:dc:7c:
         1e:bf:e7:8c:9b:19:23:8f:df:12:55:63:ec:92:64:03:1e:dc:
         f6:84:78:d8:8f:70:ad:18:d1:b8:06:88:9e:e5:c1:f3:cb:90:
         77:4b:88:6d:ef:92:25:30:73:26:25:b3:cc:ae:9a:38:31:41:
         d4:f8:27:d0:66:af:1f:a0:68:ef:1d:c0:0d:d9:03:7c:6a:b5:
         04:79:fd:1f

```

- * 确认 Issuer 字段的内容和 ca-csr.json 一致；
- * 确认 Subject 字段的内容和 kubernetes-csr.json 一致；
- * 确认 X509v3 Subject Alternative Name 字段的内容和 kubernetes-csr.json 一致；
- * 确认 X509v3 Key Usage、Extended Key Usage 字段的内容和 ca-config.json 中 kubernetes profile 一致；



使用`cfssl-certinfo`工具：

```
# cfssl-certinfo -cert kubernetes.pem
{
  "subject": {
    "common_name": "kubernetes",
    "country": "CN",
    "organization": "k8s",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "k8s",
      "System",
      "kubernetes"
    ]
  },
  "issuer": {
    "common_name": "kubernetes",
    "country": "CN",
    "organization": "k8s",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "k8s",
      "System",
      "kubernetes"
    ]
  },
  "serial_number": "107380041442520543246804407651135521354615931691",
  "sans": [
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local",
    "127.0.0.1",
    "172.25.51.115",
    "172.25.51.116",
    "172.25.51.117",
    "172.25.51.118",
    "172.25.51.119",
    "172.25.51.120",
    "172.25.51.121",
    "172.25.51.122",
    "172.25.51.123",
    "172.25.51.124",
    "172.25.47.77",
    "172.25.47.78",
    "172.25.44.6",
    "10.254.0.1"
  ],
  "not_before": "2017-07-31T13:02:00Z",
  "not_after": "2027-07-29T13:02:00Z",
  "sigalg": "SHA256WithRSA",
  "authority_key_id": "",
  "subject_key_id": "FF:3D:2E:6B:DD:5B:DA:66:EB:BF:F4:A4:DD:AA:E:7F:F7:3D:CF:7C",
  "pem": "-----BEGIN CERTIFICATE-----\nMIIEoDCCA4igAwIBAgIUEs8WbERRRGDRkMd3KUSJhikIwyswDQYJKoZIhvcNAQEL\nBQAwZTELMAkGA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0Jl\naUppbmcxDDAKBgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMwEQYDVQQDEwpr\ndWJlcm5ldGVzMB4XDTE3MDczMTEzMDIwMFoXDTI3MDcyOTEzMDIwMFowZTELMAkG\nA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0JlaUppbmcxDDAK\nBgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMwEQYDVQQDEwprdWJlcm5ldGVz\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzT3YyTRoHOPvDU9kHcgC\nZzn6Wht2Tn/f6lebc0Oi9q6HzHh7JYeEgrOd2CeHDWxScNLZqqlgFu0V27huffgM\nikyxHK4BqhxocPdh2LOMwPYxBJT7cVi8gghme2emnlxvuHyUhjxQDc0aYHQdYHtA\nau/pvuCh9cX1DemL8XK9R23OsGB6P+51SwuRj1vwwTLmayID1CrDrDXN3JKOUSMN\nFoIn26VPOKyWFLVQuSqzZsYz03YlDNQMTBV9xjMB6espI5mbgeDo7L/nRVXKogLN\nSp/RQw0LYhII6npOgzwQTci8Z9dfWPRkwth+wHaQZpAoWN2S3k7FulIz71MWMCKi\nCQIDAQABo4IBRjCCAUIwDgYDVR0PAQH/BAQDAgWgMB0GA1UdJQQWMBQGCCsGAQUF\nBwMBBggrBgEFBQcDAjAMBgNVHRMBAf8EAjAAMB0GA1UdDgQWBBT/PS5r3VvaZuu/\n9KTdqg5/9z3PfDCB4wYDVR0RBIHbMIHYggprdWJlcm5ldGVzghJrdWJlcm5ldGVz\nLmRlZmF1bHSCFmt1YmVybmV0ZXMuZGVmYXVsdC5zdmOCHmt1YmVybmV0ZXMuZGVm\nYXVsdC5zdmMuY2x1c3RlcoIka3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVy\nLmxvY2FshwR/AAABhwSsGTNzhwSsGTN0hwSsGTN1hwSsGTN2hwSsGTN3hwSsGTN4\nhwSsGTN5hwSsGTN6hwSsGTN7hwSsGTN8hwSsGS9NhwSsGS9OhwSsGSwGhwQK/gAB\nMA0GCSqGSIb3DQEBCwUAA4IBAQCmspoelCC0cRqe7yzpA1sj+UheP5kEb46No0W+\n2znZTPQbnYQo77jnOoNLWYL7M9o1kbdQr9mMQ+ClmpHWpln9x6YnRBcL7QQSAI7z\nxuqPDOeGdbeTCWRB6tJlzsQdcmjSiPaL9r9tb/fRP+KQjecnRzZ4K+hPIUPvKecN\n7YgzhEJO8ITJMzmpJpbT9DnfieS/HRd1LNF2vg5ny1Z1Rie0p8gmpzPLjikg/GuO\nqn+J8Mfj3Hwev+eMmxkjj98SVWPskmQDHtz2hHjYj3CtGNG4Boie5cHzy5B3S4ht\n75IlMHMmJbPMrpo4MUHU+CfQZq8foGjvHcAN2QN8arUEef0f\n-----END CERTIFICATE-----\n"
}

```

#### 7. 分发证书

将生成的证书和秘钥文件（后缀名为.pem）拷贝到所有机器的 /export/kubernetes/ssl 目录下备用；

在ansible主控机上下发配置(注意当前目录为证书所在目录`/export/k8s/ssl/CA`)：
```
# ansible -i ../../iphost all -m shell -a 'mkdir -p /export/kubernetes/ssl'
# for i in `ls`;do ansible -i ../../iphost all -m copy -a "src=$i dest=/export/kubernetes/ssl/";done 

查看证书：
ansible -i ../../iphost all -m shell -a 'ls /export/kubernetes/ssl'
```



