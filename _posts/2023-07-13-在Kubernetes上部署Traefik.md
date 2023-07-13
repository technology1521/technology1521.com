# 在 Kubernetes 上部署 Traefik

[TOC]



> Kubernetes版本 v1.19.14

Traefik 是一个流行的开源反向代理和负载均衡工具，专为容器化应用设计。它具有自动发现、动态配置和路由管理等功能，使得在 Kubernetes 中部署和管理应用变得更加简单和灵活。本篇博客将详细介绍如何在 Kubernetes 集群上手动部署和配置 Traefik。

## 步骤一：部署 Traefik Ingress Controller

- 创建一个名为 `traefik-config.yaml ` 的 YAML 文件，并添加以下内容（可以根据提示添加、更改想要得选项）：

```shell
kind: ConfigMap
apiVersion: v1
metadata:
  name: traefik-config
  namespace: kube-system
data:
  traefik.yaml: |-
    ping: ""                    ## 启用 Ping
    serversTransport:
      insecureSkipVerify: true  ## Traefik 忽略验证代理服务的 TLS 证书
    api:
      insecure: true            ## 允许 HTTP 方式访问 API
      dashboard: true           ## 启用 Dashboard
      debug: false              ## 启用 Debug 调试模式
    metrics:
      prometheus: ""            ## 配置 Prometheus 监控指标数据，并使用默认配置
    entryPoints:
      web:
        address: ":80"          ## 配置 80 端口，并设置入口名称为 web
      websecure:
        address: ":443"         ## 配置 443 端口，并设置入口名称为 websecure
    providers:
      kubernetesCRD: ""         ## 启用 Kubernetes CRD 方式来配置路由规则
      kubernetesIngress: ""     ## 启动 Kubernetes Ingress 方式来配置路由规则
    log:
      filePath: ""              ## 设置调试日志文件存储路径，如果为空则输出到控制台
      level: info               ## 设置调试日志级别
      format: json              ## 设置调试日志格式
    accessLog:
      filePath: ""              ## 设置访问日志文件存储路径，如果为空则输出到控制台
      format: json              ## 设置访问调试日志格式
      bufferingSize: 0          ## 设置访问日志缓存行数
      filters:
        #statusCodes: ["200"]   ## 设置只保留指定状态码范围内的访问日志
        retryAttempts: true     ## 设置代理访问重试失败时，保留访问日志
        minDuration: 20         ## 设置保留请求时间超过指定持续时间的访问日志
      fields:                   ## 设置访问日志中的字段是否保留（keep 保留、drop 不保留）
        defaultMode: keep       ## 设置默认保留访问日志字段
        names:                  ## 针对访问日志特别字段特别配置保留模式
          ClientUsername: drop  
        headers:                ## 设置 Header 中字段是否保留
          defaultMode: keep     ## 设置默认保留 Header 中字段
          names:                ## 针对 Header 中特别字段特别配置保留模式
            User-Agent: redact
            Authorization: drop
            Content-Type: keep

```

- 创建一个名为 ` traefik-crd.yaml ` 的 YAML 文件，并添加以下内容（不懂就别改）：

```shell
## IngressRoute
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
---
## IngressRouteTCP
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
---
## Middleware
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
---
## TLSOption
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
---
## TraefikService
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TraefikService
    plural: traefikservices
    singular: traefikservice

```

- 创建一个名为 `traefik-rbac.yaml ` 的 YAML 文件，并添加以下内容（不懂就别改）：

```shell
## ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: kube-system
  name: traefik-ingress-controller
---
## ClusterRole
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups: [""]
    resources: ["services","endpoints","secrets"]
    verbs: ["get","list","watch"]
  - apiGroups: ["extensions"]
    resources: ["ingresses"]
    verbs: ["get","list","watch"]
  - apiGroups: ["extensions"]
    resources: ["ingresses/status"]
    verbs: ["update"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["middlewares"]
    verbs: ["get","list","watch"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["ingressroutes"]
    verbs: ["get","list","watch"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["ingressroutetcps"]
    verbs: ["get","list","watch"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["tlsoptions"]
    verbs: ["get","list","watch"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["traefikservices"]
    verbs: ["get","list","watch"]
---
## ClusterRoleBinding
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: kube-system

```

