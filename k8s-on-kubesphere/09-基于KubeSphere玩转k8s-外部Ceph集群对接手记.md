# 基于 KubeSphere 玩转 k8s-Ceph 之 Ceph-deploy 安装手记

**大家好，我是老 Z！**

> 本系列文档是我在云原生技术领域的学习和运维实践的手记，**用输出倒逼输入**是一种高效的学习方法，能够快速积累经验和提高技术，只有把学到的知识写出来并能够让其他人理解，才能说明真正掌握了这项知识。
>
> 如果你喜欢本文，请分享给你的小伙伴！

**本系列文档内容涵盖 (但不限于) 以下技术领域：**

> - **KubeSphere**
>
> - **Kubernetes**
>
> - **Ansible**
>
> - **自动化运维**
>
> - **CNCF 技术栈**

## 1. 本文简介

本文接着上一篇 << 基于 KubeSphere 玩转 k8s-Ceph 安装手记 >>，继续实战 Kubernetes 对接外部 Ceph 集群。

本文简要介绍了 Kubernetes 对接已有 Ceph 集群的几种方案，重点演示了现在主流选用的 Ceph-CSI 方案的详细操作过程。

本文适用于学习测试环境，只是采用官方默认配置详细演示了 Kubernetes 对接已有 Ceph 集群的全部操作流程。

> **本文知识量**

- 阅读时长：5 分
- 行：500+
- 单词：1900+
- 字符：12600+
- 图片：0 张

> **本文知识点**

- 定级：**入门级**
- CSI RBD
- Ceph 的基础操作
- Kubernetes 对接 Ceph

> **演示服务器配置**

|     主机名      |      IP      | CPU  | 内存 | 系统盘 | 数据盘 |               用途               |
| :-------------: | :----------: | :--: | :--: | :----: | :----: | :------------------------------: |
|  zdeops-master  | 192.168.9.9  |  2   |  4   |   40   |  200   |       Ansible 运维控制节点        |
| ks-k8s-master-0 | 192.168.9.91 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-1 | 192.168.9.92 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-2 | 192.168.9.93 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
|   ceph-node-0   | 192.168.9.85 |  4   |  8   |   40   |  100   |               Ceph               |
|   ceph-node-1   | 192.168.9.86 |  4   |  8   |   40   |  100   |               Ceph               |
|   ceph-node-2   | 192.168.9.87 |  4   |  8   |   40   |  100   |               Ceph               |

> **演示环境涉及软件版本信息**

- 操作系统：**CentOS-7.9-x86_64**
- KubeSphere：**3.2.1**
- Ceph：**Octopus(15.2.16 )** 
- Ansible：**2.8.20**
- Ceph-CSI：**v3.6.1**

## 2. K8S 持久化存储对接 Ceph 介绍

### 2.1. 支持的对接方式

