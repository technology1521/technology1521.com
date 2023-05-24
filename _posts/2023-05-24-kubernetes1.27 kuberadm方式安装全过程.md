---
layout:  single
title:  "kubernetes1.27 kuberadm方式安装全过程"
date:   2023-05-24 
categories:   [kubernetes]
classes: wide
---



﻿[TOC]



## 配置hosts

```shell
#master
"10.102.4.13 master" >> /etc/hosts
#node1
"10.102.4.14 node1" >> /etc/hosts
#node2
"10.102.4.15 node2" >> /etc/hosts

```

## 安装 kubeadm、kubelet 和 kubectl(三台都要)

```shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-\$basearch

enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet


```

## 容器运行时（三台都要）

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

通过运行以下指令确认 `br_netfilter` 和 `overlay` 模块被加载：

```shell
lsmod | grep br_netfilter
lsmod | grep overlay
```

通过运行以下指令确认 `net.bridge.bridge-nf-call-iptables`、`net.bridge.bridge-nf-call-ip6tables` 和 `net.ipv4.ip_forward` 系统变量在你的 `sysctl` 配置中被设置为 1：

```shell
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

下载上传runc.amd64

官方：https://github.com/opencontainers/runc/releases/download/v1.1.6/runc.amd64

```shell
install -m 755 runc.amd64 /usr/local/sbin/runc
```

下载上传cni-plugins-linux-amd64-v1.2.0

官方：https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz

```shell
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.2.0.tgz 
```

下载上传containerd-1.7.0-linux-amd64.tar.gz

```shell
tar Cxzvf /usr/local containerd-1.7.0-linux-amd64.tar.gz 
```

编辑文件 `vim /lib/systemd/system/containerd.service`

```shell
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
#uncomment to enable the experimental sbservice (sandboxed) version of containerd/cri integration
#Environment="ENABLE_CRI_SANDBOXES=sandboxed"
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```



```shell
systemctl daemon-reload
systemctl enable --now containerd
```



```shell
mkdir -p /etc/containerd/
containerd config default >/etc/containerd/config.toml
```



```shell
vim /etc/containerd/config.toml

59             sandbox_image = "registry.k8s.io/pause:3.8"
137            SystemdCgroup = false
改为：
59             sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
137            SystemdCgroup = true


```



```shell
 systemctl restart containerd
 crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
 crictl config image-endpoint unix:///var/run/containerd/containerd.sock
```





```shell
cat /etc/crictl.yaml
 
