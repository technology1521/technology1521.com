---
ddolayout:  single
title:  "kubernetes二进制安装 - V1.19"
date:   2023-05-24 
categories:   [kubernetes]
classes: wide
---



[TOC]



## 一，基础环境分配

>
10.102.4.13 #hostnamectl set-hostname k8s-master1
10.102.4.15 #k8s-node1
10.102.4.14 #k8s-node2

>10.102.4.13 #etcd
10.102.4.15 #etcd
10.102.4.14 #etcd

>10.102.4.13 	#cfssl生成证书
以上机器都是2c3G
以上机器有重复利用，机器多可以分开布置
替换相应的IP即可

>docker 数据目录是本地 /data/docker   数据盘中
>etcd   数据目录是本地 /data/etcd   数据盘中
>kubernetes 集群日志存储   /data/logs/kubernetes   数据盘中
>
>>ls /data/logs/kubernetes 
>>kube-apiserver  kube-controller-manager  kube-proxy  kube-scheduler  kubelet   
>>数据盘中 
>
>cni bin目录是本地 /opt/cni
>
>containerd 目录是本地 /opt/containerd 
>
>dockerfile 目录是本地 /opt/ dockerfile
>
>etcd  目录是本地 /opt/etcd
>
>>etcd
>>|-- bin
>>|   |-- etcd
>>|   `-- etcdctl
>>|-- cfg
>>|   `-- etcd.conf
>>`-- ssl
>>    |-- ca-key.pem
>>    |-- ca.pem
>>    |-- server-key.pem
>>    `-- server.pem
>
>kubernetes 配置文件是放在 /opt/kubernetes 
>
>> kubernetes
>> |-- bin
>> |   |-- kube-apiserver
>> |   |-- kube-controller-manager
>> |   |-- kube-proxy
>> |   |-- kube-scheduler
>> |   |-- kubeadm
>> |   |-- kubectl
>> |   `-- kubelet
>> |-- cfg
>> |   |-- bootstrap.kubeconfig
>> |   |-- kube-apiserver.conf
>> |   |-- kube-controller-manager.conf
>> |   |-- kube-proxy-config.yml
>> |   |-- kube-proxy.conf
>> |   |-- kube-proxy.kubeconfig
>> |   |-- kube-proxy.sh
>> |   |-- kube-scheduler.conf
>> |   |-- kubeconfig.sh
>> |   |-- kubelet-config.yml
>> |   |-- kubelet.conf
>> |   |-- kubelet.kubeconfig
>> |   `-- token.csv
>> `-- ssl
>>     |-- ca-key.pem
>>     |-- ca.pem
>>     |-- kube-proxy-key.pem
>>     |-- kube-proxy.pem
>>     |-- kubelet-client-2021-09-11-11-09-15.pem
>>     |-- kubelet-client-current.pem -> /opt/kubernetes/ssl/kubelet-client-2021-09-11-11-09-15.pem
>>     |-- kubelet.crt
>>     |-- kubelet.key
>>     |-- proxy-client-key.pem
>>     |-- proxy-client.pem
>>     |-- server-key.pem
>>     `-- server.pem
>
>tls 配置文件是放在 /opt/tls 
>
>> |-- etcd
>> |   |-- ca-config.json
>> |   |-- ca-csr.json
>> |   |-- ca-key.pem
>> |   |-- ca.csr
>> |   |-- ca.pem
>> |   |-- server-csr.json
>> |   |-- server-key.pem
>> |   |-- server.csr
>> |   `-- server.pem
>> `-- k8s
>>     |-- admin-csr.json
>>     |-- admin-key.pem
>>     |-- admin.csr
>>     |-- admin.pem
>>     |-- ca-config.json
>>     |-- ca-csr.json
>>     |-- ca-key.pem
>>     |-- ca.csr
>>     |-- ca.pem
>>     |-- kube-proxy-csr.json
>>     |-- kube-proxy-key.pem
>>     |-- kube-proxy.csr
>>     |-- kube-proxy.pem
>>     |-- proxy-client-csr.json
>>     |-- proxy-client-key.pem
>>     |-- proxy-client.csr
>>     |-- proxy-client.pem
>>     |-- server-csr.json
>>     |-- server-key.pem
>>     |-- server.csr
>>     `-- server.pem





## 改成静态IP

## 做开机自启策略





## 二，基础环境部署

### 2.1 关闭selinux/firewalld设置

#### 2.1.1 临时关闭

``` shell 
systemctl stop firewalld #临时关闭firewalld
setenforce 0 #临时关闭