- Ceph RBD
  - Kubernetes 内置的 in-tree storage plugin
  - Ceph RBD only works on Kubernetes with **hyperkube** images, and **hyperkube** images were [deprecated since Kubernetes 1.17](https://github.com/kubernetes/kubernetes/pull/85094)
- rbd-provisioner
  - 类似于 Ceph RBD 但是属于 out-tree
  - `rbd-provisioner` is an out-of-tree dynamic provisioner for Kubernetes 1.5+
  - KubeSphere 图形化界面选择了 **rbd-provisioner**
  - 折腾了两天没搞定，一堆报错，改造成本太高，相关插件都是三年前开发的了，彻底放弃了
  - 愿意折腾的可以参考[rbd-provisioner 官方网站](https://github.com/kubernetes-retired/external-storage/tree/master/ceph/rbd)自己折腾吧
- **Ceph-CSI**
  - 如果 Ceph 集群版本在 14.0.0 (Nautilus)+，首选，
  - Container Storage Interface (CSI) driver for RBD, CephFS
  - 功能支持更好，例如 cloning、expanding 、snapshots
  - **本文的最终选择**
- Rook
  - Rook (https://rook.io/) is an orchestration tool that can run Ceph inside a Kubernetes cluster
  - 生产环境如果需要可能会选择 Rook

## 3. Ceph 集群配置

### 3.1. 创建 RBD 存储池

```shel
ceph osd pool create kube 128 128
rbd pool init kube

[root@ceph-node-0 k8s-cluster]# ceph osd pool ls
device_health_metrics
kube
```

### 3.2. 创建访问 Ceph 存储池的 client 用户

```shell
[root@ceph-node-0 k8s-cluster]# ceph auth get-or-create client.kube mon 'profile rbd' osd 'profile rbd pool=kube' mgr 'profile rbd pool=kube'
[client.kube]
        key = AQBHlIxiuRgiARAANMrLQIEXaDCm7uDSMeuZaw==
```

### 3.3. 获取 key

```shell
[root@ceph-node-0 k8s-cluster]# ceph auth list | grep client.kube -A 3
installed auth entries:

client.kube
        key: AQBHlIxiuRgiARAANMrLQIEXaDCm7uDSMeuZaw==
        caps: [mgr] profile rbd pool=kube
        caps: [mon] profile rbd
   
[root@ceph-node-0 k8s-cluster]# ceph auth get-key client.kube
AQBHlIxiuRgiARAANMrLQIEXaDCm7uDSMeuZaw==

[root@ceph-node-0 k8s-cluster]# ceph auth get-key client.kube | base64
QVFCSGxJeGl1UmdpQVJBQU5NckxRSUVYYURDbTd1RFNNZXVaYXc9PQ==
```

## 4. K8S 持久化存储对接 Ceph

### 4.1. 获取配置文件参考模板

[ceph-csi-v3.6.1 deploy](https://github.com/ceph/ceph-csi/tree/v3.6.1/deploy/rbd/kubernetes)

[ceph-csi-v3.6.1-examples](https://github.com/ceph/ceph-csi/tree/v3.6.1/examples)

直接将仓考模板下载到服务器 (本文写作时最新版是 v3.6.1)。

```shell
wget https://github.com/ceph/ceph-csi/archive/refs/tags/v3.6.1.tar.gz
tar xvf v3.6.1.tar.gz
```

**YAML manifests 在 `deploy/rbd/kubernetes` 目录中。**

```shell
cd ceph-csi-3.6.1/deploy/rbd/kubernetes/
```

### 4.2. Deploy RBACs for sidecar containers and node plugins

```yaml
kubectl create -f csi-provisioner-rbac.yaml
kubectl create -f csi-nodeplugin-rbac.yaml
```

### 4.3. Deploy ConfigMap for CSI plugins

deploy 目录中提供的配置文件是个空的配置，因此，需要根据实际情况修改。

> **修改后的 csi-config-map.yaml**

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: "ceph-csi-config"
data:
  config.json: |-
    [
      {
        "clusterID": "a614e796-6875-4134-a735-2e4db541bba8",
        "monitors": [
        "192.168.9.85:6789",
        "192.168.9.86:6789",
        "192.168.9.87:6789"
        ]
      }
    ]
```

- clusterID：通过 `ceph -s` 或是 `ceph fsid` 获取
- monitors：Ceph mon 节点的 IP，可以通过 `ceph mon dump` 获取

> **执行创建命令**

```shell
kubectl create -f csi-config-map.yaml
```

### 4.4. Deploy Ceph configuration ConfigMap for CSI pods

> **创建 ceph-config.yaml**

把实际的 ceph.conf 的内容粘贴上，配置文件模板参考 **ceph-csi-3.6.1/examples/ceph-conf.yaml**

```yaml
---
apiVersion: v1
kind: ConfigMap
data:
  ceph.conf: |
    [global]
    fsid = a614e796-6875-4134-a735-2e4db541bba8
    mon_initial_members = ceph-node-0
    mon_host = 192.168.9.85
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx
    public network = 192.168.9.0/24
  # keyring is a required key and its value should be empty
  keyring: |
metadata:
  name: ceph-config
```

> **执行创建命令**

```
kubectl create -f ceph-config.yaml
```

### 4.5. 创建 ceph-csi-encryption-kms-config

官方文档中没有提及，下面的命令执行时报错，经排查需要创建 **ceph-csi-encryption-kms-config**

Kms 是个复杂的话题，我们这里创建一个空的，有兴趣的自行参考[官方文档](https://github.com/ceph/ceph-csi/blob/devel/docs/deploy-rbd.md)

> **创建 ceph-csi-encryption-kms-config.yaml**

```yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {}
metadata:
  name: ceph-csi-encryption-kms-config
```

> **执行创建命令**

```shell
kubectl create -f ceph-csi-encryption-kms-config.yaml
```

### 4.6. Deploy CSI sidecar containers

> **部署 provision 的 deployment 包括**

- external-provisioner
- external-attacher
- csi-snapshotter sidecar containers
- CSI RBD plugin

> **执行创建命令**

```shell
kubectl create -f csi-rbdplugin-provisioner.yaml
```

### 4.7. Deploy RBD CSI driver

> **部署一个 daemon set 包含两个 containers**

- CSI node-driver-registrar
- CSI RBD driver

> **执行创建命令**

```shell
kubectl create -f csi-rbdplugin.yaml
```

### 4.8. Verifying the deployment in Kubernetes

```shell
[root@ks-k8s-master-0 rbd]# kubectl get all
NAME                                             READY   STATUS    RESTARTS   AGE
pod/csi-rbdplugin-42v8p                          3/3     Running   0          4m30s
pod/csi-rbdplugin-gm8v7                          3/3     Running   0          4m30s
pod/csi-rbdplugin-p9jvx                          3/3     Running   0          4m30s
pod/csi-rbdplugin-provisioner-85c6b9d548-526zr   7/7     Running   0          7m7s
pod/csi-rbdplugin-provisioner-85c6b9d548-sq66v   7/7     Running   0          7m7s
pod/csi-rbdplugin-provisioner-85c6b9d548-xhfkb   7/7     Running   0          7m7s

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/csi-metrics-rbdplugin       ClusterIP   10.233.10.168   <none>        8080/TCP   4m30s
service/csi-rbdplugin-provisioner   ClusterIP   10.233.5.181    <none>        8080/TCP   7m8s
service/kubernetes                  ClusterIP   10.233.0.1      <none>        443/TCP    46d

NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/csi-rbdplugin   3         3         3       3            3           <none>          4m30s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/csi-rbdplugin-provisioner   3/3     1            3           7m8s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/csi-rbdplugin-provisioner-74fd478c9d   1         1         0       5m11s
replicaset.apps/csi-rbdplugin-provisioner-85c6b9d548   3         3         3       7m7s
```

### 4.9. Deploy Secret

> **创建 csi-rbd-secret.yaml**

```yacas
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: default
stringData:
  userID: kube
  userKey: AQBHlIxiuRgiARAANMrLQIEXaDCm7uDSMeuZaw==
```

- userKey：通过 `ceph auth get-key client.kube` 获取，不要使用 `ceph auth get-key client.kube | base64`

> **执行创建命令**

```shell
kubectl create -f csi-rbd-secret.yaml
```

### 4.10. Deploy StorageClass

> **创建 csi-rbd-sc.yaml**

```yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: a614e796-6875-4134-a735-2e4db541bba8
   pool: kube
   imageFeatures: "layering"
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: default
   csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: default
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: default
   csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
   - discard
```

> **执行创建命令**

```shell
kubectl create -f csi-rbd-sc.yaml
```

### 4.11. Deploy PVC

> **创建 csi-rbd-pvc.yaml**

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc
```

> **执行创建命令**

```shel
kubectl create -f csi-rbd-pvc.yaml
```

### 4.12. Deploy POD

> **创建 csi-rbd-pod.yaml**

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: csi-rbd-demo-pod
spec:
  containers:
    - name: web-server
      image: nginx
      volumeMounts:
        - name: mypvc
          mountPath: /var/lib/www/html
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: rbd-pvc
        readOnly: false
```

> **执行创建命令**

```shell
kubectl create -f csi-rbd-pod.yaml
```

### 4.13. 验证

> **K8S 集群验证**

```shell
[root@ks-k8s-master-0 kubernetes]# kubectl get sc
NAME              PROVISIONER               RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
csi-rbd-sc        rbd.csi.ceph.com          Delete          Immediate              true                   43m
glusterfs         kubernetes.io/glusterfs   Delete          Immediate              false                  2d
local (default)   openebs.io/local          Delete          WaitForFirstConsumer   false                  47d
[root@ks-k8s-master-0 kubernetes]# kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rbd-pvc   Bound    pvc-5bf3b852-1297-4340-aebd-61895dd5705d   1Gi        RWO            csi-rbd-sc     27m
```

> **Ceph 集群验证**

```shell
[root@ceph-node-0 ~]# rbd -p kube ls
csi-vol-d0a599f0-dd5a-11ec-9ff3-768bcf86381f

[root@ceph-node-0 ~]# rbd -p kube info csi-vol-d0a599f0-dd5a-11ec-9ff3-768bcf86381f
rbd image 'csi-vol-d0a599f0-dd5a-11ec-9ff3-768bcf86381f':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 85e5fedccd17
        block_name_prefix: rbd_data.85e5fedccd17
        format: 2
        features: layering
        op_features: 
        flags: 
        create_timestamp: Fri May 27 09:18:35 2022
        access_timestamp: Fri May 27 09:18:35 2022
        modify_timestamp: Fri May 27 09:18:35 2022
```

## 5. 常见问题

### 5.1. Ceph userKey 使用错误

> **报错现象**

```she
failed to provision volume with StorageClass "csi-rbd-sc": rpc error: code = Internal desc = failed to get connection: connecting failed: rados: ret=-22, Invalid argument
```

> **解决方案**

- 出现这个报错，原因是 ceph 集群连接失败，检查 **csi-rbd-secret** 配置中的 **userKey**。
- 需要使用 `ceph auth get-key client.kube` 的原始输出值，不要进行 base64 加密。

## 6. 总结

本文首先演示了 Ceph 集群创建对接 Kubernetes 必须资源的操作配置，接下来重点演示了 Kubernetes 使用 Ceph-CSI 方式对接外部 Ceph 集群的全流程操作。本文的演示仅适用于学习测试，生产环境需要考虑更多的配置。如果是新建的 Ceph 集群，建议考虑 Rook 方案。

> **参考文档**

- [CSI RBD 安装](https://github.com/ceph/ceph-csi/blob/devel/docs/deploy-rbd.md)
- [CSI RBD examples](https://github.com/ceph/ceph-csi/tree/v3.6.1/examples)

> **Get 文档**

- Github https://github.com/devops/z-notes
- Gitee https://gitee.com/zdevops/z-notes

> **Get 代码**

- Github https://github.com/devops/ansible-zdevops
- Gitee https://gitee.com/zdevops/ansible-zdevops

> **B 站**

- [老 Z 手记](https://space.bilibili.com/1039301316) https://space.bilibili.com/1039301316

> **版权声明** 

- 所有内容均属于原创，整理不易，感谢收藏，转载请标明出处。

> **About Me**

- 昵称：老 Z
- 坐标：山东济南
- 职业：运维架构师 / 高级运维工程师 =**运维**
- 微信：zdevops
- 关注的领域：云计算 / 云原生技术运维，自动化运维
- 技能标签：OpenStack、Ansible、K8S、Python、Go、CNCF
