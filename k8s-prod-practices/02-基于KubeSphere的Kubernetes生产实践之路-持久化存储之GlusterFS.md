# 

# 基于KubeSphere的Kubernetes生产实践之路-持久化存储之GlusterFS

大家好，我是老Z

本文接着上篇 **<<基于KubeSphere的Kubernetes生产实践之路-起步篇>>** 继续打造我们Kubernetes生产环境。

## 前提说明

### Kubernetes使用GlusterFS存储的方式

1. 通过Heketi管理GlusterFS，Kubernetes调用Heketi的接口

2. GlusterFS结合NFS-Ganesha提供NFS存储，Kubernetes采用NFS的方式挂载

3. Kubernetes挂载GlusterFS提供的数据卷到本地的存储目录，Kubernetes采用hostpatch的方式

4. 利用KubeSphere初始化安装集群的时候也可以直接使用GlusterFS作为持久化存储，具体操作可以参考[安装 GlusterFS](https://kubesphere.io/zh/docs/installing-on-linux/persistent-storage-configurations/install-glusterfs/)

### 使用说明

1. 本次选型使用的是Heketi的对接方案，使用比较广泛，网上的参考用例比较多，但是该方案也存在一定弊端，各位需要根据自己的情况选择
   
   - 实现形式在底层创建了一堆的lvs卷(文中有效果展示)，如果卷太多又太小的话，后期运维会比较麻烦，有一定的未知风险。
   
   - Heketi官方已经停止更新了，项目处于维护状态，这就比较麻烦了，新入坑者慎入

2. NFS-Ganesha的方案以前对接ceph的时候使用过，但是当时的稳定度不高，经常出现进程死掉的情况，对接GlusterFS没有测试过

3. 作为本地卷挂载的方式，个人认为稳定度和性能上应该是最好的，但是由于是本地盘，使用场景受限，可根据使用需求酌情使用

## GlusterFS分布式文件系统介绍

### 简介

GlusterFS系统是一个可扩展的网络文件系统，相比其他分布式文件系统，GlusterFS具有高扩展性、高可用性、高性能、可横向扩展等特点，并且其没有元数据服务器的设计，让整个服务没有单点故障的隐患。

架构图

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/glusterfs.jpg)

### 优点

#### 1. 无元数据节点性能瓶颈

- 采用无中心对称式架构，没有专用的元数据服务器，也就不存在元数据服务器瓶颈。

- 元数据存在于文件的属性和扩展属性中。当需要访问某文件时，客户端使用DHT算法，根据文件的路径和文件名计算出文件所在brick，然后由客户端从此brick获取数据，省去了同元数据服务器通信的过程。

#### 2. 良好的可扩展性

- 使用弹性hash算法代替传统的有元数据节点服务，获得了接近线性的高扩展性。

#### 3. 高可用

- 采用副本、EC等冗余设计，保证在冗余范围内的节点掉线时，仍然可以从其它服务节点获取数据，保证高可用性。采用弱一致性的设计，当向副本中文件写入数据时，客户端计算出文件所在brick，然后通过网络把数据传给所在brick，当其中有一个成功返回，就认为数据成功写入，不必等待其它brick返回，就会避免当某个节点网络异常或磁盘损坏时因为一个brick没有成功写入而导致写操作等待。

- 服务器端还会随着存储池的启动，而开启一个glustershd进程，这个进程会定期检查副本和EC卷中各个brick之间数据的一致性，并恢复。

#### 4. 存储池类型丰富

包括粗粒度、条带、副本、条带副本和EC，可以根据用户的需求，满足不同程度的冗余。

- 粗粒度卷不带任何冗余，文件不进行切片，是完整的存放在某个brick上。

- 条带卷不带任何冗余，文件会切片存储（默认大小为128kB）在不同的brick上。这些切片可以并发读写（并发粒度是条带块），可以明显提高读写性能。该模式一般只适合用于处理超大型文件和多节点性能要求高的情况。

- 副本卷冗余度高，副本数量可以灵活配置，可以保证数据的安全性。

- 条带副本卷是条带卷和副本卷的结合。

- EC卷使用EC校验算法，提供了低于副本卷的冗余度，冗余度小于100%，满足比较低的数据安全性，例如可以使2+1（冗余度为50%）、5+3（冗余度为60%）等。这个可以满足安全性要求不高的数据。

#### 5. 高性能

- 采用弱一致性的设计，向副本中写数据时，只要有一个brick成功返回，就认为写入成功，不必等待其它brick返回，这样的方式比强一致性要快。