```

#### 2.1.2 永久关闭

```shell
systemctl disable firewalld #永久关闭firewalld

vi  /etc/selinux/config #永久关闭SElinux

SELINUX=disable


```

### 2.2 关闭swap

2.2.1 临时关闭

```shell
swapoff -a #第一步 关闭swap分区
vi  /etc/fstab
 /dev/mapper/centos-swap swap defaults 0 0 #注释此行
 #/dev/mapper/centos-swap swap                    swap    defaults        0 0

free -m #查看是否关闭 若swap行都显示 0 则表示关闭成功
```



2.2.2 永久关闭

```shell
vim /etc/sysctl.conf # 永久生效
修改 vm.swappiness 的修改为 0
vm.swappiness=0
sysctl -p # 使配置生效
vi  /etc/fstab
 /dev/mapper/centos-swap swap defaults 0 0 #注释此行
 #/dev/mapper/centos-swap swap                    swap    defaults        0 0
 ##重启init 6
```

或者

```shell
#关闭swap
swapoff -a  # 临时 
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久
```

这里要永久关闭selinux、firewalld，swap

### 2.3修改主机名、加域名解析

#### 2.3.1 修改主机名

```shell
10.102.4.13
hostnamectl set-hostname k8s-master1
10.102.4.15
hostnamectl set-hostname k8s-node1
10.102.4.14
hostnamectl set-hostname k8s-node2

```

#### 2.3.2 加域名解析

```shell
##所有机器10.102.4.13-15
vim /etc/hosts

10.102.4.13 master
10.102.4.15 node1
10.102.4.14 node2

```

#### 2.3.3做流量转发

```shell
# 将桥接的IPv4流量传递到iptables的链 
vim /etc/sysctl.d/k8s.conf 
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1

# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

## 准备cfssl证书

cfssl是一个开源的证书管理工具，使用json文件生成证书，相比openssl更方便使用。

