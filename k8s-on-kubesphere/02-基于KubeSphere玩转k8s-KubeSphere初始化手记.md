# 基于 KubeSphere 玩转 k8s-KubeSphere 初始化手记

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

本文接着上篇 **<< 基于 KubeSphere 玩转 k8s-KubeSphere 安装手记 >>** ，继续玩转 KubeSphere、k8s，本期会讲解 KubeSphere 默认安装完成后的一些必要配置。同时，会初步探究一下执行这些必要配置后 k8s 底层发生了哪些变化。

> **本文知识量**

- 阅读时长：23 分
- 行：1513
- 单词：8626
- 字符：67982
- 图片：40 张

> **本文知识点**

- 定级：**入门级**
- KubeSphere 更改默认管理用户的密码
- KubeSphere 可插拔组件的启用和配置
- KubeSphere 用户管理
- KubeSphere 企业空间管理
- KubeSphere 项目管理

> **演示服务器配置**

|      主机名      |      IP      | CPU  | 内存 | 系统盘 | 数据盘 |               用途               |
| :--------------: | :----------: | :--: | :--: | :----: | :----: | :------------------------------: |
|  zdeops-master   | 192.168.9.9  |  2   |  4   |   40   |  200   |       Ansible 运维控制节点        |
| ks-k8s-master-0  | 192.168.9.91 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-1  | 192.168.9.92 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-2  | 192.168.9.93 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| glusterfs-node-0 | 192.168.9.95 |  4   |  8   |   40   |  200   |            GlusterFS             |
| glusterfs-node-1 | 192.168.9.96 |  4   |  8   |   40   |  200   |            GlusterFS             |
| glusterfs-node-2 | 192.168.9.97 |  4   |  8   |   40   |  200   |            GlusterFS             |

---

## 2. 首次登陆配置

### 2.1. 更改默认用户密码

浏览器打开 KubeSphere 管理界面。

弹出登陆页面，使用默认用户 **admin**，密码 **P@88w0rd**，登录。

![kubesphere-login-default](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-login-default.png)

系统会提示你重置密码，按要求重置提交即可，当然你也可以选择**稍后修改**，登录以后再修改，但是**工作环境**强烈建议立即修改。

![kubesphere-login-reset](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-login-reset.png)

登录后显示默认的登录页面，即**工作台**，该页面可以看到以下信息。
- 平台信息：平台版本和集群数量

- 平台资源：最后更新时间、企业空间数量、用户数量

- 最近访问记录


![kubesphere-dashboard](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-dashboard.png)

点击左上角的**平台管理**，选择**集群管理**，进入集群管理界面，进行我们接下来的配置。

![kubesphere-clusters-manage](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-clusters-manage.png)

![kubesphere-clusters-overview](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-clusters-overview.png)

---

## 3. 启用可插拔组件

### 3.1. 可插拔组件概述

从 2.1.0 版本开始，KubeSphere 解耦了一些核心功能组件。这些组件设计成了可插拔式，您可以在安装之前或之后启用它们。如果您不启用它们，KubeSphere 会默认以最小化进行安装部署。

不同的可插拔组件部署在不同的命名空间中。您可以根据需求启用任意组件。强烈建议您安装这些可插拔组件来深度体验 KubeSphere 提供的全栈特性和功能。