- 还提供了I/O并发、write-behind、read-ahead、io-cache、条带等提高读写性能的技术。并且这些都还可以根据实际需求进行开启/关闭，i/o并发数量，cache大小都可以调整。

## Heketi介绍

### 简介

  Heketi提供了RESTful管理接口，可用于管理GlusterFS卷的生命周期。能够在[OpenStack](https://so.csdn.net/so/search?q=OpenStack&spm=1001.2101.3001.7020)，Kubernetes，Openshift等云平台上实现动态存储资源供应（动态在GlusterFS集群内选择bricks构建volume）；支持GlusterFS多集群管理。 Heketi的目标是提供一种在多个存储群集中创建，列出和删除GlusterFS卷的简单方法。 Heketi将智能地管理群集中整个磁盘的分配，创建和删除。 在满足任何请求之前，Heketi首先需要了解集群的拓扑（topologies ）也就是需要配置topologies.json文件 。 此json文件将数据资源组织为以下内容：群集、节点、设备的归属、以及块的归属。

  Heketi-cli命令行工具向Heketi提供要管理的GlusterFS集群的信息。它通过变量**HEKETI_CLI_SERVER**来找自已的服务端。

## 总体架构展示

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/glusterfs-heketi.png)

## 安装配置

### 系统基础配置

1. 检查操作系统版本并进行内核升级

```shell
[root@localhost ~]# yum update
[root@localhost ~]# cat /etc/redhat-release
CentOS Linux release 7.9.2009 (Core)
```

2. 修改主机名
- 节点一

```shell
[root@localhost ~]# hostnamectl set-hostname glusterfs-node-0
[root@localhost ~]# su
[root@glusterfs-node-0 ~]#
```

- 节点二

```shell
[root@localhost ~]# hostnamectl set-hostname glusterfs-node-1
[root@localhost ~]# su
[root@glusterfs-node-1 ~]#
```

- 节点三

```shell
[root@localhost ~]# hostnamectl set-hostname glusterfs-node-2
[root@localhost ~]# su
[root@glusterfs-node-2 ~]#
```

3. 关闭防火墙及selinux(所有节点执行)

```shell
# 关闭防火墙开机自启
[root@glusterfs-node-0 ~]# systemctl disable firewalld

# 临时关闭防火墙
[root@glusterfs-node-0 ~]# systemctl stop firewalld

# 永久关闭selinux
[root@glusterfs-node-0 ~]# sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 临时关闭selinux
[root@glusterfs-node-0 ~]# setenforce 0
```

### 安装配置GLusterFS

1. 配置glusterfs的yum源(所有节点执行)

```shell
[root@glusterfs-node-0 ~]# vi /etc/yum.repos.d/glusterfs.repo

[glusterfs]
name=glusterfs
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/7.9.2009/storage/x86_64/gluster-9/
#gpgkey=https://mirrors.tuna.tsinghua.edu.cn/centos/RPM-GPG-KEY-CentOS-7
gpgcheck=0
ebabled=1

#清除yum缓存
[root@glusterfs-node-0 ~]# yum clean all

#生成yum缓存
[root@glusterfs-node-0 ~]# yum makecache
```

2. 安装glusterfs(三个节点执行)

```shell
[root@glusterfs-node-0 ~]# yum install centos-release-gluster glusterfs-server glusterfs-client -y 
```

3. 启动glusterfs服务

```shell
[root@glusterfs-node-0 ~]#  systemctl start glusterd.service && systemctl enable glusterd.service

# 查看glusterfs服务状态
[root@glusterfs-node-0 ~]# systemctl status glusterd.service

# 查看glusterfs服务端口
[root@glusterfs-node-0 ~]# ss -lntp | grep glusterd
LISTEN     0      128          *:24007                    *:*                   users:(("glusterd",pid=108776,fd=10))
```

### 配置glusterfs集群节点

```shell
[root@glusterfs-node-0 ~]# gluster peer probe glusterfs-node-1
peer probe: success
[root@glusterfs-node-0 ~]# gluster peer probe glusterfs-node-2
peer probe: success

# 查看集群节点信息
[root@glusterfs-node-0 ~]# gluster peer status
Number of Peers: 2

Hostname: glusterfs-node-1
Uuid: 50a3da37-9f0f-49c7-aae4-e4a475beecc9
State: Peer in Cluster (Connected)

Hostname: glusterfs-node-2
Uuid: da245e40-52a7-4651-b44b-8222c237f192
State: Peer in Cluster (Connected)
```

### 安装配置Heketi（任选一节点,这里选择节点一）

1. 安装软件包

```shell
[root@glusterfs-node-0 ~]# yum install heketi heketi-client -y 
```

2. heketi服务必要配置

