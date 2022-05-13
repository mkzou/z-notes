# 基于KubeSphere玩转k8s-MySQL安装手记

**大家好，我是老Z！**

> 本系列文档是我在云原生技术领域的学习和运维实践的手记，**用输出倒逼输入**是一种高效的学习方法，能够快速积累经验和提高技术，只有把学到的知识写出来并能够让其他人理解，才能说明真正掌握了这项知识。
>
> 如果你喜欢本文，请分享给你的小伙伴！

**本系列文档内容涵盖(但不限于)以下技术领域：**

> - **KubeSphere**
>
> - **Kubernetes**
>
> - **Ansible**
>
> - **自动化运维**
>
> - **CNCF技术栈**



## 1. 本文简介

  本文实现了MySQL数据库在基于KubeSphere部署的Kubernetes集群上的安装部署，部署方式采用了图形化和手写YAML文件两种形式。部署过程涉及的所有YAML文件都会使用git进行版本管理，并存放在git仓库中。因此，本文还会涉及GitOps的基础操作。

  本文部署的MySQL选择了比较保守的5.7系列，其他版本可能会有不同。本文的操作仅适用于小规模数据量且对可靠性和性能要求不高的数据库使用场景，例如开发测试环境、例如我生产环境的Nacos服务。生产环境或是重要的数据库个人不建议将数据放到K8S上，优先采用云服务商提供的RDS，其次自己利用虚拟机搭建MySQL主从或是Galera Cluster，且一定做好备份方案。

  **数据库的可靠性、可用性是运维的重中之重，不容忽视，切记！！！**

> 本文知识量

- 阅读时长：40分
- 行：2658
- 单词：15475
- 字符：120711
- 图片：80张

> **本文知识点**

- 定级：**入门级**
- 单节点MySQL在Kubernetes上的安装配置
- KubeSphere图形化部署工作负载
- GitOps入门
- Git常用操作
- 配置代码如何实现在GitHub和Gitee保持同步
- MySQL性能测试基础
- 运维思想、思路

> **演示服务器配置**

|      主机名      |      IP      | CPU  | 内存 | 系统盘 | 数据盘 |               用途               |
| :--------------: | :----------: | :--: | :--: | :----: | :----: | :------------------------------: |
|  zdeops-master   | 192.168.9.9  |  2   |  4   |   40   |  200   |       Ansible运维控制节点        |
| ks-k8s-master-0  | 192.168.9.91 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-1  | 192.168.9.92 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-2  | 192.168.9.93 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| glusterfs-node-0 | 192.168.9.95 |  4   |  8   |   40   |  200   |     GlusterFS/Elasticsearch      |
| glusterfs-node-1 | 192.168.9.96 |  4   |  8   |   40   |  200   |     GlusterFS/Elasticsearch      |
| glusterfs-node-2 | 192.168.9.97 |  4   |  8   |   40   |  200   |     GlusterFS/Elasticsearch      |

## 2. MySQL安装之旅

### 01. 寻找参考文档

我个人查找参考文档习惯的的寻找路径

- **官方网站**-精准定位
  - 官网有时没有相关文档、或是文档不够详细
  - 英文文档、阅读困难
- **搜索关键字**-大海捞针
  - CSDN
  - 博客园
  - 某个人博客
  - 问答网站
  - 其他

