---
k8s安装之etcd集群搭建
---
# 下发证书修改权限

```
salt '*etcd*' state.sls ssl.install saltenv=gpu
salt '*master*' state.sls ssl.install saltenv=gpu
salt '*master*'  cmd.run 'chmod 600 /export/kubernetes/ssl/*.pem' 
salt '*etcd*'  cmd.run 'chmod 600 /export/kubernetes/ssl/*.pem'
```

# etcd集群搭建

```
etcd-source-install:
  file.managed:
    - source: salt://etcd/files/etcd-v3.1.5-linux-amd64.tar.gz
    - name: /usr/local/src/etcd-v3.1.5-linux-amd64.tar.gz
  cmd.run:
    - name:  cd /usr/local/src  && tar -xvf etcd-v3.1.5-linux-amd64.tar.gz && mv etcd-v3.1.5-linux-amd64/etcd* /bin/  && mkdir -pv /export/Data/etcd && mkdir /export/etcd/ && mkdir -pv /etc/etcd
/etc/etcd/etcd.conf:
  file.managed:
    - source: salt://etcd/files/etcd.conf
    - user: root
    - group: root
    - template: jinja
    - defaults:
      FQDN: {{ grains['fqdn'] }}
      FQDN_IP: {{grains.get('fqdn_ip4')[0]}}
    - mode: 644
    - require:
      - file: etcd-source-install
/usr/lib/systemd/system/etcd.service:
  file.managed:
    - source: salt://etcd/files/etcd.service
    - user: root
    - group: root
    - mode: 644
  cmd.run:
    - name: systemctl daemon-reload 
  service.running:
    - name: etcd
    - enable: True
    - reload: True
    - require: 
      - file: /usr/lib/systemd/system/etcd.service
      
```

# 集群搭建之后验证
 
 验证etcd集群的命令如下：
 
```
 export ETCDCTL_API=3 
 etcdctl  --cacert=/export/kubernetes/ssl/ca.pem --cert=/export/kubernetes/ssl/kubernetes.pem --key=/export/kubernetes/ssl/kubernetes-key.pem  --endpoints  https://10.240.29.222:2379,https://10.240.29.234:2379,https://10.240.29.235:2379 endpoint health

etcdctl  --cacert=/export/kubernetes/ssl/ca.pem --cert=/export/kubernetes/ssl/kubernetes.pem --key=/export/kubernetes/ssl/kubernetes-key.pem  --endpoints  https://10.240.29.222:2379,https://10.240.29.234:2379,https://10.240.29.235:2379 endpoint health
2017-12-27 15:01:46.194898 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
2017-12-27 15:01:46.196109 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
2017-12-27 15:01:46.197285 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
https://10.240.29.222:2379 is healthy: successfully committed proposal: took = 1.618653ms
https://10.240.29.234:2379 is healthy: successfully committed proposal: took = 1.383725ms
https://10.240.29.235:2379 is healthy: successfully committed proposal: took = 1.31283ms
 ```
证明etcd集群搭建完毕
