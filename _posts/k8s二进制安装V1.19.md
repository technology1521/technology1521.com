---
layout:  single
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
192.168.1.23
hostnamectl set-hostname k8s-master2
```

#### 2.3.2 加域名解析

```shell
##所有机器10.102.4.13-15
vim /etc/hosts

10.102.4.13 k8s-master1
10.102.4.15 k8s-node1
10.102.4.14 k8s-node2
192.168.1.23 k8s-master2
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


#  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-client-csr.json | cfssljson -bare kube-client
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes proxy-client-csr.json | cfssljson -bare proxy-client


会生成server.pem和server-key.pem文件
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


```shell
❯ cat kube-proxy-csr.json
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
      "L": "ZheJiang",
      "ST": "HangZhou",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

```shell
❯ cat proxy-client-csr.json 
{
  "CN": "aggregator",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ZheJiang",
      "L": "HangZhou",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

```



















## 安装cni

下载上传cni-plugins-linux-amd64-v1.2.0

官方：https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz

```shell
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.2.0.tgz 
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
ENDPOINTS="172.28.28.8:2379"

ETCDCTL_API=3 
/opt/etcd/bin/etcdctl \
--cacert="${CACERT}" --cert="${CERT}" --key="${EKY}" \
--endpoints=${ENDPOINTS} \
snapshot save /data/etcd_backup/etcd-snapshot-`date +%Y%m%d`.db

find /data/etcd_backup/ -name "*.db" -mtime +30 -exec rm -f {} \;

```