1. 打开[MySQL官方网站](https://dev.mysql.com/doc/)
   - 选择**MySQL5.7**版本的Reference Manual。
   - ![image-20220509141222130](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220509141222130.png)
   - 在**Installing MySQL on Linux**章节中，搜寻一番，发现在**[Deploying MySQL on Linux with Docker](https://dev.mysql.com/doc/refman/5.7/en/linux-installation-docker.html)**小节下两篇具有参考价值的文档，先去看看。
   - ![image-20220509142153500](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220509142153500.png)
   - 浏览完以后你会发现，只是学会了利用Docker Image安装MySQL的基本方法，细节不上图了。
   - 虽然官方没有提到如何在K8S上部署MySQL，但是我已经有Docker和K8S的基础知识了，先不去进行搜索吃别人的了，自己尝试在K8S上部署一个单节点的MySQL。

### 02. 尝试部署单节点MySQL

1. 先梳理一下思路，部署一个MySQL我们需要准备哪些资源。

   - 在DockerHub获取MySQL镜像。
   - 查看MySQL镜像说明，确定安装初始化参数。
   - MySQL属于有状态服务，所以我们需要定义StatefulSet类型的资源。
   - 编写StatefulSet类型的MySQL资源定义文件-YAML。

2. 查看官方镜像说明，确定初始化参数。

   > **如果之前有Docker部署MySQL的经验，这一步就很简单了，直接把参数配置搬过来就行** 

   - 打开https://hub.docker.com，搜索mysql。

   - 搜索结果中会有很多的mysql，我重点关注了两个镜像。

     - Docker官方维护的[mysql仓库](https://hub.docker.com/_/mysql)
     - Oracle的MySQL团队维护的[mysql仓库](https://hub.docker.com/r/mysql/mysql-server)

   - 本次实验我使用了Docker官方维护的仓库，进入MySQL仓库页面。

   - 大概浏览一遍，确认了几个必须要配置的地方（确定过程需要经验和技术积累）。

     > 镜像：**mysql:5.7.38**
     >
     > root密码：**MYSQL_ROOT_PASSWORD**
     >
     > 数据持久化存储目录：**/var/lib/mysql**

3. 利用KubeSphere部署MySQL（V1版）。

   - 确定了初始化的参数，接下来就开始部署MySQL。

   - 按K8S常规套路编写资源定义YAML文件？NO！我现在是小白，手写配置文件太高端了，还不适合我。

   - 我们这里投机取巧一下，利用KubeSphere的图形化操作一波，这样可以保证部署的一次成功率（还有一个隐藏的好处，先卖个关子）。

   - 使用**企业空间管理员**权限的账户，登陆KubeSphere控制台。

     > **这一步没有使用admin用户，采用多租户户形式，模拟真实的生产环境**

   - ![kubesphere-lstack-login](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-lstack-login.png)

   - ![kubesphere-workspace-lstack](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-workspace-lstack.png)

   - 点击**项目**，点击**lstack**项目，进入项目的管理页面（如无特殊说明，后面的很多界面操作都是在该页面完成）。

   - ![kubesphere-projects-lstack](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack.png)

   - ![kubesphere-projects-lstack-overview](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-overview.png)

   - **应用负载**->**工作负载**->**有状态副本集**，点击**创建**。

   - ![kubesphere-projects-lstack-statefulsets](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets.png)

   - 弹出**创建有状态副本集**页面，**基本信息**页，**名称**输入**mysql**。

   - ![kubesphere-projects-lstack-statefulsets-mysql-0](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-0.png)

   - **容器组设置**页。

     - **容器组副本数量**：1
     - 点击**添加容器**，镜像搜索栏输入**mysql:5.7.38**
     - **容器名称**： lstack-mysql
     - **CPU（Core）资源**：预留0.5，限制2
     - **内存（Mi）**：预留500i，限制4000
     - **端口设置**：协议TCP，名称tcp-mysql，容器端口：3306，服务端口3306
     - **环境变量**：
       - 引用配置字典或保密字典
       - 创建保密字典，键（MYSQL_ROOT_PASSWORD），值（P@88w0rd）
     - **同步主机时区**：勾选上
     - 其他未说明的配置采用默认值

   - ![kubesphere-projects-lstack-statefulsets-mysql-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-1.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-2.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-3](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-3.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-4](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-4.png)

   - **创建保密字典**：在**环境变量**选项中，点击**创建保密字典**，按后续图示操作。

   - ![kubesphere-projects-lstack-statefulsets-mysql-5](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-5.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-6](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-6.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-7](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-7.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-8](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-8.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-9](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-9.png)

   - 按以上信息配置完成后，点击**对号**按钮。

   - ![kubesphere-projects-lstack-statefulsets-mysql-10](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-10.png)

   - **容器组设置**完成后，点击**下一步**，进入**存储卷设置**。

   - ![kubesphere-projects-lstack-statefulsets-mysql-11](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-11.png)

   - **存储卷设置**->**存储卷模板**->**添加存储卷模板**。

     - 存储卷名称：data
       - 这个地方不要多写，系统会自动添加StatefulSet的名称作为名称后缀，生成类似data-mysql-0命名形式的存储卷
     - 存储类型：glusterfs
     - 访问模式：ReadWriteOnce
     - 存储卷容量：5Gi
     - 挂载路径：读写 /var/lib/mysql

   - ![kubesphere-projects-lstack-statefulsets-mysql-12](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-12.png)

   - 按以上信息配置完成后，点击**对号**按钮。

   - ![kubesphere-projects-lstack-statefulsets-mysql-13](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-13.png)

   - **存储卷设置**完成后，点击**下一步**，进入**高级设置**，保持默认值，点击**创建**按钮。

   - ![kubesphere-projects-lstack-statefulsets-mysql-14](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-14.png)

   - **创建**成功后，自动返回工作负载页面。第一次创建会去DockerHub下载镜像，所以初始显示状态为**更新中**。

   - ![kubesphere-projects-lstack-statefulsets-mysql-15](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-15.png)

   - 镜像下载完成并且容器配置正确时，状态变成**运行中**。

   - ![kubesphere-projects-lstack-statefulsets-mysql-16](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-16.png)

   - 点击**mysql**，进入有状态副本集详细页面。

   - ![kubesphere-projects-lstack-statefulsets-mysql-17](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-17.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-18](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-18.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-19](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-19.png)

   - **监控**，可以看到初始启动时的资源使用情况，后续可以根据监控数据调整我们的资源的配置。

   - ![kubesphere-projects-lstack-statefulsets-mysql-20](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-20.png)

   - **环境变量**，可以看到我们新增加的Secret字典生效了，并且密码是隐藏显示的。

   - ![kubesphere-projects-lstack-statefulsets-mysql-21](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-21.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-22](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-22.png)

   - 再来看看**容器组**的详细信息，在**资源状态**页面，点击容器组**mysql-0**

   - ![kubesphere-projects-lstack-statefulsets-mysql-23](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-23.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-24](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-24.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-25](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-25.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-26](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-26.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-27](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-27.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-28](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-28.png)

   - 再来看看StatefulSet对应的服务（Service），**应用负载**->**服务**。

   - 可以看到自动创建了一个StatefulSet MySQL对应的有状态服务（Headless），**mysql-2v7f(mysql)**。

   - ![kubesphere-projects-lstack-statefulsets-mysql-29](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-29.png)

   - 点击**mysql-2v7f(mysql)**，可以查看服务详情。

   - ![kubesphere-projects-lstack-statefulsets-mysql-30](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-30.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-31](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-31.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-32](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-32.png)

   - 最后验证一下，我们的MySQL服务是否正常（这里只看服务本身，先不测试外部连接）。

   - **应用负载**->**工作负载**->**有状态副本集**->**mysql**->**容器组**->**mysql-0**->**终端**

   - ![kubesphere-projects-lstack-statefulsets-mysql-33](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-33.png)

   - ![image-20220510112902273](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-34.png)

   - 至此，MySQL在K8S的基本安装就完成了，K8S集群内的其他应用可以通过svc的地址访问MySQL服务(svc地址就是**mysql-2v7f.lstack**)，此时名字看着还是很不友好，我们先不用它。

### 03. MySQL配置进阶

上面完成了MySQL的基本安装配置。但是，实际使用中我们通常还有如下需求，需要我们对MySQL进行配置。

1. 开启外部访问。

   - 开启外部访问方便管理员操作MySQL数据库，也可以满足K8S集群之外的服务访问MySQL数据库的需求。

   - 在KubeSphere中开启服务的外部访问需要先设置项目网关。

   - 用项目管理员用户登陆控制台。

   - **工作台**->**项目**->点击**具体的项目**->**项目设置**->**网关设置**，点击**开启网关**。

   - ![kubesphere-projects-lstack-statefulsets-mysql-35](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-35.png)

   - 目前访问模式有NodePort和LoadBalancer，但是LoadBalancer只支持公有云提供商云上的负载均衡器，所以我们只能选择NodePort，点击确定。

     > **NodePort模式里会创建一个采用了nginx-ingress的kubesphere-router的容器组，细节我们会在以后的专文探讨**

   - ![kubesphere-projects-lstack-statefulsets-mysql-36](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-36.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-37](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-37.png)

   - 网关设置的细节不在本文深入讨论，后续会有专文探讨。现在，做到这一步就OK了。

   - 接下来创建一个MySQL服务用于对外提供服务。

   - **应用负载**->-**服务**>**创建**->选择**自定义服务**->**指定工作负载**。

     > 这里有一个**外部服务**的选项，那个是基于Ingress使用域名访问的，不是目前我们想要的方式。

   - ![kubesphere-projects-lstack-statefulsets-mysql-38](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-38.png)

   - **指定工作负载创建服务-基本信息**。

     - **名称：**mysql-external

   - ![kubesphere-projects-lstack-statefulsets-mysql-39](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-39.png)

   - **指定工作负载创建服务-服务设置**，点击**指定工作负载**，选择**有状态副本集**->**mysql**，点击**确定**。

   - ![kubesphere-projects-lstack-statefulsets-mysql-40](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-40.png)

   - **指定工作负载创建服务-服务设置**，**端口**配置。

     - **协议：** TCP
     - **名称：** tcp-mysql-external
     - **容器端口：** 3306
     - **服务端口：** 3306

   - ![kubesphere-projects-lstack-statefulsets-mysql-41](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-41.png)

   - **指定工作负载创建服务-高级设置**。
     
     - **外部访问：** **访问模式**选择NodePort
     
   - ![kubesphere-projects-lstack-statefulsets-mysql-42](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-42.png)
     
   - 完成所有设置后，点击**创建**，创建成功会自动返回服务列表，在服务列表中可以看到我们新创建的服务**mysql-external**及自动分配的外部访问端口号。
     
   - ![kubesphere-projects-lstack-statefulsets-mysql-43](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-43.png)
     
   - 先用telnet命令测试一下，MySQL服务的连通性，能看到下面的结果就说明MySQL已经可以在K8S集群外部访问了。
     
   - ```shell
     [root@ks-k8s-master-0 ~]# telnet 192.168.9.91 32529
     Trying 192.168.9.91...
     Connected to 192.168.9.91.
     Escape character is '^]'.
     EHost '10.233.117.0' is not allowed to connect to this MySQL serverConnection closed by foreign host.
     
     # 细节！上面的EHost地址是192.168.9.91这个节点在K8S集群内部分配的IP
     [root@ks-k8s-master-0 ~]# ip add | grep 117 -B 2 -A 1
     7: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN group default qlen 1000
         link/ipip 0.0.0.0 brd 0.0.0.0
         inet 10.233.117.0/32 scope global tunl0
            valid_lft forever preferred_lft forever
     [root@ks-k8s-master-0 ~]# ip add | grep 91
         inet 192.168.9.91/24 brd 192.168.9.255 scope global noprefixroute ens160
         link/ether c6:d3:91:95:f1:0f brd ff:ff:ff:ff:ff:ff

2. 自定义MySQL配置文件。

   - 默认安装的MySQL使用的my.cnf配置文件，适配的使用场景有限，所以自定义mysql配置文件是必然要做的一项配置。

   - 这里我随机找了一份配置文件，仅仅是为了实现自定义配置的功能，请根据自己的使用场景使用合适的自定义配置文件。

   - 使用自定义配置前，我们先需要了解目前mysql容器的配置文件结构。

   - 使用KubeSphere提供的**终端**工具，进入mysql容器内部，执行下面的命令，分析执行结果（终端登录方式参考前文截图）。

   - ```shell
     # bash
     root@mysql-0:/# ls /etc/mysql/ -l
     total 8
     drwxr-xr-x 2 root root   62 Apr 28 06:20 conf.d
     lrwxrwxrwx 1 root root   24 Apr 28 06:20 my.cnf -> /etc/alternatives/my.cnf
     -rw-r--r-- 1 root root  839 Aug  3  2016 my.cnf.fallback
     -rw-r--r-- 1 root root 1200 Mar 22 01:44 mysql.cnf
     drwxr-xr-x 2 root root   24 Apr 28 06:20 mysql.conf.d
     
     root@mysql-0:/# ls /etc/mysql/conf.d/ -l
     total 12
     -rw-r--r-- 1 root root 43 Apr 28 06:20 docker.cnf
     -rw-r--r-- 1 root root  8 Aug  3  2016 mysql.cnf
     -rw-r--r-- 1 root root 55 Aug  3  2016 mysqldump.cnf
     
     root@mysql-0:/# ls /etc/mysql/mysql.conf.d/ -l
     total 4
     -rw-r--r-- 1 root root 1589 Apr 28 06:20 mysqld.cnf
     
     root@mysql-0:/# ls -l /etc/alternatives/my.cnf
     lrwxrwxrwx 1 root root 20 Apr 28 06:20 /etc/alternatives/my.cnf -> /etc/mysql/mysql.cnf
     
     root@mysql-0:/# cat /etc/mysql/mysql.cnf
     # Copyright (c) 2016, 2021, Oracle and/or its affiliates.
     #
     # This program is free software; you can redistribute it and/or modify
     # it under the terms of the GNU General Public License, version 2.0,
     # as published by the Free Software Foundation.
     #
     # This program is also distributed with certain software (including
     # but not limited to OpenSSL) that is licensed under separate terms,
     # as designated in a particular file or component or in included license
     # documentation.  The authors of MySQL hereby grant you an additional
     # permission to link the program and your derivative works with the
     # separately licensed software that they have included with MySQL.
     #
     # This program is distributed in the hope that it will be useful,
     # but WITHOUT ANY WARRANTY; without even the implied warranty of
     # MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
     # GNU General Public License, version 2.0, for more details.
     #
     # You should have received a copy of the GNU General Public License
     # along with this program; if not, write to the Free Software
     # Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301 USA
     
     !includedir /etc/mysql/conf.d/
     !includedir /etc/mysql/mysql.conf.d/
     
     ## 挑两个配置文件看看
     root@mysql-0:~# cat /etc/mysql/conf.d/docker.cnf
     [mysqld]
     skip-host-cache
     skip-name-resolve
     
     root@mysql-0:~# cat /etc/mysql/mysql.conf.d/mysqld.cnf
     # Copyright (c) 2014, 2021, Oracle and/or its affiliates.
     #
     # This program is free software; you can redistribute it and/or modify
     # it under the terms of the GNU General Public License, version 2.0,
     # as published by the Free Software Foundation.
     #
     # This program is also distributed with certain software (including
     # but not limited to OpenSSL) that is licensed under separate terms,
     # as designated in a particular file or component or in included license
     # documentation.  The authors of MySQL hereby grant you an additional
     # permission to link the program and your derivative works with the
     # separately licensed software that they have included with MySQL.
     #
     # This program is distributed in the hope that it will be useful,
     # but WITHOUT ANY WARRANTY; without even the implied warranty of
     # MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
     # GNU General Public License, version 2.0, for more details.
     #
     # You should have received a copy of the GNU General Public License
     # along with this program; if not, write to the Free Software
     # Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301 USA
     
     #
     # The MySQL  Server configuration file.
     #
     # For explanations see
     # http://dev.mysql.com/doc/mysql/en/server-system-variables.html
     
     [mysqld]
     pid-file        = /var/run/mysqld/mysqld.pid
     socket          = /var/run/mysqld/mysqld.sock
     datadir         = /var/lib/mysql
     #log-error      = /var/log/mysql/error.log
     # By default we only accept connections from localhost
     #bind-address   = 127.0.0.1
     # Disabling symbolic-links is recommended to prevent assorted security risks
     symbolic-links=0
     ```
     
   - 分析上面的输出我们得到以下结论。
   
      - 根配置文件：**/etc/mysql/mysql.cnf**
      - 自定义的配置文件可以存放在**/etc/mysql/conf.d/**或**/etc/mysql/mysql.conf.d/**目录下
     
   - 通过上面的结论，发现有两种方式实现自定义配置文件。
   
     - 直接替换**/etc/mysql/mysql.cnf**
       - 适用于个性化配置较多较复杂的场景，比如50+的配置项。
     - 将自定义的配置放在**/etc/mysql/conf.d/**或**/etc/mysql/mysql.conf.d/**目录下，根据官方配置使用情况，建议选择**/etc/mysql/conf.d/**
       - 适用于自定义配置较少的场景，比如只是为了开启个别功能，或是个别默认参数不符合使用需求。
   
   - 本文采用第二种方式，采用一个独立的custom.cnf文件配置以下参数。
   
   - ```ini
     [mysqld]
     #performance setttings
     lock_wait_timeout = 3600
     open_files_limit    = 65535
     back_log = 1024
     max_connections = 512
     max_connect_errors = 1000000
     table_open_cache = 1024
     table_definition_cache = 1024
     thread_stack = 512K
     sort_buffer_size = 4M
     join_buffer_size = 4M
     read_buffer_size = 8M
     read_rnd_buffer_size = 4M
     bulk_insert_buffer_size = 64M
     thread_cache_size = 768
     interactive_timeout = 600
     wait_timeout = 600
     tmp_table_size = 32M
     max_heap_table_size = 32M
     
     ```
     
   - 实现思路。
   
     - k8s中我们可以通过配置ConfigMap的方式将文件挂载给容器
     - 将自定义的mysql配置文件，定义为一个ConfigMap
     - 将ConfigMap挂载给mysql的容器
   
   - 创建ConfigMap配置文件，**配置**->**配置字典**，点击**创建**。
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-44](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-44.png)
   
   - **创建配置字典-基本信息**。
   
     - 名称：mysql-cnf
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-45](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-45.png)
   
   - **创建配置字典-数据设置**。
   
     - 点击**添加数据**
     - **键：** custom.cnf
     - **值：** 粘贴上面的配置参数
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-46](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-46.png)
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-47](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-47.png)
   
   - 填写完键值信息后，点击**对号**确定，最后点击**创建**，创建完成后会返回**配置字典**页面。
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-48](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-48.png)
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-49](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-49.png)
   
   - 接下来将自定义配置文件，挂载到mysql容器。
   
   - **应用负载**->**工作负载**->**有状态副本集**->点击**mysql**->进入详细配置页面->**更多操作**-点击**编辑设置**。
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-50](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-50.png)
   
   - **编辑设置**->**存储卷**->**挂载配置字典或保密字典**。
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-51](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-51.png)
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-52](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-52.png)
   
   - **存储卷**。
   
      - **选择配置字典：**mysql-cnf
      - **只读**
      - **挂载路径：**/etc/mysql/conf.d/custom.cnf
      - 指定子路径：custom.cnf
         - **此处必须这么写，否则会覆盖掉指定目录下的所有已存在文件**
         - **底层就是subPath**
         - **具体操作看下图图示，注意细节**
      - **选择特定键：** 
         - 键：custom.cnf
         - 路径：custom.cnf
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-53](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-53.png)
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-54](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-54.png)
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-55](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-55.png)
   
   - 输入完成后，点击**对号**。 
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-56](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-56.png)
   
   - 再次点击**对号**，点击**确定**，mysql容器会自动开始重建。
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-57](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-57.png)
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-58](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-58.png)
   
   - 重建成功后我们验证一下配置文件是否成功挂载。
   
   - 先看一下容器组的配置，发现新增了一个存储卷**volume-xxxx**。
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-59](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-59.png)
   
   - **终端**->进入容器内部查看。
   
   - 查看配置文件挂载和文件内容。
   
   - ```shell
      # bash
      root@mysql-0:/# ls /etc/mysql/conf.d/
      custom.cnf  docker.cnf  mysql.cnf  mysqldump.cnf
      
      root@mysql-0:/# ls -l /etc/mysql/conf.d/
      total 16
      -rw-r--r-- 1 root root 463 May 11 11:07 custom.cnf
      -rw-r--r-- 1 root root  43 Apr 28 06:20 docker.cnf
      -rw-r--r-- 1 root root   8 Aug  3  2016 mysql.cnf
      -rw-r--r-- 1 root root  55 Aug  3  2016 mysqldump.cnf
      
      root@mysql-0:/# cat /etc/mysql/conf.d/custom.cnf
      [mysqld]
      #performance setttings
      lock_wait_timeout = 3600
      open_files_limit    = 65535
      back_log = 1024
      max_connections = 512
      max_connect_errors = 1000000
      table_open_cache = 1024
      table_definition_cache = 1024
      thread_stack = 512K
      sort_buffer_size = 4M
      join_buffer_size = 4M
      read_buffer_size = 8M
      read_rnd_buffer_size = 4M
      bulk_insert_buffer_size = 64M
      thread_cache_size = 768
      interactive_timeout = 600
      wait_timeout = 600
      tmp_table_size = 32M
      max_heap_table_size = 32M
      
      ```
      
   - 查看配置参数是否生效。
   
   - ```shell
      root@mysql-0:/# mysql -u root -p
      Enter password:
      Welcome to the MySQL monitor.  Commands end with ; or \g.
      Your MySQL connection id is 2
      Server version: 5.7.38 MySQL Community Server (GPL)
      
      Copyright (c) 2000, 2022, Oracle and/or its affiliates.
      
      Oracle is a registered trademark of Oracle Corporation and/or its
      affiliates. Other names may be trademarks of their respective
      owners.
      
      Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
      
      mysql> SHOW GLOBAL VARIABLES LIKE 'max_connect%';
      +--------------------+---------+
      | Variable_name      | Value   |
      +--------------------+---------+
      | max_connect_errors | 1000000 |
      | max_connections    | 512     |
      +--------------------+---------+
      2 rows in set (0.02 sec)
      
      mysql>
      ```
   
   - 执行结果跟我们的配置一致，说明配置成功。
   
2. 导入数据库数据。

   - 将数据库文件（SQL），挂载到容器的指定目录下**/docker-entrypoint-initdb.d**，容器创建时会自动导入（**非必要不推荐**）。
   - 用数据库管理工具远程管理数据库（**推荐**）。
### 04. 原生K8S部署MySQL实现GitOps

上面我们完成了通过KubeSphere部署单实例MySQL，那么原生的K8S又该如何操作？GitOps又是什么、又该如何实现？

1. 什么是GitOps(网文摘抄)。
   - GitOps 是一套使用 Git 来管理基础架构和应用配置的实践，而 Git 指的是一个开源版控制系统。
   - GitOps 在运行过程中以 Git 为声明性基础架构和应用的单一事实来源。
   - GitOps 使用 Git 拉取请求来自动管理基础架构的置备和部署。
   - Git 存储库包含系统的全部状态，因此系统状态的修改痕迹既可查看也可审计。
   - GitOps 经常被用作 Kubernetes和云原生应用开发的运维模式，并且可以实现对 Kubernetes 的持续部署。
   - GitOps是一种持续交付的方式。它的核心思想是将应用系统的声明性基础架构和应用程序存放在Git版本库中。
   
2. 准备K8S上的MySQL资源配置清单-思路梳理。

   - 我们知道玩K8S的必备技能就是要手写资源配置清单，一般使用YAML格式的文件来创建我们预期的资源配置。
   - 此时我们也要手写MySQL的资源配置清单？我很慌，参数我记不全啊。
   - NO！NO！NO！投机取巧的时刻到了，前面卖的关子在这揭开了。
   - 前面我们已经通过KubeSphere的图形界面创建了MySQL的资源配置，而且KubeSphere一个很棒的功能就是可以直接在线编辑资源的YAML文件。
   - 我们可以在创建资源的时候，直接编辑YAML文件创建资源。也可以通过编辑YAML的方式修改已有的资源。
   - 当然啊，你不用图形界面，直接在K8S底层用命令行的方式去获取YAML格式的输出，再编辑，也是可以的。
   - 梳理一下MySQL涉及的资源配置清单包含的资源。
     - **StatefulSet(有状态副本集)**
     - **Service(服务)**
       - 集群内部（Headless）
       - 集群外部（自定义服务）
     - **ConfigMap**
     - **Secret**
   - 接下来我们就分别获取这些资源配置清单。

3. 准备K8S上的MySQL资源配置清单。

   > **01-ConfigMap**

   - **配置**->**配置字典**，找到**mysql-cnf**，点击右侧的**三个竖点**，点击**编辑YAML**。

   - ![kubesphere-projects-lstack-statefulsets-mysql-60](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-60.png)

   - 打开**编辑YAML**页面，可以直接复制所有内容，也可以点击右上角的下载图标，下载文件（也可以利用上传图标上传文件）。

   - ![kubesphere-projects-lstack-statefulsets-mysql-61](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-61.png)

   - 获取的现网配置不能完全的拿来就用，需要修改，把系统自动添加的一些元数据信息清理掉。

   - 现网的**mysql-cfm.yaml**。

   - ```yaml
     kind: ConfigMap
     apiVersion: v1
     metadata:
       name: mysql-cnf
       namespace: lstack
       annotations:
         kubesphere.io/creator: lstack
     data:
       custom.cnf: |-
         [mysqld]
         #performance setttings
         lock_wait_timeout = 3600
         open_files_limit    = 65535
         back_log = 1024
         max_connections = 512
         max_connect_errors = 1000000
         table_open_cache = 1024
         table_definition_cache = 1024
         thread_stack = 512K
         sort_buffer_size = 4M
         join_buffer_size = 4M
         read_buffer_size = 8M
         read_rnd_buffer_size = 4M
         bulk_insert_buffer_size = 64M
         thread_cache_size = 768
         interactive_timeout = 600
         wait_timeout = 600
         tmp_table_size = 32M
         max_heap_table_size = 32M
         
     ```

   - 修改后的**mysql-cfm.yaml**。

   - ```yaml
     kind: ConfigMap
     apiVersion: v1
     metadata:
       name: mysql-cnf
       namespace: lstack
     data:
       custom.cnf: |-
         [mysqld]
         #performance setttings
         lock_wait_timeout = 3600
         open_files_limit    = 65535
         back_log = 1024
         max_connections = 512
         max_connect_errors = 1000000
         table_open_cache = 1024
         table_definition_cache = 1024
         thread_stack = 512K
         sort_buffer_size = 4M
         join_buffer_size = 4M
         read_buffer_size = 8M
         read_rnd_buffer_size = 4M
         bulk_insert_buffer_size = 64M
         thread_cache_size = 768
         interactive_timeout = 600
         wait_timeout = 600
         tmp_table_size = 32M
         max_heap_table_size = 32M
     
     ```

   > **02-Secret**

   - **配置**->**保密字典**，找到**mysql-secret**，点击右侧的**三个竖点**，点击**编辑YAML**。

   - 现网的**mysql-secret.yaml**。

   - ```yaml
     kind: Secret
     apiVersion: v1
     metadata:
       name: mysql-secret
       namespace: lstack
       annotations:
         kubesphere.io/creator: lstack
     data:
       MYSQL_ROOT_PASSWORD: UEA4OHcwcmQ=
     type: Opaque
     
     ```

   - 修改后的**mysql-secret.yaml**。

   - ```yaml
     kind: Secret
     apiVersion: v1
     metadata:
       name: mysql-secret
       namespace: lstack
     data:
       MYSQL_ROOT_PASSWORD: UEA4OHcwcmQ=
     type: Opaque
     
     ```

   - 这里要说一句，Secret里的值是用base64方式加密的，所以这里的**MYSQL_ROOT_PASSWORD**，要用实际的密码用base64的方式加密。

   - base64解密。

   - ```shell
     [root@ks-k8s-master-0 ~]# echo "UEA4OHcwcmQ=" | base64 -d
     P@88w0rd
     ```

   - base加密。

   - ```shell
     [root@ks-k8s-master-0 ~]# echo -n "P@88w0rd" | base64
     UEA4OHcwcmQ=
     ```

   > **03-StatefulSet**

   - **应用负载**->**工作负载**->**有状态副本集**，找到**mysql**，点击右侧的**三个竖点**，点击**编辑YAML**。

   - 现网的**mysql-sts.yaml**。

   - ```yaml
     kind: StatefulSet
     apiVersion: apps/v1
     metadata:
       name: mysql
       namespace: lstack
       labels:
         app: mysql
       annotations:
         kubesphere.io/creator: lstack
     spec:
       replicas: 1
       selector:
         matchLabels:
           app: mysql
       template:
         metadata:
           creationTimestamp: null
           labels:
             app: mysql
           annotations:
             logging.kubesphere.io/logsidecar-config: '{}'
         spec:
           volumes:
             - name: host-time
               hostPath:
                 path: /etc/localtime
                 type: ''
             - name: volume-rca2zx
               configMap:
                 name: mysql-cnf
                 items:
                   - key: custom.cnf
                     path: custom.cnf
                 defaultMode: 420
           containers:
             - name: lstack-mysql
               image: 'mysql:5.7.38'
               ports:
                 - name: tcp-mysql
                   containerPort: 3306
                   protocol: TCP
               env:
                 - name: MYSQL_ROOT_PASSWORD
                   valueFrom:
                     secretKeyRef:
                       name: mysql-secret
                       key: MYSQL_ROOT_PASSWORD
               resources:
                 limits:
                   cpu: '2'
                   memory: 4000Mi
                 requests:
                   cpu: 500m
                   memory: 500Mi
               volumeMounts:
                 - name: host-time
                   mountPath: /etc/localtime
                 - name: data
                   mountPath: /var/lib/mysql
                 - name: volume-rca2zx
                   readOnly: true
                   mountPath: /etc/mysql/conf.d/custom.cnf
                   subPath: custom.cnf
               terminationMessagePath: /dev/termination-log
               terminationMessagePolicy: File
               imagePullPolicy: IfNotPresent
           restartPolicy: Always
           terminationGracePeriodSeconds: 30
           dnsPolicy: ClusterFirst
           serviceAccountName: default
           serviceAccount: default
           securityContext: {}
           schedulerName: default-scheduler
       volumeClaimTemplates:
         - kind: PersistentVolumeClaim
           apiVersion: v1
           metadata:
             name: data
             namespace: lstack
             creationTimestamp: null
           spec:
             accessModes:
               - ReadWriteOnce
             resources:
               requests:
                 storage: 5Gi
             storageClassName: glusterfs
             volumeMode: Filesystem
           status:
             phase: Pending
       serviceName: mysql-1dpr
       podManagementPolicy: OrderedReady
       updateStrategy:
         type: RollingUpdate
         rollingUpdate:
           partition: 0
       revisionHistoryLimit: 10
     
     ```

   - 修改后的**mysql-sts.yaml**。

   - ```yaml
     kind: StatefulSet
     apiVersion: apps/v1
     metadata:
       name: mysql
       namespace: lstack
       labels:
         app: mysql
     spec:
       replicas: 1
       selector:
         matchLabels:
           app: mysql
       template:
         metadata:
           labels:
             app: mysql
         spec:
           volumes:
             - name: host-time
               hostPath:
                 path: /etc/localtime
                 type: ''
             - name: volume-cnf
               configMap:
                 name: mysql-cnf
                 items:
                   - key: custom.cnf
                     path: custom.cnf
                 defaultMode: 420
           containers:
             - name: lstack-mysql
               image: 'mysql:5.7.38'
               ports:
                 - name: tcp-mysql
                   containerPort: 3306
                   protocol: TCP
               env:
                 - name: MYSQL_ROOT_PASSWORD
                   valueFrom:
                     secretKeyRef:
                       name: mysql-secret
                       key: MYSQL_ROOT_PASSWORD
               resources:
                 limits:
                   cpu: '2'
                   memory: 4000Mi
                 requests:
                   cpu: 500m
                   memory: 500Mi
               volumeMounts:
                 - name: host-time
                   mountPath: /etc/localtime
                 - name: data
                   mountPath: /var/lib/mysql
                 - name: volume-cnf
                   mountPath: /etc/mysql/conf.d/custom.cnf
                   subPath: custom.cnf
       volumeClaimTemplates:
         - metadata:
             name: data
             namespace: lstack
           spec:
             accessModes:
               - ReadWriteOnce
             resources:
               requests:
                 storage: 5Gi
             storageClassName: glusterfs
       serviceName: mysql-headless
     
     ```

   > **04-Service**

   - 先看看Headless服务。

   - **应用负载**->**服务**->，找到**mysql-xxxx(mysql)**，点击右侧的**三个竖点**，点击**编辑YAML**。

   - 现网的**mysql-headless.yaml**。

   - ```yaml
     kind: Service
     apiVersion: v1
     metadata:
       name: mysql-1dpr
       namespace: lstack
       labels:
         app: mysql
       annotations:
         kubesphere.io/alias-name: mysql
         kubesphere.io/creator: lstack
         kubesphere.io/serviceType: statefulservice
     spec:
       ports:
         - name: tcp-mysql
           protocol: TCP
           port: 3306
           targetPort: 3306
       selector:
         app: mysql
       clusterIP: None
       clusterIPs:
         - None
       type: ClusterIP
       sessionAffinity: None
       ipFamilies:
         - IPv4
       ipFamilyPolicy: SingleStack
     
     ```

   - 修改后的**mysql-headless.yaml**。

   - ```yaml
     kind: Service
     apiVersion: v1
     metadata:
       name: mysql-headless
       namespace: lstack
       labels:
         app: mysql
     spec:
       ports:
         - name: tcp-mysql
           protocol: TCP
           port: 3306
           targetPort: 3306
       selector:
         app: mysql
       clusterIP: None
       type: ClusterIP
     
     ```

   - 再看看自定义的**mysql-external**服务。

   - **应用负载**->**服务**->，找到**mysql-external**，点击右侧的**三个竖点**，点击**编辑YAML**。

   - 现网的**mysql-external.yaml**。

   - ```yaml
     kind: Service
     apiVersion: v1
     metadata:
       name: mysql-external
       namespace: lstack
       labels:
         app: mysql-external
       annotations:
         kubesphere.io/creator: lstack
     spec:
       ports:
         - name: tcp-mysql-external
           protocol: TCP
           port: 3306
           targetPort: 3306
           nodePort: 32529
       selector:
         app: mysql
       clusterIP: 10.233.36.71
       clusterIPs:
         - 10.233.36.71
       type: NodePort
       sessionAffinity: None
       externalTrafficPolicy: Cluster
       ipFamilies:
         - IPv4
       ipFamilyPolicy: SingleStack
     
     ```

   - 这里有一点要说明**nodePort**这个参数，如果K8S集群可控，建议规划一套服务端口使用规范，每个需要**nodePort**的服务都指定固定的端口，这样有利于运维的标准化。

   - 修改后的**mysql-external.yaml**(注意**nodePort**参数没有指定)。

   - ```yaml
     kind: Service
     apiVersion: v1
     metadata:
       name: mysql-external
       namespace: lstack
       labels:
         app: mysql-external
     spec:
       ports:
         - name: tcp-mysql-external
           protocol: TCP
           port: 3306
           targetPort: 3306
       selector:
         app: mysql
       type: NodePort
     
     ```

4. 将MySQL资源配置清单提交到Git仓库。

   - 通过上面的操作，我们获取了MySQL的资源配置清单。

     > (本人强迫症，喜欢分类存放，所以我用了4个文件，当然你也可以放到一个配置文件里)
     >
     > **mysql-headless.yaml**跟**mysql-sts.yaml**合并在一个文件

     - **mysql-external.yaml**
     - **mysql-sts.yaml**
     - **mysql-secret.yaml**
     - **mysql-cfm.yaml**

   - 接下来将资源配置清单提交到Git仓库。

     - 选择GitHub作为主仓库，Gitee作为同步仓库(人工)。
     - 本系列文档所有k8s的资源配置清单文件使用了一个公共仓库，生产环境建议每种服务创建一个配置仓库，有利于更精细化的版本控制。
     - 本文为了演示主备仓库的使用，所有选择了Github和Gitee两种Git服务，实际使用中为了更好的使用体验建议选择Gitee。

   - 在GitHub新建一个仓库，仓库名称**[k8s-yaml](https://github.com/devops/k8s-yaml)**，添加一个README文件初始化仓库，点击**Create repository**，确认创建。

   - ![kubesphere-projects-lstack-statefulsets-mysql-62](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-62.png)

   - ![kubesphere-projects-lstack-statefulsets-mysql-63](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-63.png)

   - 将代码仓库Clone回本地。

   - ```shell
     github % git clone git@github.com:devops/k8s-yaml.git
     Cloning into 'k8s-yaml'...
     Enter passphrase for key '/Users/z/.ssh/id_rsa': 
     remote: Enumerating objects: 3, done.
     remote: Counting objects: 100% (3/3), done.
     remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
     Receiving objects: 100% (3/3), done.
     
     github % ls k8s-yaml 
     README.md
     ```

   - 新创建一个文件夹，用自己喜欢的文本编辑器(推荐vscode)编辑MySQL的资源配置清单，并将文件放入新创建的文件夹。

     > 为了以后的扩展性，这里创建了一个**single**命名的二级目录，存放单实例的资源配置清单文件。

   - ```shell
     github % mkdir -p k8s-yaml/mysql/single
     github % ls -l k8s-yaml/mysql/single
     total 32
     -rw-r--r--  1 z  staff   646  5 11 19:23 mysql-cfm.yaml
     -rw-r--r--  1 z  staff   266  5 11 19:31 mysql-external.yaml
     -rw-r--r--  1 z  staff   134  5 11 19:23 mysql-secret.yaml
     -rw-r--r--  1 z  staff  1911  5 11 19:31 mysql-sts.yaml
     ```

   - 将编辑好的资源配置文件清单，提交到GitHub。

   - ```shell
     github % cd k8s-yaml
     k8s-yaml[main*] % git status
     On branch main
     Your branch is up to date with 'origin/main'.
     
     Untracked files:
       (use "git add <file>..." to include in what will be committed)
     	mysql/
     
     nothing added to commit but untracked files present (use "git add" to track)
     
     k8s-yaml[main*] % git add .
     k8s-yaml[main*] % git commit -am '添加MySQL single资源配置清单'
     [main 1d00559] 添加MySQL single资源配置清单
      4 files changed, 138 insertions(+)
      create mode 100644 mysql/single/mysql-cfm.yaml
      create mode 100644 mysql/single/mysql-external.yaml
      create mode 100644 mysql/single/mysql-secret.yaml
      create mode 100644 mysql/single/mysql-sts.yaml
     k8s-yaml[main] % git status
     On branch main
     Your branch is ahead of 'origin/main' by 1 commit.
       (use "git push" to publish your local commits)
     
     nothing to commit, working tree clean
     k8s-yaml[main] % git push
     Enter passphrase for key '/Users/z/.ssh/id_rsa':
     Enumerating objects: 9, done.
     Counting objects: 100% (9/9), done.
     Delta compression using up to 8 threads
     Compressing objects: 100% (7/7), done.
     Writing objects: 100% (8/8), 1.59 KiB | 1.59 MiB/s, done.
     Total 8 (delta 1), reused 0 (delta 0), pack-reused 0
     remote: Resolving deltas: 100% (1/1), done.
     To github.com:devops/k8s-yaml.git
        e31f780..1d00559  main -> main
     ```
     
   - 在GitHub上查看，确认代码是否提交。

   - ![kubesphere-projects-lstack-statefulsets-mysql-64](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-64.png)

   - 接下来将资源配置清单同步到Gitee备份仓库。

     - 本文采用了手工推送同步的方式(个人习惯)
     - Gitee也支持自动同步GitHub的仓库(更便捷)

   - 在Gitee新建一个仓库，仓库名称**[k8s-yaml](https://gitee.com/zdevops/k8s-yaml)**，类型默认**私有**，点击**创建**。

     > 创建完成后可去**仓库设置**中修改为开源

   - ![kubesphere-projects-lstack-statefulsets-mysql-65](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-65.png)

   - 创建完成后，因为我们创建的时候，没选择初始化仓库的配置，所以，默认会显示一个帮助页面，告诉你该如何提交代码到仓库。

   - ![kubesphere-projects-lstack-statefulsets-mysql-66](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-66.png)

   - 因为，我们已经有了代码仓库，所以我们选择**已有仓库**的配置方法，将已有代码提交到Gitee。

   - 根据帮助提示操作，要注意**origin**我们要换成**gitee**。

   - ```shell
     k8s-yaml[main] % git remote add gitee https://gitee.com/zdevops/k8s-yaml.git
     
     k8s-yaml[main] % git push -u gitee
     Enumerating objects: 11, done.
     Counting objects: 100% (11/11), done.
     Delta compression using up to 8 threads
     Compressing objects: 100% (8/8), done.
     Writing objects: 100% (11/11), 2.14 KiB | 2.14 MiB/s, done.
     Total 11 (delta 1), reused 3 (delta 0), pack-reused 0
     remote: Powered by GITEE.COM [GNK-6.3]
     remote: Create a pull request for 'main' on Gitee by visiting:
     remote:     https://gitee.com/zdevops/k8s-yaml/pull/new/zdevops:main...zdevops:master
     To https://gitee.com/zdevops/k8s-yaml.git
      * [new branch]      main -> main
     Branch 'main' set up to track remote branch 'main' from 'gitee'.
     ```

   - 在Gitee上查看，确认代码是否提交。

   - ![kubesphere-projects-lstack-statefulsets-mysql-67](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-67.png)

   - 修改Gitee仓库为开源(可选)。

   - Gitee仓库->**管理**->**仓库设置**->**基本信息**，最后面**是否开源**，选择**开源**，**仓库公开须知**，三个都勾选，点击**保存**。

   - ![kubesphere-projects-lstack-statefulsets-mysql-70](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-70.png)

   - 修改后，你的代码仓库就是开源，所有人可见的了。

5. GitOps初体验-在K8S集群上部署MySQL。

   - MySQL资源配置清单已经存放到了Git在线仓库，接下来开启我们的GitOps体验之旅。

   - 登录k8s的master节点，执行后面的操作任务。

     >  **生产环境建议打造独立的运维管理节点进行整个集群的管理,可以参考《基于KubeSphere玩转k8s-运维管理节点打造手记》**

   - 安装Git。

   - ```shell
     [root@ks-k8s-master-0 ~]# yum install git -y
     Loaded plugins: fastestmirror
     Loading mirror speeds from cached hostfile
      * base: mirrors.aliyun.com
      * centos-gluster9: mirrors.aliyun.com
      * extras: mirrors.aliyun.com
      * updates: mirrors.aliyun.com
     Resolving Dependencies
     --> Running transaction check
     ---> Package git.x86_64 0:1.8.3.1-23.el7_8 will be installed
     --> Processing Dependency: perl-Git = 1.8.3.1-23.el7_8 for package: git-1.8.3.1-23.el7_8.x86_64
     --> Processing Dependency: rsync for package: git-1.8.3.1-23.el7_8.x86_64
     --> Processing Dependency: perl(Term::ReadKey) for package: git-1.8.3.1-23.el7_8.x86_64
     --> Processing Dependency: perl(Git) for package: git-1.8.3.1-23.el7_8.x86_64
     --> Processing Dependency: perl(Error) for package: git-1.8.3.1-23.el7_8.x86_64
     --> Running transaction check
     ---> Package perl-Error.noarch 1:0.17020-2.el7 will be installed
     ---> Package perl-Git.noarch 0:1.8.3.1-23.el7_8 will be installed
     ---> Package perl-TermReadKey.x86_64 0:2.30-20.el7 will be installed
     ---> Package rsync.x86_64 0:3.1.2-10.el7 will be installed
     --> Finished Dependency Resolution
     
     Dependencies Resolved
     
     ==============================================================================================================================
      Package                            Arch                     Version                             Repository              Size
     ==============================================================================================================================
     Installing:
      git                                x86_64                   1.8.3.1-23.el7_8                    base                   4.4 M
     Installing for dependencies:
      perl-Error                         noarch                   1:0.17020-2.el7                     base                    32 k
      perl-Git                           noarch                   1.8.3.1-23.el7_8                    base                    56 k
      perl-TermReadKey                   x86_64                   2.30-20.el7                         base                    31 k
      rsync                              x86_64                   3.1.2-10.el7                        base                   404 k
     
     Transaction Summary
     ==============================================================================================================================
     Install  1 Package (+4 Dependent packages)
     
     Total download size: 4.9 M
     Installed size: 23 M
     Downloading packages:
     (1/5): perl-Error-0.17020-2.el7.noarch.rpm                                                             |  32 kB  00:00:00     
     (2/5): perl-Git-1.8.3.1-23.el7_8.noarch.rpm                                                            |  56 kB  00:00:00     
     (3/5): perl-TermReadKey-2.30-20.el7.x86_64.rpm                                                         |  31 kB  00:00:00     
     (4/5): git-1.8.3.1-23.el7_8.x86_64.rpm                                                                 | 4.4 MB  00:00:00     
     (5/5): rsync-3.1.2-10.el7.x86_64.rpm                                                                   | 404 kB  00:00:00     
     ------------------------------------------------------------------------------------------------------------------------------
     Total                                                                                          13 MB/s | 4.9 MB  00:00:00     
     Running transaction check
     Running transaction test
     Transaction test succeeded
     Running transaction
       Installing : 1:perl-Error-0.17020-2.el7.noarch                                                                          1/5 
       Installing : rsync-3.1.2-10.el7.x86_64                                                                                  2/5 
       Installing : perl-TermReadKey-2.30-20.el7.x86_64                                                                        3/5 
       Installing : perl-Git-1.8.3.1-23.el7_8.noarch                                                                           4/5 
       Installing : git-1.8.3.1-23.el7_8.x86_64                                                                                5/5 
       Verifying  : git-1.8.3.1-23.el7_8.x86_64                                                                                1/5 
       Verifying  : 1:perl-Error-0.17020-2.el7.noarch                                                                          2/5 
       Verifying  : perl-TermReadKey-2.30-20.el7.x86_64                                                                        3/5 
       Verifying  : perl-Git-1.8.3.1-23.el7_8.noarch                                                                           4/5 
       Verifying  : rsync-3.1.2-10.el7.x86_64                                                                                  5/5 
     
     Installed:
       git.x86_64 0:1.8.3.1-23.el7_8                                                                                               
     
     Dependency Installed:
       perl-Error.noarch 1:0.17020-2.el7       perl-Git.noarch 0:1.8.3.1-23.el7_8       perl-TermReadKey.x86_64 0:2.30-20.el7      
       rsync.x86_64 0:3.1.2-10.el7            
     
     Complete!
     ```

   - 创建devops目录，我选择/opt目录作为devops的根目录。

   - ```shell
     [root@ks-k8s-master-0 ~]# mkdir /opt/devops
     [root@ks-k8s-master-0 ~]# cd /opt/devops/
     [root@ks-k8s-master-0 devops]# 
     ```

   - 从Gitee下载**k8s-yaml**仓库的代码。

   - ```shell
     [root@ks-k8s-master-0 devops]# git clone https://gitee.com/zdevops/k8s-yaml.git
     Cloning into 'k8s-yaml'...
     remote: Enumerating objects: 11, done.
     remote: Counting objects: 100% (11/11), done.
     remote: Compressing objects: 100% (8/8), done.
     remote: Total 11 (delta 1), reused 0 (delta 0), pack-reused 0
     Unpacking objects: 100% (11/11), done.
     
     [root@ks-k8s-master-0 devops]# ls k8s-yaml/
     mysql  README.md
     ```

   - 由于是同一个测试环境，先清理掉现有的MySQL服务。

   - ```shell
     [root@ks-k8s-master-0 devops]# kubectl get secrets -n lstack 
     NAME                  TYPE                                  DATA   AGE
     default-token-x2gzv   kubernetes.io/service-account-token   3      31d
     mysql-secret          Opaque                                1      2d20h
     
     [root@ks-k8s-master-0 devops]# kubectl get configmaps -n lstack 
     NAME               DATA   AGE
     kube-root-ca.crt   1      31d
     mysql-cnf          1      47h
     
     [root@ks-k8s-master-0 devops]# kubectl get service -n lstack 
     NAME                                                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
     glusterfs-dynamic-afe88cf4-86b1-4215-833a-534c5f779a22   ClusterIP   10.233.13.188   <none>        1/TCP            2d
     mysql-1dpr                                               ClusterIP   None            <none>        3306/TCP         2d
     mysql-external                                           NodePort    10.233.36.71    <none>        3306:32529/TCP   47h
     
     [root@ks-k8s-master-0 devops]# kubectl get statefulsets -n lstack 
     NAME    READY   AGE
     mysql   1/1     2d
     
     # 清理
     [root@ks-k8s-master-0 devops]# kubectl delete statefulsets mysql -n lstack
     statefulset.apps "mysql" deleted
     [root@ks-k8s-master-0 devops]# kubectl delete service mysql-external -n lstack
     service "mysql-external" deleted
     [root@ks-k8s-master-0 devops]# kubectl delete service mysql-1dpr -n lstack
     service "mysql-1dpr" deleted
     [root@ks-k8s-master-0 devops]# kubectl delete secrets mysql-secret -n lstack
     secret "mysql-secret" deleted
     [root@ks-k8s-master-0 devops]# kubectl delete configmaps mysql-cnf -n lstack
     configmap "mysql-cnf" deleted
     
     ```

   - 利用资源配置清单一键部署MySQL。

   - ```shell
     [root@ks-k8s-master-0 devops]# cd /opt/devops/k8s-yaml/
     [root@ks-k8s-master-0 k8s-yaml]# ls
     mysql  README.md
     
     [root@ks-k8s-master-0 k8s-yaml]# kubectl apply -f mysql/single/
     configmap/mysql-cnf created
     service/mysql-external created
     secret/mysql-secret created
     service/mysql-headless created
     ```

   - 验证结果，发现**StatefulSet**没有创建，分析问题。

   - ```shell
     [root@ks-k8s-master-0 k8s-yaml]# kubectl get statefulsets -n lstack
     No resources found in lstack namespace.
     
     # 一开始我以为我遗漏了配置文件，ls看一眼，发现文件都在
     [root@ks-k8s-master-0 k8s-yaml]# ls
     mysql  README.md
     [root@ks-k8s-master-0 k8s-yaml]# cd mysql/
     [root@ks-k8s-master-0 mysql]# ls
     single
     [root@ks-k8s-master-0 mysql]# cd single/
     [root@ks-k8s-master-0 single]# ls
     mysql-cfm.yaml  mysql-external.yaml  mysql-secret.yaml  mysql-sts.yaml
     
     # 确认一下文件内容，发现文件也有内容
     [root@ks-k8s-master-0 single]# vi mysql-sts.yaml
     
     # 再次执行，发现了端倪，为啥只有service/mysql-headless 的资源配置，没有statefulset
     [root@ks-k8s-master-0 single]# kubectl apply -f mysql-sts.yaml 
     service/mysql-headless unchanged
     
     # 再次确认，发现编辑文件的时候遗漏了一点，当一个配置文件有多种资源定义时，不同资源的配置直接需要用"---"分隔。修改配置文件再次执行，发现执行成功。
     [root@ks-k8s-master-0 single]# vi mysql-sts.yaml
     [root@ks-k8s-master-0 single]# cd ..
     
     [root@ks-k8s-master-0 mysql]# kubectl apply -f single/
     configmap/mysql-cnf unchanged
     service/mysql-external unchanged
     secret/mysql-secret unchanged
     statefulset.apps/mysql created
     service/mysql-headless unchanged
     
     [root@ks-k8s-master-0 mysql]# kubectl get statefulsets -n lstack -o wide
     NAME    READY   AGE   CONTAINERS     IMAGES
     mysql   1/1     31s   lstack-mysql   mysql:5.7.38
     [root@ks-k8s-master-0 mysql]# kubectl get pods -n lstack -o wide
     NAME      READY   STATUS    RESTARTS   AGE   IP              NODE              NOMINATED NODE   READINESS GATES
     mysql-0   1/1     Running   0          35s   10.233.116.59   ks-k8s-master-2   <none>           <none>
     ```

   - 回到我们的KubeSphere的管理控制台，发现mysql的工作负载也能在界面中显示，这也验证了在原生k8s上的操作也会直接反应到KubeSphere的管理控制台。

   - ![kubesphere-projects-lstack-statefulsets-mysql-71](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-71.png)

   

6. 二次体验GitOps。

   - 正好借着上面出现的问题，二次体验一下GitOps。我们直接在部署服务器上修改了**mysql-sts.yaml**，且修改后的结果验证成功。

     > 为了演示GitOps的更多场景，直接在部署服务器上修改，然后提交到在线代码仓库。
     >
     > 实际工作中我都是在自己的办公电脑上修改，提交到在线代码仓库，然后部署服务器拉取更新代码。

   - 修改后的**mysql-sts.yaml**，由于篇幅问题这里只演示关键部分，StatefulSet的完整配置见Gitee仓库或是前文。

   - ```yaml
     ---
     kind: StatefulSet
     apiVersion: apps/v1
     metadata:
       name: mysql
       namespace: lstack
       labels:
         app: mysql
     ...
     
     ---
     kind: Service
     apiVersion: v1
     metadata:
       name: mysql-headless
       namespace: lstack
       labels:
         app: mysql
     spec:
       ports:
         - name: tcp-mysql
           protocol: TCP
           port: 3306
           targetPort: 3306
       selector:
         app: mysql
       clusterIP: None
       type: ClusterIP
     ```

   - 提交修改后的代码到代码仓库。

   - ```shell
     # 修改后查看git仓库的变化
     [root@ks-k8s-master-0 single]# git status
     # On branch main
     # Changes not staged for commit:
     #   (use "git add <file>..." to update what will be committed)
     #   (use "git checkout -- <file>..." to discard changes in working directory)
     #
     #       modified:   mysql-sts.yaml
     #
     no changes added to commit (use "git add" and/or "git commit -a")
     
     [root@ks-k8s-master-0 single]# git diff 
     diff --git a/mysql/single/mysql-sts.yaml b/mysql/single/mysql-sts.yaml
     index f775920..1eded9c 100644
     --- a/mysql/single/mysql-sts.yaml
     +++ b/mysql/single/mysql-sts.yaml
     @@ -1,3 +1,4 @@
     +---
      kind: StatefulSet
      apiVersion: apps/v1
      metadata:
     @@ -68,6 +69,7 @@ spec:
              storageClassName: glusterfs
        serviceName: mysql-headless
      
     +---
      kind: Service
      apiVersion: v1
      metadata:
     
     # 本地提交代码变更
     [root@ks-k8s-master-0 single]# git commit -am '修复mysql statefulset配置不生效问题'
     
     *** Please tell me who you are.
     
     Run
     
       git config --global user.email "you@example.com"
       git config --global user.name "Your Name"
     
     to set your account's default identity.
     Omit --global to set the identity only in this repository.
     
     fatal: unable to auto-detect email address (got 'root@ks-k8s-master-0.(none)')
     
     # 发现报错了，因为我们这是新环境，还没有配置git的user.email和user.name。按提示配置。 
     [root@ks-k8s-master-0 single]# git config --global user.email devops@163.com
     [root@ks-k8s-master-0 single]# git config --global user.name "devops"
     
     # 再次提交
     [root@ks-k8s-master-0 single]# git commit -am '修复mysql statefulset配置不生效问题'
     [main f5ba03b] 修复mysql statefulset配置不生效问题
      1 file changed, 2 insertions(+)
      
     # push到在线代码仓库，有一个warning可以忽略，也可以按提示执行
     [root@ks-k8s-master-0 single]# git push
     warning: push.default is unset; its implicit value is changing in
     Git 2.0 from 'matching' to 'simple'. To squelch this message
     and maintain the current behavior after the default changes, use:
     
       git config --global push.default matching
     
     To squelch this message and adopt the new behavior now, use:
     
       git config --global push.default simple
     
     See 'git help config' and search for 'push.default' for further information.
     (the 'simple' mode was introduced in Git 1.7.11. Use the similar mode
     'current' instead of 'simple' if you sometimes use older versions of Git)
     
     Username for 'https://gitee.com': zdevops
     Password for 'https://zdevops@gitee.com': 
     Counting objects: 9, done.
     Delta compression using up to 8 threads.
     Compressing objects: 100% (4/4), done.
     Writing objects: 100% (5/5), 456 bytes | 0 bytes/s, done.
     Total 5 (delta 2), reused 0 (delta 0)
     remote: Powered by GITEE.COM [GNK-6.3]
     To https://gitee.com/zdevops/k8s-yaml.git
        1d00559..f5ba03b  main -> main
     ```
     
   - 查看Gitee在线代码仓库是否有变更。
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-72](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-72.png)
   
   - 在个人的办公电脑上，同步更新后的代码。
   
   - ```shell
     # 更新代码
     k8s-yaml[main] % git pull
     remote: Enumerating objects: 9, done.
     remote: Counting objects: 100% (9/9), done.
     remote: Compressing objects: 100% (4/4), done.
     remote: Total 5 (delta 2), reused 0 (delta 0), pack-reused 0
     Unpacking objects: 100% (5/5), 436 bytes | 87.00 KiB/s, done.
     From https://gitee.com/zdevops/k8s-yaml
        1d00559..f5ba03b  main       -> gitee/main
     Updating 1d00559..f5ba03b
     Fast-forward
      mysql/single/mysql-sts.yaml | 2 ++
      1 file changed, 2 insertions(+)
      
     # 同步更新后的代码到Github
     k8s-yaml[main] % git push -u origin
     Enter passphrase for key '/Users/z/.ssh/id_rsa':
     Enumerating objects: 9, done.
     Counting objects: 100% (9/9), done.
     Delta compression using up to 8 threads
     Compressing objects: 100% (4/4), done.
     Writing objects: 100% (5/5), 456 bytes | 456.00 KiB/s, done.
     Total 5 (delta 2), reused 0 (delta 0), pack-reused 0
     remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
     To github.com:devops/k8s-yaml.git
        1d00559..f5ba03b  main -> main
     Branch 'main' set up to track remote branch 'main' from 'origin'.
     ```
   
   - 查看GitHub在线代码仓库是否有变更。
   
   - ![kubesphere-projects-lstack-statefulsets-mysql-73](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-73.png)
   
7. 再次体验GitOps

   - 模拟一个业务场景，再次体验一下GitOps。

     - MySQL上线运行后，由于业务量上涨，初始配置参数中的**max_connections**太小了，需要增大。
     - 配置参数调整完成后，更新线上配置，并重启服务(生产环境数据库不要轻易重启，这种需求可以用临时修改解决)。
     - 这里只是模拟一个简单的例子，带大家体验GitOps，实际使用中所有的配置文件都建议使用Git进行版本控制。

   - 编辑本地Git仓库MySQL资源配置清单中的**mysql-cfm.yaml**文件，修改**max_connections**，从512变成1024。

   - 提交修改到Git在线仓库。

   - ```shell
     # 查看修改状态
     k8s-yaml[main*] % git status
     On branch main
     Your branch is up to date with 'origin/main'.
     
     Changes not staged for commit:
       (use "git add <file>..." to update what will be committed)
       (use "git restore <file>..." to discard changes in working directory)
     	modified:   mysql/single/mysql-cfm.yaml
     
     no changes added to commit (use "git add" and/or "git commit -a")
     
     k8s-yaml[main*] % git diff
     diff --git a/mysql/single/mysql-cfm.yaml b/mysql/single/mysql-cfm.yaml
     index e24d96d..50d1778 100644
     --- a/mysql/single/mysql-cfm.yaml
     +++ b/mysql/single/mysql-cfm.yaml
     @@ -10,7 +10,7 @@ data:
          lock_wait_timeout = 3600
          open_files_limit    = 65535
          back_log = 1024
     -    max_connections = 512
     +    max_connections = 1024
          max_connect_errors = 1000000
          table_open_cache = 1024
          table_definition_cache = 1024
     
     # 提交本地修改
     k8s-yaml[main*] % git commit -am '修改mysql-cnf中max_connections的值'
     [main 180f97a] 修改mysql-cnf中max_connections的值
      1 file changed, 1 insertion(+), 1 deletion(-)
      
     # 提交到Github
     k8s-yaml[main] % git push
     Enter passphrase for key '/Users/z/.ssh/id_rsa':
     Enumerating objects: 9, done.
     Counting objects: 100% (9/9), done.
     Delta compression using up to 8 threads
     Compressing objects: 100% (4/4), done.
     Writing objects: 100% (5/5), 447 bytes | 447.00 KiB/s, done.
     Total 5 (delta 2), reused 0 (delta 0), pack-reused 0
     remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
     To github.com:devops/k8s-yaml.git
        f5ba03b..180f97a  main -> main
     
     # 同步到Gitee
     k8s-yaml[main] % git push -u gitee
     Enumerating objects: 9, done.
     Counting objects: 100% (9/9), done.
     Delta compression using up to 8 threads
     Compressing objects: 100% (4/4), done.
     Writing objects: 100% (5/5), 447 bytes | 447.00 KiB/s, done.
     Total 5 (delta 2), reused 0 (delta 0), pack-reused 0
     remote: Powered by GITEE.COM [GNK-6.3]
     To https://gitee.com/zdevops/k8s-yaml.git
        f5ba03b..180f97a  main -> main
     Branch 'main' set up to track remote branch 'main' from 'gitee'.
     ```

   - 登录运维管理节点，更新Git代码，并重新运行。

   - ```shell
     [root@ks-k8s-master-0 k8s-yaml]# git pull
     remote: Enumerating objects: 9, done.
     remote: Counting objects: 100% (9/9), done.
     remote: Compressing objects: 100% (4/4), done.
     remote: Total 5 (delta 2), reused 0 (delta 0), pack-reused 0
     Unpacking objects: 100% (5/5), done.
     From https://gitee.com/zdevops/k8s-yaml
        f5ba03b..180f97a  main       -> origin/main
     Updating f5ba03b..180f97a
     Fast-forward
      mysql/single/mysql-cfm.yaml | 2 +-
      1 file changed, 1 insertion(+), 1 deletion(-)
     
     [root@ks-k8s-master-0 k8s-yaml]# kubectl apply -f mysql/single/
     configmap/mysql-cnf configured
     service/mysql-external unchanged
     secret/mysql-secret unchanged
     statefulset.apps/mysql configured
     service/mysql-headless unchanged
     
     # 查看ConfigMap的变化
     [root@ks-k8s-master-0 k8s-yaml]# kubectl get configmaps mysql-cnf -n lstack -o yaml
     apiVersion: v1
     data:
       custom.cnf: |-
         [mysqld]
         #performance setttings
         lock_wait_timeout = 3600
         open_files_limit    = 65535
         back_log = 1024
         max_connections = 1024
         max_connect_errors = 1000000
         table_open_cache = 1024
         table_definition_cache = 1024
         thread_stack = 512K
         sort_buffer_size = 4M
         join_buffer_size = 4M
         read_buffer_size = 8M
         read_rnd_buffer_size = 4M
         bulk_insert_buffer_size = 64M
         thread_cache_size = 768
         interactive_timeout = 600
         wait_timeout = 600
         tmp_table_size = 32M
         max_heap_table_size = 32M
     kind: ConfigMap
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"v1","data":{"custom.cnf":"[mysqld]\n#performance setttings\nlock_wait_timeout = 3600\nopen_files_limit    = 65535\nback_log = 1024\nmax_connections = 1024\nmax_connect_errors = 1000000\ntable_open_cache = 1024\ntable_definition_cache = 1024\nthread_stack = 512K\nsort_buffer_size = 4M\njoin_buffer_size = 4M\nread_buffer_size = 8M\nread_rnd_buffer_size = 4M\nbulk_insert_buffer_size = 64M\nthread_cache_size = 768\ninteractive_timeout = 600\nwait_timeout = 600\ntmp_table_size = 32M\nmax_heap_table_size = 32M"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"mysql-cnf","namespace":"lstack"}}
       creationTimestamp: "2022-05-12T07:20:07Z"
       name: mysql-cnf
       namespace: lstack
       resourceVersion: "8928391"
       uid: 1b7322cf-f11e-445d-a2ba-b42a90ade469
     
     # 重启mysql pod使配置生效
     [root@ks-k8s-master-0 k8s-yaml]# kubectl delete -f mysql/single/mysql-sts.yaml 
     statefulset.apps "mysql" deleted
     service "mysql-headless" deleted
     
     [root@ks-k8s-master-0 k8s-yaml]# kubectl apply -f mysql/single/mysql-sts.yaml 
     statefulset.apps/mysql created
     service/mysql-headless created
     
     # 查看mysql容器内部配置是否更新
     [root@ks-k8s-master-0 k8s-yaml]# kubectl exec  mysql-0  -n lstack -- cat /etc/mysql/conf.d/custom.cnf
     [mysqld]
     #performance setttings
     lock_wait_timeout = 3600
     open_files_limit    = 65535
     back_log = 1024
     max_connections = 1024
     max_connect_errors = 1000000
     table_open_cache = 1024
     table_definition_cache = 1024
     thread_stack = 512K
     sort_buffer_size = 4M
     join_buffer_size = 4M
     read_buffer_size = 8M
     read_rnd_buffer_size = 4M
     bulk_insert_buffer_size = 64M
     thread_cache_size = 768
     interactive_timeout = 600
     wait_timeout = 600
     tmp_table_size = 32M
     ```

   - 切记！上面的例子只是让大家体验GitOps，生产环境不要轻易重启数据库服务器，除非你知道自己在干什么。

   - 现在经过验证，我们的MySQL的配置可用且比较稳定，我们把这个好的状态记录下来，避免以后修改变更弄坏了，再找不回原来正确的配置。

   - 在我们的个人电脑上给当前的Git代码打个Tag，记录当前的状态(也可以通过在线仓库的管理界面操作)。

   - ```shell
     # 打tag -a tag名字 -m tag描述
     k8s-yaml[main] % git tag -a v0.1  -m 'mysql version v0.1'
     
     # 查看现有tag
     k8s-yaml[main] % git tag -l
     v0.1
     
     # 查看tag详细信息
     k8s-yaml[main] % git show v0.1
     tag v0.1
     Tagger: devops <devops@163.com>
     Date:   Thu May 12 18:15:34 2022 +0800
     
     mysql version v0.1
     
     commit 180f97ac96da504a0b46eb4871ef423f64fde093 (HEAD -> main, tag: v0.1, origin/main, origin/HEAD, gitee/main)
     Author: devops <devops@163.com>
     Date:   Thu May 12 17:48:18 2022 +0800
     
         修改mysql-cnf中max_connections的值
     
     diff --git a/mysql/single/mysql-cfm.yaml b/mysql/single/mysql-cfm.yaml
     index e24d96d..50d1778 100644
     --- a/mysql/single/mysql-cfm.yaml
     +++ b/mysql/single/mysql-cfm.yaml
     @@ -10,7 +10,7 @@ data:
          lock_wait_timeout = 3600
          open_files_limit    = 65535
          back_log = 1024
     -    max_connections = 512
     +    max_connections = 1024
          max_connect_errors = 1000000
          table_open_cache = 1024
          table_definition_cache = 1024
          
     # 将tag推送到远程服务器
     k8s-yaml[main] % git push -u origin --tags
     Enter passphrase for key '/Users/z/.ssh/id_rsa':
     Enumerating objects: 1, done.
     Counting objects: 100% (1/1), done.
     Writing objects: 100% (1/1), 157 bytes | 157.00 KiB/s, done.
     Total 1 (delta 0), reused 0 (delta 0), pack-reused 0
     To github.com:devops/k8s-yaml.git
      * [new tag]         v0.1 -> v0.1
     
     k8s-yaml[main] % git push -u gitee --tags
     Enumerating objects: 1, done.
     Counting objects: 100% (1/1), done.
     Writing objects: 100% (1/1), 157 bytes | 157.00 KiB/s, done.
     Total 1 (delta 0), reused 0 (delta 0), pack-reused 0
     remote: Powered by GITEE.COM [GNK-6.3]
     To https://gitee.com/zdevops/k8s-yaml.git
      * [new tag]         v0.1 -> v0.1
      
     # 线上服务器验证（图略）
     ```

   - 运维管理服务器更新代码，并切换到指定tag(**注意！使用Git一定要养成每次操作前git pull**这种习惯)。

   - ```shell
     ## 更新代码
     [root@ks-k8s-master-0 k8s-yaml]# git pull
     remote: Enumerating objects: 1, done.
     remote: Counting objects: 100% (1/1), done.
     remote: Total 1 (delta 0), reused 0 (delta 0), pack-reused 0
     Unpacking objects: 100% (1/1), done.
     From https://gitee.com/zdevops/k8s-yaml
      * [new tag]         v0.1       -> v0.1
     Already up-to-date.
     
     [root@ks-k8s-master-0 k8s-yaml]# git status
     # On branch main
     nothing to commit, working directory clean
     
     ## 切换到v0.1
     [root@ks-k8s-master-0 k8s-yaml]# git checkout -b v0.1
     Switched to a new branch 'v0.1'
     
     [root@ks-k8s-master-0 k8s-yaml]# git status
     # On branch v0.1
     nothing to commit, working directory clean
     ```

   - 通过上面的几波操作，我们可以看到，我们所有的配置变更都采用了Git管理，完整的记录了配置的全生命周期管理，通过给仓库打分支或是tag，可以方便我们切换到任意已记录状态。

### 05. 高可用部署MySQL（预留占坑）

暂时没有高可用部署的需求，因此不涉及高可用模式的MySQL的部署，但是有一些思考留着占坑。

1. 目前的做法。
   - 不给自己找麻烦，有高可用需求直接买云服务商的RDS。
   - 实在需要自己搭建，在K8S集群之外部署主从。

2. 以后可能的方向。
   - K8S上的MySQL主从部署
   - Operator
   - Helm

### 06. 遗留问题

此部分内容也是运维MySQL必备的技能，有些内容我也没有经验无法分享，有些内容会在<<基于KubeSphere的Kubernetes生产实践之路>>系列文档中介绍。

- MySQL数据库备份
- MySQL高可用部署
- MySQL安全加固
- MySQL调优

## 3. MySQL性能（基准）测试

运维一定要做到对自己的运维环境**心中有数**，MySQL上线前一定要进行性能(基准测试)，有助于了解我们的数据库服务器能达到的理想状态。本次介绍的只是皮毛，只是告诉大家一些基本入门的知识，更细节、更深入的内容请参考其他更专业的文档。

### 01. 性能（基准）测试工具安装

1. 工具选型(**sysbench**)。

   - 云厂商展示自家数据库产品性能都用这个工具。
   - 据说很多DBA也喜欢用。

2. 安装工具

   - 安装工具。

   - ```shell
     # 导入软件源
     [root@ks-k8s-master-0 ~]# curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
     
     Detected operating system as centos/7.
     Checking for curl...
     Detected curl...
     Downloading repository file: https://packagecloud.io/install/repositories/akopytov/sysbench/config_file.repo?os=centos&dist=7&source=script
     done.
     Installing pygpgme to verify GPG signatures...
     Loaded plugins: fastestmirror
     Loading mirror speeds from cached hostfile
      * base: mirrors.aliyun.com
      * centos-gluster9: mirrors.aliyun.com
      * extras: mirrors.aliyun.com
      * updates: mirrors.aliyun.com
     akopytov_sysbench-source/signature                                                 |  833 B  00:00:00     
     Retrieving key from https://packagecloud.io/akopytov/sysbench/gpgkey
     Importing GPG key 0x04DCFD39:
      Userid     : "https://packagecloud.io/akopytov/sysbench-prerelease (https://packagecloud.io/docs#gpg_signing) <support@packagecloud.io>"
      Fingerprint: 9789 8d69 f99e e5ca c462 a0f8 cf10 4890 04dc fd39
      From       : https://packagecloud.io/akopytov/sysbench/gpgkey
     akopytov_sysbench-source/signature                                                 | 1.0 kB  00:00:01 !!! 
     base                                                                               | 3.6 kB  00:00:00     
     centos-gluster9                                                                    | 3.0 kB  00:00:00     
     extras                                                                             | 2.9 kB  00:00:00     
     updates                                                                            | 2.9 kB  00:00:00     
     akopytov_sysbench-source/primary                                                                                                | 2.0 kB  00:00:09     
     akopytov_sysbench-source                                                                                                                         15/15
     Package pygpgme-0.3-9.el7.x86_64 already installed and latest version
     Nothing to do
     Installing yum-utils...
     Loaded plugins: fastestmirror
     Loading mirror speeds from cached hostfile
      * base: mirrors.aliyun.com
      * centos-gluster9: mirrors.aliyun.com
      * extras: mirrors.aliyun.com
      * updates: mirrors.aliyun.com
     Package yum-utils-1.1.31-54.el7_8.noarch already installed and latest version
     Nothing to do
     Generating yum cache for akopytov_sysbench...
     Importing GPG key 0x04DCFD39:
      Userid     : "https://packagecloud.io/akopytov/sysbench-prerelease (https://packagecloud.io/docs#gpg_signing) <support@packagecloud.io>"
      Fingerprint: 9789 8d69 f99e e5ca c462 a0f8 cf10 4890 04dc fd39
      From       : https://packagecloud.io/akopytov/sysbench/gpgkey
     Generating yum cache for akopytov_sysbench-source...
     
     The repository is setup! You can now install packages.
     
     # 安装sysbench
     [root@ks-k8s-master-0 ~]# yum install sysbench
     Loaded plugins: fastestmirror
     Loading mirror speeds from cached hostfile
      * base: mirrors.aliyun.com
      * centos-gluster9: mirrors.aliyun.com
      * extras: mirrors.aliyun.com
      * updates: mirrors.aliyun.com
     akopytov_sysbench/x86_64/signature                                                                                  |  833 B  00:00:00     
     akopytov_sysbench/x86_64/signature                                                                                  | 1.0 kB  00:00:00 !!! 
     akopytov_sysbench-source/signature                                                                                  |  833 B  00:00:00     
     akopytov_sysbench-source/signature                                                                                  | 1.0 kB  00:00:00 !!! 
     Resolving Dependencies
     --> Running transaction check
     ---> Package sysbench.x86_64 0:1.0.20-1.el7 will be installed
     --> Processing Dependency: libpq.so.5()(64bit) for package: sysbench-1.0.20-1.el7.x86_64
     --> Running transaction check
     ---> Package postgresql-libs.x86_64 0:9.2.24-7.el7_9 will be installed
     --> Finished Dependency Resolution
     
     Dependencies Resolved
     
     ===========================================================================================================================================
      Package                            Arch                      Version                           Repository                            Size
     ===========================================================================================================================================
     Installing:
      sysbench                           x86_64                    1.0.20-1.el7                      akopytov_sysbench                    430 k
     Installing for dependencies:
      postgresql-libs                    x86_64                    9.2.24-7.el7_9                    updates                              235 k
     
     Transaction Summary
     ===========================================================================================================================================
     Install  1 Package (+1 Dependent package)
     
     Total download size: 665 k
     Installed size: 1.8 M
     Is this ok [y/d/N]: y
     Downloading packages:
     (1/2): postgresql-libs-9.2.24-7.el7_9.x86_64.rpm                                                                    | 235 kB  00:00:00     
     (2/2): sysbench-1.0.20-1.el7.x86_64.rpm                                                                             | 430 kB  00:00:03     
     -------------------------------------------------------------------------------------------------------------------------------------------
     Total                                                                                                      204 kB/s | 665 kB  00:00:03     
     Running transaction check
     Running transaction test
     Transaction test succeeded
     Running transaction
       Installing : postgresql-libs-9.2.24-7.el7_9.x86_64                                                                                   1/2 
       Installing : sysbench-1.0.20-1.el7.x86_64                                                                                            2/2 
       Verifying  : postgresql-libs-9.2.24-7.el7_9.x86_64                                                                                   1/2 
       Verifying  : sysbench-1.0.20-1.el7.x86_64                                                                                            2/2 
     
     Installed:
       sysbench.x86_64 0:1.0.20-1.el7                                                                                                           
     
     Dependency Installed:
       postgresql-libs.x86_64 0:9.2.24-7.el7_9                                                                                                  
     
     Complete!
     ```

   - 验证-执行命令查看版本

   - ```shell
     [root@ks-k8s-master-0 ~]# sysbench --version
     sysbench 1.0.20
     ```

     

### 02. 性能（基准）测试

1. 测试方案。

   - 测试参数。

   - |    指标    |   值    |
     | :--------: | :-----: |
     |   线程数   | 8/16/32 |
     | 单表数据量 | 100000  |
     |   表数量   |   16    |

     性能指标。

     |        指标         |                             说明                             |
     | :-----------------: | :----------------------------------------------------------: |
     |         TPS         | Transactions Per Second ，即数据库每秒执行的事务数，以 commit 成功次数为准。 |
     |         QPS         | Queries Per Second ，即数据库每秒执行的 SQL 数（含 insert、select、update、delete 等）。 |
     |         RT          | Response Time ，响应时间。包括平均响应时间、最小响应时间、最大响应时间、每个响应时间的查询占比。比较需要重点关注的是，前 95-99% 的最大响应时间。因为它决定了大多数情况下的短板。 |
     | Concurrency Threads |             并发量，每秒可处理的查询请求的数量。             |

     

2. 准备测试数据。

   - 使用我们在k8s上创建的数据库，涉及数据库操作命令，需要**终端**登录到容器内运行。

   - 提前创建测试用数据库**sbtest**，并赋予root从任意IP远程管理所有数据库的权限。

     > **生产环境千万不要这么搞，一定要遵循最小化原则！**

   - ```mysql
     # bash
     root@mysql-0:/# mysql -u root -p
     Enter password:
     Welcome to the MySQL monitor.  Commands end with ; or \g.
     Your MySQL connection id is 4
     Server version: 5.7.38 MySQL Community Server (GPL)
     
     Copyright (c) 2000, 2022, Oracle and/or its affiliates.
     
     Oracle is a registered trademark of Oracle Corporation and/or its
     affiliates. Other names may be trademarks of their respective
     owners.
     
     Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
     
     mysql> create database sbtest;
     Query OK, 1 row affected (0.02 sec)
     
     mysql> grant all privileges on *.* to 'root'@'%' identified by 'P@88w0rd' with grant option;
     Query OK, 0 rows affected, 1 warning (0.02 sec)
     
     ```

   - 测试数据库是否能连接。

   - ```shell
     # 安装mysql客户端，下面的示例是在k8s节点上安装的，由于系统是最小化安装，所有会安装很多依赖。实际测试可以起一个mysql的pod或是用其他的mysql客户端工具。
     
     [root@ks-k8s-master-0 ~]# yum install mysql
     Loaded plugins: fastestmirror
     Loading mirror speeds from cached hostfile
      * base: mirrors.aliyun.com
      * centos-gluster9: mirrors.aliyun.com
      * extras: mirrors.aliyun.com
      * updates: mirrors.aliyun.com
     akopytov_sysbench/x86_64/signature                                                                                  |  833 B  00:00:00     
     akopytov_sysbench/x86_64/signature                                                                                  | 1.0 kB  00:00:00 !!! 
     akopytov_sysbench-source/signature                                                                                  |  833 B  00:00:00     
     akopytov_sysbench-source/signature                                                                                  | 1.0 kB  00:00:00 !!! 
     Resolving Dependencies
     --> Running transaction check
     ---> Package mariadb.x86_64 1:5.5.68-1.el7 will be installed
     --> Processing Dependency: perl(Sys::Hostname) for package: 1:mariadb-5.5.68-1.el7.x86_64
     --> Processing Dependency: perl(IPC::Open3) for package: 1:mariadb-5.5.68-1.el7.x86_64
     --> Processing Dependency: perl(Getopt::Long) for package: 1:mariadb-5.5.68-1.el7.x86_64
     --> Processing Dependency: perl(File::Temp) for package: 1:mariadb-5.5.68-1.el7.x86_64
     --> Processing Dependency: perl(Fcntl) for package: 1:mariadb-5.5.68-1.el7.x86_64
     --> Processing Dependency: perl(Exporter) for package: 1:mariadb-5.5.68-1.el7.x86_64
     --> Processing Dependency: /usr/bin/perl for package: 1:mariadb-5.5.68-1.el7.x86_64
     --> Running transaction check
     ---> Package perl.x86_64 4:5.16.3-299.el7_9 will be installed
     --> Processing Dependency: perl-libs = 4:5.16.3-299.el7_9 for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Socket) >= 1.3 for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Scalar::Util) >= 1.10 for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl-macros for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl-libs for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(threads::shared) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(threads) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(constant) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Time::Local) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Time::HiRes) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Storable) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Socket) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Scalar::Util) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Pod::Simple::XHTML) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Pod::Simple::Search) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Filter::Util::Call) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(File::Spec::Unix) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(File::Spec::Functions) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(File::Spec) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(File::Path) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Cwd) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: perl(Carp) for package: 4:perl-5.16.3-299.el7_9.x86_64
     --> Processing Dependency: libperl.so()(64bit) for package: 4:perl-5.16.3-299.el7_9.x86_64
     ---> Package perl-Exporter.noarch 0:5.68-3.el7 will be installed
     ---> Package perl-File-Temp.noarch 0:0.23.01-3.el7 will be installed
     ---> Package perl-Getopt-Long.noarch 0:2.40-3.el7 will be installed
     --> Processing Dependency: perl(Pod::Usage) >= 1.14 for package: perl-Getopt-Long-2.40-3.el7.noarch
     --> Processing Dependency: perl(Text::ParseWords) for package: perl-Getopt-Long-2.40-3.el7.noarch
     --> Running transaction check
     ---> Package perl-Carp.noarch 0:1.26-244.el7 will be installed
     ---> Package perl-File-Path.noarch 0:2.09-2.el7 will be installed
     ---> Package perl-Filter.x86_64 0:1.49-3.el7 will be installed
     ---> Package perl-PathTools.x86_64 0:3.40-5.el7 will be installed
     ---> Package perl-Pod-Simple.noarch 1:3.28-4.el7 will be installed
     --> Processing Dependency: perl(Pod::Escapes) >= 1.04 for package: 1:perl-Pod-Simple-3.28-4.el7.noarch
     --> Processing Dependency: perl(Encode) for package: 1:perl-Pod-Simple-3.28-4.el7.noarch
     ---> Package perl-Pod-Usage.noarch 0:1.63-3.el7 will be installed
     --> Processing Dependency: perl(Pod::Text) >= 3.15 for package: perl-Pod-Usage-1.63-3.el7.noarch
     --> Processing Dependency: perl-Pod-Perldoc for package: perl-Pod-Usage-1.63-3.el7.noarch
     ---> Package perl-Scalar-List-Utils.x86_64 0:1.27-248.el7 will be installed
     ---> Package perl-Socket.x86_64 0:2.010-5.el7 will be installed
     ---> Package perl-Storable.x86_64 0:2.45-3.el7 will be installed
     ---> Package perl-Text-ParseWords.noarch 0:3.29-4.el7 will be installed
     ---> Package perl-Time-HiRes.x86_64 4:1.9725-3.el7 will be installed
     ---> Package perl-Time-Local.noarch 0:1.2300-2.el7 will be installed
     ---> Package perl-constant.noarch 0:1.27-2.el7 will be installed
     ---> Package perl-libs.x86_64 4:5.16.3-299.el7_9 will be installed
     ---> Package perl-macros.x86_64 4:5.16.3-299.el7_9 will be installed
     ---> Package perl-threads.x86_64 0:1.87-4.el7 will be installed
     ---> Package perl-threads-shared.x86_64 0:1.43-6.el7 will be installed
     --> Running transaction check
     ---> Package perl-Encode.x86_64 0:2.51-7.el7 will be installed
     ---> Package perl-Pod-Escapes.noarch 1:1.04-299.el7_9 will be installed
     ---> Package perl-Pod-Perldoc.noarch 0:3.20-4.el7 will be installed
     --> Processing Dependency: perl(parent) for package: perl-Pod-Perldoc-3.20-4.el7.noarch
     --> Processing Dependency: perl(HTTP::Tiny) for package: perl-Pod-Perldoc-3.20-4.el7.noarch
     ---> Package perl-podlators.noarch 0:2.5.1-3.el7 will be installed
     --> Running transaction check
     ---> Package perl-HTTP-Tiny.noarch 0:0.033-3.el7 will be installed
     ---> Package perl-parent.noarch 1:0.225-244.el7 will be installed
     --> Finished Dependency Resolution
     
     Dependencies Resolved
     
     ===========================================================================================================================================
      Package                                  Arch                     Version                                 Repository                 Size
     ===========================================================================================================================================
     Installing:
      mariadb                                  x86_64                   1:5.5.68-1.el7                          base                      8.8 M
     Installing for dependencies:
      perl                                     x86_64                   4:5.16.3-299.el7_9                      updates                   8.0 M
      perl-Carp                                noarch                   1.26-244.el7                            base                       19 k
      perl-Encode                              x86_64                   2.51-7.el7                              base                      1.5 M
      perl-Exporter                            noarch                   5.68-3.el7                              base                       28 k
      perl-File-Path                           noarch                   2.09-2.el7                              base                       26 k
      perl-File-Temp                           noarch                   0.23.01-3.el7                           base                       56 k
      perl-Filter                              x86_64                   1.49-3.el7                              base                       76 k
      perl-Getopt-Long                         noarch                   2.40-3.el7                              base                       56 k
      perl-HTTP-Tiny                           noarch                   0.033-3.el7                             base                       38 k
      perl-PathTools                           x86_64                   3.40-5.el7                              base                       82 k
      perl-Pod-Escapes                         noarch                   1:1.04-299.el7_9                        updates                    52 k
      perl-Pod-Perldoc                         noarch                   3.20-4.el7                              base                       87 k
      perl-Pod-Simple                          noarch                   1:3.28-4.el7                            base                      216 k
      perl-Pod-Usage                           noarch                   1.63-3.el7                              base                       27 k
      perl-Scalar-List-Utils                   x86_64                   1.27-248.el7                            base                       36 k
      perl-Socket                              x86_64                   2.010-5.el7                             base                       49 k
      perl-Storable                            x86_64                   2.45-3.el7                              base                       77 k
      perl-Text-ParseWords                     noarch                   3.29-4.el7                              base                       14 k
      perl-Time-HiRes                          x86_64                   4:1.9725-3.el7                          base                       45 k
      perl-Time-Local                          noarch                   1.2300-2.el7                            base                       24 k
      perl-constant                            noarch                   1.27-2.el7                              base                       19 k
      perl-libs                                x86_64                   4:5.16.3-299.el7_9                      updates                   690 k
      perl-macros                              x86_64                   4:5.16.3-299.el7_9                      updates                    44 k
      perl-parent                              noarch                   1:0.225-244.el7                         base                       12 k
      perl-podlators                           noarch                   2.5.1-3.el7                             base                      112 k
      perl-threads                             x86_64                   1.87-4.el7                              base                       49 k
      perl-threads-shared                      x86_64                   1.43-6.el7                              base                       39 k
     
     Transaction Summary
     ===========================================================================================================================================
     Install  1 Package (+27 Dependent packages)
     
     Total download size: 20 M
     Installed size: 85 M
     Is this ok [y/d/N]: y
     Downloading packages:
     (1/28): perl-Carp-1.26-244.el7.noarch.rpm                                                                           |  19 kB  00:00:00     
     (2/28): perl-Encode-2.51-7.el7.x86_64.rpm                                                                           | 1.5 MB  00:00:00     
     (3/28): perl-Exporter-5.68-3.el7.noarch.rpm                                                                         |  28 kB  00:00:00     
     (4/28): perl-File-Path-2.09-2.el7.noarch.rpm                                                                        |  26 kB  00:00:00     
     (5/28): perl-File-Temp-0.23.01-3.el7.noarch.rpm                                                                     |  56 kB  00:00:00     
     (6/28): perl-Filter-1.49-3.el7.x86_64.rpm                                                                           |  76 kB  00:00:00     
     (7/28): perl-Getopt-Long-2.40-3.el7.noarch.rpm                                                                      |  56 kB  00:00:00     
     (8/28): perl-HTTP-Tiny-0.033-3.el7.noarch.rpm                                                                       |  38 kB  00:00:00     
     (9/28): perl-PathTools-3.40-5.el7.x86_64.rpm                                                                        |  82 kB  00:00:00     
     (10/28): perl-5.16.3-299.el7_9.x86_64.rpm                                                                           | 8.0 MB  00:00:00     
     (11/28): perl-Pod-Perldoc-3.20-4.el7.noarch.rpm                                                                     |  87 kB  00:00:00     
     (12/28): mariadb-5.5.68-1.el7.x86_64.rpm                                                                            | 8.8 MB  00:00:00     
     (13/28): perl-Pod-Escapes-1.04-299.el7_9.noarch.rpm                                                                 |  52 kB  00:00:00     
     (14/28): perl-Pod-Simple-3.28-4.el7.noarch.rpm                                                                      | 216 kB  00:00:00     
     (15/28): perl-Scalar-List-Utils-1.27-248.el7.x86_64.rpm                                                             |  36 kB  00:00:00     
     (16/28): perl-Socket-2.010-5.el7.x86_64.rpm                                                                         |  49 kB  00:00:00     
     (17/28): perl-Storable-2.45-3.el7.x86_64.rpm                                                                        |  77 kB  00:00:00     
     (18/28): perl-Text-ParseWords-3.29-4.el7.noarch.rpm                                                                 |  14 kB  00:00:00     
     (19/28): perl-Time-HiRes-1.9725-3.el7.x86_64.rpm                                                                    |  45 kB  00:00:00     
     (20/28): perl-Pod-Usage-1.63-3.el7.noarch.rpm                                                                       |  27 kB  00:00:00     
     (21/28): perl-Time-Local-1.2300-2.el7.noarch.rpm                                                                    |  24 kB  00:00:00     
     (22/28): perl-constant-1.27-2.el7.noarch.rpm                                                                        |  19 kB  00:00:00     
     (23/28): perl-podlators-2.5.1-3.el7.noarch.rpm                                                                      | 112 kB  00:00:00     
     (24/28): perl-threads-1.87-4.el7.x86_64.rpm                                                                         |  49 kB  00:00:00     
     (25/28): perl-threads-shared-1.43-6.el7.x86_64.rpm                                                                  |  39 kB  00:00:00     
     (26/28): perl-macros-5.16.3-299.el7_9.x86_64.rpm                                                                    |  44 kB  00:00:00     
     (27/28): perl-libs-5.16.3-299.el7_9.x86_64.rpm                                                                      | 690 kB  00:00:00     
     (28/28): perl-parent-0.225-244.el7.noarch.rpm                                                                       |  12 kB  00:00:00     
     -------------------------------------------------------------------------------------------------------------------------------------------
     Total                                                                                                       14 MB/s |  20 MB  00:00:01     
     Running transaction check
     Running transaction test
     Transaction test succeeded
     Running transaction
       Installing : 1:perl-parent-0.225-244.el7.noarch                                                                                     1/28 
       Installing : perl-HTTP-Tiny-0.033-3.el7.noarch                                                                                      2/28 
       Installing : perl-podlators-2.5.1-3.el7.noarch                                                                                      3/28 
       Installing : perl-Pod-Perldoc-3.20-4.el7.noarch                                                                                     4/28 
       Installing : 1:perl-Pod-Escapes-1.04-299.el7_9.noarch                                                                               5/28 
       Installing : perl-Encode-2.51-7.el7.x86_64                                                                                          6/28 
       Installing : perl-Text-ParseWords-3.29-4.el7.noarch                                                                                 7/28 
       Installing : perl-Pod-Usage-1.63-3.el7.noarch                                                                                       8/28 
       Installing : 4:perl-macros-5.16.3-299.el7_9.x86_64                                                                                  9/28 
       Installing : perl-Storable-2.45-3.el7.x86_64                                                                                       10/28 
       Installing : perl-Exporter-5.68-3.el7.noarch                                                                                       11/28 
       Installing : perl-constant-1.27-2.el7.noarch                                                                                       12/28 
       Installing : perl-Socket-2.010-5.el7.x86_64                                                                                        13/28 
       Installing : perl-Time-Local-1.2300-2.el7.noarch                                                                                   14/28 
       Installing : perl-Carp-1.26-244.el7.noarch                                                                                         15/28 
       Installing : 4:perl-Time-HiRes-1.9725-3.el7.x86_64                                                                                 16/28 
       Installing : perl-PathTools-3.40-5.el7.x86_64                                                                                      17/28 
       Installing : perl-Scalar-List-Utils-1.27-248.el7.x86_64                                                                            18/28 
       Installing : 1:perl-Pod-Simple-3.28-4.el7.noarch                                                                                   19/28 
       Installing : perl-File-Temp-0.23.01-3.el7.noarch                                                                                   20/28 
       Installing : perl-File-Path-2.09-2.el7.noarch                                                                                      21/28 
       Installing : perl-threads-shared-1.43-6.el7.x86_64                                                                                 22/28 
       Installing : perl-threads-1.87-4.el7.x86_64                                                                                        23/28 
       Installing : perl-Filter-1.49-3.el7.x86_64                                                                                         24/28 
       Installing : 4:perl-libs-5.16.3-299.el7_9.x86_64                                                                                   25/28 
       Installing : perl-Getopt-Long-2.40-3.el7.noarch                                                                                    26/28 
       Installing : 4:perl-5.16.3-299.el7_9.x86_64                                                                                        27/28 
       Installing : 1:mariadb-5.5.68-1.el7.x86_64                                                                                         28/28 
       Verifying  : perl-HTTP-Tiny-0.033-3.el7.noarch                                                                                      1/28 
       Verifying  : perl-threads-shared-1.43-6.el7.x86_64                                                                                  2/28 
       Verifying  : perl-Storable-2.45-3.el7.x86_64                                                                                        3/28 
       Verifying  : perl-Exporter-5.68-3.el7.noarch                                                                                        4/28 
       Verifying  : perl-constant-1.27-2.el7.noarch                                                                                        5/28 
       Verifying  : perl-PathTools-3.40-5.el7.x86_64                                                                                       6/28 
       Verifying  : perl-Socket-2.010-5.el7.x86_64                                                                                         7/28 
       Verifying  : 1:perl-parent-0.225-244.el7.noarch                                                                                     8/28 
       Verifying  : 4:perl-macros-5.16.3-299.el7_9.x86_64                                                                                  9/28 
       Verifying  : perl-File-Temp-0.23.01-3.el7.noarch                                                                                   10/28 
       Verifying  : 1:perl-Pod-Simple-3.28-4.el7.noarch                                                                                   11/28 
       Verifying  : perl-Time-Local-1.2300-2.el7.noarch                                                                                   12/28 
       Verifying  : 1:perl-Pod-Escapes-1.04-299.el7_9.noarch                                                                              13/28 
       Verifying  : perl-Carp-1.26-244.el7.noarch                                                                                         14/28 
       Verifying  : 4:perl-Time-HiRes-1.9725-3.el7.x86_64                                                                                 15/28 
       Verifying  : perl-Scalar-List-Utils-1.27-248.el7.x86_64                                                                            16/28 
       Verifying  : perl-Pod-Usage-1.63-3.el7.noarch                                                                                      17/28 
       Verifying  : perl-Encode-2.51-7.el7.x86_64                                                                                         18/28 
       Verifying  : perl-Pod-Perldoc-3.20-4.el7.noarch                                                                                    19/28 
       Verifying  : perl-podlators-2.5.1-3.el7.noarch                                                                                     20/28 
       Verifying  : 4:perl-5.16.3-299.el7_9.x86_64                                                                                        21/28 
       Verifying  : perl-File-Path-2.09-2.el7.noarch                                                                                      22/28 
       Verifying  : perl-threads-1.87-4.el7.x86_64                                                                                        23/28 
       Verifying  : 1:mariadb-5.5.68-1.el7.x86_64                                                                                         24/28 
       Verifying  : perl-Filter-1.49-3.el7.x86_64                                                                                         25/28 
       Verifying  : perl-Getopt-Long-2.40-3.el7.noarch                                                                                    26/28 
       Verifying  : perl-Text-ParseWords-3.29-4.el7.noarch                                                                                27/28 
       Verifying  : 4:perl-libs-5.16.3-299.el7_9.x86_64                                                                                   28/28 
     
     Installed:
       mariadb.x86_64 1:5.5.68-1.el7                                                                                                            
     
     Dependency Installed:
       perl.x86_64 4:5.16.3-299.el7_9            perl-Carp.noarch 0:1.26-244.el7              perl-Encode.x86_64 0:2.51-7.el7                 
       perl-Exporter.noarch 0:5.68-3.el7         perl-File-Path.noarch 0:2.09-2.el7           perl-File-Temp.noarch 0:0.23.01-3.el7           
       perl-Filter.x86_64 0:1.49-3.el7           perl-Getopt-Long.noarch 0:2.40-3.el7         perl-HTTP-Tiny.noarch 0:0.033-3.el7             
       perl-PathTools.x86_64 0:3.40-5.el7        perl-Pod-Escapes.noarch 1:1.04-299.el7_9     perl-Pod-Perldoc.noarch 0:3.20-4.el7            
       perl-Pod-Simple.noarch 1:3.28-4.el7       perl-Pod-Usage.noarch 0:1.63-3.el7           perl-Scalar-List-Utils.x86_64 0:1.27-248.el7    
       perl-Socket.x86_64 0:2.010-5.el7          perl-Storable.x86_64 0:2.45-3.el7            perl-Text-ParseWords.noarch 0:3.29-4.el7        
       perl-Time-HiRes.x86_64 4:1.9725-3.el7     perl-Time-Local.noarch 0:1.2300-2.el7        perl-constant.noarch 0:1.27-2.el7               
       perl-libs.x86_64 4:5.16.3-299.el7_9       perl-macros.x86_64 4:5.16.3-299.el7_9        perl-parent.noarch 1:0.225-244.el7              
       perl-podlators.noarch 0:2.5.1-3.el7       perl-threads.x86_64 0:1.87-4.el7             perl-threads-shared.x86_64 0:1.43-6.el7         
     
     Complete!
     
     # 测试MySQL服务连通性 -h 是k8s节点的IP -P 是mysql外部服务的端口号
     
     [root@ks-k8s-master-0 ~]# mysql -h 192.168.9.91 -P 32529 -u root -p
     Enter password: 
     Welcome to the MariaDB monitor.  Commands end with ; or \g.
     Your MySQL connection id is 5
     Server version: 5.7.38 MySQL Community Server (GPL)
     
     Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
     
     Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
     
     MySQL [(none)]> 
     ```

   - 准备测试数据。

   - ```shell
     [root@ks-k8s-master-0 ~]# sysbench --db-driver=mysql --mysql-host=192.168.9.91 --mysql-port=32529 --mysql-user=root --mysql-password=P@88w0rd --mysql-db=sbtest --table-size=100000 --tables=16 --threads=8 --events=999999999 --report-interval=10 --time=100 /usr/share/sysbench/oltp_common.lua prepare
     sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)
     
     Initializing worker threads...
     
     Creating table 'sbtest6'...
     Creating table 'sbtest2'...
     Creating table 'sbtest8'...
     Creating table 'sbtest3'...
     Creating table 'sbtest7'...
     Creating table 'sbtest5'...
     Creating table 'sbtest1'...
     Creating table 'sbtest4'...
     Inserting 100000 records into 'sbtest3'
     Inserting 100000 records into 'sbtest6'
     Inserting 100000 records into 'sbtest1'
     Inserting 100000 records into 'sbtest4'
     Inserting 100000 records into 'sbtest7'
     Inserting 100000 records into 'sbtest5'
     Inserting 100000 records into 'sbtest2'
     Inserting 100000 records into 'sbtest8'
     Creating a secondary index on 'sbtest3'...
     Creating table 'sbtest11'...
     Inserting 100000 records into 'sbtest11'
     Creating a secondary index on 'sbtest5'...
     Creating a secondary index on 'sbtest1'...
     Creating a secondary index on 'sbtest6'...
     Creating a secondary index on 'sbtest4'...
     Creating a secondary index on 'sbtest7'...
     Creating a secondary index on 'sbtest2'...
     Creating a secondary index on 'sbtest8'...
     Creating table 'sbtest13'...
     Inserting 100000 records into 'sbtest13'
     Creating table 'sbtest9'...
     Inserting 100000 records into 'sbtest9'
     Creating table 'sbtest14'...
     Creating table 'sbtest12'...
     Inserting 100000 records into 'sbtest14'
     Inserting 100000 records into 'sbtest12'
     Creating table 'sbtest15'...
     Inserting 100000 records into 'sbtest15'
     Creating table 'sbtest16'...
     Creating table 'sbtest10'...
     Inserting 100000 records into 'sbtest16'
     Inserting 100000 records into 'sbtest10'
     Creating a secondary index on 'sbtest11'...
     Creating a secondary index on 'sbtest13'...
     Creating a secondary index on 'sbtest9'...
     Creating a secondary index on 'sbtest12'...
     Creating a secondary index on 'sbtest14'...
     Creating a secondary index on 'sbtest15'...
     Creating a secondary index on 'sbtest10'...
     Creating a secondary index on 'sbtest16'...
     ```

3. 执行测试。

   - 8线程测试。

   - ```shell
     [root@ks-k8s-master-0 ~]# sysbench --db-driver=mysql --mysql-host=192.168.9.91 --mysql-port=32529 --mysql-user=root --mysql-password=P@88w0rd --mysql-db=sbtest --table-size=100000 --tables=16 --threads=8 --events=999999999 --report-interval=10 --time=100  /usr/share/sysbench/oltp_read_write.lua run
     sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)
     
     Running the test with following options:
     Number of threads: 8
     Report intermediate results every 10 second(s)
     Initializing random number generator from current time
     
     Initializing worker threads...
     
     Threads started!
     
     [ 10s ] thds: 8 tps: 88.46 qps: 1782.38 (r/w/o: 1249.19/355.46/177.73) lat (ms,95%): 267.41 err/s: 0.00 reconn/s: 0.00
     [ 20s ] thds: 8 tps: 84.31 qps: 1678.47 (r/w/o: 1173.42/336.43/168.62) lat (ms,95%): 277.21 err/s: 0.00 reconn/s: 0.00
     [ 30s ] thds: 8 tps: 70.20 qps: 1413.82 (r/w/o: 990.21/283.20/140.40) lat (ms,95%): 369.77 err/s: 0.00 reconn/s: 0.00
     [ 40s ] thds: 8 tps: 47.30 qps: 946.00 (r/w/o: 662.20/189.20/94.60) lat (ms,95%): 484.44 err/s: 0.00 reconn/s: 0.00
     [ 50s ] thds: 8 tps: 43.80 qps: 875.99 (r/w/o: 613.19/175.20/87.60) lat (ms,95%): 484.44 err/s: 0.00 reconn/s: 0.00
     [ 60s ] thds: 8 tps: 60.70 qps: 1213.08 (r/w/o: 849.69/242.00/121.40) lat (ms,95%): 411.96 err/s: 0.00 reconn/s: 0.00
     [ 70s ] thds: 8 tps: 53.90 qps: 1078.22 (r/w/o: 754.42/216.00/107.80) lat (ms,95%): 376.49 err/s: 0.00 reconn/s: 0.00
     [ 80s ] thds: 8 tps: 56.49 qps: 1127.98 (r/w/o: 790.11/224.88/112.99) lat (ms,95%): 397.39 err/s: 0.00 reconn/s: 0.00
     [ 90s ] thds: 8 tps: 50.60 qps: 1014.59 (r/w/o: 709.56/203.82/101.21) lat (ms,95%): 434.83 err/s: 0.00 reconn/s: 0.00
     [ 100s ] thds: 8 tps: 54.70 qps: 1093.12 (r/w/o: 765.22/218.50/109.40) lat (ms,95%): 390.30 err/s: 0.00 reconn/s: 0.00
     SQL statistics:
         queries performed:
             read:                            85582
             write:                           24452
             other:                           12226
             total:                           122260
         transactions:                        6113   (61.10 per sec.)
         queries:                             122260 (1221.96 per sec.)
         ignored errors:                      0      (0.00 per sec.)
         reconnects:                          0      (0.00 per sec.)
     
     General statistics:
         total time:                          100.0494s
         total number of events:              6113
     
     Latency (ms):
              min:                                   35.63
              avg:                                  130.89
              max:                                  951.86
              95th percentile:                      390.30
              sum:                               800129.59
     
     Threads fairness:
         events (avg/stddev):           764.1250/4.14
         execution time (avg/stddev):   100.0162/0.01
     ```
     
   - 16线程测试。

   - ```shell
     [root@ks-k8s-master-0 ~]# sysbench --db-driver=mysql --mysql-host=192.168.9.91 --mysql-port=32529 --mysql-user=root --mysql-password=P@88w0rd --mysql-db=sbtest --table-size=100000 --tables=16 --threads=16 --events=999999999 --report-interval=10 --time=100  /usr/share/sysbench/oltp_read_write.lua run
     sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)
     
     Running the test with following options:
     Number of threads: 16
     Report intermediate results every 10 second(s)
     Initializing random number generator from current time
     
     Initializing worker threads...
     
     Threads started!
     
     [ 10s ] thds: 16 tps: 114.41 qps: 2310.22 (r/w/o: 1621.18/458.63/230.41) lat (ms,95%): 369.77 err/s: 0.00 reconn/s: 0.00
     [ 20s ] thds: 16 tps: 106.35 qps: 2111.86 (r/w/o: 1474.74/424.41/212.71) lat (ms,95%): 383.33 err/s: 0.00 reconn/s: 0.00
     [ 30s ] thds: 16 tps: 80.40 qps: 1612.01 (r/w/o: 1129.21/322.00/160.80) lat (ms,95%): 623.33 err/s: 0.00 reconn/s: 0.00
     [ 40s ] thds: 16 tps: 63.40 qps: 1266.80 (r/w/o: 886.80/253.20/126.80) lat (ms,95%): 539.71 err/s: 0.00 reconn/s: 0.00
     [ 50s ] thds: 16 tps: 57.20 qps: 1145.91 (r/w/o: 802.74/228.78/114.39) lat (ms,95%): 549.52 err/s: 0.00 reconn/s: 0.00
     [ 60s ] thds: 16 tps: 69.91 qps: 1408.31 (r/w/o: 987.57/280.92/139.81) lat (ms,95%): 511.33 err/s: 0.00 reconn/s: 0.00
     [ 70s ] thds: 16 tps: 78.00 qps: 1547.22 (r/w/o: 1080.51/310.70/156.00) lat (ms,95%): 484.44 err/s: 0.00 reconn/s: 0.00
     [ 80s ] thds: 16 tps: 79.50 qps: 1599.87 (r/w/o: 1122.58/318.29/159.00) lat (ms,95%): 520.62 err/s: 0.00 reconn/s: 0.00
     [ 90s ] thds: 16 tps: 67.80 qps: 1354.83 (r/w/o: 947.62/271.61/135.60) lat (ms,95%): 539.71 err/s: 0.00 reconn/s: 0.00
     [ 100s ] thds: 16 tps: 73.90 qps: 1474.10 (r/w/o: 1030.80/295.50/147.80) lat (ms,95%): 502.20 err/s: 0.00 reconn/s: 0.00
     SQL statistics:
         queries performed:
             read:                            110950
             write:                           31700
             other:                           15850
             total:                           158500
         transactions:                        7925   (79.00 per sec.)
         queries:                             158500 (1580.05 per sec.)
         ignored errors:                      0      (0.00 per sec.)
         reconnects:                          0      (0.00 per sec.)
     
     General statistics:
         total time:                          100.3103s
         total number of events:              7925
     
     Latency (ms):
              min:                                   41.24
              avg:                                  202.44
              max:                                 1198.81
              95th percentile:                      511.33
              sum:                              1604328.52
     
     Threads fairness:
         events (avg/stddev):           495.3125/4.03
         execution time (avg/stddev):   100.2705/0.03
     
     ```
     
   - 32线程测试。

   - ```shell
     [root@ks-k8s-master-0 ~]# sysbench --db-driver=mysql --mysql-host=192.168.9.91 --mysql-port=32529 --mysql-user=root --mysql-password=P@88w0rd --mysql-db=sbtest --table-size=100000 --tables=16 --threads=32 --events=999999999 --report-interval=10 --time=100  /usr/share/sysbench/oltp_read_write.lua run
     sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)
     
     Running the test with following options:
     Number of threads: 32
     Report intermediate results every 10 second(s)
     Initializing random number generator from current time
     
     Initializing worker threads...
     
     Threads started!
     
     [ 10s ] thds: 32 tps: 140.10 qps: 2825.04 (r/w/o: 1981.25/560.39/283.39) lat (ms,95%): 450.77 err/s: 0.00 reconn/s: 0.00
     [ 20s ] thds: 32 tps: 124.41 qps: 2515.49 (r/w/o: 1763.43/503.24/248.82) lat (ms,95%): 549.52 err/s: 0.00 reconn/s: 0.00
     [ 30s ] thds: 32 tps: 95.90 qps: 1887.10 (r/w/o: 1316.70/378.60/191.80) lat (ms,95%): 733.00 err/s: 0.00 reconn/s: 0.00
     [ 40s ] thds: 32 tps: 81.80 qps: 1656.59 (r/w/o: 1164.89/328.10/163.60) lat (ms,95%): 707.07 err/s: 0.00 reconn/s: 0.00
     [ 50s ] thds: 32 tps: 82.60 qps: 1638.41 (r/w/o: 1143.51/329.70/165.20) lat (ms,95%): 657.93 err/s: 0.00 reconn/s: 0.00
     [ 60s ] thds: 32 tps: 94.34 qps: 1905.84 (r/w/o: 1336.62/380.65/188.58) lat (ms,95%): 623.33 err/s: 0.00 reconn/s: 0.00
     [ 70s ] thds: 32 tps: 87.86 qps: 1739.86 (r/w/o: 1215.31/348.73/175.82) lat (ms,95%): 634.66 err/s: 0.00 reconn/s: 0.00
     [ 80s ] thds: 32 tps: 84.40 qps: 1705.48 (r/w/o: 1196.49/340.20/168.80) lat (ms,95%): 759.88 err/s: 0.00 reconn/s: 0.00
     [ 90s ] thds: 32 tps: 80.50 qps: 1580.71 (r/w/o: 1101.70/318.00/161.00) lat (ms,95%): 612.21 err/s: 0.00 reconn/s: 0.00
     [ 100s ] thds: 32 tps: 81.40 qps: 1661.90 (r/w/o: 1167.00/332.10/162.80) lat (ms,95%): 707.07 err/s: 0.00 reconn/s: 0.00
     SQL statistics:
         queries performed:
             read:                            133924
             write:                           38264
             other:                           19132
             total:                           191320
         transactions:                        9566   (95.33 per sec.)
         queries:                             191320 (1906.56 per sec.)
         ignored errors:                      0      (0.00 per sec.)
         reconnects:                          0      (0.00 per sec.)
     
     General statistics:
         total time:                          100.3457s
         total number of events:              9566
     
     Latency (ms):
              min:                                   51.94
              avg:                                  335.14
              max:                                 1405.78
              95th percentile:                      657.93
              sum:                              3205913.85
     
     Threads fairness:
         events (avg/stddev):           298.9375/5.15
         execution time (avg/stddev):   100.1848/0.14
     
     ```
     
   - MySQL容器性能监控图。

   - ![kubesphere-projects-lstack-statefulsets-mysql-74](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-projects-lstack-statefulsets-mysql-74.png)

   - 清理测试数据(为了保证数据更精准，建议每次测试前都清理数据，准备数据，测试)。

   - ```shell
     [root@ks-k8s-master-0 ~]# sysbench --db-driver=mysql --mysql-host=192.168.9.91 --mysql-port=32529 --mysql-user=root --mysql-password=P@88w0rd --mysql-db=sbtest --table-size=100000 --tables=16 --threads=32 --events=999999999 --report-interval=10 --time=100  /usr/share/sysbench/oltp_read_write.lua cleanup
     sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)
     
     Dropping table 'sbtest1'...
     Dropping table 'sbtest2'...
     Dropping table 'sbtest3'...
     Dropping table 'sbtest4'...
     Dropping table 'sbtest5'...
     Dropping table 'sbtest6'...
     Dropping table 'sbtest7'...
     Dropping table 'sbtest8'...
     Dropping table 'sbtest9'...
     Dropping table 'sbtest10'...
     Dropping table 'sbtest11'...
     Dropping table 'sbtest12'...
     Dropping table 'sbtest13'...
     Dropping table 'sbtest14'...
     Dropping table 'sbtest15'...
     Dropping table 'sbtest16'...
     ```

4. 测试结果。

   - 结果汇总对比。

   - | 压测线程数量 | TPS  | QPS  | 延迟 |
     | ------------ | ---- | ---- | ---- |
     | 8            | 61   | 1221 | 130  |
     | 16           | 79   | 1580 | 202  |
     | 32           | 95   | 1906 | 335  |

   - 建议根据测试结果，调优！

## 4. 总结

本文详细介绍了KubeSphere图形化部署单节点MySQL上的安装配置过程，如何利用KubeSphere的图形化功能创建资源配置清单YAML文件的思路和具体操作过程，以后再部署其他在官网找不到详细配置指南的服务都可以借鉴这个方法。

本文还详细介绍了Git常用操作、如果将代码在多个在线代码仓库中存储并保持同步，还介绍了GitOps的基本概念并演示了如何用GitOps理念在原生K8S上部署MySQL服务。

最后，演示了MySQL常用性能测试工具sysbench的安装和基础使用。

我多年的一些运维经验和运维思路贯穿了全文。


> **参考文档**

- 都在文档里直接链接了


> **Get文档**

- Github https://github.com/devops/z-notes
- Gitee https://gitee.com/zdevops/z-notes

> **Get代码**

- Github https://github.com/devops/k8s-yaml
- Gitee https://gitee.com/zdevops/k8s-yaml

> **B站**

- [老Z手记](https://space.bilibili.com/1039301316)

> **版权声明** 

- 所有内容均属于原创，整理不易，感谢收藏，转载请标明出处。

> About Me

- 昵称：老Z
- 坐标：山东济南
- 职业：运维架构师/高级运维工程师=**运维**
- 联系方式：微信zdevops
- 关注的领域：云计算/云原生技术运维，自动化运维
- 技能标签：OpenStack、Ansible、K8S、Python、Go、CNCF