```shell
# 生成密钥并配置免密登录
[root@glusterfs-node-0 ~]$ ssh-keygen -t rsa -q -f /etc/heketi/private_key -N ""
[root@glusterfs-node-0 ~]$ ssh-copy-id -i /etc/heketi/private_key.pub glusterfs-node-0
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/etc/heketi/private_key.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'glusterfs-node-0'"
and check to make sure that only the key(s) you wanted were added.

[root@glusterfs-node-0 ~]$ ssh-copy-id -i /etc/heketi/private_key.pub glusterfs-node-1
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/etc/heketi/private_key.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'glusterfs-node-1'"
and check to make sure that only the key(s) you wanted were added.

[root@glusterfs-node-0 ~]$ ssh-copy-id -i /etc/heketi/private_key.pub glusterfs-node-2
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/etc/heketi/private_key.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'glusterfs-node-2'"
and check to make sure that only the key(s) you wanted were added.


#测试免密登录
[root@glusterfs-node-0 ~]$ ssh glusterfs-node-0
Last login: Wed Mar  2 09:12:30 2022
[root@glusterfs-node-0 ~]$ exit
[root@glusterfs-node-0 ~]$ ssh glusterfs-node-1
Last login: Wed Mar  2 09:12:30 2022
[root@glusterfs-node-1 ~]$ exit
[root@glusterfs-node-0 ~]$ ssh glusterfs-node-2
Last login: Wed Mar  2 09:12:30 2022
[root@glusterfs-node-2 ~]$ exit
#修改目录权限
[root@glusterfs-node-0 ~]# chown heketi:heketi /etc/heketi/ -R
[root@glusterfs-node-0 ~]# chown heketi:heketi /var/lib/heketi -R
[root@glusterfs-node-0 ~]# chown heketi.heketi /etc/heketi/private_key
```

### 编写heketi配置文件

```shell
#修改heketi的配置文件
[root@glusterfs-node-0 ~]# vi /etc/heketi/heketi.json
{
  "_port_comment": "Heketi Server Port Number",
  "port": "48080",               #######heketi服务端口可以自定义

  "_use_auth": "Enable JWT authorization. Please enable for deployment",
  "use_auth": true,              ########启动用户认证模式

  "_jwt": "Private keys for access",
  "jwt": {
    "_admin": "Admin has access to all APIs",
    "admin": {
      "key": "admin@P@ssW0rd"          ######heketi所有接口权限admin用户的密码
    },
    "_user": "User only has access to /volumes endpoint",
    "user": {
      "key": "user@P@ssW0rd"           #####heketi仅仅对卷相关API有权限的user用户密码
    }
  },

  "_glusterfs_comment": "GlusterFS Configuration",
  "glusterfs": {
    "_executor_comment": [
      "Execute plugin. Possible choices: mock, ssh",
      "mock: This setting is used for testing and development.",
      "      It will not send commands to any node.",
      "ssh:  This setting will notify Heketi to ssh to the nodes.",
      "      It will need the values in sshexec to be configured.",
      "kubernetes: Communicate with GlusterFS containers over",
      "            Kubernetes exec api."
    ],
    "executor": "ssh",              #########配置heketi使用ssh认证模式

    "_sshexec_comment": "SSH username and private key file information",
    "sshexec": {
      "keyfile": "/etc/heketi/private_key",          #######heketi使用主机用户的认证密钥文件
      "user": "root",                                  ########heketi使用主机的用户root
      "port": "22",                                     ######主机的ssh服务端口
      "fstab": "/etc/fstab"
    },

    "_kubeexec_comment": "Kubernetes configuration",
    "kubeexec": {
      "host" :"https://kubernetes.host:8443",
      "cert" : "/path/to/crt.file",
      "insecure": false,
      "user": "kubernetes username",
      "password": "password for kubernetes user",
      "namespace": "OpenShift project or Kubernetes namespace",
      "fstab": "Optional: Specify fstab file on node.  Default is /etc/fstab"
    },

    "_db_comment": "Database file name",
    "db": "/var/lib/heketi/heketi.db",       #####heketi数据存储文件

    "_loglevel_comment": [
      "Set log level. Choices are:",
      "  none, critical, error, warning, info, debug",
      "Default is warning"
    ],
    "loglevel" : "debug"
  }
}
```

### 启动服务

