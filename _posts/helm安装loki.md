---
layout:  single
title:  "Helm安装loki"
date:   2023-05-24 
categories:   [kubernetes]
classes: wide
---

[TOC]

## 从ceph分设备块给集群存储类使用

从Ceph切出一个块设备，并在Kubernetes中使用PV/PVC方式挂载，以下是详细的步骤：

在Ceph集群上的步骤：

1. 创建一个Ceph存储池：使用以下命令创建一个Ceph存储池（如果你已经有存储池可以跳过此步骤）：

   ```shell
   ceph osd pool create <pool-name> <pg-num>
   ```

   其中，`<pool-name>`是你要创建的存储池的名称，`<pg-num>`是存储池的初始PG数量。

   如：

   ```
   ceph osd pool create kube
   ```

   

2. 创建一个RBD镜像：使用以下命令在Ceph存储池中创建一个RBD镜像：

   ```
   rbd create <image-name> --size <image-size> --pool <pool-name>
   ```

   其中，`<image-name>`是你要创建的RBD镜像的名称，`<image-size>`是镜像的大小（以字节、KB、MB或GB为单位），`<pool-name>`是存储池的名称。

   如：

   ```
   rbd create loki-rdb --size 20Gi --pool kube
   ```

   

在Kubernetes集群上的步骤：

1. 创建一个Kubernetes的Secret：在Kubernetes中，需要创建一个Secret来存储Ceph集群的认证信息。首先，创建一个包含Ceph认证信息的文件，例如 `ceph-secret.yaml`：

   ```
   apiVersion: v1
   kind: Secret
   metadata:
     name: ceph-secret
   type: kubernetes.io/rbd
   data:
     key: BASE64_ENCODED_SECRET_KEY
   ```

   将 `BASE64_ENCODED_SECRET_KEY` 替换为经过Base64编码的Ceph集群密钥。

   如

   ```
   ---
   apiVersion: v1
   kind: Secret
   metadata:
     name: csi-rbd-kube-secret
     namespace: default
   stringData:
     userID: k8s
     userKey: AQCTaVRkFAA5BhAAuFCyZJ3HSM7p5B01ipn6NA==
   
   ```

   

   然后，使用以下命令创建Secret：

   ```
   kubectl apply -f ceph-secret.yaml
   ```

2. 创建一个存储类（StorageClass）：创建一个与Ceph RBD集成的存储类，定义PV的属性和配置。例如，创建一个名为 `ceph-rbd-storageclass.yaml` 的文件：

   ```
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: ceph-rbd-storageclass
   provisioner: rbd.csi.ceph.com
   parameters:
     monitors: <ceph-monitors>
     pool: <pool-name>
     adminSecretName: ceph-secret
     adminSecretNamespace: default
     imageFormat: "2"
     imageFeatures: "layering"
   ```

   将 `<ceph-monitors>` 替换为Ceph监视器的地址（多个地址使用逗号分隔），`<pool-name>` 替换为之前创建的存储池的名称。

   如

   ```yaml
   ---
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
      name: csi-rbd-kube-sc
   provisioner: rbd.csi.ceph.com
   parameters:
      clusterID: c3a92433-8431-491b-8595-f9a455b7b75b
      pool: kube
      imageFeatures: layering
      csi.storage.k8s.io/provisioner-secret-name: csi-rbd-kube-secret
      csi.storage.k8s.io/provisioner-secret-namespace: default
      csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-kube-secret
      csi.storage.k8s.io/controller-expand-secret-namespace: default
      csi.storage.k8s.io/node-stage-secret-name: csi-rbd-kube-secret
      csi.storage.k8s.io/node-stage-secret-namespace: default
   reclaimPolicy: Delete
   allowVolumeExpansion: true
   mountOptions:
      - discard
   ```

   

   使用以下命令创建存储类：

   ```
   
   kubectl apply -f ceph-rbd-storageclass.yaml
   ```




## 修改文件

 cat values.yaml 



```yaml
loki:
  enabled: true
  persistence:
    enabled: true
    storageClassName: csi-rbd-kube-sc
    accessModes:
    - ReadWriteOnce
    size: 20Gi
  config:
    ruler:
      storage:
        type: local
        local:
          directory: /rules
      rule_path: /tmp/scratch
      alertmanager_url: http://alertmanager.kube-system.svc.cluster.local
      ring:
        kvstore:
          store: inmemory
      enable_api: true
  alerting_groups:
  - name: dyxt-exception
    rules:
    - alert: DyxtException
      expr: '{k8s_app="dyxt-tag-default", namespace="prod", pod="dyxt-tag-default-66b56657fd-5wmwt"} |= "Exception"'
      labels:
        severity: critical
      annotations:
        summary: dxyt exception
        description: dxyt exception

promtail:
  enabled: true

fluent-bit:
  enabled: false

grafana:
  enabled: false
  sidecar:
    datasources:
      enabled: true
  image:
    tag: 8.3.5

prometheus:
  enabled: false

filebeat:
  enabled: false
  filebeatConfig:
    filebeat.yml: |
      # logging.level: debug
      filebeat.inputs:
      - type: container
        paths:
          - /var/log/containers/*.log
        processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"
      output.logstash:
        hosts: ["logstash-loki:5044"]

logstash:
  enabled: false
  image: grafana/logstash-output-loki
  imageTag: 1.0.1
  filters:
    main: |-
      filter {
        if [kubernetes] {
          mutate {
            add_field => {
              "container_name" => "%{[kubernetes][container][name]}"
              "namespace" => "%{[kubernetes][namespace]}"
              "pod" => "%{[kubernetes][pod][name]}"
            }
            replace => { "host" => "%{[kubernetes][node][name]}"}
          }
        }
        mutate {
          remove_field => ["tags"]
        }
      }
  outputs:
    main: |-
      output {
        loki {
          url => "http://loki:3100/loki/api/v1/push"
          #username => "test"
          #password => "test"
        }
        # stdout { codec => rubydebug }
      }

```

