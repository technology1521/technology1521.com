# 在 Kubernetes 上部署 Nexus

Nexus 是一个流行的仓库管理系统，用于存储和管理软件包、容器镜像等资源。它提供了强大的功能，包括快速部署、安全性和可扩展性。本篇博客将详细介绍如何在 Kubernetes 集群上手动部署和配置 Nexus。

## 准备工作

在开始之前，确保以下准备工作已完成：

- 一个运行的 Kubernetes 集群
- 已安装并配置好 kubectl 命令行工具

## 步骤一：部署 Nexus

1. 创建一个名为 `nexus-deployment.yaml` 的 YAML 文件，并添加以下内容：

```
yamlCopy codeapiVersion: apps/v1
kind: Deployment
metadata:
  name: nexus-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nexus
  template:
    metadata:
      labels:
        app: nexus
    spec:
      containers:
        - name: nexus
          image: sonatype/nexus3:latest
          ports:
            - containerPort: 8081
          volumeMounts:
            - name: nexus-data
              mountPath: /nexus-data
      volumes:
        - name: nexus-data
          emptyDir: {}
```

该 YAML 文件定义了一个 Deployment，使用 Sonatype 提供的 Nexus3 镜像。它将容器的 8081 端口映射到主机上。

1. 使用 kubectl 命令创建 Nexus Deployment：

```
shellCopy code
kubectl apply -f nexus-deployment.yaml
```

1. 创建一个名为 `nexus-service.yaml` 的 YAML 文件，并添加以下内容：

```
yamlCopy codeapiVersion: v1
kind: Service
metadata:
  name: nexus-service
spec:
  type: LoadBalancer
  ports:
    - port: 8081
      targetPort: 8081
  selector:
    app: nexus
```

该 YAML 文件定义了一个 LoadBalancer 类型的 Service，将流量导向 Nexus Deployment 的 Pod。

1. 使用 kubectl 命令创建 Nexus Service：

```
shellCopy code
kubectl apply -f nexus-service.yaml
```

1. 等待 Nexus 部署完成，并获取外部 IP 地址：

```
shellCopy code
kubectl get services
```

找到名称为 "nexus-service" 的 Service，并记录其外部 IP 地址。

## 步骤二：访问 Nexus

1. 在浏览器中访问 Nexus：

```
arduinoCopy code
http://<external-ip>:8081
```

将 `<external-ip>` 替换为上一步获取到的外部 IP 地址。

1. 完成 Nexus 首次设置：
   - 设置管理员密码
   - 配置存储库和仓库组

## 结论

通过手动部署和配置 Nexus，你可以在 Kubernetes 集群中快速搭建一个强大的仓库管理系统。请根据你的需求和实际环境进行自定义配置和扩展。

希望这篇博客对你有所帮助！如有其他问题，请随时提问或参考相关文档和资源。

参考资源：

- [Nexus 官方文档](https://help.sonatype.com/repomanager3)