```shell
#启动heketi服务
[root@glusterfs-node-0 ~]# systemctl enable heketi.service && systemctl start heketi.service

#查看服务状态
[root@glusterfs-node-0 ~]# systemctl status heketi
● heketi.service - Heketi Server
   Loaded: loaded (/usr/lib/systemd/system/heketi.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-03-02 12:02:59 IST; 14min ago
 Main PID: 120229 (heketi)
   CGroup: /system.slice/heketi.service
           └─120229 /usr/bin/heketi --config=/etc/heketi/heketi.json

Mar 02 12:17:01 glusterfs-node-0 heketi[120229]: Main PID: 234479 (glusterd)
Mar 02 12:17:01 glusterfs-node-0 heketi[120229]: Tasks: 9
Mar 02 12:17:01 glusterfs-node-0 heketi[120229]: Memory: 6.0M
Mar 02 12:17:01 glusterfs-node-0 heketi[120229]: CGroup: /system.slice/glusterd.service
Mar 02 12:17:01 glusterfs-node-0 heketi[120229]: └─234479 /usr/sbin/glusterd -p /var/run/glusterd.pid --log-level INFO
Mar 02 12:17:01 glusterfs-node-0 heketi[120229]: Mar 02 13:52:19 glusterfs-node-2 systemd[1]: Starting GlusterFS, a clustered file-system server...
Mar 02 12:17:01 glusterfs-node-0 heketi[120229]: Mar 02 13:52:19 glusterfs-node-2 systemd[1]: Started GlusterFS, a clustered file-system server.
Mar 02 12:17:01 glusterfs-node-0 heketi[120229]: ]: Stderr []
Mar 02 12:17:01 glusterfs-node-0 heketi[120229]: [heketi] INFO 2022/03/02 12:17:01 Periodic health check status: node cf3ad692cc008e697eb68f2863843391 up=true
Mar 02 12:17:01 glusterfs-node-0 heketi[120229]: [heketi] INFO 2022/03/02 12:17:01 Cleaned 0 nodes from health cache
```

### 配置topology

```shell
[root@glusterfs-node-0 ~]# vi /etc/heketi/topology.json #配置三个节点的使用信息，这里/dev/db表示后端存储所要使用的数据盘
{
    "clusters": [
        {
            "nodes": [
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.10.1"
                            ],
                            "storage": [
                                "192.168.10.1"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.10.2"
                            ],
                            "storage": [
                                "192.168.10.2"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.10.3"
                            ],
                            "storage": [
                                "192.168.10.3"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb"
                    ]
                }
            ]
        }
    ]
}
```

### 配置环境变量，便于后续管理

```shell
[root@glusterfs-node-0 ~]# echo "export HEKETI_CLI_SERVER=http://192.168.10.1:48080" >> /etc/profile.d/heketi.sh 
#192.168.10.1为heketi地址
#--user指定的用户为heketi配置文件中配置的admin用户--secret指定的密码为heketi配置文件中配置的admin密码
[root@glusterfs-node-0 ~]# echo "alias heketi-cli='heketi-cli --server '$HEKETI_CLI_SERVER' --user admin --secret admin@P@ssW0rd'" >> ~/.bashrc 
#加载环境变量
[root@glusterfs-node-0 ~]# source /etc/profile.d/heketi.sh
[root@glusterfs-node-0 ~]# source ~/.bashrc
```

### 根据topology.json来创建集群

```shell
#创建集群
[root@glusterfs-node-0 ~]# heketi-cli topology load --json=/etc/heketi/topology.json
        Found node 192.168.10.1 on cluster b89a07c2c6e8c533322591bf2a4aa613
                Adding device /dev/sdb ... OK
        Found node 192.168.10.2 on cluster b89a07c2c6e8c533322591bf2a4aa613
                Adding device /dev/sdb ... OK
        Found node 192.168.10.3 on cluster b89a07c2c6e8c533322591bf2a4aa613
                Adding device /dev/sdb ... OK
#查看集群列表
[root@glusterfs-node-0 ~]# heketi-cli cluster list
Clusters:
Id:b89a07c2c6e8c533322591bf2a4aa613 [file][block]

#查看集群详情，此命令用到的id为查看集群状态时返回的id
[root@glusterfs-node-0 ~]# heketi-cli cluster info b89a07c2c6e8c533322591bf2a4aa613
Cluster id: b89a07c2c6e8c533322591bf2a4aa613
Nodes:
1218c773299856ebaef6f2461051640c
8bffed2d55f652a412c098ee31246559
cf3ad692cc008e697eb68f2863843391
Volumes:

Block: true

File: true

#查看集群节点
[root@glusterfs-node-0 heketi]# heketi-cli node list
Id:1218c773299856ebaef6f2461051640c     Cluster:b89a07c2c6e8c533322591bf2a4aa613
Id:8bffed2d55f652a412c098ee31246559     Cluster:b89a07c2c6e8c533322591bf2a4aa613
Id:cf3ad692cc008e697eb68f2863843391     Cluster:b89a07c2c6e8c533322591bf2a4aa613
```