有关如何启用每个组件的更多信息，请参见[官方文档](https://kubesphere.com.cn/docs/pluggable-components/overview/)的各个教程。

### 3.2. 可插拔组件资源要求

在您启用可插拔组件之前，请确保您的环境中有足够的资源 . 否则，可能会因为缺乏资源导致组件崩溃。具体参见下表：

- **CPU 和内存的资源请求和限制均指单个副本的要求。**

- **官方列出的可插拔组件的资源需求，并不完整，没有涵盖所有组件，因此建议把服务器资源配置的尽可能大一些。**

> **官方资源要求**

- **KubeSphere 应用商店**

| 命名空间 | openpitrix-system                        |
| :------- | :--------------------------------------- |
| CPU 请求 | 0.3 核                                   |
| CPU 限制 | 无                                       |
| 内存请求 | 300 MiB                                  |
| 内存限制 | 无                                       |
| 安装     | 可选                                     |
| 备注     | 该组件可用于管理应用生命周期。建议安装。 |

- **KubeSphere DevOps 系统**

| 命名空间 | kubesphere-devops-system                                     | kubesphere-devops-system         |
| :------- | :----------------------------------------------------------- | :------------------------------- |
| 安装模式 | All-in-One 安装                                              | 多节点安装                       |
| CPU 请求 | 34 m                                                         | 0.47 核                          |
| CPU 限制 | 无                                                           | 无                               |
| 内存请求 | 2.69 G                                                       | 8.6 G                            |
| 内存限制 | 无                                                           | 无                               |
| 安装     | 可选                                                         | 可选                             |
| 备注     | 提供一站式 DevOps 解决方案，包括 Jenkins 流水线、B2I 和 S2I。 | 其中一个节点的内存必须大于 8 G。 |

- **KubeSphere 监控系统**

| 命名空间 | kubesphere-monitoring-system                                 | kubesphere-monitoring-system | kubesphere-monitoring-system |
| -------- | ------------------------------------------------------------ | ---------------------------- | ---------------------------- |
| 子组件   | 2 x Prometheus                                               | 3 x Alertmanager             | Notification Manager         |
| CPU 请求 | 100 m                                                        | 10 m                         | 100 m                        |
| CPU 限制 | 4 core                                                       | 无                           | 500 m                        |
| 内存请求 | 400 MiB                                                      | 30 MiB                       | 20 MiB                       |
| 内存限制 | 8 GiB                                                        |                              | 1 GiB                        |
| 安装     | 必需                                                         | 必需                         | 必需                         |
| 备注     | Prometheus 的内存消耗取决于集群大小。8 GiB 可满足 200 个节点 /16,000 个容器组的集群规模。 |                              |                              |

**KubeSphere 监控系统不是可插拔组件，会默认安装。** 它与其他组件（例如日志系统）紧密关联，因此将其资源请求和限制也列在本页中，供您参考。

- **KubeSphere 日志系统**

| 命名空间 | kubesphere-logging-system                                    | kubesphere-logging-system                    | kubesphere-logging-system               | kubesphere-logging-system                           |
| -------- | ------------------------------------------------------------ | -------------------------------------------- | --------------------------------------- | --------------------------------------------------- |
| 子组件   | 3 x Elasticsearch                                            | fluent bit                                   | kube-events                             | kube-auditing                                       |
| CPU 请求 | 50 m                                                         | 20 m                                         | 90 m                                    | 20 m                                                |
| CPU 限制 | 1 core                                                       | 200 m                                        | 900 m                                   | 200 m                                               |
| 内存请求 | 2 G                                                          | 50 MiB                                       | 120 MiB                                 | 50 MiB                                              |
| 内存限制 | 无                                                           | 100 MiB                                      | 1200 MiB                                | 100 MiB                                             |
| 安装     | 可选                                                         | 必需                                         | 可选                                    | 可选                                                |
| 备注     | 可选组件，用于存储日志数据。不建议在生产环境中使用内置 Elasticsearch。 | 日志收集代理。启用日志系统后，它是必需组件。 | Kubernetes 事件收集、过滤、导出和告警。 | Kubernetes 和 KubeSphere 审计日志收集、过滤和告警。 |

- **KubeSphere 告警和通知**

| 命名空间 | kubesphere-alerting-system |
| -------- | -------------------------- |
| CPU 请求 | 0.08 core                  |
| CPU 限制 | 无                         |
| 内存请求 | 80 M                       |
| 内存限制 | 无                         |
| 安装     | 可选                       |
| 备注     | 告警和通知需要同时启用。   |

- **KubeSphere 服务网格**

| 命名空间 | istio-system                                           |
| -------- | ------------------------------------------------------ |
| CPU 请求 | 1 core                                                 |
| CPU 限制 | 无                                                     |
| 内存请求 | 3.5 G                                                  |
| 内存限制 | 无                                                     |
| 安装     | 可选                                                   |
| 备注     | 支持灰度发布策略、流量拓扑、流量管理和分布式链路追踪。 |

> **官方可插拔组件汇总**

| 组件名称                                                     | 官方是否提供资源要求建议 |                             备注                             |
| ------------------------------------------------------------ | :----------------------: | :----------------------------------------------------------: |
| KubeSphere 应用商店                                          |            是            | KubeSphere 在 [OpenPitrix](https://github.com/openpitrix/openpitrix) 的基础上，为用户提供了一个基于 Helm 的应用商店，用于应用生命周期管理。 |
| KubeSphere DevOps                                            |            是            | 基于 [Jenkins](https://jenkins.io/) 的 KubeSphere DevOps 系统是专为 Kubernetes 中的 CI/CD 工作流设计的，它还具有插件管理、[Binary-to-Image (B2I)](https://kubesphere.com.cn/docs/project-user-guide/image-builder/binary-to-image/)、[Source-to-Image (S2I)](https://kubesphere.com.cn/docs/project-user-guide/image-builder/source-to-image/)、代码依赖缓存、代码质量分析、流水线日志等功能。 |
| KubeSphere 监控系统                                          |            是            | KubeSphere 在 Prometheus 和 Alertmanager 的基础上构建了监控系统，不是可插拔组件，会默认安装。 |
| KubeSphere 日志系统                                          |            是            | KubeSphere 为日志收集、查询和管理提供了一个强大的、全面的、易于使用的日志系统。它涵盖了不同层级的日志，包括租户、基础设施资源和应用。 |
| KubeSphere 告警系统                                          |            是            | KubeSphere 中的告警系统与其主动式故障通知 (Proactive Failure Notification)  系统相结合，使用户可以基于告警策略了解感兴趣的活动。当达到某个指标的预定义阈值时，会向预先配置的收件人发出告警。因此，您需要预先配置通知方式，包括邮件、Slack、钉钉、企业微信和 Webhook。 |
| KubeSphere 服务网格                                          |            是            | KubeSphere 服务网格基于 [Istio](https://istio.io/)，将微服务治理和流量管理可视化。 |
| [网络策略](https://kubesphere.com.cn/docs/pluggable-components/network-policy/) |            否            | 网络策略是一种以应用为中心的结构，使您能够指定如何允许容器组通过网络与各种网络实体进行通信。 |
| [Metrics Server](https://kubesphere.com.cn/docs/pluggable-components/metrics-server/) |            否            | KubeSphere 支持用于[部署](https://kubesphere.com.cn/docs/project-user-guide/application-workloads/deployments/)的容器组（Pod）弹性伸缩程序 (HPA)。在 KubeSphere 中，Metrics Server 控制着 HPA 是否启用。 |
| [服务拓扑图](https://kubesphere.com.cn/docs/pluggable-components/service-topology/) |            否            | 您可以启用服务拓扑图以集成 [Weave Scope](https://www.weave.works/oss/scope/)（Docker 和 Kubernetes 的可视化和监控工具）。 |
| [容器组 IP 池](https://kubesphere.com.cn/docs/pluggable-components/pod-ip-pools/) |            否            | 容器组 IP 池用于规划容器组网络地址空间，每个容器组 IP 池之间的地址空间不能重叠。创建工作负载时，可选择特定的容器组 IP 池，这样创建出的容器组将从该容器组 IP 池中分配 IP 地址。 |
| [KubeEdge](https://kubesphere.com.cn/docs/pluggable-components/kubeedge/) |            否            | [KubeEdge](https://kubeedge.io/zh/) 是一个开源系统，用于将容器化应用程序编排功能扩展到边缘的主机。KubeEdge 支持多个边缘协议，旨在对部署于云端和边端的应用程序与资源等进行统一管理。 |

> **官方资源要求汇总**

| 组件名称                                 | CPU 请求 (核) | CPU 限制 (核) | 内存请求 (MiB) | 内存限制 (MiB) | 默认安装 |              备注              |
| ---------------------------------------- | :---------: | :---------: | :-----------: | :-----------: | :------: | :----------------------------: |
| KubeSphere 应用商店                      |     0.3     |      0      |      300      |       0       |    否    |         可选，建议安装         |
| KubeSphere DevOps                        |    0.47     |      0      |     8600      |       0       |    否    |         可选，建议安装         |
| KubeSphere 监控系统-Prometheus           |     0.1     |      4      |      400      |     8000      |    是    |              2 台               |
| KubeSphere 监控系统-Alertmanager         |    0.01     |      0      |      30       |       0       |    是    |              3 台               |
| KubeSphere 监控系统-Notification Manager |     0.1     |     0.5     |      20       |     1000      |    是    |                                |
| KubeSphere 日志系统-Elasticsearch        |    0.05     |      1      |     2000      |       0       |    否    | 可选，生产不建议开启，建议外置 |
| KubeSphere 日志系统-fluent bit           |    0.02     |     0.2     |      50       |      100      |    否    |      必须，启用日志系统后      |
| KubeSphere 日志系统-kube-events          |    0.09     |     0.9     |      120      |     1200      |    否    |              可选              |
| KubeSphere 日志系统-kube-auditing        |    0.02     |     0.2     |      50       |      100      |    否    |              可选              |
| KubeSphere 告警系统                      |    0.08     |      0      |      80       |       0       |    否    |              可选              |
| KubeSphere 服务网格                      |      1      |      0      |     3500      |       0       |    否    |              可选              |
| 合计                                     |  **2.24**   |  **8.66**   |   **15150**   |   **24910**   |          |                                |

> **本文启用插件资源要求汇总**

| 组件名称                                 | CPU 请求 (核) | CPU 限制 (核) | 内存请求 (MiB) | 内存限制 (MiB) | 是否启用 |                             备注                             |
| ---------------------------------------- | :---------: | :---------: | :-----------: | :-----------: | :------: | :----------------------------------------------------------: |
| KubeSphere 应用商店                      |     0.3     |      0      |      300      |       0       |    是    |                             必选                             |
| KubeSphere DevOps 系统                   |    0.47     |      0      |     8600      |       0       |    是    |                             必选                             |
| KubeSphere 监控系统-Prometheus           |     0.1     |      4      |      400      |     8000      |    是    |                           默认已开                           |
| KubeSphere 监控系统-Alertmanager         |    0.01     |      0      |      30       |       0       |    是    |                           默认已开                           |
| KubeSphere 监控系统-Notification Manager |     0.1     |     0.5     |      20       |     1000      |    是    |                           默认已开                           |
| KubeSphere 日志系统-Elasticsearch        |    0.05     |      1      |     2000      |       0       |    否    |                     不开启，外部独立部署                     |
| KubeSphere 日志系统-fluent bit           |    0.02     |     0.2     |      50       |      100      |    否    |      暂时不开启，等外部的 Elasticsearch 部署完成后再开启       |
| KubeSphere 日志系统-kube-events          |    0.09     |     0.9     |      120      |     1200      |    否    |      暂时不开启，等外部的 Elasticsearch 部署完成后再开启       |
| KubeSphere 日志系统-kube-auditing        |    0.02     |     0.2     |      50       |      100      |    否    |      暂时不开启，等外部的 Elasticsearch 部署完成后再开启       |
| KubeSphere 告警系统                      |    0.08     |      0      |      80       |       0       |    是    |                          可选，建议                          |
| KubeSphere 服务网格                      |      1      |      0      |     3500      |       0       |    否    | 暂时不开启，等后期再考虑 , 生产环境计划使用服务网功能的建议开启 |
| 网络策略                                 |             |             |               |               |    否    |                     没有需求，暂时不开启                     |
| Metrics Server                           |             |             |               |               |    否    |                     没有需求，暂时不开启                     |
| 服务拓扑图                               |             |             |               |               |    否    |                     没有需求，暂时不开启                     |
| 容器组 IP 池                             |             |             |               |               |    否    |                     没有需求，暂时不开启                     |
| KubeEdge                                 |             |             |               |               |    否    |                     没有需求，暂时不开启                     |
| 合计                                     |  **1.06**   |  **5.36**   |   **9430**    |   **18010**   |          |                                                              |

### **3.3. 开启可插拔组件具体配置**

上文介绍了可插拔组件的概念、官方可用的可插拔组件、可插拔组件的资源要求等，下面开始介绍可插拔组件具体的开启配置方法。

**可插拔组件的启用可以在部署 KubeSphere 的配置文件中开启，也可以在安装完 KubeSphere 系统后再开启，本文介绍的是安装之后的启用方式**。

使用 **admin** 用户登录控制台,点击左上角**平台管理**-> 选择**集群管理**-> 集群管理界面中，左侧菜单点击 **CRD**。

![kubesphere-clustesr-crd](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-clusters-crd.png)

搜索栏中输入 **clusterconfiguration**，点击结果查看其详细页面。

![kubesphere-clusters-crd-clusterconfiguration](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-clusters-crd-clusterconfiguration.png)

![kubesphere-clusters-crd-clusterconfiguration-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-clusters-crd-clusterconfiguration-1.png)

在**自定义资源**中，点击 **ks-installer** 右侧的 ![img](https://kubesphere.com.cn/images/docs/zh-cn/enable-pluggable-components/kubesphere-app-store/three-dots.png)，选择**编辑 YAML**。

![kubesphere-clusters-crd-clusterconfiguration-2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-clusters-crd-clusterconfiguration-2.png)

![kubesphere-clusters-crd-clusterconfiguration-3](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-clusters-crd-clusterconfiguration-3.png)

在该 **YAML** 文件中，启用需要开启的可插拔组件。每个组件的详细启用方法如下：

> **KubeSphere 应用商店**

在 YAML 文件中，搜索 **openpitrix**，并将 **enabled** 的 **false** 改为 **true**。

```yaml
openpitrix:
  store:
    enabled: true
```

> **KubeSphere DevOps**

在 YAML 文件中，搜索 **devops**，将 **enabled** 的 **false** 改为 **true**。

```yaml
devops:
  enabled: true
  jenkinsJavaOpts_MaxRAM: 2g
  jenkinsJavaOpts_Xms: 512m
  jenkinsJavaOpts_Xmx: 512m
  jenkinsMemoryLim: 2Gi
  jenkinsMemoryReq: 1500Mi
  jenkinsVolumeSize: 8Gi
```

**其他参数使用了默认值，各位可以根据自己的服务器配置和业务规模做出适合的调整，主要就是控制内存和存储空间，业务规模大的一定要加大内存的分配**。

> **KubeSphere 告警系统**

在 YAML 文件中，搜寻 **alerting**，将 **enabled** 的 **false** 更改为 **true**。完成后，点击右下角的**确定**，保存配置。

```yaml
alerting:
  enabled: true
```

所有配置完成后，点击右下角的**确定**，保存配置。

点击**控制台右下角**的 ![img](https://kubesphere.com.cn/images/docs/zh-cn/enable-pluggable-components/kubesphere-app-store/hammer.png) 找到 kubectl 工具。

![kubesphere-kubectl](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-kubectl.png)

![kubesphere-kubectl-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-kubectl-1.png)

在 kubectl 中执行以下命令检查安装过程。

```shell
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
```

看到如下状态，说明安装成功。

```yaml
PLAY RECAP *********************************************************************
localhost                  : ok=26   changed=14   unreachable=0    failed=0    skipped=12   rescued=0    ignored=0

Start installing monitoring
Start installing multicluster
Start installing openpitrix
Start installing network
Start installing alerting
Start installing devops
**************************************************
Waiting for all tasks to be completed ...
task alerting status is successful  (1/6)
task network status is successful  (2/6)
task multicluster status is successful  (3/6)
task openpitrix status is successful  (4/6)
task devops status is successful  (5/6)
task monitoring status is successful  (6/6)
**************************************************
Collecting installation results ...
#####################################################
###              Welcome to KubeSphere!           ###
#####################################################

Console: http://192.168.9.91:30880
Account: admin
Password: P@88w0rd

NOTES：
  1. After you log into the console, please check the
     monitoring status of service components in
     "Cluster Management". If any service is not
     ready, please wait patiently until all components
     are up and running.
  2. Please change the default password after login.

#####################################################
https://kubesphere.io             2022-04-07 13:48:19
#####################################################
```

详细过程分析。

```yaml
2022-04-07T13:38:11+08:00 INFO     : EVENT Kube event '3ededddf-6c59-48dc-a865-e38c7bdf6a3f'
2022-04-07T13:38:11+08:00 INFO     : QUEUE add TASK_HOOK_RUN@KUBE_EVENTS kubesphere/installRunner.py
2022-04-07T13:38:11+08:00 INFO     : TASK_RUN HookRun@KUBE_EVENTS kubesphere/installRunner.py
2022-04-07T13:38:11+08:00 INFO     : Running hook 'kubesphere/installRunner.py' binding 'KUBE_EVENTS' ...
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that
the implicit localhost does not match 'all'

PLAY [localhost] ***************************************************************

TASK [download : Generating images list] ***************************************
skipping: [localhost]

TASK [download : Synchronizing images] *****************************************

TASK [kubesphere-defaults : KubeSphere | Setting images' namespace override] ***
skipping: [localhost]

TASK [kubesphere-defaults : KubeSphere | Configuring defaults] *****************
ok: [localhost] => {
    "msg": "Check roles/kubesphere-defaults/defaults/main.yml"
}

TASK [preinstall : KubeSphere | Stopping if Kubernetes version is nonsupport] ***
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [preinstall : KubeSphere | Checking StorageClass] *************************
changed: [localhost]

TASK [preinstall : KubeSphere | Stopping if StorageClass was not found] ********
skipping: [localhost]

TASK [preinstall : KubeSphere | Checking default StorageClass] *****************
changed: [localhost]

TASK [preinstall : KubeSphere | Stopping if default StorageClass was not found] ***
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [preinstall : KubeSphere | Checking KubeSphere component] *****************
changed: [localhost]

TASK [preinstall : KubeSphere | Getting KubeSphere component version] **********
changed: [localhost]

TASK [preinstall : KubeSphere | Getting KubeSphere component version] **********
skipping: [localhost] => (item=ks-openldap)
skipping: [localhost] => (item=ks-redis)
skipping: [localhost] => (item=ks-minio)
skipping: [localhost] => (item=ks-openpitrix)
skipping: [localhost] => (item=elasticsearch-logging)
skipping: [localhost] => (item=elasticsearch-logging-curator)
skipping: [localhost] => (item=istio)
skipping: [localhost] => (item=istio-init)
skipping: [localhost] => (item=jaeger-operator)
skipping: [localhost] => (item=ks-jenkins)
skipping: [localhost] => (item=ks-sonarqube)
skipping: [localhost] => (item=logging-fluentbit-operator)
skipping: [localhost] => (item=uc)
skipping: [localhost] => (item=metrics-server)

PLAY RECAP *********************************************************************
localhost                  : ok=7    changed=4    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0

[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that
the implicit localhost does not match 'all'

PLAY [localhost] ***************************************************************

TASK [download : Generating images list] ***************************************
skipping: [localhost]

TASK [download : Synchronizing images] *****************************************

TASK [kubesphere-defaults : KubeSphere | Setting images' namespace override] ***
skipping: [localhost]

TASK [kubesphere-defaults : KubeSphere | Configuring defaults] *****************
ok: [localhost] => {
    "msg": "Check roles/kubesphere-defaults/defaults/main.yml"
}

TASK [Metrics-Server | Getting metrics-server installation files] **************
skipping: [localhost]

TASK [metrics-server : Metrics-Server | Creating manifests] ********************
skipping: [localhost] => (item={'file': 'metrics-server.yaml'})

TASK [metrics-server : Metrics-Server | Checking Metrics-Server] ***************
skipping: [localhost]

TASK [Metrics-Server | Uninstalling old metrics-server] ************************
skipping: [localhost]

TASK [Metrics-Server | Installing new metrics-server] **************************
skipping: [localhost]

TASK [metrics-server : Metrics-Server | Waitting for metrics.k8s.io ready] *****
skipping: [localhost]

TASK [Metrics-Server | Importing metrics-server status] ************************
skipping: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=10   rescued=0    ignored=0

[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that
the implicit localhost does not match 'all'

PLAY [localhost] ***************************************************************

TASK [download : Generating images list] ***************************************
skipping: [localhost]

TASK [download : Synchronizing images] *****************************************

TASK [kubesphere-defaults : KubeSphere | Setting images' namespace override] ***
skipping: [localhost]

TASK [kubesphere-defaults : KubeSphere | Configuring defaults] *****************
ok: [localhost] => {
    "msg": "Check roles/kubesphere-defaults/defaults/main.yml"
}

TASK [common : KubeSphere | Checking kube-node-lease namespace] ****************
changed: [localhost]

TASK [common : KubeSphere | Getting system namespaces] *************************
ok: [localhost]

TASK [common : set_fact] *******************************************************
ok: [localhost]

TASK [common : debug] **********************************************************
ok: [localhost] => {
    "msg": [
        "kubesphere-system",
        "kubesphere-controls-system",
        "kubesphere-monitoring-system",
        "kubesphere-monitoring-federated",
        "kube-node-lease",
        "kubesphere-devops-system"
    ]
}

TASK [common : KubeSphere | Creating KubeSphere namespace] *********************
changed: [localhost] => (item=kubesphere-system)
changed: [localhost] => (item=kubesphere-controls-system)
changed: [localhost] => (item=kubesphere-monitoring-system)
changed: [localhost] => (item=kubesphere-monitoring-federated)
changed: [localhost] => (item=kube-node-lease)
changed: [localhost] => (item=kubesphere-devops-system)

TASK [common : KubeSphere | Labeling system-workspace] *************************
changed: [localhost] => (item=default)
changed: [localhost] => (item=kube-public)
changed: [localhost] => (item=kube-system)
changed: [localhost] => (item=kubesphere-system)
changed: [localhost] => (item=kubesphere-controls-system)
changed: [localhost] => (item=kubesphere-monitoring-system)
changed: [localhost] => (item=kubesphere-monitoring-federated)
changed: [localhost] => (item=kube-node-lease)
changed: [localhost] => (item=kubesphere-devops-system)

TASK [common : KubeSphere | Labeling namespace for network policy] *************
changed: [localhost]

TASK [common : KubeSphere | Getting Kubernetes master num] *********************
changed: [localhost]

TASK [common : KubeSphere | Setting master num] ********************************
ok: [localhost]

TASK [KubeSphere | Getting common component installation files] ****************
ok: [localhost] => (item=common)

TASK [common : KubeSphere | Checking Kubernetes version] ***********************
changed: [localhost]

TASK [KubeSphere | Getting common component installation files] ****************
ok: [localhost] => (item=snapshot-controller)

TASK [common : KubeSphere | Creating snapshot controller values] ***************
ok: [localhost] => (item={'name': 'custom-values-snapshot-controller', 'file': 'custom-values-snapshot-controller.yaml'})

TASK [common : KubeSphere | Updating snapshot crd] *****************************
changed: [localhost]

TASK [common : KubeSphere | Deploying snapshot controller] *********************
changed: [localhost]

TASK [KubeSphere | Checking openpitrix common component] ***********************
changed: [localhost]

TASK [common : include_tasks] **************************************************
skipping: [localhost] => (item={'op': 'openpitrix-db', 'ks': 'mysql-pvc'})
skipping: [localhost] => (item={'op': 'openpitrix-etcd', 'ks': 'etcd-pvc'})

TASK [common : Getting PersistentVolumeName (mysql)] ***************************
skipping: [localhost]

TASK [common : Getting PersistentVolumeSize (mysql)] ***************************
skipping: [localhost]

TASK [common : Setting PersistentVolumeName (mysql)] ***************************
skipping: [localhost]

TASK [common : Setting PersistentVolumeSize (mysql)] ***************************
skipping: [localhost]

TASK [common : Getting PersistentVolumeName (etcd)] ****************************
skipping: [localhost]

TASK [common : Getting PersistentVolumeSize (etcd)] ****************************
skipping: [localhost]

TASK [common : Setting PersistentVolumeName (etcd)] ****************************
skipping: [localhost]

TASK [common : Setting PersistentVolumeSize (etcd)] ****************************
skipping: [localhost]

TASK [common : KubeSphere | Checking mysql PersistentVolumeClaim] **************
changed: [localhost]

TASK [common : KubeSphere | Setting mysql db pv size] **************************
skipping: [localhost]

TASK [common : KubeSphere | Checking redis PersistentVolumeClaim] **************
changed: [localhost]

TASK [common : KubeSphere | Setting redis db pv size] **************************
skipping: [localhost]

TASK [common : KubeSphere | Checking minio PersistentVolumeClaim] **************
changed: [localhost]

TASK [common : KubeSphere | Setting minio pv size] *****************************
skipping: [localhost]

TASK [common : KubeSphere | Checking openldap PersistentVolumeClaim] ***********
changed: [localhost]

TASK [common : KubeSphere | Setting openldap pv size] **************************
skipping: [localhost]

TASK [common : KubeSphere | Checking etcd db PersistentVolumeClaim] ************
changed: [localhost]

TASK [common : KubeSphere | Setting etcd pv size] ******************************
skipping: [localhost]

TASK [common : KubeSphere | Checking redis ha PersistentVolumeClaim] ***********
changed: [localhost]

TASK [common : KubeSphere | Setting redis ha pv size] **************************
ok: [localhost]

TASK [common : KubeSphere | Checking es-master PersistentVolumeClaim] **********
changed: [localhost]

TASK [common : KubeSphere | Setting es master pv size] *************************
skipping: [localhost]

TASK [common : KubeSphere | Checking es data PersistentVolumeClaim] ************
changed: [localhost]

TASK [common : KubeSphere | Setting es data pv size] ***************************
skipping: [localhost]

TASK [KubeSphere | Creating common component manifests] ************************
ok: [localhost] => (item={'path': 'redis', 'file': 'redis.yaml'})

TASK [common : KubeSphere | Deploying etcd and mysql] **************************
skipping: [localhost] => (item=etcd.yaml)
skipping: [localhost] => (item=mysql.yaml)

TASK [common : KubeSphere | Getting minio installation files] ******************
skipping: [localhost] => (item=minio-ha)

TASK [common : KubeSphere | Creating manifests] ********************************
skipping: [localhost] => (item={'name': 'custom-values-minio', 'file': 'custom-values-minio.yaml'})

TASK [common : KubeSphere | Checking minio] ************************************
skipping: [localhost]

TASK [common : KubeSphere | Deploying minio] ***********************************
skipping: [localhost]

TASK [common : debug] **********************************************************
skipping: [localhost]

TASK [common : fail] ***********************************************************
skipping: [localhost]

TASK [common : KubeSphere | Importing minio status] ****************************
skipping: [localhost]

TASK [common : KubeSphere | Generet Random password] ***************************
skipping: [localhost]

TASK [common : KubeSphere | Creating Redis Password Secret] ********************
skipping: [localhost]

TASK [common : KubeSphere | Getting redis installation files] ******************
skipping: [localhost] => (item=redis-ha)

TASK [common : KubeSphere | Creating manifests] ********************************
skipping: [localhost] => (item={'name': 'custom-values-redis', 'file': 'custom-values-redis.yaml'})

TASK [common : KubeSphere | Checking old redis status] *************************
skipping: [localhost]

TASK [common : KubeSphere | Deleting and backup old redis svc] *****************
skipping: [localhost]

TASK [common : KubeSphere | Deploying redis] ***********************************
skipping: [localhost]

TASK [common : KubeSphere | Deploying redis] ***********************************
skipping: [localhost] => (item=redis.yaml)

TASK [common : KubeSphere | Importing redis status] ****************************
skipping: [localhost]

TASK [common : KubeSphere | Getting openldap installation files] ***************
skipping: [localhost] => (item=openldap-ha)

TASK [common : KubeSphere | Creating manifests] ********************************
skipping: [localhost] => (item={'name': 'custom-values-openldap', 'file': 'custom-values-openldap.yaml'})

TASK [common : KubeSphere | Checking old openldap status] **********************
skipping: [localhost]

TASK [common : KubeSphere | Shutdown ks-account] *******************************
skipping: [localhost]

TASK [common : KubeSphere | Deleting and backup old openldap svc] **************
skipping: [localhost]

TASK [common : KubeSphere | Checking openldap] *********************************
skipping: [localhost]

TASK [common : KubeSphere | Deploying openldap] ********************************
skipping: [localhost]

TASK [common : KubeSphere | Loading old openldap data] *************************
skipping: [localhost]

TASK [common : KubeSphere | Checking openldap-ha status] ***********************
skipping: [localhost]

TASK [common : KubeSphere | Getting openldap-ha pod list] **********************
skipping: [localhost]

TASK [common : KubeSphere | Getting old openldap data] *************************
skipping: [localhost]

TASK [common : KubeSphere | Migrating openldap data] ***************************
skipping: [localhost]

TASK [common : KubeSphere | Disabling old openldap] ****************************
skipping: [localhost]

TASK [common : KubeSphere | Restarting openldap] *******************************
skipping: [localhost]

TASK [common : KubeSphere | Restarting ks-account] *****************************
skipping: [localhost]

TASK [common : KubeSphere | Importing openldap status] *************************
skipping: [localhost]

TASK [common : KubeSphere | Generet Random password] ***************************
ok: [localhost]

TASK [common : KubeSphere | Creating Redis Password Secret] ********************
changed: [localhost]

TASK [common : KubeSphere | Getting redis installation files] ******************
ok: [localhost] => (item=redis-ha)

TASK [common : KubeSphere | Creating manifests] ********************************
ok: [localhost] => (item={'name': 'custom-values-redis', 'file': 'custom-values-redis.yaml'})

TASK [common : KubeSphere | Checking old redis status] *************************
changed: [localhost]

TASK [common : KubeSphere | Deleting and backup old redis svc] *****************
skipping: [localhost]

TASK [common : KubeSphere | Deploying redis] ***********************************
changed: [localhost]

TASK [common : KubeSphere | Deploying redis] ***********************************
skipping: [localhost] => (item=redis.yaml)

TASK [common : KubeSphere | Importing redis status] ****************************
changed: [localhost]

TASK [common : KubeSphere | Getting openldap installation files] ***************
changed: [localhost] => (item=openldap-ha)

TASK [common : KubeSphere | Creating manifests] ********************************
changed: [localhost] => (item={'name': 'custom-values-openldap', 'file': 'custom-values-openldap.yaml'})

TASK [common : KubeSphere | Checking old openldap status] **********************
changed: [localhost]

TASK [common : KubeSphere | Shutdown ks-account] *******************************
skipping: [localhost]

TASK [common : KubeSphere | Deleting and backup old openldap svc] **************
skipping: [localhost]

TASK [common : KubeSphere | Checking openldap] *********************************
changed: [localhost]

TASK [common : KubeSphere | Deploying openldap] ********************************
changed: [localhost]

TASK [common : KubeSphere | Loading old openldap data] *************************
skipping: [localhost]

TASK [common : KubeSphere | Checking openldap-ha status] ***********************
skipping: [localhost]

TASK [common : KubeSphere | Getting openldap-ha pod list] **********************
skipping: [localhost]

TASK [common : KubeSphere | Getting old openldap data] *************************
skipping: [localhost]

TASK [common : KubeSphere | Migrating openldap data] ***************************
skipping: [localhost]

TASK [common : KubeSphere | Disabling old openldap] ****************************
skipping: [localhost]

TASK [common : KubeSphere | Restarting openldap] *******************************
skipping: [localhost]

TASK [common : KubeSphere | Restarting ks-account] *****************************
skipping: [localhost]

TASK [common : KubeSphere | Importing openldap status] *************************
changed: [localhost]

TASK [common : KubeSphere | Getting minio installation files] ******************
changed: [localhost] => (item=minio-ha)

TASK [common : KubeSphere | Creating manifests] ********************************
changed: [localhost] => (item={'name': 'custom-values-minio', 'file': 'custom-values-minio.yaml'})

TASK [common : KubeSphere | Checking minio] ************************************
changed: [localhost]

TASK [common : KubeSphere | Deploying minio] ***********************************
changed: [localhost]

TASK [common : debug] **********************************************************
skipping: [localhost]

TASK [common : fail] ***********************************************************
skipping: [localhost]

TASK [common : KubeSphere | Importing minio status] ****************************
changed: [localhost]

TASK [common : KubeSphere | Getting elasticsearch and curator installation files] ***
skipping: [localhost]

TASK [common : KubeSphere | Creating custom manifests] *************************
skipping: [localhost] => (item={'name': 'custom-values-elasticsearch', 'file': 'custom-values-elasticsearch.yaml'})
skipping: [localhost] => (item={'name': 'custom-values-elasticsearch-curator', 'file': 'custom-values-elasticsearch-curator.yaml'})

TASK [common : KubeSphere | Checking elasticsearch data StatefulSet] ***********
skipping: [localhost]

TASK [common : KubeSphere | Checking elasticsearch storageclass] ***************
skipping: [localhost]

TASK [common : KubeSphere | Commenting elasticsearch storageclass parameter] ***
skipping: [localhost]

TASK [common : KubeSphere | Creating elasticsearch credentials secret] *********
skipping: [localhost]

TASK [common : KubeSphere | Checking internal es] ******************************
skipping: [localhost]

TASK [common : KubeSphere | Deploying elasticsearch-logging] *******************
skipping: [localhost]

TASK [common : KubeSphere | Getting PersistentVolume Name] *********************
skipping: [localhost]

TASK [common : KubeSphere | Patching PersistentVolume (persistentVolumeReclaimPolicy)] ***
skipping: [localhost]

TASK [common : KubeSphere | Deleting elasticsearch] ****************************
skipping: [localhost]

TASK [common : KubeSphere | Waiting for seconds] *******************************
skipping: [localhost]

TASK [common : KubeSphere | Deploying elasticsearch-logging] *******************
skipping: [localhost]

TASK [common : KubeSphere | Importing es status] *******************************
skipping: [localhost]

TASK [common : KubeSphere | Deploying elasticsearch-logging-curator] ***********
skipping: [localhost]

TASK [common : KubeSphere | Getting fluentbit installation files] **************
skipping: [localhost]

TASK [common : KubeSphere | Creating custom manifests] *************************
skipping: [localhost] => (item={'path': 'fluentbit', 'file': 'custom-fluentbit-fluentBit.yaml'})
skipping: [localhost] => (item={'path': 'init', 'file': 'custom-fluentbit-operator-deployment.yaml'})

TASK [common : KubeSphere | Preparing fluentbit operator setup] ****************
skipping: [localhost]

TASK [common : KubeSphere | Deploying new fluentbit operator] ******************
skipping: [localhost]

TASK [common : KubeSphere | Importing fluentbit status] ************************
skipping: [localhost]

TASK [common : Setting persistentVolumeReclaimPolicy (mysql)] ******************
skipping: [localhost]

TASK [common : Setting persistentVolumeReclaimPolicy (etcd)] *******************
skipping: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=45   changed=32   unreachable=0    failed=0    skipped=88   rescued=0    ignored=0

[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that
the implicit localhost does not match 'all'

PLAY [localhost] ***************************************************************

TASK [download : Generating images list] ***************************************
skipping: [localhost]

TASK [download : Synchronizing images] *****************************************

TASK [kubesphere-defaults : KubeSphere | Setting images' namespace override] ***
skipping: [localhost]

TASK [kubesphere-defaults : KubeSphere | Configuring defaults] *****************
ok: [localhost] => {
    "msg": "Check roles/kubesphere-defaults/defaults/main.yml"
}

TASK [ks-core/init-token : KubeSphere | Creating KubeSphere directory] *********
ok: [localhost]

TASK [ks-core/init-token : KubeSphere | Getting installation init files] *******
ok: [localhost] => (item=jwt-script)

TASK [ks-core/init-token : KubeSphere | Creating KubeSphere Secret] ************
changed: [localhost]

TASK [ks-core/init-token : KubeSphere | Creating KubeSphere Secret] ************
ok: [localhost]

TASK [ks-core/init-token : KubeSphere | Creating KubeSphere Secret] ************
skipping: [localhost]

TASK [ks-core/init-token : KubeSphere | Enabling Token Script] *****************
ok: [localhost]

TASK [ks-core/init-token : KubeSphere | Getting KubeSphere Token] **************
changed: [localhost]

TASK [ks-core/init-token : KubeSphere | Checking KubeSphere secrets] ***********
changed: [localhost]

TASK [ks-core/init-token : KubeSphere | Deleting KubeSphere secret] ************
skipping: [localhost]

TASK [ks-core/init-token : KubeSphere | Creating components token] *************
changed: [localhost]

TASK [ks-core/ks-core : KubeSphere | Setting Kubernetes version] ***************
ok: [localhost]

TASK [ks-core/ks-core : KubeSphere | Getting Kubernetes master num] ************
changed: [localhost]

TASK [ks-core/ks-core : KubeSphere | Setting master num] ***********************
ok: [localhost]

TASK [ks-core/ks-core : KubeSphere | Override master num] **********************
skipping: [localhost]

TASK [ks-core/ks-core : KubeSphere | Setting enableHA] *************************
ok: [localhost]

TASK [ks-core/ks-core : KubeSphere | Checking ks-core Helm Release] ************
changed: [localhost]

TASK [ks-core/ks-core : KubeSphere | Checking ks-core Exsit] *******************
changed: [localhost]

TASK [ks-core/ks-core : KubeSphere | Convert ks-core to helm mananged] *********
skipping: [localhost] => (item={'ns': 'kubesphere-controls-system', 'kind': 'serviceaccounts', 'resource': 'kubesphere-cluster-admin', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-controls-system', 'kind': 'serviceaccounts', 'resource': 'kubesphere-router-serviceaccount', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-controls-system', 'kind': 'role', 'resource': 'system:kubesphere-router-role', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-controls-system', 'kind': 'rolebinding', 'resource': 'nginx-ingress-role-nisa-binding', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-controls-system', 'kind': 'deployment', 'resource': 'default-http-backend', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-controls-system', 'kind': 'service', 'resource': 'default-http-backend', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'secrets', 'resource': 'ks-controller-manager-webhook-cert', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'serviceaccounts', 'resource': 'kubesphere', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'configmaps', 'resource': 'ks-console-config', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'configmaps', 'resource': 'ks-router-config', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'configmaps', 'resource': 'sample-bookinfo', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'clusterroles', 'resource': 'system:kubesphere-router-clusterrole', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'clusterrolebindings', 'resource': 'system:nginx-ingress-clusterrole-nisa-binding', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'clusterrolebindings', 'resource': 'system:kubesphere-cluster-admin', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'clusterrolebindings', 'resource': 'kubesphere', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'services', 'resource': 'ks-apiserver', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'services', 'resource': 'ks-console', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'services', 'resource': 'ks-controller-manager', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'deployments', 'resource': 'ks-apiserver', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'deployments', 'resource': 'ks-console', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'deployments', 'resource': 'ks-controller-manager', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'validatingwebhookconfigurations', 'resource': 'users.iam.kubesphere.io','release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'validatingwebhookconfigurations', 'resource': 'resourcesquotas.quota.kubesphere.io', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'validatingwebhookconfigurations', 'resource': 'network.kubesphere.io', 'release': 'ks-core'})
skipping: [localhost] => (item={'ns': 'kubesphere-system', 'kind': 'users.iam.kubesphere.io', 'resource': 'admin', 'release': 'ks-core'})

TASK [ks-core/ks-core : KubeSphere | Patch admin user] *************************
changed: [localhost]

TASK [ks-core/ks-core : KubeSphere | Getting ks-core helm charts] **************
ok: [localhost] => (item=ks-core)

TASK [ks-core/ks-core : KubeSphere | Creating manifests] ***********************
ok: [localhost] => (item={'path': 'ks-core', 'file': 'custom-values-ks-core.yaml'})

TASK [ks-core/ks-core : KubeSphere | Upgrade CRDs] *****************************
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/app_v1beta1_application.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/application.kubesphere.io_helmapplications.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/application.kubesphere.io_helmapplicationversions.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/application.kubesphere.io_helmcategories.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/application.kubesphere.io_helmreleases.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/application.kubesphere.io_helmrepos.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/cluster.kubesphere.io_clusters.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/gateway.kubesphere.io_gateways.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/gateway.kubesphere.io_nginxes.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_federatedrolebindings.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_federatedroles.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_federatedusers.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_globalrolebindings.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_globalroles.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_groupbindings.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_groups.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_loginrecords.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_rolebases.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_users.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_workspacerolebindings.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/iam.kubesphere.io_workspaceroles.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/network.kubesphere.io_ipamblocks.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/network.kubesphere.io_ipamhandles.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/network.kubesphere.io_ippools.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/network.kubesphere.io_namespacenetworkpolicies.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/quota.kubesphere.io_resourcequotas.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/servicemesh.kubesphere.io_servicepolicies.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/servicemesh.kubesphere.io_strategies.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/tenant.kubesphere.io_workspaces.yaml)
changed: [localhost] => (item=/kubesphere/kubesphere/ks-core/crds/tenant.kubesphere.io_workspacetemplates.yaml)

TASK [ks-core/ks-core : KubeSphere | Creating ks-core] *************************
changed: [localhost]

TASK [ks-core/ks-core : KubeSphere | Importing ks-core status] *****************
changed: [localhost]

TASK [ks-core/prepare : KubeSphere | Checking core components (1)] *************
changed: [localhost]

TASK [ks-core/prepare : KubeSphere | Checking core components (2)] *************
changed: [localhost]

TASK [ks-core/prepare : KubeSphere | Checking core components (3)] *************
skipping: [localhost]

TASK [ks-core/prepare : KubeSphere | Checking core components (4)] *************
skipping: [localhost]

TASK [ks-core/prepare : KubeSphere | Updating ks-core status] ******************
skipping: [localhost]

TASK [ks-core/prepare : set_fact] **********************************************
skipping: [localhost]

TASK [ks-core/prepare : KubeSphere | Creating KubeSphere directory] ************
ok: [localhost]

TASK [ks-core/prepare : KubeSphere | Getting installation init files] **********
ok: [localhost] => (item=ks-init)

TASK [ks-core/prepare : KubeSphere | Initing KubeSphere] ***********************
changed: [localhost] => (item=role-templates.yaml)

TASK [ks-core/prepare : KubeSphere | Generating kubeconfig-admin] **************
skipping: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=26   changed=14   unreachable=0    failed=0    skipped=12   rescued=0    ignored=0

Start installing monitoring
Start installing multicluster
Start installing openpitrix
Start installing network
Start installing alerting
Start installing devops
**************************************************
Waiting for all tasks to be completed ...
task alerting status is successful  (1/6)
task network status is successful  (2/6)
task multicluster status is successful  (3/6)
task openpitrix status is successful  (4/6)
task devops status is successful  (5/6)
task monitoring status is successful  (6/6)
**************************************************
Collecting installation results ...
#####################################################
###              Welcome to KubeSphere!           ###
#####################################################

Console: http://192.168.9.91:30880
Account: admin
Password: P@88w0rd

NOTES：
  1. After you log into the console, please check the
     monitoring status of service components in
     "Cluster Management". If any service is not
     ready, please wait patiently until all components
     are up and running.
  2. Please change the default password after login.

#####################################################
https://kubesphere.io             2022-04-07 13:48:19
#####################################################
```

验证各个组件的安装结果(可以刷新页面也可以重新登录控制台)。

> **KubeSphere 应用商店**

登录 **KubeSphere** 控制台，如果您能看到页面左上角的**应用商店**以及其中的应用，则说明安装成功。

![kubesphere-openpitrix](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-openpitrix.png)

![kubesphere-openpitrix-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-openpitrix-1.png)

> **KubeSphere DevOps 系统**

**方法一：** 登录控制台，**平台管理**->**集群管理**->**系统组件**，检查是否 **DevOps** 标签页中的所有组件都处于**健康**状态。如果是，表明组件安装成功。

![kubesphere-devops](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-devops.png)

**方法二：** 通过 kubectl 工具验证。

```shell
/ # kubectl get pod -n kubesphere-devops-system
NAME                                READY   STATUS      RESTARTS   AGE
devops-27488520-s52rq               0/1     Completed   0          4m16s
devops-apiserver-7c6774fff5-qtnl2   1/1     Running     0          19m
devops-controller-98975d478-nbcj9   1/1     Running     0          19m
devops-jenkins-5d744f66b9-cfmrj     1/1     Running     0          19m
s2ioperator-0                       1/1     Running     0          19m
/ #
```

> **KubeSphere 告警系统**

登录控制台，**平台管理**->**集群管理**->**监控告警**，看到**告警消息**和**告警策略**菜单，则说明安装成功。

![kubesphere-clusters-alerts](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-clusters-alerts.png)

![kubesphere-clusters-alert-rules](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-clusters-alert-rules.png)

以上配置开启了本文用到的可插拔组件，其他组件我们后期根据需求开启。

---

## 4. 多租户系统

### 4.1. 多租户系统架构

KubeSphere 的多租户系统分**三个**层级，即集群、企业空间和项目。KubeSphere 中的项目等同于 Kubernetes 的[命名空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/)。

>  **架构**

![multi-tenancy-architecture](https://kubesphere.com.cn/images/docs/zh-cn/access-control-and-account-management/multi-tanancy-in-kubesphere/multi-tenancy-architecture.png)

   - [企业空间](https://kubesphere.com.cn/docs/workspace-administration/what-is-workspace/)是最小的租户单元，企业空间提供了跨集群、跨项目（即 Kubernetes 中的命名空间）共享资源的能力。企业空间中的成员可以在授权集群中创建项目，并通过邀请授权的方式参与项目协同。
   - **用户**是 KubeSphere 的帐户实例，可以被设置为平台层面的管理员参与集群的管理，也可以被添加到企业空间中参与项目协同。

> **详情查看官方文档 [KubeSphere 中的多租户](https://kubesphere.com.cn/docs/access-control-and-account-management/multi-tenancy-in-kubesphere/)**

### 4.2. 平台角色

平台角色概览

以 admin 登录 KubeSphere Web 控制台

点击左上角的**平台管理**，然后选择**访问控制**。

![kubesphere-dashboard-access](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-dashboard-access.png)

在左侧导航栏中，选择**平台角色**。四个内置角色的描述信息如下所示。

![kubesphere-access-roles](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-access-roles.png)

内置角色说明。

| 内置角色             | 描述                                                         |
| -------------------- | ------------------------------------------------------------ |
| `workspaces-manager` | 企业空间管理员，管理平台所有企业空间。                       |
| `users-manager`      | 用户管理员，管理平台所有用户。                               |
| `platform-regular`   | 平台普通用户，在被邀请加入企业空间或集群之前没有任何资源操作权限。 |
| `platform-admin`     | 平台管理员，可以管理平台内的所有资源。                       |

**内置角色由 KubeSphere 自动创建，无法编辑或删除。**

**常用角色**。

- **platform-regular：** 新建的用户赋予该角色，由企业空间管理员，邀请进具体的企业空间管理对应的资源
- **platform-admin：** 管理员比较多的时候使用

除了内置角色外，用户也可以根据需求创建自定义的角色，用兴趣的读者可以参考官方文档。

### 4.3. 用户管理

> **创建用户**

以 admin 登录 KubeSphere Web 控制台。

点击左上角的**平台管理**，然后选择**访问控制**。

在左侧导航栏中，选择**用户**页面，点击**创建**。

![kubesphere-access-accounts](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-access-accounts.png)

在弹出的**添加用户**对话框中，填写必要（带有*号）信息，点击**确定**。

- **用户名：** lstack 

  >  正好最近备案了一个域名，用来做图床，这里就用域名当用户名了，后面需要名称的地方也都用这个命名。

- **邮箱：** 建议写实际可用的邮箱。

- **平台角色：** platform-regular，后期由管理员邀请加入具体的企业空间，获取相应的权限。

- **密码：** 密码必须包含数字、大写字母和小写字母，长度为 6 至 64 个字符。

- **描述：** 可选，生产环境建议填写有意义的描述，说明该用户的归属、权限等信息。

- ![kubesphere-access-accounts-add](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-access-accounts-add.png)

新创建的用户将显示在**用户**中的用户列表中。

![kubesphere-access-accounts-list](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-access-accounts-list.png)

接下来我们再创建一个**管理员**用户，日常管理使用，**admin** 用户可以把密码设置的超级复杂然后遗忘吧，省的别人惦记。

创建过程跟上文创建普通用户都是一样的，唯一的区别在于**平台角色**选择 **platform-admin**。

![kubesphere-access-accounts-add2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-access-accounts-add2.png)

---

## 5. 企业空间管理

### 5.1. 企业空间概述

企业空间是用来管理 [项目](https://kubesphere.com.cn/docs/project-administration/)、[DevOps 项目](https://kubesphere.com.cn/docs/devops-user-guide/)、[应用模板](https://kubesphere.com.cn/docs/workspace-administration/upload-helm-based-application/) 和应用仓库的一种逻辑单元。您可以在企业空间中控制资源访问权限，也可以安全地在团队内部分享资源。

最佳的做法是为租户（集群管理员除外）创建新的企业空间。同一名租户可以在多个企业空间中工作，并且多个租户可以通过不同方式访问同一个企业空间。

### 5.2. 企业空间管理

> **创建企业空间**

以 **admin** 登录 KubeSphere Web 控制台。

点击左上角的**平台管理**，然后选择**访问控制**。

在左侧导航栏中，选择**企业空间**页面，点击**创建**。

![kubesphere-access-workspaces](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-access-workspaces.png)

在弹出的**创建企业空间**对话框中，填写必要的**基本信息**（带有*号），点击**创建**。

- **名称**：企业空间名称，本文使用 **lstack**
- **别名**：该企业空间的别名 (从未用过)。
- **管理员**：管理该企业空间的用户，本文选择刚创建的 **lstack** 用户
- **描述**：企业空间的简短介绍（建议填写）。

![kubesphere-access-workspaces-add](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-access-workspaces-add.png)企业空间创建后将显示在企业空间列表中。

![kubesphere-access-workspaces-list](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-access-workspaces-list.png)

点击该企业空间，可以在**概览**页面查看企业空间中的资源状态。后续对企业空间的操作多数也是在该页面完成。

![kubesphere-workspaces-overview](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-workspaces-overview.png)

---

## 6. 项目管理

### 6.1. 项目管理

KubeSphere 中的项目与 Kubernetes 中的 **namespaces** 相同，为资源提供了虚拟隔离。有关更多信息，请参见[命名空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/)。

> **创建项目**

以 **lstack** 登录 KubeSphere Web 控制台。

**第一次登录会提示你重置密码**

登录后默认显示企业空间**概览**页面。

![kubesphere-workspaces-user-overview](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-workspaces-user-overview.png)

在左侧导航栏中，选择**项目**页面，点击**创建**。

![kubesphere-workspaces-projects](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-workspaces-projects.png)

在弹出的**创建项目**对话框中，填写必要的**信息**（带有*号），点击**确定**。

- **名称**：项目名称，本文使用 **lstack**。
- **别名**：该项目的别名 (从未用过)。
- **描述**：项目的简短介绍（建议填写）。

**项目**创建后将显示在项目列表中。

![kubesphere-workspaces-projects-list](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-workspaces-projects-list.png)

点击该**项目**，进入项目管理页面，默认显示**概览**页面。可以在该页面查看项目的资源状态。

**下面重点展示一下项目管理菜单的功能**

> **概览**

![kubesphere-projects-overview](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-overview.png)

> **应用负载**

![kubesphere-projects-applications](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-applications.png)

> **存储**

![kubesphere-projects-volumes](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-volumes.png)

> **配置**

![kubesphere-projects-secrets](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-secrets.png)

> **镜像构建器**

![kubesphere-projects-s2ibuilders](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-s2ibuilders.png)

> **监控告警**

![kubesphere-projects-alerts](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-alerts.png)

> **项目设置**

![kubesphere-projects-base-info](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-base-info.png)

---

## 7. 底层初探

回顾一下我们上面做了哪些主要操作，探究一下 k8s 底层又有了哪些变化。

以下涉及 **kubectl** 的操作使用 **admin** 登录

> **启用可插拔组件-KubeSphere 应用商店**

```shell
# 搜索 namespaces(结果跟官方说的居然不一样，官方说的是在 openpitrix-system)
 
/ # kubectl get namespaces
NAME                              STATUS   AGE
default                           Active   5d2h
kube-node-lease                   Active   5d2h
kube-public                       Active   5d2h
kube-system                       Active   5d2h
kubekey-system                    Active   5d2h
kubesphere-controls-system        Active   5d2h
kubesphere-devops-system          Active   26h
kubesphere-devops-worker          Active   26h
kubesphere-monitoring-federated   Active   5d2h
kubesphere-monitoring-system      Active   5d2h
kubesphere-system                 Active   5d2h
lstack                            Active   84m
 
# 搜索 pods, 只有一个名称类似的，居然还是个 job
 
/ # kubectl get pods -A | grep -i openpitrix
kubesphere-system              openpitrix-import-job-tz6gb                        0/1     Completed   0          26h
 
```

> **启用可插拔组件-KubeSphere DevOps**

```shell
# 搜索 namespaces
/ # kubectl get namespaces
NAME                              STATUS   AGE
default                           Active   5d2h
kube-node-lease                   Active   5d2h
kube-public                       Active   5d2h
kube-system                       Active   5d2h
kubekey-system                    Active   5d2h
kubesphere-controls-system        Active   5d2h
kubesphere-devops-system          Active   26h
kubesphere-devops-worker          Active   26h
kubesphere-monitoring-federated   Active   5d2h
kubesphere-monitoring-system      Active   5d2h
kubesphere-system                 Active   5d2h
lstack                            Active   94m
 
# 搜索 deployments
/ # kubectl get deployments -n kubesphere-devops-system
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
devops-apiserver    1/1     1            1           31h
devops-controller   1/1     1            1           31h
devops-jenkins      1/1     1            1           31h
 
# 搜索 statefulsets
/ # kubectl get statefulsets -n kubesphere-devops-system
NAME          READY   AGE
s2ioperator   1/1     31h
 
# 搜索 services
/ # kubectl get svc -n kubesphere-devops-system
NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
devops-apiserver              ClusterIP   10.233.0.8      <none>        9090/TCP       31h
devops-jenkins                NodePort    10.233.55.62    <none>        80:30180/TCP   31h
devops-jenkins-agent          ClusterIP   10.233.36.52    <none>        50000/TCP      31h
s2ioperator-metrics-service   ClusterIP   10.233.58.210   <none>        8080/TCP       31h
s2ioperator-trigger-service   ClusterIP   10.233.10.208   <none>        8081/TCP       31h
webhook-server-service        ClusterIP   10.233.31.138   <none>        443/TCP        31h
 
# 搜索 pods
/ # kubectl get pods -n kubesphere-devops-system
NAME                                READY   STATUS      RESTARTS   AGE
devops-27490020-gxmjx               0/1     Completed   0          61m
devops-27490050-qckmj               0/1     Completed   0          31m
devops-27490080-vmfzc               0/1     Completed   0          68s
devops-apiserver-7c6774fff5-qtnl2   1/1     Running     0          26h
devops-controller-98975d478-nbcj9   1/1     Running     0          26h
devops-jenkins-5d744f66b9-cfmrj     1/1     Running     0          26h
s2ioperator-0                       1/1     Running     0          26h
/ # kubectl get pods -n kubesphere-devops-worker
No resources found in kubesphere-devops-worker namespace.
```

> **启用可插拔组件-KubeSphere 告警系统**

```shell
# 找了一圈没找到，有待更新
```

> **创建用户**

```shell
/ # kubectl get users
NAME     EMAIL                 STATUS
admin    admin@kubesphere.io   Active
lstack   z@lstack.cn           Active
z        admin@lstack.cn       Active
```

> **创建企业空间**

**KubeSphere 的概念，k8s 中并没有对应资源**

> **创建项目**

**项目等同于 namespaces**

```shell
# 搜索 namespaces
/ # kubectl get namespaces
NAME                              STATUS   AGE
default                           Active   5d2h
kube-node-lease                   Active   5d2h
kube-public                       Active   5d2h
kube-system                       Active   5d2h
kubekey-system                    Active   5d2h
kubesphere-controls-system        Active   5d2h
kubesphere-devops-system          Active   26h
kubesphere-devops-worker          Active   26h
kubesphere-monitoring-federated   Active   5d2h
kubesphere-monitoring-system      Active   5d2h
kubesphere-system                 Active   5d2h
lstack                            Active   94m
```

---

## 8. 总结

本文详细介绍了 KubeSphere 默认安装后，需要执行的一系列操作。主要涉及可插拔组件管理、用户管理、企业空间管理、项目管理，并初步探究了 k8s 底层的变化，更深层的技术细节会在后面实际项目中进行探究。

本文在启用可插拔组件的配置中有一个重要的组件没有启用，那就是 **KubeSphere 日志系统**，该插件依赖于 **Elasticsearch**，KubeSphere 本身内置 **Elasticsearch** 组件的安装，但是不建议生产环境使用，为了更贴近于生产环境，我们下一期单独介绍 **Elasticsearch** 的安装配置以及与 KubeSphere 的对接配置。

> **参考文档**

- [启用可插拔组件](https://kubesphere.com.cn/docs/pluggable-components/overview/)
- [KubeSphere 中的多租户](https://kubesphere.com.cn/docs/access-control-and-account-management/multi-tenancy-in-kubesphere/)

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

> About Me

- 昵称：老 Z
- 坐标：山东济南
- 职业：运维架构师 / 高级运维工程师 =**运维**
- 微信：zdevops
- 关注的领域：云计算 / 云原生技术运维，自动化运维
- 技能标签：OpenStack、Ansible、K8S、Python、Go、CNCF