crictl version
```

## kubeadm init 配置类型(master)

> 以下是参数分析

```shell
kubeadm config print init-defaults --component-configs KubeletConfiguration
```

官网：https://kubernetes.io/zh-cn/docs/reference/config-api/kubeadm-config.v1beta3/#kubeadm-k8s-io-v1beta3-APIEndpoint

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
bootstrapTokens:                           #BootstrapToken 描述的是一个启动引导令牌，以 Secret 形式存储在集群中
- groups:                                  #groups 设定此令牌被用于身份认证时对应的附加用户组。
  - system:bootstrappers:kubeadm:default-node-token  #用户组为bootstrappers:kubeadm:default-node-token  
  token: abcdef.0123456789abcdef #token 用来在节点与控制面之间建立双向的信任关系。 在向集群中添加节点时使用。
  ttl: 24h0m0s           #ttl 定义此令牌的声明周期。默认为 24h。 expires 和 ttl 是互斥的。
 #expires:              #expires 设置此令牌过期的时间戳。默认为在运行时基于 ttl 来决定。 expires 和 ttl 是互斥的
  usages:               #usages 描述此令牌的可能使用方式。默认情况下，令牌可用于 建立双向的信任关系；不过这里可以改变默认用途。
  - signing
  - authentication
localAPIEndpoint:      #localAPIEndpoint 所代表的的是在此控制面节点上要部署的 API 服务器 的端点。在高可用（HA）配置中，此字段与 ClusterConfiguration.controlPlaneEndpoint 的取值不同：后者代表的是整个集群的全局端点，该端点上的请求会被负载均衡到每个 API 服务器。 此配置对象允许你定制本地 API 服务器所公布的、可访问的 IP/DNS 名称和端口。 默认情况下，kubeadm 会尝试自动检测默认接口上的 IP 并使用该地址。 不过，如果这种检测失败，你可以在此字段中直接设置所期望的值。
  advertiseAddress: 1.2.3.4              #advertiseAddress 设置 API 服务器要公布的 IP 地址。
  bindPort: 6443                        #bindPort 设置 API 服务器要绑定到的安全端口。默认值为 6443。
nodeRegistration:                       #nodeRegistration 包含与向集群注册控制面节点相关的字段。
  criSocket: unix:///var/run/containerd/containerd.sock  #criSocket 用来读取容器运行时的信息。 此信息会被以注解的方式添加到 Node API 对象至上，用于后续用途。
  imagePullPolicy: IfNotPresent        #imagePullPolicy 设定 "kubeadm init" 和 "kubeadm join" 操作期间的镜像拉取策略。此字段的取值可以是 "Always"、"IfNotPresent" 或 "Never" 之一。 若此字段未设置，则 kubeadm 使用 "IfNotPresent" 作为其默认值，换言之， 当镜像在主机上不存在时才执行拉取操作。
  name: node                         #name 是 Node API 对象的 .metadata.name 字段值； 该 API 对象会在此 kubeadm init 或 kubeadm join 操作期间创建。 在提交给 API 服务器的 kubelet 客户端证书中，此字段也用作其 CommonName。 如果未指定则默认为节点的主机名。                 
  taints: null                      #taints 设定 Node API 对象被注册时要附带的污点。 若未设置此字段（即字段值为 null），默认为控制平面节点添加控制平面污点。 如果你不想污染你的控制平面节点，可以将此字段设置为空列表（即 YAML 文件中的 taints: []）， 这个字段只用于节点注册。
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration              #ClusterConfiguration 包含一个 kubadm 集群的集群范围配置信息
apiServer:                          #apiServer 包含 API 服务器的一些额外配置。
  timeoutForControlPlane: 4m0s      #timeoutForControlPlane 用来控制我们等待 API 服务器开始运行的超时时间。
certificatesDir: /etc/kubernetes/pki  #certificatesDir 设置在何处存放或者查找所需证书。
clusterName: kubernetes             #集群名称。
controllerManager: {}          #controllerManager 中包含控制器管理器的额外配置。
dns: {}                        #dns 定义在集群中安装的 DNS 插件的选项。
etcd:                         #etcd 中包含 etcd 服务的配置。
  local:
    dataDir: /var/lib/etcd     #local 提供配置本地 etcd 实例的选项。local 和 external 是互斥的。
imageRepository: registry.k8s.io  #imageRepository 设置用来拉取镜像的容器仓库。 如果此字段为空，默认使用 registry.k8s.io。 当 Kubernetes 用来执行 CI 构造时（Kubernetes 版本以 ci/ 开头）， 将默认使用 gcr.io/k8s-staging-ci-images 来拉取控制面组件镜像， 而使用 registry.k8s.io 来拉取所有其他镜像。
kubernetesVersion: 1.27.0           #kubernetesVersion 设置控制面的目标版本。
networking:                        #networking 字段包含集群的网络拓扑配置。
  dnsDomain: cluster.local        #dnsDomain 是 Kubernetes 服务所使用的的 DNS 域名。 默认值为 "cluster.local"
  serviceSubnet: 10.96.0.0/12    #serviceSubnet 是 Kubernetes 服务所使用的的子网。 默认值为 "10.96.0.0/12"。
  #podSubnet: 10.96.0.0/12      #podSubnet 为 Pod 所使用的子网。
scheduler: {}                   #scheduler 包含调度器的额外配置。
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:       #authorization设置发送给 kubelet 服务器的请求是如何进行身份认证的
  anonymous:            #anonymous包含与匿名身份认证相关的配置信息。
    enabled: false        #enabled是否允许匿名用户向 kubelet 服务器发送请求。 未被其他身份认证方法拒绝的请求都会被当做匿名请求。 匿名请求对应的用户名为system:anonymous，对应的用户组名为 system:unauthenticated。
  webhook:               #webhook包含与 Webhook 持有者令牌认证相关的配置。
    cacheTTL: 0s        #cacheTTL启用对身份认证结果的缓存。
    enabled: true       #enabled允许使用tokenreviews.authentication.k8s.io API 来提供持有者令牌身份认证
  x509:                #x509包含与 x509 客户端证书认证相关的配置。
    clientCAFile: /etc/kubernetes/pki/ca.crt #clientCAFile是一个指向 PEM 编发的证书包的路径。 如果设置了此字段，则能够提供由此证书包中机构之一所签名的客户端证书的请求会被成功认证， 并且其用户名对应于客户端证书的CommonName、组名对应于客户端证书的 Organization。
authorization:
  mode: Webhook       #mode>是应用到 kubelet 服务器所接收到的请求上的鉴权模式。合法值包括 AlwaysAllow和Webhook。 Webhook 模式使用 SubjectAccessReview API 来确定鉴权。
  webhook:           #webhook包含与 Webhook 鉴权相关的配置信息。
    cacheAuthorizedTTL: 0s   #cacheAuthorizedTTL设置来自 Webhook 鉴权组件的 'authorized' 响应的缓存时长
    cacheUnauthorizedTTL: 0s  #cacheUnauthorizedTTL设置来自 Webhook 鉴权组件的 'unauthorized' 响应的缓存时长。
cgroupDriver: systemd      #cgroupDriver是 kubelet 用来操控宿主系统上控制组（CGroup） 的驱动程序（cgroupfs 或 systemd）。默认值："cgroupfs"
clusterDNS:     #clusterDNS是集群 DNS 服务器的 IP 地址的列表。 如果设置了，kubelet 将会配置所有容器使用这里的 IP 地址而不是宿主系统上的 DNS 服务器来完成 DNS 解析。
- 10.96.0.10
clusterDomain: cluster.local    #clusterDomain是集群的 DNS 域名。如果设置了此字段，kubelet 会配置所有容器，使之在搜索主机的搜索域的同时也搜索这里指定的 DNS 域。
containerRuntimeEndpoint: ""
cpuManagerReconcilePeriod: 0s      #cpuManagerReconcilePeriod是 CPU 管理器的协调周期时长。 需要启用CPUManager特性门控。默认值："10s"
evictionPressureTransitionPeriod: 0s #evictionPressureTransitionPeriod设置 kubelet 离开驱逐压力状况之前必须要等待的时长。
fileCheckFrequency: 0s   #fileCheckFrequency 是对配置文件中新数据进行检查的时间间隔值。
healthzBindAddress: 127.0.0.1  #healthzBindAddress是healthz服务器用来提供服务的 IP 地址。
healthzPort: 10248             #healthzPort是本地主机上提供healthz端点的端口 （设置值为 0 时表示禁止）。合法值介于 1 和 65535 之间。
httpCheckFrequency: 0s       #httpCheckFrequency 是对 HTTP 服务器上新数据进行检查的时间间隔值。
imageMinimumGCAge: 0s      #imageMinimumGCAge是对未使用镜像进行垃圾搜集之前允许其存在的时长。
logging:                   #logging设置日志机制选项
  flushFrequency: 0     #对日志进行清洗的最大间隔纳秒数（例如，1s = 1000000000）。 如果所选的日志后端在写入日志消息时不提供缓存，则此配置会被忽略。
  options:
    json:               #[Alpha] JSON 包含记录 "json" 格式日志的选项。 只有 LoggingAlphaOptions 特性门控被启用时才可用。
      infoBufferSize: "0" #Alpha] infoBufferSize 在分离数据流时用来设置提示数据流的大小。 默认值为 0，相当于禁止缓存。只有 LoggingAlphaOptions 特性门控被启用时才可用。
  verbosity: 0          #verbosity 用来确定日志消息记录的详细程度阈值。默认值为 0， 意味着仅记录最重要的消息。数值越大，额外的消息越多。出错消息总是会被记录下来。
memorySwap: {}      #memorySwap配置容器负载可用的交换内存。
nodeStatusReportFrequency: 0s #nodeStatusReportFrequency是节点状态未发生变化时，kubelet 向控制面更新节点状态的频率。如果节点状态发生变化，则 kubelet 会忽略这一频率设置， 立即更新节点状态。
#此字段仅当启用了节点租约特性时才被使用。nodeStatusReportFrequency 的默认值是"5m"。不过，如果nodeStatusUpdateFrequency 被显式设置了，则nodeStatusReportFrequency的默认值会等于 nodeStatusUpdateFrequency值，这是为了实现向后兼容。
nodeStatusUpdateFrequency: 0s #nodeStatusUpdateFrequency是 kubelet 计算节点状态的频率。 如果未启用节点租约特性，这一字段设置的也是 kubelet 向控制面投递节点状态的频率。
#注意：如果节点租约特性未被启用，更改此参数设置时要非常小心， 所设置的参数值必须与节点控制器的nodeMonitorGracePeriod协同。
rotateCertificates: true   #rotateCertificates用来启用客户端证书轮换。kubelet 会调用 certificates.k8s.io API 来请求新的证书。需要有一个批复人批准证书签名请求。
runtimeRequestTimeout: 0s     #runtimeRequestTimeout用来设置除长期运行的请求（pull、 logs、exec和attach）之外所有运行时请求的超时时长。
shutdownGracePeriod: 0s  #shutdownGracePeriod设置节点关闭期间，节点自身需要延迟以及为 Pod 提供的宽限期限的总时长
shutdownGracePeriodCriticalPods: 0s #shutdownGracePeriodCriticalPods设置节点关闭期间用来终止关键性 Pod 的时长。此时长要短于shutdownGracePeriod。 例如，如果shutdownGracePeriod=30s，shutdownGracePeriodCriticalPods=10s， 在节点关闭期间，前 20 秒钟被预留用来体面终止普通 Pod，后 10 秒钟用来终止关键 Pod。
staticPodPath: /etc/kubernetes/manifests #staticPodPath 是指向要运行的本地（静态）Pod 的目录， 或者指向某个静态 Pod 文件的路径
streamingConnectionIdleTimeout: 0s #streamingConnectionIdleTimeout设置流式连接在被自动关闭之前可以空闲的最长时间
syncFrequency: 0s #syncFrequency 是对运行中的容器和配置进行同步的最长周期
volumeStatsAggPeriod: 0s # volumeStatsAggPeriod是计算和缓存所有 Pod 磁盘用量的频率。

```