### 创建集群时报错解决办法

如果数据盘不是新盘曾经被使用过，创建集群时会报错，可按下面的方法处理

```shell
[root@glusterfs-node-0 ~]# heketi-cli topology load --json=/etc/heketi/topology.json
Creating cluster ... ID: b89a07c2c6e8c533322591bf2a4aa613
        Allowing file volumes on cluster.
        Allowing block volumes on cluster.
        Creating node 10.4.11.38 ... ID: 8bffed2d55f652a412c098ee31246559
                Adding device /dev/sdb ... Unable to add device: Setup of device /dev/sdb failed (already initialized or contains data?):   Can't initialize physical volume "/dev/sdb" of volume group "vg_423c25b95cf33cb94fa192aedd325b26" without -ff
  /dev/sdb: physical volume not initialized.
        Creating node 10.4.11.39 ... ID: 1218c773299856ebaef6f2461051640c
                Adding device /dev/sdb ... Unable to add device: Setup of device /dev/sdb failed (already initialized or contains data?):   Can't initialize physical volume "/dev/sdb" of volume group "vg_564e3fb62e748f1451f387deb469541e" without -ff
  /dev/sdb: physical volume not initialized.
        Creating node 10.4.11.40 ... ID: cf3ad692cc008e697eb68f2863843391
                Adding device /dev/sdb ... Unable to add device: Setup of device /dev/sdb failed (already initialized or contains data?):   Can't initialize physical volume "/dev/sdb" of volume group "vg_5bc298db6e8cc335d545c96eed7150f0" without -ff

#所有glusterfs节点执行
[root@glusterfs-node-0 ~]# mkfs.xfs -f /dev/sdb
meta-data=/dev/sdb               isize=512    agcount=4, agsize=183105408 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=732421632, imaxpct=5
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=357627, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@glusterfs-node-0 ~]# pvcreate -ff --metadatasize=128M --dataalignment=256K /dev/sdb
  Wiping xfs signature on /dev/sdb.
  Physical volume "/dev/sdb" successfully created.
```

### 验证测试

1. 创建卷
   
   ```shell
   [root@glusterfs-node-0 ~]# heketi-cli volume create --size=2 --replica=3
   Name: vol_aa8a1280b5133a36b32cf552ec9dd3f3
   Size: 2
   Volume Id: aa8a1280b5133a36b32cf552ec9dd3f3
   Cluster Id: b89a07c2c6e8c533322591bf2a4aa613
   Mount: 192.168.10.2:vol_aa8a1280b5133a36b32cf552ec9dd3f3
   Mount Options: backup-volfile-servers=192.168.10.1,192.168.10.3
   Block: false
   Free Size: 0
   Reserved Size: 0
   Block Hosting Restriction: (none)
   Block Volumes: []
   Durability Type: replicate
   Distribute Count: 1
   Replica Count: 3
   ```

2. 查看卷
   
   ```shell
   #查看卷
   [root@glusterfs-node-0 heketi]# heketi-cli volume list
   Id:aa8a1280b5133a36b32cf552ec9dd3f3    Cluster:b89a07c2c6e8c533322591bf2a4aa613    Name:vol_aa8a1280b5133a36b32cf552ec9dd3f
   #挂载测试，下列命令中的192.168.10.2:vol_aa8a1280b5133a36b32cf552ec9dd3f3为上方创建卷时Mount的返回值，可以用heketi-cli volum info 卷id进行查看
   [root@glusterfs-node-0 heketi]# mount -t glusterfs 192.168.10.2:vol_aa8a1280b5133a36b32cf552ec9dd3f3 /mnt
   #查看挂载详情
   [root@glusterfs-node-0 heketi]# df -h
   Filesystem                                                                              Size  Used Avail Use% Mounted on
   .............................
   192.168.10.2:vol_aa8a1280b5133a36b32cf552ec9dd3f3                                       2.0G   54M  2.0G   3% /mnt
   ```

