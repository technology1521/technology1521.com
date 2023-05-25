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







#下载地址： https://storage.googleapis.com/kubernetes-release/release/v1.19.14/kubernetes-server-linux-amd64.tar.gz

[kubernetes-v1.19.14-server-linux-amd64.tar.gz](https://technology1521.github.io/technology1521.com/soft/kubernetes-v1.19.14-server-linux-amd64.tar.gz)



下载二进制文件

```shell
#创建目录
[root@k8s-master1 ~]# mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}

#解压二进制包
[root@k8s-master1 ~]# tar -zxf kubernetes-v1.19.14-server-linux-amd64.tar.gz

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
[root@k8s-master1 bin]# cp /opt/tls/k8s/ca*pem /opt/tls/k8s/server*pem /opt/kubernetes/ssl/
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
--master=127.0.0.1:8080 \
--bind-address=127.0.0.1 \
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



```shell
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
  --client-certificate=./kube-controller-manager.pem \
  --client-key=./kube-controller-manager-key.pem \
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
ExecStart=/usr/local/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
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
--log-dir=/opt/kubernetes/logs \
--leader-elect \
--kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \
--bind-address=127.0.0.1"

参数说明
•	--kubeconfig：连接apiserver配置文件
•	--leader-elect：当该组件启动多个时，自动选举（HA）
```

生成kubeconfig文件

```shell
[root@k8s-master1 k8s]# KUBE_CONFIG="/opt/kubernetes/cfg/kube-scheduler.kubeconfig"
[root@k8s-master1 k8s]# KUBE_APISERVER="https://192.168.1.20:6443"


·终端执行（4条）
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials kube-scheduler \
  --client-certificate=./kube-scheduler.pem \
  --client-key=./kube-scheduler-key.pem \
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
[root@k8s-master1 k8s]# mkdir /root/.kube

[root@k8s-master1 k8s]# KUBE_CONFIG="/root/.kube/config"
[root@k8s-master1 k8s]# KUBE_APISERVER="https://192.168.1.20:6443"

·终端执行（4条）
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials cluster-admin \
  --client-certificate=./admin.pem \
  --client-key=./admin-key.pem \
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





#若出现下列情况，可按下面操作
[root@k8s-master1 k8s]# kubectl get cs
NAME                 AGE
etcd-0               <unknown>
scheduler            <unknown>
controller-manager   <unknown>
etcd-2               <unknown>
etcd-1               <unknown>

#从1.16开始就显示为unknow 具体原因:https://segmentfault.com/a/1190000020912684


#临时解决办法（通过模板）
[root@k8s-master1 k8s]# kubectl get cs -o=go-template='{{printf "|NAME|STATUS|MESSAGE|\n"}}{{range .items}}{{$name := .metadata.name}}{{range .conditions}}{{printf "|%s|%s|%s|\n" $name .status .message}}{{end}}{{end}}'
|NAME|STATUS|MESSAGE|
|scheduler|True|ok|
|controller-manager|True|ok|
|etcd-1|True|{"health":"true"}|
|etcd-0|True|{"health":"true"}|
|etcd-2|True|{"health":"true"}|

#查看k8s的名称空间
[root@k8s-master1 k8s]# kubectl get ns
NAME              STATUS   AGE
default           Active   3h21m
kube-node-lease   Active   3h21m
kube-public       Active   3h21m
kube-system       Active   3h21m
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
ENDPOINTS="10.102.4.13:2379"

ETCDCTL_API=3 
/opt/etcd/bin/etcdctl \
--cacert="${CACERT}" --cert="${CERT}" --key="${EKY}" \
--endpoints=${ENDPOINTS} \
snapshot save /data/etcd_backup/etcd-snapshot-`date +%Y%m%d`.db

find /data/etcd_backup/ -name "*.db" -mtime +30 -exec rm -f {} \;

```

