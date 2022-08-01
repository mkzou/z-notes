# 基于 KubeSphere 玩转 k8s｜60 分钟 Rook 实战入门

**大家好，我是老 Z！**

> **欢迎来到云原生技术栈实战系列之基于 KubeSphere 玩转 K8s**

## 前言

> **导图**

![rook-architecture](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/rook-architecture.png)

> **知识量**

- 阅读时长：20 分
- 行：944
- 单词：4800+
- 字符：30900+
- 图片：7 张

> **知识点**

- 定级：**入门级**
- Rook 概览
- Rook 集群的部署
- Rook Block 存储的配置
- Ceph Dashboard 的配置使用

> **演示服务器配置**

|     主机名      |      IP      | CPU  | 内存 | 系统盘 | 数据盘  |                 用途                  |
| :-------------: | :----------: | :--: | :--: | :----: | :-----: | :-----------------------------------: |
|  zdeops-master  | 192.168.9.9  |  2   |  4   |   40   |   200   |          Ansible 运维控制节点          |
| ks-k8s-master-0 | 192.168.9.91 |  4   |  16  |   40   | 200+200 | KubeSphere/k8s-master/k8s-worker/Ceph |
| ks-k8s-master-1 | 192.168.9.92 |  4   |  16  |   40   | 200+200 | KubeSphere/k8s-master/k8s-worker/Ceph |
| ks-k8s-master-2 | 192.168.9.93 |  4   |  16  |   40   | 200+200 | KubeSphere/k8s-master/k8s-worker/Ceph |
|    storage-node-0    | 192.168.9.95 |  2   |  8   |   40   | 200+200 |        ElasticSearch/GlusterFS        |
|    storage-node-1    | 192.168.9.96 |  2   |  8   |   40   | 200+200 |        ElasticSearch/GlusterFS        |
|    storage-node-2    | 192.168.9.97 |  2   |  8   |   40   | 200+200 |        ElasticSearch/GlusterFS        |
|     harbor      | 192.168.9.89 |  2   |  8   |   40   |   200   |                Harbor                 |
|      合计       |      8       |  22  |  84  |  320   |  2800   |                                       |

> **演示环境涉及软件版本信息**

- 操作系统：**CentOS-7.9-x86_64**
- Ansible：**2.8.20**
- KubeSphere：**3.3.0**
- Kubernetes：**v1.24.1**
- Rook：**v1.9.7**
- GlusterFS：**9.5.1**
- ElasticSearch：**7.17.5**
- Harbor：**2.5.1**

## 1. 简介

### 1.1. Rook 是什么？

> **官方定义**

- Rook 是一个开源的云原生存储编排器，为各种存储解决方案提供平台、框架和支持，以便与云原生环境进行原生集成。
- Rook 将存储软件转变为自管理、自扩展和自修复的存储服务。它通过自动化部署、引导、置备、配置、伸缩、升级、迁移、灾难恢复、监控和资源管理来实现这一点。
- Rook 使用底层云原生容器提供的工具执行其管理、调度和编排平台的职责。
- Rook 利用扩展点深度集成到云原生环境中，为调度、生命周期管理、资源管理、安全、监控和用户体验提供无缝体验。
- Ceph operator 于 2018 年 12 月在 Rook **v0.9** 版本中宣布**稳定**，已经提供了多年的生产存储平台。

![rook-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/rook-1.png)

![rook-2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/rook-2.png)

![rook-3](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/rook-3.png)

> **多种存储解决方案**

- Rook 编排多个存储解决方案，每个解决方案都有一个专门的 Kubernetes Operator 来自动化管理。
- 为您的使用场景选择最好的存储提供商，Rook 确保它们在 Kubernetes 上都能良好运行，具有相同、一致的体验。
- 目前支持 **Ceph** 和 **NFS**

### 1.2. Rook Architecture

![rook-kubernetes](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/rook-kubernetes.png)

## 2. 前提条件

### 2.1.  Kubernetes Minimum Version

- Rook 可以安装在任何现有的 Kubernetes 集群上，只要它满足最低版本，并且授予 Rook 所需的特权 .

- Kubernetes **v1.17** or higher is supported for the Ceph operator.

### 2.2. CPU Architecture

Architectures released are `amd64 / x86_64` and `arm64`.

### 2.3. Ceph Prerequisites

为了配置 Ceph 存储集群，至少需要以下任意一种本地存储选项 :

- Raw devices (no partitions or formatted filesystems，没有分区和格式化文件系统)
- Raw partitions (no formatted filesystem，已分区但是没有格式化文件系统)
- PVs available from a storage class in `block` mode

可以使用以下命令确认分区或设备是否使用文件系统进行了格式化。