3. 节点详细信息查看
   
   可以看出底层vg和lv的创建详情，了解底层的分配细节
   
   ```shell
   ##节点一
   
   #查看节点创建出的pv
   [root@glusterfs-node-0 ~]# pvdisplay
   
     --- Physical volume ---
     PV Name               /dev/sdb
     VG Name               vg_10a23554385cca32a01d7fc3373faf2d
     PV Size               <2.73 TiB / not usable 130.00 MiB
     Allocatable           yes
     PE Size               4.00 MiB
     Total PE              715223
     Free PE              714705
     Allocated PE          518
     PV UUID               mkDi5V-JHnR-xy2Y-5FcW-2JKg-K9dQ-JeXQCF
   
   #查看节点创建出的vg
   [root@glusterfs-node-0 ~]# vgdisplay
   
     --- Volume group ---
     VG Name               vg_10a23554385cca32a01d7fc3373faf2d
     System ID
     Format                lvm2
     Metadata Areas        1
     Metadata Sequence No  22
     VG Access             read/write
     VG Status             resizable
     MAX LV                0
     Cur LV                2
     Open LV               1
     Max PV                0
     Cur PV                1
     Act PV                1
     VG Size               <2.73 TiB
     PE Size               4.00 MiB
     Total PE              715223
     Alloc PE / Size       518 / 2.02 GiB
     Free  PE / Size       714705 / <2.73 TiB
     VG UUID               BzMbsL-FiPz-oVZO-j6o1-Du67-N094-JIbdPB
   
   #查看节点创建出的lv
   [root@glusterfs-node-0 ~]# lvdisplay
   
     --- Logical volume ---
     LV Name                tp_8b932b033db44bf8d6b202ec3cd9b6f3
     VG Name                vg_10a23554385cca32a01d7fc3373faf2d
     LV UUID                hesO25-OdId-1eYo-OLVc-Phe8-3Y3L-z0RcsH
     LV Write Access        read/write (activated read only)
     LV Creation host, time glusterfs-node-0, 2022-03-03 07:21:30 +0530
     LV Pool metadata       tp_8b932b033db44bf8d6b202ec3cd9b6f3_tmeta
     LV Pool data           tp_8b932b033db44bf8d6b202ec3cd9b6f3_tdata
     LV Status              available
     # open                 2
     LV Size                2.00 GiB
     Allocated pool data    0.70%
     Allocated metadata     10.32%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     256
     Block device           253:6
   
     --- Logical volume ---
     LV Path                /dev/vg_10a23554385cca32a01d7fc3373faf2d/brick_8b932b033db44bf8d6b202ec3cd9b6f3
     LV Name                brick_8b932b033db44bf8d6b202ec3cd9b6f3
     VG Name                vg_10a23554385cca32a01d7fc3373faf2d
     LV UUID                VpkvpB-cmQo-jYgf-g83G-XA8C-1yxG-gAXksl
     LV Write Access        read/write
     LV Creation host, time glusterfs-node-0, 2022-03-03 07:21:30 +0530
     LV Pool name           tp_8b932b033db44bf8d6b202ec3cd9b6f3
     LV Status              available
     # open                 1
     LV Size                2.00 GiB
     Mapped size            0.70%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     256
     Block device           253:8
   
   ##节点二
   
   [root@glusterfs-node-1 ~]# pvdisplay
     --- Physical volume ---
     PV Name               /dev/sdb
     VG Name               vg_c74562fedbcf8ea46923918cf3ac8958
     PV Size               <2.73 TiB / not usable 130.00 MiB
     Allocatable           yes
     PE Size               4.00 MiB
     Total PE              715223
     Free PE               714705
     Allocated PE          518
     PV UUID               cyZ6Ur-HIiQ-tREG-Ueb9-ZKBK-4Wkx-Pgc7FE
   
   [root@glusterfs-node-1 ~]# lvdisplay
     --- Logical volume ---
     LV Name                tp_0524f0ec9f3ab36e11a06320c6a1afa2
     VG Name                vg_c74562fedbcf8ea46923918cf3ac8958
     LV UUID                jpG0fi-SvNf-Tufz-IEn8-VSBi-ChTf-t54IEN
     LV Write Access        read/write (activated read only)
     LV Creation host, time glusterfs-node-1, 2022-03-03 07:20:45 +0530
     LV Pool metadata       tp_0524f0ec9f3ab36e11a06320c6a1afa2_tmeta
     LV Pool data           tp_0524f0ec9f3ab36e11a06320c6a1afa2_tdata
     LV Status              available
     # open                 2
     LV Size                2.00 GiB
     Allocated pool data    0.70%
     Allocated metadata     10.32%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     256
     Block device           253:6
   
     --- Logical volume ---
     LV Path                /dev/vg_c74562fedbcf8ea46923918cf3ac8958/brick_7bb72d28b432248bcf9de23346d9ea3d
     LV Name                brick_7bb72d28b432248bcf9de23346d9ea3d
     VG Name                vg_c74562fedbcf8ea46923918cf3ac8958
     LV UUID                gsvc2l-JDHA-0L0e-oeNE-N0hY-p1Zd-aGAXo4
     LV Write Access        read/write
     LV Creation host, time glusterfs-node-1, 2022-03-03 07:20:45 +0530
     LV Pool name           tp_0524f0ec9f3ab36e11a06320c6a1afa2
     LV Status              available
     # open                 1
     LV Size                2.00 GiB
     Mapped size            0.70%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     256
     Block device           253:8
   
   [root@glusterfs-node-1 ~]# vgdisplay
     --- Volume group ---
     VG Name               vg_c74562fedbcf8ea46923918cf3ac8958
     System ID
     Format                lvm2
     Metadata Areas        1
     Metadata Sequence No  22
     VG Access             read/write
     VG Status             resizable
     MAX LV                0
     Cur LV                2
     Open LV               1
     Max PV                0
     Cur PV                1
     Act PV                1
     VG Size               <2.73 TiB
     PE Size               4.00 MiB
     Total PE              715223
     Alloc PE / Size       518 / 2.02 GiB
     Free  PE / Size       714705 / <2.73 TiB
     VG UUID               ImSA9l-xtcY-q1ol-ERmA-kDtc-xGs5-6MIgNW
   
   ##节点三
   
   [root@glusterfs-node-2 ~]# pvdisplay
     --- Physical volume ---
     PV Name               /dev/sdb
     VG Name               vg_96fce8c076ca08f24c256e12e8cbbde0
     PV Size               <2.73 TiB / not usable 130.00 MiB
     Allocatable           yes
     PE Size               4.00 MiB
     Total PE              715223
     Free PE               714705
     Allocated PE          518
     PV UUID               cA2GW7-1EoO-w00U-8OKT-qZtF-J7xK-dRaqwd
   
   [root@glusterfs-node-2 ~]# vgdisplay
     --- Volume group ---
     VG Name               vg_96fce8c076ca08f24c256e12e8cbbde0
     System ID
     Format                lvm2
     Metadata Areas        1
     Metadata Sequence No  22
     VG Access             read/write
     VG Status             resizable
     MAX LV                0
     Cur LV                2
     Open LV               1
     Max PV                0
     Cur PV                1
     Act PV                1
     VG Size               <2.73 TiB
     PE Size               4.00 MiB
     Total PE              715223
     Alloc PE / Size       518 / 2.02 GiB
     Free  PE / Size       714705 / <2.73 TiB
     VG UUID               PbEPjn-8zn6-wQjF-1Uvq-qaTv-aQ1J-BJFtZu
   
   [root@glusterfs-node-2 ~]# lvdisplay
     --- Logical volume ---
     LV Name                tp_be0ace1a68127b0537199b654c521b2b
     VG Name                vg_96fce8c076ca08f24c256e12e8cbbde0
     LV UUID                nETAXU-HVYn-JZHq-afwh-2W74-YMZ4-m26xyw
     LV Write Access        read/write (activated read only)
     LV Creation host, time glusterfs-node-2, 2022-03-03 09:51:14 +0800
     LV Pool metadata       tp_be0ace1a68127b0537199b654c521b2b_tmeta
     LV Pool data           tp_be0ace1a68127b0537199b654c521b2b_tdata
     LV Status              available
     # open                 2
     LV Size                2.00 GiB
     Allocated pool data    0.70%
     Allocated metadata     10.32%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     256
     Block device           253:6
   
     --- Logical volume ---
     LV Path                /dev/vg_96fce8c076ca08f24c256e12e8cbbde0/brick_be0ace1a68127b0537199b654c521b2b
     LV Name                brick_be0ace1a68127b0537199b654c521b2b
     VG Name                vg_96fce8c076ca08f24c256e12e8cbbde0
     LV UUID                yXhTeX-4r7D-5rcE-rqKw-8NZb-z158-fchMLq
     LV Write Access        read/write
     LV Creation host, time glusterfs-node-2, 2022-03-03 09:51:15 +0800
     LV Pool name           tp_be0ace1a68127b0537199b654c521b2b
     LV Status              available
     # open                 1
     LV Size                2.00 GiB
     Mapped size            0.70%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     256
     Block device           253:8
   ```