- 创建一个名为 `traefik-svc.yaml ` 的 YAML 文件，并添加以下内容（可以根据提示添加、更改想要得选项）：

```shell
apiVersion: v1
kind: Service
metadata:
  name: traefik
  namespace: kube-system
spec:
  ports:
    - name: web
      port: 80
    - name: websecure
      port: 443
  selector:
    app: traefik
```

这个 YAML 文件定义了一个 Service 对象，用于公开 Traefik Ingress Controller 的服务。它使用负载均衡器类型的 Service，并将 HTTP 和 HTTPS 的端口映射到 Traefik Ingress Controller 的端口。

- 创建一个名为 `traefik-deployment.yaml` 的 YAML 文件，并添加以下内容：

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    app: traefik
spec:
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      name: traefik
      labels:
        app: traefik
      annotations:
        prometheus_io_path: "/metrics"
        prometheus_io_port: "8080"
        prometheus_io_scheme: "traefik"
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 1
      containers:
        - image: traefik:v2.1.9
          name: traefik-ingress-lb
          ports:
            - name: web
              containerPort: 80
              hostPort: 80         ## 将容器端口绑定所在服务器的 80 端口
            - name: websecure
              containerPort: 443
              hostPort: 443        ## 将容器端口绑定所在服务器的 443 端口
            - name: admin
              containerPort: 8080  ## Traefik Dashboard 端口
          resources:
            limits:
              cpu: 2000m
              memory: 1024Mi
            requests:
              cpu: 1000m
              memory: 1024Mi
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
          args:
            - --configfile=/config/traefik.yaml
          volumeMounts:
            - mountPath: "/config"
              name: "config"
      volumes:
        - name: config
          configMap:
            name: traefik-config 
      tolerations:              ## 设置容忍所有污点，防止节点被设置污点
        - operator: "Exists"
      nodeSelector:             ## 设置node筛选器，在特定label的节点上启动
        IngressProxy: "true"
```

这个 YAML 文件定义了一个 Deployment 对象，用于部署 Traefik Ingress Controller。它使用 Traefik 的容器镜像，并配置了 HTTP 和 HTTPS 的入口点，以及证书解析器的相关配置。

- 使用 `kubectl` 命令部署 Traefik Ingress Controller：

```
kubectl apply -f traefik-deployment.yaml
```



- 等待一段时间，直到 Traefik Ingress Controller 的 Pod 和 Service 部署完成：

```
kubectl get pods
kubectl get services
```

确保 Traefik 的 Pod 状态为 "Running"，并且 Traefik 的 Service 具有外部 IP 地址。

## 步骤二：配置 Ingress 路由

- 创建一个名为 `traefik-dashboard.yaml ` 的 YAML 文件，并添加以下内容：

```
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard-route
  namespace: kube-system
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`traefik.dyxt.com`)
      kind: Rule
      services:
        - name: traefik
          port: 8080

```

这个 YAML 文件定义了一个 Ingress 对象，用于配置路由规则和将流量路由到后端的服务。它使用了 Traefik 的注解来指定 entrypoints 和中间件。

- 使用 `kubectl` 命令创建 Ingress 路由：

```

kubectl apply -f myapp-ingress.yaml
```

- 验证 Ingress 是否成功创建：

```

kubectl get ingress
```

确保 Ingress 的状态为 "READY"，并且规则正确配置。

## 结论

通过手动部署和配置 Traefik Ingress Controller，你可以在 Kubernetes 集群中实现高级的负载均衡和路由管理。Traefik 提供了强大的自动化和动态配置功能，使得在容器化环境中部署和管理应用变得更加简单和灵活。