```shell
kubeadm config print init-defaults --component-configs KubeProxyConfiguration


```

```yaml

apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
bindAddress: 0.0.0.0        #bindAddress 字段是代理服务器提供服务时所用 IP 地址（设置为 0.0.0.0 时意味着在所有网络接口上提供服务）。
bindAddressHardFail: false  #bindAddressHardFail 字段设置为 true 时， kube-proxy 将无法绑定到某端口这类问题视为致命错误并直接退出。
clientConnection:         #clientConnection 字段给出代理服务器与 API 服务器通信时要使用的 kubeconfig 文件和客户端链接设置    
  acceptContentTypes: ""   #acceptContentTypes 字段定义客户端在连接到服务器时所发送的 Accept 头部字段。 此设置值会覆盖默认配置 'application/json'。 此字段会控制某特定客户端与指定服务器的所有链接。
  burst: 0                 # burst 字段允许客户端超出其速率限制时可以临时累积的额外查询个数。
  contentType: ""           #   contentType 字段是从此客户端向服务器发送数据时使用的内容类型（Content Type）。
  kubeconfig: /var/lib/kube-proxy/kubeconfig.conf     #
  qps: 0               #qps 字段控制此连接上每秒钟可以发送的查询请求个数。
clusterCIDR: ""          #clusterCIDR 字段是集群中 Pod 所使用的 CIDR 范围。 这一地址范围用于对来自集群外的请求流量进行桥接。 如果未设置，则 kube-proxy 不会对非集群内部的流量做桥接
configSyncPeriod: 0s     #configSyncPeriod 字段是从 API 服务器刷新配置的频率。此值必须大于 0。
conntrack:              #conntrack 字段包含与 conntrack 相关的配置选项。
  maxPerCore: null        #maxPerCore 字段是每个 CPU 核所跟踪的 NAT 链接个数上限 （0 意味着保留当前上限限制并忽略 min 字段设置值）。
  min: null               #min 字段给出要分配的链接跟踪记录个数下限。 设置此值时会忽略 maxPerCore 的值（将 maxPerCore 设置为 0 时不会调整上限值）。
  tcpCloseWaitTimeout: null #tcpCloseWaitTimeout 字段用来设置空闲的、处于 CLOSE_WAIT 状态的 conntrack 条目 保留在 conntrack 表中的时间长度（例如，'60s'）。 此设置值必须大于 0。
  tcpEstablishedTimeout: null #tcpEstablishedTimeout 字段给出空闲 TCP 连接的保留时间（例如，'2s'）。 此值必须大于 0。
detectLocal:           #
  bridgeInterface: ""    #
  interfaceNamePrefix: ""   #
detectLocalMode: ""           #
enableProfiling: false       #
healthzBindAddress: ""       #
hostnameOverride: ""      #
iptables:               #
  localhostNodePorts: null #
  masqueradeAll: false     #
  masqueradeBit: null    #    
  minSyncPeriod: 0s        #
  syncPeriod: 0s         #
ipvs:                    #
  excludeCIDRs: null #
  minSyncPeriod: 0s  #
  scheduler: ""     #
  strictARP: false  #
  syncPeriod: 0s #
  tcpFinTimeout: 0s #
  tcpTimeout: 0s #
  udpTimeout: 0s #
metricsBindAddress: "" #
mode: ""           #mode 字段用来设置将使用的代理模式。
nodePortAddresses: null #
oomScoreAdj: null  #
portRange: "" #
showHiddenMetricsForVersion: "" #
winkernel: #
  enableDSR: false #
  forwardHealthCheckVip: false #
  networkName: "" #
  rootHnsEndpointName: "" #
  sourceVip: "" #

```



