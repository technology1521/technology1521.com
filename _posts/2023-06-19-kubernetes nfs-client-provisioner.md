---
layout:  single
title:  "kubernetes nfs-client-provisioner"
date:   2023-07-13
categories:   [kubernetes]
classes: wide
---



## 在 Kubernetes 上部署 NFS Client Provisioner

NFS Client Provisioner 是一个 Kubernetes 存储类的实现，它利用 NFS（Network File System）来动态创建持久卷（Persistent Volume）。在本篇博客中，我们将介绍如何在 Kubernetes 集群上部署和配置 NFS Client Provisioner。

### 步骤一：部署 NFS 服务器

首先，我们需要准备一个 NFS 服务器来提供存储服务。可以使用一个现有的 NFS 服务器或者自己搭建一个。

1. 安装并配置 NFS 服务器：

```
# 在 Ubuntu 上安装 NFS 服务器
sudo apt update
sudo apt install nfs-kernel-server

# 创建共享目录
sudo mkdir /shared_directory

# 配置 NFS 导出
sudo nano /etc/exports
# 在文件中添加以下行
/shared_directory *(rw,sync,no_subtree_check)

# 重新加载 NFS 配置
sudo exportfs -a

# 启动 NFS 服务
sudo systemctl start nfs-kernel-server
sudo systemctl enable nfs-kernel-server
```

确保将 `/shared_directory` 替换为实际的共享目录路径，并根据需要调整其他配置。

1. 验证 NFS 服务器是否正常运行：

```
shellCopy code# 检查 NFS 服务状态
sudo systemctl status nfs-kernel-server

# 确保 NFS 导出可用
showmount -e localhost
```

确保显示出 `/shared_directory` 的导出信息。

更多内容可参考作者[NFS介绍](https://blog.csdn.net/qq_42704442/article/details/130060542?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522168923550516800182149714%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=168923550516800182149714&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-130060542-null-null.268),或者官方网站内容

### 步骤二：部署 NFS Client Provisioner

在部署 NFS Client Provisioner 之前，确保你的 Kubernetes 集群已经配置好，并且具有可用的存储类。

- 编写rbac.yaml文件,进行权限分配

```shell
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

- 编写storageclass.yaml文件

```shell
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: "k8s-sigs.io/nfs-subdir-external-provisioner"                 
parameters:
  archiveOnDelete: "true"
```

- 编写deployment.yaml文件

  > 修改配置
  > 将 <nfs-server-ip> 替换为 NFS 服务器的 IP 地址
  > 将 <shared-directory> 替换为 NFS 服务器上的共享目录路径
  > 可根据需要修改其他配置

```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: kube-system
  labels:
    app: nfs-client-provisioner
spec:
  replicas: 1
  strategy: 
    type: Recreate                   ## 设置升级策略为删除再创建(默认为滚动更新)
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
      - name: nfs-client-provisioner
        image: eipwork/nfs-subdir-external-provisioner:4.0.2
        volumeMounts:
        - name: nfs-client-root
          mountPath: /persistentvolumes
        env:
        - name: PROVISIONER_NAME 
          value: k8s-sigs.io/nfs-subdir-external-provisioner
        # 设置高可用允许选举
        #- name: ENABLE_LEADER_ELECTION
        #  value: "True"
        - name: NFS_SERVER 
          value: 192.168.0.80
        - name: NFS_PATH 
          value: /data/nfs
      imagePullSecrets:
      - name: default-secret
      volumes:
      - name: nfs-client-root
        nfs:
          server: nfs-server-ip
          path: shared-directory
```



1. 部署 NFS Client Provisioner：

```
kubectl apply -f rbac.yaml
kubectl apply -f storageclass.yaml
kubectl apply -f deployment.yaml
```

1. 验证部署是否成功：

```shell
kubectl get pods -n  kube-system
```

确保 NFS Client Provisioner 的 Pod 运行正常。



### 步骤三：创建持久卷

现在，你可以创建使用 NFS Client Provisioner 的持久卷了。下面是一个示例的持久卷声明 YAML 文件的例子 pvc.yaml：

```shell
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Mi

```

根据需要，调整存储容量、访问模式等配置。然后，使用 `kubectl apply -f pvc.yaml` 命令创建持久卷声明。

测试挂载卷，编写pod.yaml

```shell
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: busybox:latest
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"  ## 创建一个名称为"SUCCESS"的文件
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-pvc
```

根据需要，调整镜像、测试命令、挂载路径等配置。然后，使用 `kubectl apply -f pod.yaml` 命令创建持久卷声明。





### 结论

通过部署和配置 NFS Client Provisioner，你可以在 Kubernetes 集群中轻松创建动态的 NFS 持久卷。这提供了一种方便且可扩展的方式来管理存储，并为应用程序提供持久化的存储支持。

希望这篇博客对你部署 NFS Client Provisioner 有所帮助！

参考资源：

- [Kubernetes NFS Client Provisioner GitHub 仓库](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)
- [Kubernetes NFS 存储类文档](https://kubernetes.io/docs/concepts/storage/storage-classes/#nfs)
- [NFS 官方文档](https://nfs.sourceforge.io/)