4. 删除卷
   
   ```shell
   [root@glusterfs-node-0 heketi]# heketi-cli volume delete 951523a39e6d926764984279dbd7e4bc
   ```

## k8s集群的调用（两种方式）

### 命令行手动配置(方式一)

1. 所有的k8s节点均安装glusterfs客户端，以保证正常挂载

```shell
[root@k8s-master-0 ~]# yum install glusterfs-fuse -y
```

2. 创建secret，heketi的认证密码,或者在kebesphere平台添加密钥,key的值,使用base64转码生成`echo -n "admin@P@ssW0rd" | base64` 这里的用户名密码为heketi配置文件中创建的用户密码

```shell
[root@k8s-master-1 ~]# vi heketi_secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
  namespace: kube-system
data:
  key: YWRtaW5AUEBzc1cwcmQ=
type: kubernetes.io/glusterfs
[root@k8s-master-1 ~]# kubectl apply -f heketi_secret.yaml
```

3. 创建storageclass

```shell
[root@k8s-master-1 ~]# vi heketi-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs
  namespace: kube-system
parameters:
  resturl: "http://192.168.10.1:48080"     #heketi的地址
  clusterid: "b89a07c2c6e8c533322591bf2a4aa613"  #在heketi节点使用heketi-cli cluster list命令返回的集群id
  restauthenabled: "true" 
  restuser: "admin" #heketi配置文件中创建的用户密
  secretName: "heketi-secret" #与secret资源中定义一致
  secretNamespace: "kube-system" #与secret资源中定义一致
  volumetype: "replicate:3" 
provisioner: kubernetes.io/glusterfs
reclaimPolicy: Delete
[root@k8s-master-1 ~]# kubectl apply -f heketi-storageclass.yaml
```

