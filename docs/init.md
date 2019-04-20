---
title: k8smaster节点高可用搭建（一）
date: 2017-12-26 17:47:01
categories: k8
tags: kubernetes，k8s，高可用方案
---
# 简介 
Kubernetes作为容器编排管理系统，通过Scheduler、Replication Controller等组件实现了应用层的高可用，通过service实现了pod的故障透明和负载均衡。但是针对Kubernetes集群，还需要实现各个
组件和服务的高可用。下面主要介绍一下搭建高可用k8s集群的步骤和配置（以下步骤是使用非容器方式启动k8s的组件和服务）。

   Keepalived是一个类似于layer3, 4 & 5交换机制的软件，也就是我们平时说的第3层、第4层和第5层交换。Keepalived的作用是检测web服务器的状态，如果有一台web服务器死机，或工作出现故障，
Keepalived将检测到，并将有故障的web服务器从系统中剔除，当web服务器工作正常后Keepalived自动将web服务器加入到服务器群中，这些工作全部自动完成，不需要人工干涉，需要人工做的只是修复
故障的web服务器.

  HAProxy提供高可用性、负载均衡以及基于TCP和HTTP应用的代理，支持虚拟主机，它是免费、快速并且可靠的一种解决方案。HAProxy特别适用于那些负载特大的web站点，这些站点通常又需要会话保持
或七层处理。HAProxy运行在当前的硬件上，完全可以支持数以万计的并发连接。并且它的运行模式使得它可以很简单安全的整合进您当前的架构中， 同时可以保护你的web服务器不被暴露到网络上。

# ip地址规划
物理机ip        |虚拟ip     | 名字  |  realserver |
 :--------:   | :-----:   | :----: | :-----:   |
 10.240.29.226     | 10.240.29.249      |  k8s-haproxy2-gpu-m6 |10.240.29.221 10.240.29.233  10.240.29.220 |
 10.249.29.239        | 10.240.29.249    |   k8s-haproxy1-gpu-m6  |10.240.29.221 10.240.29.233  10.240.29.220 | 
 
# 软件版本

软件名称|版本 |
 :--------:   | :-----:   
keepalived  | v1.3.4    
haproxy | v1.7.3    
```
wget http://www.keepalived.org/software/keepalived-1.3.4.tar.gz
wget http://www.haproxy.org/download/1.7/src/haproxy-1.7.3.tar.gz

```
# 安装步骤

首先进行keepalived的安装。

## 安装haproxy
```
[root@JXQ-23-58-39 haproxy]# tree 
.
├── files
│   ├── haproxy-1.7.3.tar.gz
│   ├── haproxy.cfg
│   └── set_log.sh
└── install.sls

1 directory, 4 files
```


```
haproxy:
  group.present:
    - name: haproxy
  user.present:
    - fullname: haproxy
    - shell: /sbin/nologin
    - groups:
      - haproxy
haproxy-source-install:
  file.managed:
    - source: salt://haproxy/files/haproxy-1.7.3.tar.gz
    - name: /usr/local/src/haproxy-1.7.3.tar.gz
    - user: root
    - mode: 644
  cmd.run:
    - name: cd /usr/local/src && tar -xvf haproxy-1.7.3.tar.gz -C /usr/local && ln -s /usr/local/haproxy-1.7.3 /usr/local/haproxy && cd /usr/local/haproxy && make TARGET=linux26 PREFIX=/usr/local/haproxy &&  make install PREFIX=/usr/local/haproxy && mkdir /var/lib/haproxy
    - unless: test -d /usr/local/haproxy
haproxy-conf-managed:
  file.managed:
    - source: salt://haproxy/files/haproxy.cfg
    - name: /usr/local/haproxy/haproxy.cfg
    - user: root
    - root: 644
set-haproxy-log:
  file.managed:
    - source: salt://haproxy/files/set_log.sh
    - name: /usr/local/src/set_log.sh
    - user: root
    - group: root
    - mode: 755
  cmd.run:
    - name: cd /usr/local/src && sh set_log.sh && sleep 2 && touch set_log.sh.lock
    - unless: test -e /usr/local/src/set_log.sh.lock
    - require:
      - file: set-haproxy-log
```