当带有 `--config` 选项来执行 kubeadm init 命令时，可以使用下面的配置类型： `InitConfiguration`、`ClusterConfiguration`、`KubeProxyConfiguration`、`KubeletConfiguration`， 但 `InitConfiguration` 和 `ClusterConfiguration` 之间只有一个是必须提供的。

> 以下是正式应用

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.102.4.13  #改变为自己master机器上的IP（内网IP，不是弹性公网的）
  bindPort: 6443                 #apisever的端口，没改，想改可以改
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock #这个没改但是你要用RIO就要改了
  taints: [] #控制节点也调度
---
 apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
etcd:
  local:
    dataDir: /opt/etcd          #改变etcd地址
imageRepository: registry.aliyuncs.com/google_containers #改变镜像源
kubernetesVersion: 1.27.0    #版本号没改
networking:
  serviceSubnet: 10.96.0.0/12      #集群内部地址域没改
  podSubnet: 172.168.0.0/16        #pod域
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
clusterDNS:
- 10.96.0.20
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
```



```shell
kubeadm config images pull --config kubeadm.yaml
kubeadm init --config kubeadm.yaml
```



```shell
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.102.4.13:6443 --token k6obb9.kl6xqu5sdjgwxu0e \
	--discovery-token-ca-cert-hash sha256:3b6fcc42fc736a0d2b***********************************3dd1
```



```shell
export KUBECONFIG=/etc/kubernetes/admin.conf
```

## 配置CNI（master）

下载上传

```shell
https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml
https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml

kubectl create -f custom-resources.yaml
kubectl create -f tigera-operator.yaml

```

custom-resources.yaml

```yaml
# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 172.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer 
metadata: 
  name: default 
spec: {}

```

查看结果

```shell
 kubectl  get pod -n  calico-system
 kubectl  get nodes
```





## node 节点加入

```
kubeadm join 10.102.4.13:6443 --token k6obb9.kl6xqu5sdjgwxu0e \
	--discovery-token-ca-cert-hash sha256:3b6fcc42fc736a0d2b***********************************3dd1
```