```shell
[root@ks-k8s-master-0 ~]# lsblk -f
NAME            FSTYPE      LABEL UUID                                   MOUNTPOINT
sda                                                                      
├─sda1          xfs               6a66c441-78cf-4d86-a9a5-201fcccaa0ba   /boot
└─sda2          LVM2_member       4Qx7ir-P9HJ-eqRf-c4ZA-v3hA-YCtr-8Ufgbo 
  ├─centos-root xfs               ea556f53-4505-42d7-953e-a20aacb799d5   /
  └─centos-swap swap              306bc365-0c12-4c12-b3d2-971ce22caa3b   
sdb                                                                      
└─sdb1          xfs               1e644d09-507c-45ca-8d3f-57adbae6e600   /var/lib/containerd
sdc  
```

- 如果 FSTYPE 字段不为空，说明该设备已经格式化为文件系统，对应的值就是文件系统类型
- 如果 FSTYPE 字段为空，说明该设备还没有被格式化，可以被 Ceph 使用
- 本例中可以使用的设备为 **sdc**

### 2.4. Admission Controller

本文忽略，详情见[官方文档](https://rook.io/docs/rook/v1.9/Getting-Started/Prerequisites/prerequisites/#admission-controller)

### 2.5. LVM package

Ceph OSDs 在以下场景依赖 LVM。

- OSDs are created on raw devices or partitions
- If encryption is enabled (`encryptedDevice: "true"` in the cluster CR)
- A `metadata` device is specified

Ceph OSDs 在以下场景不需要 LVM。

- Creating OSDs on PVCs using the `storageClassDeviceSets`

CentOS 默认已经安装 LVM，如果没有装，使用下面的命令安装。

```shell
yum install -y lvm2
```

### 2.6. Kernel

- RBD

Ceph 需要使用构建了 RBD 模块的 Linux 内核。许多 Linux 发行版都有这个模块，但不是所有发行版都有。例如，GKE Container-Optimised OS (COS) 就没有 RBD。

在 Kubernetes 节点使用 `modprober rbd` 命令验证，正常情况下该命名没有任何输出，如果输出提示 'not found'，则需要重新编译内核或是更换操作系统。

- CephFS

如果您将从 Ceph shared file system (CephFS) 创建卷，推荐的最低内核版本是 4.17。如果内核版本小于 4.17，则不会强制执行请求的 PVC sizes。存储配额只会在更新的内核上执行。

## 3.  ROOK 资源准备

### 3.1. 下载 ROOK 部署代码

```shell
 git clone --single-branch --branch v1.9.7 https://github.com/rook/rook.git
```

### 3.2. 离线镜像制作

此过程为可选项，离线内网环境可用，如果是直连互联网的场景，可以直接使用互联网的镜像。

在一台能同时访问互联网和内网 Harbor 仓库的服务器上进行下面的操作。

- Harbor 创建项目

```shell
#!/usr/bin/env bash

# Harbor 仓库地址
url="https://registry.zdevops.com.cn"

# 访问 Harbor 仓库用户
user="admin"

# 访问 Harbor 仓库用户密码
passwd="Harbor12345"

# 需要创建的项目名列表，正常只需要创建一个**kubesphereio**即可，这里为了保留变量可扩展性多写了两个。
harbor_projects=(csiaddons
    sig-storage
    rook
    cephcsi
    ceph
)

for project in "${harbor_projects[@]}"; do
    echo "creating $project"
    curl -u "${user}:${passwd}" -X POST -H "Content-Type: application/json" "${url}/api/v2.0/projects" -d "{ \"project_name\": \"${project}\", \"public\": true}"
done
```

- 下载镜像

```shell
cd rook/deploy/examples

# 两条命令二选一 都可以
for i in `cat images.txt`;do docker pull $i;done

# cat images.txt | awk -F "/" '{ print "docker pull "$0 }' | bash
```

- images.txt 文件内容

```shell
quay.io/ceph/ceph:v16.2.9
quay.io/cephcsi/cephcsi:v3.6.2
quay.io/csiaddons/k8s-sidecar:v0.4.0
quay.io/csiaddons/volumereplication-operator:v0.3.0
registry.k8s.io/sig-storage/csi-attacher:v3.4.0
registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.5.1
registry.k8s.io/sig-storage/csi-provisioner:v3.1.0
registry.k8s.io/sig-storage/csi-resizer:v1.4.0
registry.k8s.io/sig-storage/csi-snapshotter:v6.0.1
registry.k8s.io/sig-storage/nfsplugin:v4.0.0
rook/ceph:v1.9.7
```

- 重新打 tag

```shell
docker tag quay.io/ceph/ceph:v16.2.9 registry.zdevops.com.cn/ceph/ceph:v16.2.9
docker tag quay.io/cephcsi/cephcsi:v3.6.2 registry.zdevops.com.cn/cephcsi/cephcsi:v3.6.2
docker tag quay.io/csiaddons/k8s-sidecar:v0.4.0 registry.zdevops.com.cn/csiaddons/k8s-sidecar:v0.4.0
docker tag quay.io/csiaddons/volumereplication-operator:v0.3.0 registry.zdevops.com.cn/csiaddons/volumereplication-operator:v0.3.0
docker tag registry.k8s.io/sig-storage/csi-attacher:v3.4.0 registry.zdevops.com.cn/sig-storage/csi-attacher:v3.4.0
docker tag registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.5.1 registry.zdevops.com.cn/sig-storage/csi-node-driver-registrar:v2.5.1
docker tag registry.k8s.io/sig-storage/csi-provisioner:v3.1.0 registry.zdevops.com.cn/sig-storage/csi-provisioner:v3.1.0
docker tag registry.k8s.io/sig-storage/csi-resizer:v1.4.0 registry.zdevops.com.cn/sig-storage/csi-resizer:v1.4.0
docker tag registry.k8s.io/sig-storage/csi-snapshotter:v6.0.1 registry.zdevops.com.cn/sig-storage/csi-snapshotter:v6.0.1
docker tag registry.k8s.io/sig-storage/nfsplugin:v4.0.0 registry.zdevops.com.cn/sig-storage/nfsplugin:v4.0.0
docker tag rook/ceph:v1.9.7 registry.zdevops.com.cn/rook/ceph:v1.9.7
```

- 推送到私有镜像仓库

```shell
docker push registry.zdevops.com.cn/ceph/ceph:v16.2.9
docker push registry.zdevops.com.cn/cephcsi/cephcsi:v3.6.2
docker push registry.zdevops.com.cn/csiaddons/k8s-sidecar:v0.4.0
docker push registry.zdevops.com.cn/csiaddons/volumereplication-operator:v0.3.0
docker push registry.zdevops.com.cn/sig-storage/csi-attacher:v3.4.0
docker push registry.zdevops.com.cn/sig-storage/csi-node-driver-registrar:v2.5.1
docker push registry.zdevops.com.cn/sig-storage/csi-provisioner:v3.1.0
docker push registry.zdevops.com.cn/sig-storage/csi-resizer:v1.4.0
docker push registry.zdevops.com.cn/sig-storage/csi-snapshotter:v6.0.1
docker push registry.zdevops.com.cn/sig-storage/nfsplugin:v4.0.0
docker push registry.zdevops.com.cn/rook/ceph:v1.9.7
```

- 清理临时镜像

```shell
docker rmi registry.zdevops.com.cn/ceph/ceph:v16.2.9
docker rmi registry.zdevops.com.cn/cephcsi/cephcsi:v3.6.2
docker rmi registry.zdevops.com.cn/csiaddons/k8s-sidecar:v0.4.0
docker rmi registry.zdevops.com.cn/csiaddons/volumereplication-operator:v0.3.0
docker rmi registry.zdevops.com.cn/sig-storage/csi-attacher:v3.4.0
docker rmi registry.zdevops.com.cn/sig-storage/csi-node-driver-registrar:v2.5.1
docker rmi registry.zdevops.com.cn/sig-storage/csi-provisioner:v3.1.0
docker rmi registry.zdevops.com.cn/sig-storage/csi-resizer:v1.4.0
docker rmi registry.zdevops.com.cn/sig-storage/csi-snapshotter:v6.0.1
docker rmi registry.zdevops.com.cn/sig-storage/nfsplugin:v4.0.0
docker rmi registry.zdevops.com.cn/rook/ceph:v1.9.7
docker rmi quay.io/ceph/ceph:v16.2.9
docker rmi quay.io/cephcsi/cephcsi:v3.6.2
docker rmi quay.io/csiaddons/k8s-sidecar:v0.4.0
docker rmi quay.io/csiaddons/volumereplication-operator:v0.3.0
docker rmi registry.k8s.io/sig-storage/csi-attacher:v3.4.0
docker rmi registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.5.1
docker rmi registry.k8s.io/sig-storage/csi-provisioner:v3.1.0
docker rmi registry.k8s.io/sig-storage/csi-resizer:v1.4.0
docker rmi registry.k8s.io/sig-storage/csi-snapshotter:v6.0.1
docker rmi registry.k8s.io/sig-storage/nfsplugin:v4.0.0
docker rmi rook/ceph:v1.9.7
```

- 自动化脚本

```shell
# pull | tag | push
# cat images.txt | awk -F "/" '{sub("^ *","");print "docker pull "$0" && docker tag "$0" registry.zdevops.com.cn/"$(NF-1)"/"$NF " && docker push registry.zdevops.com.cn/"$(NF-1)"/"$NF}'
cat images.txt | awk -F "/" '{print "docker pull "$0" && docker tag "$0" registry.zdevops.com.cn/"$(NF-1)"/"$NF " && docker push registry.zdevops.com.cn/"$(NF-1)"/"$NF " && docker rmi " $0" && docker rmi registry.zdevops.com.cn/"$(NF-1)"/"$NF }' | bash
```

### 3.3. 服务器镜像下载

此步骤本文并未执行，适用于**「4.1 修改镜像地址」**方案二。

- 拉取内网镜像

```shell
ctr images pull registry.zdevops.com.cn/ceph/ceph:v16.2.9
ctr images pull registry.zdevops.com.cn/cephcsi/cephcsi:v3.6.2
ctr images pull registry.zdevops.com.cn/csiaddons/k8s-sidecar:v0.4.0
ctr images pull registry.zdevops.com.cn/csiaddons/volumereplication-operator:v0.3.0
ctr images pull registry.zdevops.com.cn/sig-storage/csi-attacher:v3.4.0
ctr images pull registry.zdevops.com.cn/sig-storage/csi-node-driver-registrar:v2.5.1
ctr images pull registry.zdevops.com.cn/sig-storage/csi-provisioner:v3.1.0
ctr images pull registry.zdevops.com.cn/sig-storage/csi-resizer:v1.4.0
ctr images pull registry.zdevops.com.cn/sig-storage/csi-snapshotter:v6.0.1
ctr images pull registry.zdevops.com.cn/sig-storage/nfsplugin:v4.0.0
ctr images pull registry.zdevops.com.cn/rook/ceph:v1.9.7
```

- 重新打 tag

```shell
ctr -n k8s.io images tag registry.zdevops.com.cn/ceph/ceph:v16.2.9 quay.io/ceph/ceph:v16.2.9
ctr -n k8s.io images tag registry.zdevops.com.cn/cephcsi/cephcsi:v3.6.2 quay.io/cephcsi/cephcsi:v3.6.2
ctr -n k8s.io images tag registry.zdevops.com.cn/csiaddons/k8s-sidecar:v0.4.0 quay.io/csiaddons/k8s-sidecar:v0.4.0
ctr -n k8s.io images tag registry.zdevops.com.cn/csiaddons/volumereplication-operator:v0.3.0 quay.io/csiaddons/volumereplication-operator:v0.3.0
ctr -n k8s.io images tag registry.zdevops.com.cn/sig-storage/csi-attacher:v3.4.0 registry.k8s.io/sig-storage/csi-attacher:v3.4.0
ctr -n k8s.io images tag registry.zdevops.com.cn/sig-storage/csi-node-driver-registrar:v2.5.1 registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.5.1
ctr -n k8s.io images tag registry.zdevops.com.cn/sig-storage/csi-provisioner:v3.1.0 registry.k8s.io/sig-storage/csi-provisioner:v3.1.0
ctr -n k8s.io images tag registry.zdevops.com.cn/sig-storage/csi-resizer:v1.4.0 registry.k8s.io/sig-storage/csi-resizer:v1.4.0
ctr -n k8s.io images tag registry.zdevops.com.cn/sig-storage/csi-snapshotter:v6.0.1 registry.k8s.io/sig-storage/csi-snapshotter:v6.0.1
ctr -n k8s.io images tag registry.zdevops.com.cn/sig-storage/nfsplugin:v4.0.0 registry.k8s.io/sig-storage/nfsplugin:v4.0.0
ctr -n k8s.io images tag registry.zdevops.com.cn/rook/ceph:v1.9.7 rook/ceph:v1.9.7
```

- 清理临时镜像

```shell
ctr -n k8s.io images rm registry.zdevops.com.cn/ceph/ceph:v16.2.9
ctr -n k8s.io images rm registry.zdevops.com.cn/cephcsi/cephcsi:v3.6.2
ctr -n k8s.io images rm registry.zdevops.com.cn/csiaddons/k8s-sidecar:v0.4.0
ctr -n k8s.io images rm registry.zdevops.com.cn/csiaddons/volumereplication-operator:v0.3.0
ctr -n k8s.io images rm registry.zdevops.com.cn/sig-storage/csi-attacher:v3.4.0
ctr -n k8s.io images rm registry.zdevops.com.cn/csi-node-driver-registrar:v2.5.1
ctr -n k8s.io images rm registry.zdevops.com.cn/sig-storage/csi-provisioner:v3.1.0
ctr -n k8s.io images rm registry.zdevops.com.cn/sig-storage/csi-resizer:v1.4.0
ctr -n k8s.io images rm registry.zdevops.com.cn/sig-storage/csi-snapshotter:v6.0.1
ctr -n k8s.io images rm registry.zdevops.com.cn/sig-storage/nfsplugin:v4.0.0
ctr -n k8s.io images rm registry.zdevops.com.cn/rook/ceph:v1.9.7
```

## 4. Rook 部署 Ceph 集群

### 4.1. 修改镜像地址

默认的配置文件中，使用的镜像为互联网地址的镜像，使用本地镜像仓库时，需要做一些特殊处理。

有两种可选方案，

- 方案一：编辑配置文件，替换引用的镜像仓库地址。
- 方案二：利用内网镜像仓库将镜像下载到服务器，然后重新打 tag，将前缀换为互联网的原始地址，具体操作参考**「3.3 服务器镜像下载」**。

**本文采用方案一，具体操作如下：**

- 编辑配置文件**「operator.yaml」**，修改镜像为本地镜像

```yaml
...
ROOK_CSI_CEPH_IMAGE: "registry.zdevops.com.cn/cephcsi/cephcsi:v3.6.2"
ROOK_CSI_REGISTRAR_IMAGE: "registry.zdevops.com.cn/sig-storage/csi-node-driver-registrar:v2.5.1"
ROOK_CSI_RESIZER_IMAGE: "registry.zdevops.com.cn/sig-storage/csi-resizer:v1.4.0"
ROOK_CSI_PROVISIONER_IMAGE: "registry.zdevops.com.cn/sig-storage/csi-provisioner:v3.1.0"
ROOK_CSI_SNAPSHOTTER_IMAGE: "registry.zdevops.com.cn/sig-storage/csi-snapshotter:v6.0.1"
ROOK_CSI_ATTACHER_IMAGE: "registry.zdevops.com.cn/sig-storage/csi-attacher:v3.4.0"
ROOK_CSI_NFS_IMAGE: "registry.zdevops.com.cn/sig-storage/nfsplugin:v4.0.0"
CSI_VOLUME_REPLICATION_IMAGE: "registry.zdevops.com.cn/csiaddons/volumereplication-operator:v0.3.0"
ROOK_CSIADDONS_IMAGE: "registry.zdevops.com.cn/csiaddons/k8s-sidecar:v0.4.0"

image: registry.zdevops.com.cn/rook/ceph:v1.9.7
```

- 编辑配置文件**「cluster.yaml」**，修改镜像为本地镜像

```yaml
image: registry.zdevops.com.cn/ceph/ceph:v16.2.9
```

- 自动化操作脚本

```shell
# 取消镜像注释
sed -i '91,97s/^.*#/ /g' operator.yaml
sed -i '434,434s/^.*#/ /g' operator.yaml
sed -i '437,437s/^.*#/ /g' operator.yaml

# 替换镜像地址前缀
sed -i -e 's/registry.k8s.io/registry.zdevops.com.cn/g' -e 's/quay.io/registry.zdevops.com.cn/g' operator.yaml
sed -i 's|rook/ceph:v1.9.7|registry.zdevops.com.cn/rook/ceph:v1.9.7|g' operator.yaml
sed -i '24,24s/quay.io/registry.zdevops.com.cn/g' cluster.yaml
```

### 4.2. Deploy the Rook Operator

- 根据资源配置清单创建资源

```shell
cd deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

- 验证 **rook-ceph-operator ** Pod 的状态是否为 `Running`

```shell
[root@ks-k8s-master-0 examples]# kubectl -n rook-ceph get pod
NAME                                                        READY   STATUS      RESTARTS   AGE
rook-ceph-operator-85dcb9d489-98mx8                         1/1     Running     0          50s
```

### 4.3. Create a Ceph Cluster

- 修改集群配置文件**「cluster.yaml」**，增加磁盘配置

```yaml
storage: # cluster level storage configuration and selection
  useAllNodes: false
  useAllDevices: false
  deviceFilter:
  config:
    storeType: bluestore
  nodes:
    - name: ks-k8s-master-0
      devices:
        - name: "sdc"
    - name: ks-k8s-master-1
      devices:
        - name: "sdc"
    - name: ks-k8s-master-2
      devices:
        - name: "sdc"
```

- 创建集群

```shell
kubectl create -f cluster.yaml
```

- 查看资源状态，确保所有相关 Pod 均为 `Running`

```shell
[root@ks-k8s-master-0 examples]# kubectl -n rook-ceph get pod
NAME                                                        READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-njmdn                                      3/3     Running     0          3m24s
csi-cephfsplugin-provisioner-84bc77ff84-j9vgz               6/6     Running     0          3m24s
csi-cephfsplugin-provisioner-84bc77ff84-t52pm               6/6     Running     0          3m24s
csi-cephfsplugin-rwmjm                                      3/3     Running     0          3m24s
csi-cephfsplugin-xhc67                                      3/3     Running     0          3m24s
csi-rbdplugin-2xpvs                                         3/3     Running     0          3m24s
csi-rbdplugin-4m898                                         3/3     Running     0          3m24s
csi-rbdplugin-provisioner-845945c9cb-2sclk                  6/6     Running     0          3m24s
csi-rbdplugin-provisioner-845945c9cb-dl4vx                  6/6     Running     0          3m24s
csi-rbdplugin-xhkpv                                         3/3     Running     0          3m24s
rook-ceph-crashcollector-ks-k8s-master-0-75fc5f778c-9pq6b   1/1     Running     0          90s
rook-ceph-crashcollector-ks-k8s-master-1-7bc7cc898d-75rr4   1/1     Running     0          57s
rook-ceph-crashcollector-ks-k8s-master-2-74f7fc9b5-k55w4    1/1     Running     0          57s
rook-ceph-mgr-a-6cc97dc547-lrtzq                            2/2     Running     0          99s
rook-ceph-mgr-b-556f474b5f-jshp8                            2/2     Running     0          98s
rook-ceph-mon-a-6cc594c678-8lms5                            1/1     Running     0          3m14s
rook-ceph-mon-b-c54846f8c-2lffx                             1/1     Running     0          2m15s
rook-ceph-mon-c-5fcf8f98b8-hs6w6                            1/1     Running     0          2m
rook-ceph-operator-85dcb9d489-98mx8                         1/1     Running     0          3h34m
rook-ceph-osd-0-57687f7bf9-qc6f5                            1/1     Running     0          59s
rook-ceph-osd-1-6bdf65b96b-lw2sv                            1/1     Running     0          57s
rook-ceph-osd-2-78676b7bf-c9tct                             1/1     Running     0          57s
rook-ceph-osd-prepare-ks-k8s-master-0-vq24c                 0/1     Completed   0          75s
rook-ceph-osd-prepare-ks-k8s-master-1-w9vjg                 0/1     Completed   0          75s
rook-ceph-osd-prepare-ks-k8s-master-2-zj89n                 0/1     Completed   0          75s
```

### 4.4. 创建 Rook toolbox

通过 Rook 提供的 toolbox，我们可以实现对 Ceph 集群的管理。

- 创建 toolbox

```shell
kubectl apply -f toolbox.yaml
```

- 查看 toolbox 状态，确认状态为 **Running**

```shell
kubectl -n rook-ceph rollout status deploy/rook-ceph-tools
```

- 登录 Toolbox

```shell
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

- 验证 Ceph 集群状态

```shell
[rook@rook-ceph-tools-6c6974f44c-pq49w /]$ ceph -s 
  cluster:
    id:     b670b67a-83a4-42e7-905d-ed3f97759bb6
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 20h)
    mgr: b(active, since 3m), standbys: a
    osd: 3 osds: 3 up (since 20h), 3 in (since 20h)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   15 MiB used, 600 GiB / 600 GiB avail
    pgs:     1 active+clean
```

> 观察 Ceph 集群状态，需要满足下面的条件才会认为集群状态是健康的。
>
> - health 的值为 HEALTH_OK
> - Mons 的数量和状态
> - Mgr 有一个是 active 状态
> - OSD 状态都是 up

- 其他常用的 Ceph 命令

```shell
# 查看 OSD 状态
ceph osd status
ceph osd df
ceph osd utilization
ceph osd pool stats
ceph osd tree

# 查看 Ceph 容量
ceph df

# 查看 Rados 状态
rados df

# 查看 PG 状态
ceph pg stat
```

- 删除 toolbox(可选)

```shell
kubectl -n rook-ceph delete deploy/rook-ceph-tools
```

### 4.5. Storage 介绍

Rock 提供了三种存储类型，请参考官方指南了解详情：

- **[Block](https://rook.io/docs/rook/v1.9/Storage-Configuration/Block-Storage-RBD/block-storage/)**: Create block storage to be consumed by a pod (RWO)
- **[Shared Filesystem](https://rook.io/docs/rook/v1.9/Storage-Configuration/Shared-Filesystem-CephFS/filesystem-storage/)**: Create a filesystem to be shared across multiple pods (RWX)
- **[Object](https://rook.io/docs/rook/v1.9/Storage-Configuration/Object-Storage-RGW/object-storage/)**: Create an object store that is accessible inside or outside the Kubernetes cluster

## 5. Block Storage

### 5.1. Block 存储

Rook 允许通过自定义资源定义 (crd) 创建和自定义 Block 存储池。支持 Replicated 和 Erasure Coded 类型。本文演示 Replicated 的创建过程。

- 创建 Ceph 块存储，编辑 `CephBlockPool` CR 资源清单 , ceph-replicapool.yaml

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
```

上面的操作会创建一个 3 副本的块存储池子

- 创建 CephBlockPool 资源

```shell
[root@ks-k8s-master-0 examples]# kubectl create -f ceph-replicapool.yaml
```

- 查看资源创建情况

```shell
[root@ks-k8s-master-0 examples]# kubectl get cephBlockPool -n rook-ceph -o wide
NAME          PHASE
replicapool   Ready
```

- 在 ceph toolbox 中查看 Ceph 集群状态

```shell
# 登录
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

# 查看集群
[rook@rook-ceph-tools-6c6974f44c-pq49w /]$ ceph -s
  cluster:
    id:     b670b67a-83a4-42e7-905d-ed3f97759bb6
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 27h)
    mgr: b(active, since 7h), standbys: a
    osd: 3 osds: 3 up (since 27h), 3 in (since 27h)
 
  data:
    pools:   2 pools, 33 pgs
    objects: 1 objects, 19 B
    usage:   16 MiB used, 600 GiB / 600 GiB avail
    pgs:     33 active+clean
    
    
# 查看集群存储池 
[rook@rook-ceph-tools-6c6974f44c-pq49w /]$ceph osd pool ls
device_health_metrics
replicapool

[rook@rook-ceph-tools-6c6974f44c-pq49w /]$ rados df
POOL_NAME                USED  OBJECTS  CLONES  COPIES  MISSING_ON_PRIMARY  UNFOUND  DEGRADED  RD_OPS   RD  WR_OPS     WR  USED COMPR  UNDER COMPR
device_health_metrics     0 B        0       0       0                   0        0         0       0  0 B       0    0 B         0 B          0 B
replicapool            12 KiB        1       0       3                   0        0         0       0  0 B       2  2 KiB         0 B          0 B

total_objects    1
total_used       16 MiB
total_avail      600 GiB
total_space      600 GiB

# 查看存储池的 pg number
[rook@rook-ceph-tools-6c6974f44c-pq49w /]$ ceph osd pool get replicapool pg_num
pg_num: 32
```

- 编辑 StorageClass 资源清单 ,storageclass-rook-ceph-block.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
    # clusterID is the namespace where the rook cluster is running
    clusterID: rook-ceph
    # Ceph pool into which the RBD image shall be created
    pool: replicapool

    # RBD image format. Defaults to "2".
    imageFormat: "2"

    # RBD image features. Available for imageFormat: "2". CSI RBD currently supports only `layering` feature.
    imageFeatures: layering

    # The secrets contain Ceph admin credentials.
    csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
    csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
    csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

    # Specify the filesystem type of the volume. If not specified, csi-provisioner
    # will set default as `ext4`. Note that `xfs` is not recommended due to potential deadlock
    # in hyperconverged settings where the volume is mounted on the same node as the osds.
    csi.storage.k8s.io/fstype: ext4

# Delete the rbd volume when a PVC is deleted
reclaimPolicy: Delete

# Optional, if you want to add dynamic resize for PVC.
# For now only ext3, ext4, xfs resize support provided, like in Kubernetes itself.
allowVolumeExpansion: true

```

- 创建 StorageClass 资源

```shell
[root@ks-k8s-master-0 examples]# kubectl create -f storageclass-rook-ceph-block.yaml
```

**examples/csi/rbd** 目录中有更多的参考用例。

- 验证资源

```shell
[root@ks-k8s-master-0 examples]# kubectl get sc
NAME              PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local (default)   openebs.io/local             Delete          WaitForFirstConsumer   false                  10d
rook-ceph-block   rook-ceph.rbd.csi.ceph.com   Delete          Immediate              true                   3m36s
```

### 5.2. 创建测试应用

我们使用经典的 wordpress 和 mysql 应用程序创建一个使用 Rook 提供块存储的示例应用程序，这两个应用程序都使用由 Rook 提供的块存储卷。

- 创建 mysql 和 wordpress

```shell
kubectl create -f mysql.yaml
kubectl create -f wordpress.yaml
```

- 查看 PVC 资源

```shell
[root@ks-k8s-master-0 examples]# kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
mysql-pv-claim   Bound    pvc-33a2c666-a001-447c-81d4-9a93b80cd782   20Gi       RWO            rook-ceph-block   53s
wp-pv-claim      Bound    pvc-d9d1e10c-dd94-402d-bf0d-ca9bc9d2b6af   20Gi       RWO            rook-ceph-block   51s
```

- 查看 SVC 资源

```shell
[root@ks-k8s-master-0 examples]# kubectl get svc
NAME              TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes        ClusterIP      10.233.0.1     <none>        443/TCP        10d
wordpress         LoadBalancer   10.233.6.210   <pending>     80:30463/TCP   16s
wordpress-mysql   ClusterIP      None           <none>        3306/TCP       18s
```

- 查看 Pod 资源

 ```shell
 [root@ks-k8s-master-0 examples]# kubectl get pod -o wide
 NAME                               READY   STATUS    RESTARTS   AGE     IP              NODE              NOMINATED NODE   READINESS GATES
 wordpress-7964897bd9-7gjdp         1/1     Running   0          2m21s   10.233.116.75   ks-k8s-master-2   <none>           <none>
 wordpress-mysql-776b4f56c4-sphrk   1/1     Running   0          2m24s   10.233.87.162   ks-k8s-master-1   <none>           <none>
 ```

**上面的细节文档中不做过多展示，更多细节会在直播中演示讲解**。

## 6. Ceph Dashboard

Ceph 提供了一个 Dashboard 工具，我们可以在上面查看集群的状态，包括集群整体运行状态、Mgr、Mon、OSD 和其他 Ceph 进程的状态，查看存储池和 PG 状态，以及显示守护进程的日志等。

具体使用流程如下：

### 6.1. Enable the Ceph Dashboard

可以通过 cluster.yaml 中 Ceph Cluster CRD 中的设置启用 Dashboard 功能，默认的清单文件中已经启用该功能，配置示例如下：

```yaml
[...]
spec:
  dashboard:
    enabled: true
```

Dashboard 启用后，Rook operator 将启用 ceph-mgr 的 dashboard 模块。

### 6.2. 获取 Dashboard 的 service 地址

```shell
[root@ks-k8s-master-0 examples]# kubectl -n rook-ceph get service
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
rook-ceph-mgr              ClusterIP   10.233.31.147   <none>        9283/TCP            21h
rook-ceph-mgr-dashboard    ClusterIP   10.233.18.59    <none>        8443/TCP            21h
```

### 6.3. 配置在集群外部访问 Dashboard

通常我们需要在 K8s 集群外部访问 Ceph Dashboard，可以通过 NodePort 或是 Ingress 的方式 .

本文采用 NodePort 的方式，并且按集群规划使用固定的端口号 `31443`。

- 创建资源清单文件 `ceph-dashboard-external-https.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-mgr-dashboard-external-https
  namespace: rook-ceph
  labels:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
spec:
  ports:
  - name: dashboard
    port: 8443
    protocol: TCP
    targetPort: 8443
    nodePort: 31443
  selector:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
  sessionAffinity: None
  type: NodePort
```

**官方 example 也提供了参考样例 `dashboard-external-https.yaml`，可以直接使用**

- 创建资源

```shell
[root@ks-k8s-master-0 examples]# kubectl create -f ceph-dashboard-external-https.yaml
```

- 验证创建的资源

```shell
[root@ks-k8s-master-0 examples]# kubectl -n rook-ceph get service rook-ceph-mgr-dashboard-external-https
NAME                                     TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
rook-ceph-mgr-dashboard-external-https   NodePort   10.233.28.87   <none>        8443:31443/TCP   55s
```

### 6.4. 获取 Login Credentials

登陆 Dashboard 时需要身份验证，Rook 创建了一个默认用户，用户名 admin。创建了一个名为 rook-ceph-dashboard-password 的 secret 存储密码，使用下面的命令获取随机生成的密码

```shell
[root@ks-k8s-master-0 examples]# kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
/Vl^679ck440/nf6~G"l
```

### 6.5. 通过浏览器打开 Dashboard

访问 K8s 集群中任意节点的 IP，`https://192.168.9.91:31443`，默认用户名 `admin`，密码通过上面的命令获取。

![rook-dashboard-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/rook-dashboard-1.png)

![rook-dashboard-2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/rook-dashboard-2.png)

## 7. 生产环境思考

- 磁盘 SSD 和 SAS/SATA
- 存储类型选择
- Block 存储池类型选择
- 存储节点规划
- ...

## 8. 常见问题

### 8.1. OSD pod 创建失败

- invalid main GPT header

这个盘是新添加的，并没有创建 GPT 分区信息，手动给各个盘创建 GPT header

```shell
# 问题细节
# kubectl -n rook-ceph log rook-ceph-osd-prepare-ke-dev1-worker1-bbm9t provision
...
2018-11-29 03:28:36.533532 I | exec: Running command: lsblk /dev/vde --bytes --nodeps --pairs --output SIZE,ROTA,RO,TYPE,PKNAME
2018-11-29 03:28:36.537270 I | exec: Running command: sgdisk --print /dev/vde
2018-11-29 03:28:36.547839 W | inventory: skipping device vde with an unknown uuid. Failed to complete 'get disk vde uuid': exit status 2. ^GCaution: invalid main GPT header, but valid backup; regenerating main header
from backup!

Invalid partition data!
```

### 8.2. 部署失败后清理重新部署

- 执行下面的命令清理环境，再重新部署

```shell
rm -rf /var/lib/rook/
dd if=/dev/zero of=/dev/sdc bs=512k count=1
wipefs -af /dev/sdc
```

### 8.3. Dashboard 打不开

当你配置完在集群外部访问 Dashboard，用浏览器打开页面，出现以下问题

- 页面一致处于加载状态，浏览器的左下角会显示一个 10开头的K8s 内部网络IP
- 用浏览器的开发者工具观察网络请求，也会有访问 10开头的K8s 内部网络IP 的请求
- 过一段时间就会超时并跳转到 Mgr 的 POD IP

使用如下的资源清单重新创建服务能暂时解决问题，深层次的原因还需要再分析

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-mgr-dashboard-external-https
  namespace: rook-ceph
  labels:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
spec:
  ports:
  - name: dashboard
    port: 8443
    protocol: TCP
    targetPort: 8443
    nodePort: 31443
  selector:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
    mgr: a
  sessionAffinity: None
  type: NodePort
```

> 主要是加了一个`mgr: a`的selector，因为mgr是主备模式的，默认a处于活动状态，因此要选择a才能打开



## 结束语

本系列文档是我在云原生技术领域的学习和运维实践的手记，**用输出倒逼输入**是一种高效的学习方法，能够快速积累经验和提高技术，只有把学到的知识写出来并能够让其他人理解，才能说明真正掌握了这项知识。

> **本系列文档内容涵盖 (但不限于) 以下技术领域：**

- **KubeSphere**
- **Kubernetes**
- **Ansible**
- **自动化运维**
- **CNCF 技术栈**

**如果你喜欢本文，请分享给你的小伙伴！**

> **Get 文档**

- Github https://github.com/devops/z-notes
- Gitee https://gitee.com/zdevops/z-notes
- 知乎 https://www.zhihu.com/people/zdevops/

> **Get 代码**

- Github https://github.com/devops/ansible-zdevops
- Gitee https://gitee.com/zdevops/ansible-zdevops

> **Get 视频 B 站**

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