## 安装keepalived
目录结构如下：

```
[root@JXQ-23-58-39 keepalived]# tree 
.
├── files
│   ├── check_haproxy.sh
│   ├── keepalived
│   ├── keepalived-1.3.4.tar.gz
│   └── keepalived.conf
└── install.sls

1 directory, 5 files
```

安装过程如下：

```
keepalived-source-install:
  file.managed:
    - source: salt://keepalived/files/keepalived-1.3.4.tar.gz 
    - name: /usr/local/src/keepalived-1.3.4.tar.gz 
    - user: root
    - mode: 644
  cmd.run:
    - name: yum -y install openssl-devel && cd /usr/local/src  &&  tar -xvf keepalived-1.3.4.tar.gz -C /usr/local  &&ln -s /usr/local/keepalived-1.3.4 /usr/local/keepalived  && cd /usr/local/keepalived && ./configure  && make && make install && mkdir -pv /etc/keepalived && ln -s /usr/local/sbin/keepalived /usr/sbin/keepalived 
    - unless: test -d /usr/local/keepalived
keepalived-conf-managed:
  file.managed:
    - source: salt://keepalived/files/keepalived.conf
    - name: /etc/keepalived/keepalived.conf
    - mode: 644
    - user: root
    - template: jinja
      {% if grains['fqdn_ip4'][0] == '10.240.29.226' %}
      STATE: MASTER
      PRIORITY: 150
      {% else %}
      STATE: BACKUP
      PRIORITY: 120
      {% endif %}
      VIP: 10.240.26.251
keepalived-init-managed:
  file.managed:
    - source: salt://keepalived/files/keepalived
    - name: /etc/init.d/keepalived
    - mode: 774
    - user: root
  service.running:
    - name: keepalived
    - enable: True
    - reload: True
    - watch:
      - file: keepalived-conf-managed
keepalived-check-haproxy-conf:
  file.managed:
    - source: salt://keepalived/files/check_haproxy.sh
    - name: /etc/keepalived/check_haproxy.sh
    - user: root
    - mode: 774

```
配置文件说明：将10.240.29.226这台主机设置为master主机；其PRIORITY值为150；vip为10.240.26.251。

## 执行安装命令
```
salt '*ha*'  state.sls keepalived.install  
```
## 检查服务是否正常启动
```
k8s-haproxy1-gpu-m6:
    * keepalived.service - LVS and VRRP High Availability Monitor
       Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
       Active: active (running) since Tue 2017-12-26 17:25:39 CST; 16min ago
      Process: 30177 ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
     Main PID: 30178 (keepalived)
       CGroup: /system.slice/keepalived.service
               |-30178 /usr/sbin/keepalived
               |-30179 /usr/sbin/keepalived
               `-30180 /usr/sbin/keepalived
    
    Dec 26 17:25:39 JXQ-240-29-226.h.chinabank.com.cn Keepalived_vrrp[30180]: Unable to access script `/etc/keepalived/check.sh`
    Dec 26 17:25:39 JXQ-240-29-226.h.chinabank.com.cn Keepalived_vrrp[30180]: Disabling track script chk_haproxy since not found
    Dec 26 17:25:39 JXQ-240-29-226.h.chinabank.com.cn Keepalived_vrrp[30180]: Using LinkWatch kernel netlink reflector...
    Dec 26 17:25:39 JXQ-240-29-226.h.chinabank.com.cn Keepalived_healthcheckers[30179]: Registering Kernel netlink reflector
    Dec 26 17:25:39 JXQ-240-29-226.h.chinabank.com.cn Keepalived_healthcheckers[30179]: Registering Kernel netlink command channel
    Dec 26 17:25:39 JXQ-240-29-226.h.chinabank.com.cn Keepalived_healthcheckers[30179]: Opening file '/etc/keepalived/keepalived.conf'.
    Dec 26 17:25:39 JXQ-240-29-226.h.chinabank.com.cn Keepalived_healthcheckers[30179]: Using LinkWatch kernel netlink reflector...
    Dec 26 17:25:41 JXQ-240-29-226.h.chinabank.com.cn Keepalived_vrrp[30180]: VRRP_Instance(sce_master) Transition to MASTER STATE
    Dec 26 17:25:41 JXQ-240-29-226.h.chinabank.com.cn Keepalived_vrrp[30180]: VRRP_Instance(sce_master) Changing effective priority from 150 to 152
    Dec 26 17:25:43 JXQ-240-29-226.h.chinabank.com.cn Keepalived_vrrp[30180]: VRRP_Instance(sce_master) Entering MASTER STATE