文件：[cfssl](https://technology1521.github.io/technology1521.com/soft/cfssl)

[cfssljson](https://technology1521.github.io/technology1521.com/soft/cfssljson)

[cfssl-certinfo](https://technology1521.github.io/technology1521.com/soft/cfssl-certinfo)

```shell
chmod +x cfssl cfssljson cfssl-certinfo #赋权
mv cfssl /usr/local/bin/cfssl
mv cfssljson /usr/local/bin/cfssljson
mv cfssl-certinfo /usr/bin/cfssl-certinfo #移动
```



### 生成Etcd证书

1)自签证书颁发机构（CA）

```shell
#创建目录
[root@k8s-master2 ~]# mkdir -p /opt/tls/etcd 
[root@k8s-master2 ~]# cd /opt/tls/etcd 

#自签CA
[root@k8s-master2 etcd]# vim ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}

[root@k8s-master1 etcd]# vim ca-csr.json
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "wangzhuangzhuang",
            "ST": "chengdu"
        }
    ]
}

#生成证书
[root@k8s-master1 etcd]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -


2)使用自签CA签发Etcd HTTPS证书

#创建证书申请文件
[root@k8s-master1 etcd]# vim server-csr.json
{
    "CN": "etcd",
    "hosts": [
    "10.102.4.13", 
    "10.102.4.14",
    "10.102.4.15"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "wangzhuangzhuang",
            "ST": "ChengDu"
        }
    ]
}

*注：上述文件hosts字段中IP为所有etcd节点的集群内部通信IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP

#生成证书
[root@k8s-master1 etcd]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
```

### 生成kube-apiserver证书

```shell
#创建目录
[root@k8s-master1 ~]# mkdir /opt/tls/k8s -p
[root@k8s-master1 ~]# cd /opt/tls/k8s

#自签证书颁发机构（CA）
[root@k8s-master k8s]# vim ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
[root@k8s-master k8s]# vim ca-csr.json
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "wangzhuangzhuang",
            "ST": "ChengDu",
            "O": "k8s",
            "OU": "System"
        }
    ]
}

#生成证书
[root@k8s-master1 k8s]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

会生成ca.pem和ca-key.pem文件
```



### 使用自签CA签发kube-apiserver HTTPS证书

```shell
#创建证书申请文件
[root@k8s-master1 k8s]# vim server-csr.json
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "10.96.0.1",
      "10.102.4.13",    
      "10.102.4.14", 
      "10.102.4.15",
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
            "L": "wangzhuangzhuang",
            "ST": "ChengDu",
            "O": "k8s",
            "OU": "System"
        }
    ]
}


#生成证书
[root@k8s-master1 k8s]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server


会生成server.pem和server-key.pem文件

cp  server.pem proxy-client.pem
cp  server-key.pem proxy-client-key.pem
```

###  使用自签CA签发kube-apiserver HTTPS证书

### 生成kubectl连接集群的证书
```json
[root@k8s-master1 k8s]# vim admin-csr.json
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
      "L": "wangzhuangzhuang",
      "ST": "ChengDu",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}


[root@k8s-master1 k8s]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

使用自签CA签发 kube-controller-manager

```shell
[root@k8s-master1 k8s]# vim kube-controller-manager-csr.json
{
  "CN": "system:kube-controller-manager",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "wangzhuangzhuang",
      "ST": "ChengDu",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}

[root@k8s-master1 k8s]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

```



```shell
 vim kube-scheduler-csr.json
{
  "CN": "system:kube-scheduler",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "wangzhuangzhuang",
      "ST": "ChengDu",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}


#生成证书
[root@k8s-master1 k8s]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```





```shell

```



### 3.3、部署Etcd集群

```shell
-------------------------master1节点------------------------

#创建工作目录并解压二进制包
[root@k8s-master1 ~]# mkdir -p /opt/etcd/{bin,cfg,ssl}
[root@k8s-master1 ~]# tar zxvf etcd-v3.5.0-linux-amd64.tar.gz
[root@k8s-master1 ~]# mv etcd-v3.5.0-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/

#创建etcd配置文件
[root@k8s-master1 ~]# vim /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/data/etcd"
ETCD_LISTEN_PEER_URLS="https://10.102.4.13:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.102.4.13:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.102.4.13:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.102.4.13:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.102.4.13:2380,etcd-2=https://10.102.4.14:2380,etcd-3=https://10.102.4.15:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"


*参数解释
•	ETCD_NAME：节点名称，集群中唯一
•	ETCD_DATA_DIR：数据目录
•	ETCD_LISTEN_PEER_URLS：集群通信监听地址
•	ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
•	ETCD_INITIAL_ADVERTISE_PEERURLS：集群通告地址
•	ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
•	ETCD_INITIAL_CLUSTER：集群节点地址
•	ETCD_INITIALCLUSTER_TOKEN：集群Token
•	ETCD_INITIALCLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群


#systemd管理etcd
[root@k8s-master1 ~]# vim /usr/lib/systemd/system/etcd.service


[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap                                
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target


#拷贝刚才生成的证书
[root@k8s-master1 ~]# cp /opt/tls/etcd/*.pem /opt/etcd/ssl


#将上面Master节点所有生成的文件拷贝到节点Node2和节点Node3上
 scp -r /opt/etcd/ node1:/opt/
 scp /usr/lib/systemd/system/etcd.service node1:/usr/lib/systemd/system/
scp -r /opt/etcd/ node2:/opt/
scp /usr/lib/systemd/system/etcd.service node2:/usr/lib/systemd/system/


#在节点Node2和节点Node3分别修改etcd.conf配置文件
[root@k8s-node1 ~]# vim /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-2"
ETCD_DATA_DIR="/data/etcd"
ETCD_LISTEN_PEER_URLS="https://10.102.4.14:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.102.4.14:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.102.4.14:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.102.4.14:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.102.4.13:2380,etcd-2=https://10.102.4.14:2380,etcd-3=https://10.102.4.15:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

[root@k8s-node2 ~]# vim /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-3"
ETCD_DATA_DIR="/data/etcd"
ETCD_LISTEN_PEER_URLS="https://10.102.4.15:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.102.4.15:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.102.4.15:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.102.4.15:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.102.4.13:2380,etcd-2=https://10.102.4.14:2380,etcd-3=https://10.102.4.15:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"


---------------master1、node1、node2节点------------------

#启动并设置开机启动
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd

*注：master如果先启动了，会出现start悬停等待的现象，这时候可以先把node1、node2的etcd启动，随后master的etcd会正常启动。

#查看集群状态
[root@k8s-master1 ~]# ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://10.102.4.13:2379,https://10.102.4.14:2379,https://10.102.4.15:2379" endpoint health --write-out=table
+---------------------------+--------+-------------+-------+
|         ENDPOINT          | HEALTH |    TOOK     | ERROR |
+---------------------------+--------+-------------+-------+
| https://10.102.4.13:2379 |   true |  8.280002ms |       |
| https://10.102.4.14:2379 |   true |  8.113125ms |       |
| https://10.102.4.15:2379 |   true | 17.479134ms |       |
+---------------------------+--------+-------------+-------+

如果输出上面信息，就说明集群部署成功。
如果有问题第一步先看日志：/var/log/message 或 journalctl -u etcd
```







## 四 ，安装docker

### 1、下载并解压二进制包

```shell
以下在所有节点操作。这里采用二进制安装，用yum安装也一样
------------------------master1、node1、node2节点--------------------------

[root@k8s-node1 ~]# wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.9.tgz
[root@k8s-node1 ~]#tar xvf docker-20.10.12.tgz
[root@k8s-node1 ~]# mv docker/* /usr/bin
```

### 2、创建配置文件

```shell
------------------------master1、node1、node2节点--------------------------

[root@k8s-node1 ~]# mkdir /etc/docker
[root@k8s-node1 ~]# vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
```

### 3、systemd管理docker

```shell
------------------------master1、node1、node2节点--------------------------

[root@k8s-node1 ~]# vim /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target

#启动并设置开机启动
systemctl daemon-reload
systemctl enable docker
systemctl start docker

ps -aux |grep  docker|grep -v grep

```




#下载地址： https://storage.googleapis.com/kubernetes-release/release/v1.19.14/kubernetes-server-linux-amd64.tar.gz

[kubernetes-v1.19.14-server-linux-amd64.tar.gz](https://technology1521.github.io/technology1521.com/soft/kubernetes-v1.19.14-server-linux-amd64.tar.gz)



下载二进制文件

```shell
#创建目录
[root@k8s-master1 ~]# mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}

#解压二进制包
[root@k8s-master1 ~]# tar -xf kubernetes-v1.19.14-server-linux-amd64.tar.gz

#拷贝
[root@k8s-master1 ~]# cd kubernetes/server/bin
[root@k8s-master1 bin]# cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin
[root@k8s-master1 bin]# cp kubectl /usr/bin/
```

### 4、部署kube-apiserver

```shell
#创建配置文件
[root@k8s-master1 bin]# vim /opt/kubernetes/cfg/kube-apiserver.conf
KUBE_APISERVER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/data/logs/kubernetes/kube-apiserver \
--etcd-servers=https://10.102.4.13:2379,https://10.102.4.14:2379,https://10.102.4.15:2379 \
--bind-address=10.102.4.13 \
--secure-port=6443 \
--advertise-address=10.102.4.13 \
--allow-privileged=true \
--service-cluster-ip-range=10.96.0.0/16 \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--enable-bootstrap-token-auth=true \
--token-auth-file=/opt/kubernetes/cfg/token.csv \
--service-node-port-range=30000-65535 \
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \
--client-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
--etcd-cafile=/opt/etcd/ssl/ca.pem \
--etcd-certfile=/opt/etcd/ssl/server.pem \
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/data/logs/kubernetes/k8s-audit.log" \
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \
--requestheader-allowed-names=aggregator \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--proxy-client-cert-file=/opt/kubernetes/ssl/proxy-client.pem \
--proxy-client-key-file=/opt/kubernetes/ssl/proxy-client-key.pem \
--runtime-config=api/all=true \
--enable-aggregator-routing=true


参考说明
•	--logtostderr：启用日志
•	---v：日志等级
•	--log-dir：日志目录
•	--etcd-servers：etcd集群地址
•	--bind-address：监听地址
•	--secure-port：https安全端口
•	--advertise-address：集群通告地址
•	--allow-privileged：启用授权
•	--service-cluster-ip-range：Service虚拟IP地址段
•	--enable-admission-plugins：准入控制模块
•	--authorization-mode：认证授权，启用RBAC授权和节点自管理
•	--enable-bootstrap-token-auth：启用TLS bootstrap机制
•	--token-auth-file：bootstrap token文件
•	--service-node-port-range：Service nodeport类型默认分配端口范围
•	--kubelet-client-xxx：apiserver访问kubelet客户端证书
•	--tls-xxx-file：apiserver https证书
•	1.20版本必须加的参数：--service-account-issuer，--service-account-signing-key-file
•	--etcd-xxxfile：连接Etcd集群证书
•	--audit-log-xxx：审计日志
•	启动聚合层相关配置：--requestheader-client-ca-file，--proxy-client-cert-file，--proxy-client-key-file，--requestheader-allowed-names，--requestheader-extra-headers-prefix，--requestheader-group-headers，--requestheader-username-headers，--enable-aggregator-routing

#拷贝刚才生成的证书
[root@k8s-master1 bin]# cp /opt/tls/k8s/*pem /opt/kubernetes/ssl/
```



  ```shell
  #配置token文件
  [root@k8s-master1 bin]# vim /opt/kubernetes/cfg/token.csv
  #####8e4908667d4d495dd8b9367aa1301317,kubelet-bootstrap,10001,"system:node-bootstrapper"
  783e934d73901e0791216e591bd6636f,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
  
  *注：上述token可自行生成替换，但一定要与后续配置对应
  head -c 16 /dev/urandom | od -An -t x | tr -d ' '
  ```

**systemd管理apiserver**

```shell
[root@k8s-master1 bin]# vim /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target



#启动并设置开机启动
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
```

### 5、部署kube-controller-manager

```shell
#创建配置文件
❯ cat /opt/kubernetes/cfg/kube-controller-manager.conf      
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/data/logs/kubernetes/kube-controller-manager \
--leader-elect=true \
--bind-address=127.0.0.1 \
--kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \
--allocate-node-cidrs=true \
--cluster-cidr=10.244.0.0/16 \
--service-cluster-ip-range=10.96.0.0/16 \
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \
--root-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
--experimental-cluster-signing-duration=87600h0m0s"




参数说明
•	--kubeconfig：连接apiserver配置文件
•	--leader-elect：当该组件启动多个时，自动选举（HA）
•	--cluster-signing-cert-file/--cluster-signing-key-file：自动为kubelet颁发证书的CA，与apiserver保持一致
```

mkdir -p /data/logs/kubernetes/

```shell
#生成kubeconfig文件
KUBE_CONFIG="/opt/kubernetes/cfg/kube-controller-manager.kubeconfig"
KUBE_APISERVER="https://10.102.4.13:6443"
#生成kubeconfig文件
[root@k8s-master1 k8s]# KUBE_CONFIG="/opt/kubernetes/cfg/kube-controller-manager.kubeconfig"
[root@k8s-master1 k8s]# KUBE_APISERVER="https://192.168.1.20:6443"

·终端执行（4条）
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials kube-controller-manager \
  --client-certificate=/opt/kubernetes/ssl/kube-controller-manager.pem \
  --client-key=/opt/kubernetes/ssl/kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-controller-manager \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

systemd管理controller-manager

```shell
[root@k8s-master1 k8s]# vim /usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target

#启动并设置开机启动
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
```

### 6、部署kube-scheduler

```shell
#创建配置文件
[root@k8s-master1 k8s]# vim /opt/kubernetes/cfg/kube-scheduler.conf
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/data/logs/kubernetes/kube-scheduler \
--kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \
--leader-elect \
--bind-address=127.0.0.1"



参数说明
•	--kubeconfig：连接apiserver配置文件
•	--leader-elect：当该组件启动多个时，自动选举（HA）
```

生成kubeconfig文件

```shell
 KUBE_CONFIG="/opt/kubernetes/cfg/kube-scheduler.kubeconfig"
KUBE_APISERVER="https://10.102.4.13:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials kube-scheduler \
  --client-certificate=/opt/kubernetes/ssl/kube-scheduler.pem \
  --client-key=/opt/kubernetes/ssl/kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-scheduler \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}

```

systemd管理scheduler

```shell
[root@k8s-master1 k8s]# vim /usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target


#启动并设置开机启动
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
```

查看集群状态

```shell
#生成kubeconfig文件
mkdir /root/.kube

 KUBE_CONFIG="/root/.kube/config"
 KUBE_APISERVER="https://10.102.4.13:6443"


kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials cluster-admin \
  --client-certificate=/opt/tls/k8s/admin.pem \
  --client-key=/opt/tls/k8s/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user=cluster-admin \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}


#通过kubectl工具查看当前集群组件状态
root@k8s-master1 k8s]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-2               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"} 


#查看k8s的名称空间
[root@k8s-master1 k8s]# kubectl get ns
NAME              STATUS   AGE
default           Active   3h21m
kube-node-lease   Active   3h21m
kube-public       Active   3h21m
kube-system       Active   3h21m
```



## 部署Worker Node

### 1、创建工作目录并拷贝文件

```shell
--------------------node1、node2节点-------------------
# mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}
#master将kubelet和kube-proxy拷贝给node1、node2节点
[root@k8s-master k8s]# cd /date/kubernetes/server/bin/
[root@k8s-master bin]# scp kubelet kube-proxy node1:/opt/kubernetes/bin/
```

### 2、部署kubelet

```shell
----------------------下面这些操作在master节点完成：---------------------------
将kubelet-bootstrap用户绑定到系统集群角色
[root@k8s-master1 ~]# kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
  
clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created

创建kubeconfig文件:
在生成kubernetes证书的目录下执行以下命令生成kubeconfig文件：
[root@k8s-master1 ~]#  cd /opt/tls/k8s/
指定apiserver 内网负载均衡地址
[root@k8s-master1 crt]# KUBE_APISERVER="https://10.102.4.13:6443"  #写你master的ip地址，集群中就写负载均衡的ip地址
[root@k8s-master1 crt]# BOOTSTRAP_TOKEN=783e934d73901e0791216e591bd6636f

# 设置集群参数
[root@k8s-master1 crt]# kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig

# 设置客户端认证参数
[root@k8s-master crt]# kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig

# 设置上下文参数
[root@k8s-master crt]# kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig

# 设置默认上下文
[root@k8s-master crt]# kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

#====================================================================================

# 创建kube-proxy kubeconfig文件

[root@k8s-master1 crt]#kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig

[root@k8s-master1 crt]# kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

[root@k8s-master1 crt]# kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

[root@k8s-master1 crt]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

[root@k8s-master1 crt]# ls
bootstrap.kubeconfig  kube-proxy.kubeconfig

#必看：将这两个文件拷贝到Node节点/opt/kubernetes/cfg目录下。
[root@k8s-master1 crt]# scp *.kubeconfig node1:/opt/kubernetes/cfg/
[root@k8s-master1 crt]# scp *.kubeconfig node2:/opt/kubernetes/cfg/

 scp /opt/kubernetes/ssl/ca* node1:/opt/kubernetes/ssl
 scp /opt/kubernetes/ssl/ca* node2:/opt/kubernetes/ssl
```





```shell
在两个node节点创建kubelet配置文件：
[root@k8s-node1 ~]# vim /opt/kubernetes/cfg/kubelet.config
KUBELET_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/data/logs/kubernetes/kubelet \
--hostname-override=10.102.4.14 \
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
--config=/opt/kubernetes/cfg/kubelet-config.yml \
--cert-dir=/opt/kubernetes/ssl \
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"

[root@k8s-node1 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0
[root@k8s-node2 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0
参数说明：
* --hostname-override 在集群中显示的主机名
* --kubeconfig 指定kubeconfig文件位置，会自动生成
* --bootstrap-kubeconfig 指定刚才生成的bootstrap.kubeconfig文件
* --cert-dir 颁发证书存放位置
* --pod-infra-container-image 管理Pod网络的镜像

其中/opt/kubernetes/cfg/kubelet.config配置文件如下：
[root@k8s-node1 ~]# vim /opt/kubernetes/cfg/kub-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 10.102.4.14   #写你机器的ip地址
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS: ["10.96.0.2"]      #不要改，就是这个ip地址
clusterDomain: cluster.local.
failSwapOn: false
authentication:
  anonymous:
    enabled: true
  webhook:
    enabled: false
    
systemd管理kubelet组件：
# vim /usr/lib/systemd/system/kubelet.service 
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet $KUBELET_OPTS
Restart=on-failure
KillMode=process

[Install]
WantedBy=multi-user.target

启动：
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
[root@k8s-master ~]# /opt/kubernetes/bin/kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-F5AQ8SeoyloVrjPuzSbzJnFKQaUsier7EGvNFXLKTqM   17s       kubelet-bootstrap   Pending
node-csr-bjeHSWXOuUDSHganJPL_hDz_8jjYhM2FQyTkbA9pM0Q   18s       kubelet-bootstrap   Pending

在Master审批Node加入集群：
启动后还没加入到集群中，需要手动允许该节点才可以。在Master节点查看请求签名的Node：
[root@k8s-master1 ~]# /opt/kubernetes/bin/kubectl certificate approve XXXXID
注意：xxxid 指的是上面的NAME这一列
[root@k8s-master1 ~]# /opt/kubernetes/bin/kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr--1TVDzcozo7NoOD3WS2t9xLQqNunsVXj_i2AQ5x1mbs   1m        kubelet-bootstrap   Approved,Issued
node-csr-L0wqvr69oy8rzXwFm1u1uNx4aEMOOvd_RWPxaAERn_w   27m       kubelet-bootstrap   Approved,Issued


查看集群节点信息：
[root@k8s-master1 ~]# /opt/kubernetes/bin/kubectl get node
NAME              STATUS    ROLES     AGE       VERSION
192.168.246.164   Ready     <none>    1m        v1.11.10
192.168.246.165   Ready     <none>    17s       v1.11.10



----------------------下面这些操作在node节点完成：---------------------------
mkdir -p /var/lib/kubelet/ /data/logs/kubernetes/



vim /opt/kubernetes/cfg/kubelet-config.yml



在两个node节点创建kubelet配置文件：
[root@k8s-node1 ~]# vim /opt/kubernetes/cfg/kubelet.conf
KUBELET_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/data/logs/kubernetes/kubelet \
--hostname-override=172.28.28.8 \  
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
--config=/opt/kubernetes/cfg/kubelet-config.yml \
--cert-dir=/opt/kubernetes/ssl \
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"




[root@k8s-node1 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0
[root@k8s-node2 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0
参数说明：
* --hostname-override 在集群中显示的主机名
* --kubeconfig 指定kubeconfig文件位置，会自动生成
* --bootstrap-kubeconfig 指定刚才生成的bootstrap.kubeconfig文件
* --cert-dir 颁发证书存放位置
* --pod-infra-container-image 管理Pod网络的镜像


systemd管理kubelet组件：
# vim /usr/lib/systemd/system/kubelet.service 
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet
ExecStart=/opt/kubernetes/bin/kubelet $KUBELET_OPTS
Restart=on-failure
KillMode=process

[Install]
WantedBy=multi-user.target

启动：
 systemctl daemon-reload
 systemctl enable kubelet
 systemctl start kubelet
[root@k8s-master ~]# /opt/kubernetes/bin/kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-F5AQ8SeoyloVrjPuzSbzJnFKQaUsier7EGvNFXLKTqM   17s       kubelet-bootstrap   Pending
node-csr-bjeHSWXOuUDSHganJPL_hDz_8jjYhM2FQyTkbA9pM0Q   18s       kubelet-bootstrap   Pending

在Master审批Node加入集群：
启动后还没加入到集群中，需要手动允许该节点才可以。在Master节点查看请求签名的Node：
[root@k8s-master1 ~]# kubectl certificate approve XXXXID
注意：xxxid 指的是上面的NAME这一列
[root@k8s-master1 ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr--1TVDzcozo7NoOD3WS2t9xLQqNunsVXj_i2AQ5x1mbs   1m        kubelet-bootstrap   Approved,Issued
node-csr-L0wqvr69oy8rzXwFm1u1uNx4aEMOOvd_RWPxaAERn_w   27m       kubelet-bootstrap   Approved,Issued


查看集群节点信息：
[root@k8s-master1 ~]#kubectl get node
NAME              STATUS    ROLES     AGE       VERSION
10.102.4.14   NotReady     <none>    1m        v1.11.10
10.102.4.15   NotReady     <none>    17s       v1.11.10
```

### 3.部署kube-proxy组件

```shell
创建kube-proxy配置文件：还是在所有node节点

[root@k8s-node1 ~]#  vim /opt/kubernetes/cfg/kube-proxy-config.yml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: 10.102.4.14
clusterCIDR: 10.244.0.0/16
mode: "ipvs"


[root@k8s-node1 ~]# vim /opt/kubernetes/cfg/kube-proxy.conf
#❯ cat /opt/kubernetes/cfg/kube-proxy.conf
KUBE_PROXY_OPTS="--logtostderr=false \
 --v=2 \
 --log-dir=/data/logs/kubernetes/kube-proxy \
 --config=/opt/kubernetes/cfg/kube-proxy-config.yml"


systemd管理kube-proxy组件：
# cat /usr/lib/systemd/system/kube-proxy.service 
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

启动：

systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy

在master查看集群状态
[root@k8s-master1 ~]# kubectl get node
NAME              STATUS    ROLES     AGE       VERSION
192.168.246.164   Ready     <none>    19m       v1.11.10
192.168.246.165   Ready     <none>    18m       v1.11.10

查看集群状态
[root@k8s-master1 ~]# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}
=====================================================================================
```





## 安装cni

下载上传cni-plugins-linux-amd64-v1.2.0

官方：https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz

```shell
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.2.0.tgz 
```







## fla

```shell

/opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://10.102.4.13:2379,https://10.102.4.14:2379,https://10.102.4.15:2379" \
put /coreos.com/network/config  '{ "Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'


/opt/etcd/bin/etcdctl \
--ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem  --endpoints="https://10.102.4.13:2379,https://10.102.4.14:2379,https://10.102.4.15:2379" \
set /coreos.com/network/config  '{ "Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'

 ETCDCTL_API=2 /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem --endpoints="https://10.102.4.13:2379,https://10.102.4.14:2379,https://10.102.4.15:2379" cluster-health

 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://10.102.4.13:2379,https://10.102.4.14:2379,https://10.102.4.15:2379" del /coreos.com/network/config
 
 
 ETCDCTL_API=2 /opt/etcd/bin/etcdctl \
--ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem \
--endpoints="https://10.8.165.101:2379,https://10.8.165.102:2379,https://10.8.165.103:2379" \
set /coreos.com/network/config '{ "Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'
 
ETCDCTL_API=2 /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem --endpoints="https://10.102.4.13:2379,https://10.102.4.14:2379,https://10.102.4.15:2379"  set /coreos.com/network/config '{ "Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'
```





## 脚本



```shell
❯ cat shell/deploy.sh                
#! /bin/bash
#set -x

if [ $# -lt 1 ];then
    echo "Usage: `basename $0` [PROJECT]"
    exit 1
fi


PROJECT=$1
N_VERSION=`date +%Y%m%d%H%M%S`
HARBOR="harbor.dyxt.com"

cd /opt/dockerfile/$PROJECT && docker build -t $HARBOR/app/$PROJECT:1.0.0.$N_VERSION . && docker push $HARBOR/app/$PROJECT:1.0.0.$N_VERSION

if [ $PROJECT == "dsfsjcj" ];then
    O_VERSION=`awk -F ':' /image:.*$PROJECT/'{print $NF}' /opt/k8s-yaml/$PROJECT/${PROJECT}.yaml|head -n 1`
    sed -i "s/$O_VERSION/1.0.0.$N_VERSION/g" /opt/k8s-yaml/$PROJECT/${PROJECT}.yaml
    kubectl apply -f /opt/k8s-yaml/$PROJECT/${PROJECT}.yaml
else
    O_VERSION=`awk -F ':' /image:.*$PROJECT/'{print $NF}' /opt/k8s-yaml/$PROJECT/prod/${PROJECT}.yaml|head -n 1`  
    sed -i "s/$O_VERSION/1.0.0.$N_VERSION/g" /opt/k8s-yaml/$PROJECT/prod/${PROJECT}.yaml
    kubectl apply -f /opt/k8s-yaml/$PROJECT/prod/${PROJECT}.yaml
fi

/opt                                                                                                                                                           with root@dy-28-8 at 16:47:24
❯ cat shell/etcd_backup.sh 
#! /bin/bash

CACERT="/opt/etcd/ssl/ca.pem"
CERT="/opt/etcd/ssl/server.pem"
EKY="/opt/etcd/ssl/server-key.pem"
ENDPOINTS="10.102.4.13:2379"

ETCDCTL_API=3 
/opt/etcd/bin/etcdctl \
--cacert="${CACERT}" --cert="${CERT}" --key="${EKY}" \
--endpoints=${ENDPOINTS} \
snapshot save /data/etcd_backup/etcd-snapshot-`date +%Y%m%d`.db

find /data/etcd_backup/ -name "*.db" -mtime +30 -exec rm -f {} \;

```