4. 创建pvc测试

```shell
[root@k8s-master-1 ~]# vi heketi-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: heketi-pvc
  annotations:
    volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/glusterfs
spec:
  storageClassName: "glusterfs"
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
[root@k8s-master-1 ~]# kubectl apply -f heketi-pvc.yaml
```

5. 查看sc和pvc的信息

```shell
[root@k8s-master-1 ~]# kubectl get sc
[root@k8s-master-1 ~]# kubectl get pvc
```

6. 创建Pod挂载pvc

```yaml
[root@k8s-master-1 ~]# vi heketi-pod.yaml
kind: Pod
apiVersion: v1
metadata:
  name: heketi-pod
spec:
  containers:
  - name: heketi-container
    image: busybox
    command:
    - sleep
    - "3600"
    volumeMounts:
    - name: heketi-volume
      mountPath: "/pv-data"
      readOnly: false
  volumes:
  - name: heketi-volume
    persistentVolumeClaim:
      claimName: heketi-pvc
[root@k8s-master-1 ~]# kubectl apply -f  heketi-pod.yaml
```

7. 查看POD状态,若状态为runing，则挂载成功

```shell
[root@k8s-master-1 ~]# kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
heketi-pod                         1/1     Running   0          4s

#清理测试pod
[root@k8s-master-1 ~]# kubectl delete -f  heketi-pod.yaml
```

### kubesphere界面配置(方式二)

1. 创建密钥
- 平台管理->配置中心->密钥->创建

- 名称：heketi-secret

- 项目：kube-system

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/kubesphere-glusterfs-0.jpg)

- 类型选择默认

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/kubesphere-glusterfs-1.jpg)

- 键：key

- 值：YWRtaW5AUEBzc1cwcmQ=

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/kubesphere-glusterfs-2.jpg)

2. 创建存储类型
- 存储名称：glusterfs

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/kubesphere-glusterfs-3.jpg)

- 存储类型：glusterfs

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/kubesphere-glusterfs-4.jpg)

- 创建存储类型的详细信息
  
  - 允许    存储卷扩容：是
  
  - 支持的访问模式：ReadWriteOnce、ReadOnlyMany、ReadWriteMany
  
  - 存储系统：kubernetes.io/glusterfs
  
  - resturl：192.168.10.1:48080 （heketi的访问地址）
  
  - clusterid：b89a07c2c6e8c533322591bf2a4aa613（heketi-cli cluster list命令返回的集群id ）
  
  - restauthenabled：true
  
  - restuser：admin (heketi服务的admin用户)
  
  - secretNamespace：kube-system
  
  - secretName：heketi-secret （上方创建的secert名称）
  
  - volumetype：replicate:3

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/kubesphere-glusterfs-5.jpg)

- 查看存储类型是否创建成功

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/kubesphere-glusterfs-6.jpg)

3. 创建存储卷进行测试
- 名称：test （可自定义）

- 项目：kube-system （可自定义）

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/kubesphere-glusterfs-7.jpg)

- 存储类型选择glusterfs

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/kubesphere-glusterfs-8.jpg)

4. 状态为准备就绪即为成功

![](https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/kubesphere-glusterfs-9.jpg)

## 参考文档

1. [GlusterFS](https://www.gluster.org)

2. https://github.com/heketi/heketi

## 后续

1. 基于KubeSphere的Kubernetes生产实践之路-SpingCloud架构依赖的中间件的安装部署