k8s-haproxy2-gpu-m6:
    * keepalived.service - LVS and VRRP High Availability Monitor
       Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
       Active: active (running) since Tue 2017-12-26 17:27:19 CST; 14min ago
     Main PID: 17181 (keepalived)
       CGroup: /system.slice/keepalived.service
               |-17181 /usr/sbin/keepalived
               |-17182 /usr/sbin/keepalived
               `-17183 /usr/sbin/keepalived
    
    Dec 26 17:27:19 JXQ-240-29-239.h.chinabank.com.cn Keepalived_vrrp[17183]: VRRP parsed invalid IP {. skipping IP...
    Dec 26 17:27:19 JXQ-240-29-239.h.chinabank.com.cn Keepalived_vrrp[17183]: Unable to access script `/etc/keepalived/check.sh`
    Dec 26 17:27:19 JXQ-240-29-239.h.chinabank.com.cn Keepalived_vrrp[17183]: Disabling track script chk_haproxy since not found
    Dec 26 17:27:19 JXQ-240-29-239.h.chinabank.com.cn Keepalived_vrrp[17183]: Using LinkWatch kernel netlink reflector...
    Dec 26 17:27:19 JXQ-240-29-239.h.chinabank.com.cn Keepalived_vrrp[17183]: VRRP_Instance(sce_master) Entering BACKUP STATE
    Dec 26 17:27:19 JXQ-240-29-239.h.chinabank.com.cn Keepalived_healthcheckers[17182]: Registering Kernel netlink reflector
    Dec 26 17:27:19 JXQ-240-29-239.h.chinabank.com.cn Keepalived_healthcheckers[17182]: Registering Kernel netlink command channel
    Dec 26 17:27:19 JXQ-240-29-239.h.chinabank.com.cn Keepalived_healthcheckers[17182]: Opening file '/etc/keepalived/keepalived.conf'.
    Dec 26 17:27:19 JXQ-240-29-239.h.chinabank.com.cn Keepalived_healthcheckers[17182]: Using LinkWatch kernel netlink reflector...
    Dec 26 17:27:21 JXQ-240-29-239.h.chinabank.com.cn Keepalived_vrrp[17183]: VRRP_Instance(sce_master) Changing effective priority from 120 to 122
```
继续检查看vip是否正常启动。
```
k8s-haproxy1-gpu-m6:
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
        link/ether f0:00:0a:f0:1d:e2 brd ff:ff:ff:ff:ff:ff
        inet 10.240.29.226/23 brd 10.240.29.255 scope global eth0
           valid_lft forever preferred_lft forever
        inet 10.240.26.251/24 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::f200:aff:fef0:1de2/64 scope link 
           valid_lft forever preferred_lft forever
k8s-haproxy2-gpu-m6:
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
        link/ether f0:00:0a:f0:1d:ef brd ff:ff:ff:ff:ff:ff
        inet 10.240.29.239/23 brd 10.240.29.255 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::f200:aff:fef0:1def/64 scope link 
           valid_lft forever preferred_lft forever
[root@JXQ-23-58-39 files]# 
```
经过检查发现10.240.26.251已经在haproxy1上正常启动。keepalived会自动启动haproxy文件。
配置完成之后做一次连通性测试。
```
[root@JXQ-23-58-39 gpu]# telnet 10.240.29.249 8080      
Trying 10.240.29.249...
Connected to 10.240.29.249.
Escape character is '^]'.
^CConnection closed by foreign host
```





