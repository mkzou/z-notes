# 基于 KubeSphere 玩转 k8s-GlusterFS 扩容手记

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

本文来源于生产环境实际需求，生产环境部署时只挂载了一块 1T 的数据盘，随着业务量的上涨，存储空间分配完毕，需要增加磁盘，因此有了本文。

> 本文知识量

- 阅读时长：7 分
- 行：800+
- 单词：2600+
- 字符：35200+
- 图片：0 张

> **本文知识点**

- 定级：**入门级**
- Heketi 的常用操作
- 利用 Heketi 给 GlusterFS 增加磁盘的方法

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

> **演示环境涉及软件版本信息**

- 操作系统：**CentOS-7.9-x86_64**
- KubeSphere：**3.2.1**
- Kubernetes：**1.21.5**
- GlusterFS：**9.5** 
- Ansible：**2.8.20**

## 2. 查看现有存储集群信息

### 2.1. Topology 信息

```shell
[root@glusterfs-node-0 ~]# heketi-cli topology info

Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f

    File:  true
    Block: true

    Volumes:

        Name: vol_744e76a230868653857bd44d17ec350c
        Size: 90
        Id: 744e76a230868653857bd44d17ec350c
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Mount: 192.168.9.95:vol_744e76a230868653857bd44d17ec350c
        Mount Options: backup-volfile-servers=192.168.9.97,192.168.9.96
        Durability Type: replicate
        Replica: 3
        Snapshot: Enabled
        Snapshot Factor: 1.00

                Bricks:
                        Id: a95e3ef850140a5561b075b9a3cb9728
                        Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_a95e3ef850140a5561b075b9a3cb9728/brick
                        Size (GiB): 90
                        Node: 0ece0d8cc9e3b69dd6d1940107cee0ef
                        Device: 4d81fdb2e784633d56a5dadf74cdb2df

                        Id: aeec9456e6b171d5f8f839f4fb47dad2
                        Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_aeec9456e6b171d5f8f839f4fb47dad2/brick
                        Size (GiB): 90
                        Node: 6a2b14ba0b802a9a0cd6981639a314e2
                        Device: da3a8e59a6667fc54cea25d5f32bdb73

                        Id: c9a9234da08003a5b23aeba06ed1c39b
                        Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_c9a9234da08003a5b23aeba06ed1c39b/brick
                        Size (GiB): 90
                        Node: 7acfa91bd0fd96a2f13aef3ff816a75e
                        Device: ad739f5de43e2139d95677cda807a71f


        Name: vol_751d2c23c1dd25265a97967aaa7c0a97
        Size: 5
        Id: 751d2c23c1dd25265a97967aaa7c0a97
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Mount: 192.168.9.95:vol_751d2c23c1dd25265a97967aaa7c0a97
        Mount Options: backup-volfile-servers=192.168.9.97,192.168.9.96
        Durability Type: replicate
        Replica: 3
        Snapshot: Enabled
        Snapshot Factor: 1.00

                Bricks:
                        Id: 2d8465c9a80faf1353b511d0da5651d2
                        Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_2d8465c9a80faf1353b511d0da5651d2/brick
                        Size (GiB): 5
                        Node: 6a2b14ba0b802a9a0cd6981639a314e2
                        Device: da3a8e59a6667fc54cea25d5f32bdb73

                        Id: abb90d3dd295f6ce4b9f559d0592cbef
                        Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_abb90d3dd295f6ce4b9f559d0592cbef/brick
                        Size (GiB): 5
                        Node: 7acfa91bd0fd96a2f13aef3ff816a75e
                        Device: ad739f5de43e2139d95677cda807a71f

                        Id: c737738b593a82793e8695572f7cff07
                        Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_c737738b593a82793e8695572f7cff07/brick
                        Size (GiB): 5
                        Node: 0ece0d8cc9e3b69dd6d1940107cee0ef
                        Device: 4d81fdb2e784633d56a5dadf74cdb2df



    Nodes:

        Node Id: 0ece0d8cc9e3b69dd6d1940107cee0ef
        State: online
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Zone: 1
        Management Hostnames: 192.168.9.95
        Storage Hostnames: 192.168.9.95
        Devices:
                Id:4d81fdb2e784633d56a5dadf74cdb2df   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       
                        Bricks:
                                Id:a95e3ef850140a5561b075b9a3cb9728   Size (GiB):90      Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_a95e3ef850140a5561b075b9a3cb9728/brick
                                Id:c737738b593a82793e8695572f7cff07   Size (GiB):5       Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_c737738b593a82793e8695572f7cff07/brick

        Node Id: 6a2b14ba0b802a9a0cd6981639a314e2
        State: online
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Zone: 1
        Management Hostnames: 192.168.9.97
        Storage Hostnames: 192.168.9.97
        Devices:
                Id:da3a8e59a6667fc54cea25d5f32bdb73   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       
                        Bricks:
                                Id:2d8465c9a80faf1353b511d0da5651d2   Size (GiB):5       Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_2d8465c9a80faf1353b511d0da5651d2/brick
                                Id:aeec9456e6b171d5f8f839f4fb47dad2   Size (GiB):90      Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_aeec9456e6b171d5f8f839f4fb47dad2/brick

        Node Id: 7acfa91bd0fd96a2f13aef3ff816a75e
        State: online
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Zone: 1
        Management Hostnames: 192.168.9.96
        Storage Hostnames: 192.168.9.96
        Devices:
                Id:ad739f5de43e2139d95677cda807a71f   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       
                        Bricks:
                                Id:abb90d3dd295f6ce4b9f559d0592cbef   Size (GiB):5       Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_abb90d3dd295f6ce4b9f559d0592cbef/brick
                                Id:c9a9234da08003a5b23aeba06ed1c39b   Size (GiB):90      Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_c9a9234da08003a5b23aeba06ed1c39b/brick

```

### 2.2. 集群信息

```shell
[root@glusterfs-node-0 ~]# heketi-cli cluster list
Clusters:
Id:94fd8a9991e4d7fd6792d0408d28033f [file][block]
[root@glusterfs-node-0 ~]# heketi-cli cluster info 94fd8a9991e4d7fd6792d0408d28033f
Cluster id: 94fd8a9991e4d7fd6792d0408d28033f
Nodes:
0ece0d8cc9e3b69dd6d1940107cee0ef
6a2b14ba0b802a9a0cd6981639a314e2
7acfa91bd0fd96a2f13aef3ff816a75e
Volumes:
751d2c23c1dd25265a97967aaa7c0a97
Block: true

File: true
```

### 2.3. Node 信息

```shell
[root@glusterfs-node-0 ~]# heketi-cli node list 
Id:0ece0d8cc9e3b69dd6d1940107cee0ef     Cluster:94fd8a9991e4d7fd6792d0408d28033f
Id:6a2b14ba0b802a9a0cd6981639a314e2     Cluster:94fd8a9991e4d7fd6792d0408d28033f
Id:7acfa91bd0fd96a2f13aef3ff816a75e     Cluster:94fd8a9991e4d7fd6792d0408d28033f

[root@glusterfs-node-0 ~]# heketi-cli node info 0ece0d8cc9e3b69dd6d1940107cee0ef
Node Id: 0ece0d8cc9e3b69dd6d1940107cee0ef
State: online
Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
Zone: 1
Management Hostname: 192.168.9.95
Storage Hostname: 192.168.9.95
Devices:
Id:4d81fdb2e784633d56a5dadf74cdb2df   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       Bricks:2       
[root@glusterfs-node-0 ~]# heketi-cli node info 6a2b14ba0b802a9a0cd6981639a314e2
Node Id: 6a2b14ba0b802a9a0cd6981639a314e2
State: online
Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
Zone: 1
Management Hostname: 192.168.9.97
Storage Hostname: 192.168.9.97
Devices:
Id:da3a8e59a6667fc54cea25d5f32bdb73   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       Bricks:2       
[root@glusterfs-node-0 ~]# heketi-cli node info 7acfa91bd0fd96a2f13aef3ff816a75e
Node Id: 7acfa91bd0fd96a2f13aef3ff816a75e
State: online
Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
Zone: 1
Management Hostname: 192.168.9.96
Storage Hostname: 192.168.9.96
Devices:
Id:ad739f5de43e2139d95677cda807a71f   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       Bricks:2
```

### 2.4. VG 信息

```shell
[root@glusterfs-node-0 heketi]# vgs
  VG                                  #PV #LV #SN Attr   VSize   VFree 
  centos                                1   2   0 wz--n- <39.00g  4.00m
  vg_4d81fdb2e784633d56a5dadf74cdb2df   1   4   0 wz--n-  99.87g <3.94g
```

### 2.5. LV 信息

```shell
[root@glusterfs-node-0 heketi]# lvs
  LV                                     VG                                  Attr       LSize  Pool                                Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root                                   centos                              -wi-ao---- 36.99g                                                                                   
  swap                                   centos                              -wi-ao----  2.00g                                                                                   
  brick_a95e3ef850140a5561b075b9a3cb9728 vg_4d81fdb2e784633d56a5dadf74cdb2df Vwi-aotz-- 90.00g tp_a95e3ef850140a5561b075b9a3cb9728        0.29                                   
  brick_c737738b593a82793e8695572f7cff07 vg_4d81fdb2e784633d56a5dadf74cdb2df Vwi-aotz--  5.00g tp_c737738b593a82793e8695572f7cff07        4.38                                   
  tp_a95e3ef850140a5561b075b9a3cb9728    vg_4d81fdb2e784633d56a5dadf74cdb2df twi-aotz-- 90.00g                                            0.29   3.49                            
  tp_c737738b593a82793e8695572f7cff07    vg_4d81fdb2e784633d56a5dadf74cdb2df twi-aotz--  5.00g                                            4.38   10.23  
```

## 3. 扩容方案之调整 Topology 配置文件

- 扩容盘符：/dev/sdc
- 扩容容量：200G

### 3.1. 现有 topology.json 配置文件

```json
{
    "clusters": [
        {
            "nodes": [
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.9.95"
                            ],
                            "storage": [
                                "192.168.9.95"
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
                                "192.168.9.96"
                            ],
                            "storage": [
                                "192.168.9.96"
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
                                "192.168.9.97"
                            ],
                            "storage": [
                                "192.168.9.97"
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

### 3.2. 修改 Topology 文件

在每一个 node 的 devices 的配置下面增加 **/dev/sdc**，注意 **/dev/sdb** 后面的标点配置。

```json
{
    "clusters": [
        {
            "nodes": [
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.9.95"
                            ],
                            "storage": [
                                "192.168.9.95"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb",
                        "/dev/sdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.9.96"
                            ],
                            "storage": [
                                "192.168.9.96"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb",
                        "/dev/sdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.9.97"
                            ],
                            "storage": [
                                "192.168.9.97"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb",
                        "/dev/sdc"
                    ]
                }
            ]
        }
    ]
}
```

### 3.3 重新加载 Topology

```shell
[root@glusterfs-node-0 heketi]# heketi-cli topology load --json=/etc/heketi/topology.json 
        Found node 192.168.9.95 on cluster 94fd8a9991e4d7fd6792d0408d28033f
                Found device /dev/sdb
                Adding device /dev/sdc ... OK
        Found node 192.168.9.96 on cluster 94fd8a9991e4d7fd6792d0408d28033f
                Found device /dev/sdb
                Adding device /dev/sdc ... OK
        Found node 192.168.9.97 on cluster 94fd8a9991e4d7fd6792d0408d28033f
                Found device /dev/sdb
                Adding device /dev/sdc ... OK
```

### 3.4. 查看更新后的 Topology 信息

```shell
[root@glusterfs-node-0 heketi]# heketi-cli topology info

Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f

    File:  true
    Block: true

    Volumes:

        Name: vol_744e76a230868653857bd44d17ec350c
        Size: 90
        Id: 744e76a230868653857bd44d17ec350c
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Mount: 192.168.9.95:vol_744e76a230868653857bd44d17ec350c
        Mount Options: backup-volfile-servers=192.168.9.97,192.168.9.96
        Durability Type: replicate
        Replica: 3
        Snapshot: Enabled
        Snapshot Factor: 1.00

                Bricks:
                        Id: a95e3ef850140a5561b075b9a3cb9728
                        Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_a95e3ef850140a5561b075b9a3cb9728/brick
                        Size (GiB): 90
                        Node: 0ece0d8cc9e3b69dd6d1940107cee0ef
                        Device: 4d81fdb2e784633d56a5dadf74cdb2df

                        Id: aeec9456e6b171d5f8f839f4fb47dad2
                        Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_aeec9456e6b171d5f8f839f4fb47dad2/brick
                        Size (GiB): 90
                        Node: 6a2b14ba0b802a9a0cd6981639a314e2
                        Device: da3a8e59a6667fc54cea25d5f32bdb73

                        Id: c9a9234da08003a5b23aeba06ed1c39b
                        Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_c9a9234da08003a5b23aeba06ed1c39b/brick
                        Size (GiB): 90
                        Node: 7acfa91bd0fd96a2f13aef3ff816a75e
                        Device: ad739f5de43e2139d95677cda807a71f


        Name: vol_751d2c23c1dd25265a97967aaa7c0a97
        Size: 5
        Id: 751d2c23c1dd25265a97967aaa7c0a97
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Mount: 192.168.9.95:vol_751d2c23c1dd25265a97967aaa7c0a97
        Mount Options: backup-volfile-servers=192.168.9.97,192.168.9.96
        Durability Type: replicate
        Replica: 3
        Snapshot: Enabled
        Snapshot Factor: 1.00

                Bricks:
                        Id: 2d8465c9a80faf1353b511d0da5651d2
                        Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_2d8465c9a80faf1353b511d0da5651d2/brick
                        Size (GiB): 5
                        Node: 6a2b14ba0b802a9a0cd6981639a314e2
                        Device: da3a8e59a6667fc54cea25d5f32bdb73

                        Id: abb90d3dd295f6ce4b9f559d0592cbef
                        Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_abb90d3dd295f6ce4b9f559d0592cbef/brick
                        Size (GiB): 5
                        Node: 7acfa91bd0fd96a2f13aef3ff816a75e
                        Device: ad739f5de43e2139d95677cda807a71f

                        Id: c737738b593a82793e8695572f7cff07
                        Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_c737738b593a82793e8695572f7cff07/brick
                        Size (GiB): 5
                        Node: 0ece0d8cc9e3b69dd6d1940107cee0ef
                        Device: 4d81fdb2e784633d56a5dadf74cdb2df



    Nodes:

        Node Id: 0ece0d8cc9e3b69dd6d1940107cee0ef
        State: online
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Zone: 1
        Management Hostnames: 192.168.9.95
        Storage Hostnames: 192.168.9.95
        Devices:
                Id:3401fa3979682be3d6d29b991a9a46bc   Name:/dev/sdc            State:online    Size (GiB):199     Used (GiB):0       Free (GiB):199     
                        Bricks:
                Id:4d81fdb2e784633d56a5dadf74cdb2df   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       
                        Bricks:
                                Id:a95e3ef850140a5561b075b9a3cb9728   Size (GiB):90      Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_a95e3ef850140a5561b075b9a3cb9728/brick
                                Id:c737738b593a82793e8695572f7cff07   Size (GiB):5       Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_c737738b593a82793e8695572f7cff07/brick

        Node Id: 6a2b14ba0b802a9a0cd6981639a314e2
        State: online
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Zone: 1
        Management Hostnames: 192.168.9.97
        Storage Hostnames: 192.168.9.97
        Devices:
                Id:aa2e7f0cac4c3c4eecb6de2ce3c77165   Name:/dev/sdc            State:online    Size (GiB):199     Used (GiB):0       Free (GiB):199     
                        Bricks:
                Id:da3a8e59a6667fc54cea25d5f32bdb73   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       
                        Bricks:
                                Id:2d8465c9a80faf1353b511d0da5651d2   Size (GiB):5       Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_2d8465c9a80faf1353b511d0da5651d2/brick
                                Id:aeec9456e6b171d5f8f839f4fb47dad2   Size (GiB):90      Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_aeec9456e6b171d5f8f839f4fb47dad2/brick

        Node Id: 7acfa91bd0fd96a2f13aef3ff816a75e
        State: online
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Zone: 1
        Management Hostnames: 192.168.9.96
        Storage Hostnames: 192.168.9.96
        Devices:
                Id:806243bc466118e9eff0d8b08133cafc   Name:/dev/sdc            State:online    Size (GiB):199     Used (GiB):0       Free (GiB):199     
                        Bricks:
                Id:ad739f5de43e2139d95677cda807a71f   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       
                        Bricks:
                                Id:abb90d3dd295f6ce4b9f559d0592cbef   Size (GiB):5       Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_abb90d3dd295f6ce4b9f559d0592cbef/brick
                                Id:c9a9234da08003a5b23aeba06ed1c39b   Size (GiB):90      Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_c9a9234da08003a5b23aeba06ed1c39b/brick
```

### 3.5. 查看更新后的 Node 信息

```shell
[root@glusterfs-node-0 heketi]# heketi-cli node list
Id:0ece0d8cc9e3b69dd6d1940107cee0ef     Cluster:94fd8a9991e4d7fd6792d0408d28033f
Id:6a2b14ba0b802a9a0cd6981639a314e2     Cluster:94fd8a9991e4d7fd6792d0408d28033f
Id:7acfa91bd0fd96a2f13aef3ff816a75e     Cluster:94fd8a9991e4d7fd6792d0408d28033f

[root@glusterfs-node-0 heketi]# heketi-cli node info 0ece0d8cc9e3b69dd6d1940107cee0ef
Node Id: 0ece0d8cc9e3b69dd6d1940107cee0ef
State: online
Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
Zone: 1
Management Hostname: 192.168.9.95
Storage Hostname: 192.168.9.95
Devices:
Id:3401fa3979682be3d6d29b991a9a46bc   Name:/dev/sdc            State:online    Size (GiB):199     Used (GiB):0       Free (GiB):199     Bricks:0       
Id:4d81fdb2e784633d56a5dadf74cdb2df   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       Bricks:2       
[root@glusterfs-node-0 heketi]# heketi-cli node info 6a2b14ba0b802a9a0cd6981639a314e2
Node Id: 6a2b14ba0b802a9a0cd6981639a314e2
State: online
Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
Zone: 1
Management Hostname: 192.168.9.97
Storage Hostname: 192.168.9.97
Devices:
Id:aa2e7f0cac4c3c4eecb6de2ce3c77165   Name:/dev/sdc            State:online    Size (GiB):199     Used (GiB):0       Free (GiB):199     Bricks:0       
Id:da3a8e59a6667fc54cea25d5f32bdb73   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       Bricks:2       
[root@glusterfs-node-0 heketi]# heketi-cli node info 7acfa91bd0fd96a2f13aef3ff816a75e
Node Id: 7acfa91bd0fd96a2f13aef3ff816a75e
State: online
Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
Zone: 1
Management Hostname: 192.168.9.96
Storage Hostname: 192.168.9.96
Devices:
Id:806243bc466118e9eff0d8b08133cafc   Name:/dev/sdc            State:online    Size (GiB):199     Used (GiB):0       Free (GiB):199     Bricks:0       
Id:ad739f5de43e2139d95677cda807a71f   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       Bricks:2  
```

### 3.6. 查看更新后的 VG 信息

```shell
[root@glusterfs-node-0 heketi]# vgs
  VG                                  #PV #LV #SN Attr   VSize   VFree  
  centos                                1   2   0 wz--n- <39.00g   4.00m
  vg_3401fa3979682be3d6d29b991a9a46bc   1   0   0 wz--n- 199.87g 199.87g
  vg_4d81fdb2e784633d56a5dadf74cdb2df   1   4   0 wz--n-  99.87g  <3.94g
```

## 4. 扩容方案之 heketi cli

### 4.1. 添加 Device

```shell
[root@glusterfs-node-0 heketi]# heketi-cli device add --name /dev/sdc --node 0ece0d8cc9e3b69dd6d1940107cee0ef
Device added successfully
[root@glusterfs-node-0 heketi]# heketi-cli device add --name /dev/sdc --node 6a2b14ba0b802a9a0cd6981639a314e2
Device added successfully
[root@glusterfs-node-0 heketi]# heketi-cli device add --name /dev/sdc --node 7acfa91bd0fd96a2f13aef3ff816a75e
Device added successfully
```

### 4.2. 查看更新后的 Topology 信息

```shell
heketi-cli topology info
```

```shell
[root@glusterfs-node-0 heketi]# heketi-cli topology info

Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f

    File:  true
    Block: true

    Volumes:

        Name: vol_744e76a230868653857bd44d17ec350c
        Size: 90
        Id: 744e76a230868653857bd44d17ec350c
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Mount: 192.168.9.95:vol_744e76a230868653857bd44d17ec350c
        Mount Options: backup-volfile-servers=192.168.9.97,192.168.9.96
        Durability Type: replicate
        Replica: 3
        Snapshot: Enabled
        Snapshot Factor: 1.00

                Bricks:
                        Id: a95e3ef850140a5561b075b9a3cb9728
                        Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_a95e3ef850140a5561b075b9a3cb9728/brick
                        Size (GiB): 90
                        Node: 0ece0d8cc9e3b69dd6d1940107cee0ef
                        Device: 4d81fdb2e784633d56a5dadf74cdb2df

                        Id: aeec9456e6b171d5f8f839f4fb47dad2
                        Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_aeec9456e6b171d5f8f839f4fb47dad2/brick
                        Size (GiB): 90
                        Node: 6a2b14ba0b802a9a0cd6981639a314e2
                        Device: da3a8e59a6667fc54cea25d5f32bdb73

                        Id: c9a9234da08003a5b23aeba06ed1c39b
                        Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_c9a9234da08003a5b23aeba06ed1c39b/brick
                        Size (GiB): 90
                        Node: 7acfa91bd0fd96a2f13aef3ff816a75e
                        Device: ad739f5de43e2139d95677cda807a71f


        Name: vol_751d2c23c1dd25265a97967aaa7c0a97
        Size: 5
        Id: 751d2c23c1dd25265a97967aaa7c0a97
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Mount: 192.168.9.95:vol_751d2c23c1dd25265a97967aaa7c0a97
        Mount Options: backup-volfile-servers=192.168.9.97,192.168.9.96
        Durability Type: replicate
        Replica: 3
        Snapshot: Enabled
        Snapshot Factor: 1.00

                Bricks:
                        Id: 2d8465c9a80faf1353b511d0da5651d2
                        Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_2d8465c9a80faf1353b511d0da5651d2/brick
                        Size (GiB): 5
                        Node: 6a2b14ba0b802a9a0cd6981639a314e2
                        Device: da3a8e59a6667fc54cea25d5f32bdb73

                        Id: abb90d3dd295f6ce4b9f559d0592cbef
                        Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_abb90d3dd295f6ce4b9f559d0592cbef/brick
                        Size (GiB): 5
                        Node: 7acfa91bd0fd96a2f13aef3ff816a75e
                        Device: ad739f5de43e2139d95677cda807a71f

                        Id: c737738b593a82793e8695572f7cff07
                        Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_c737738b593a82793e8695572f7cff07/brick
                        Size (GiB): 5
                        Node: 0ece0d8cc9e3b69dd6d1940107cee0ef
                        Device: 4d81fdb2e784633d56a5dadf74cdb2df



    Nodes:

        Node Id: 0ece0d8cc9e3b69dd6d1940107cee0ef
        State: online
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Zone: 1
        Management Hostnames: 192.168.9.95
        Storage Hostnames: 192.168.9.95
        Devices:
                Id:4d81fdb2e784633d56a5dadf74cdb2df   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       
                        Bricks:
                                Id:a95e3ef850140a5561b075b9a3cb9728   Size (GiB):90      Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_a95e3ef850140a5561b075b9a3cb9728/brick
                                Id:c737738b593a82793e8695572f7cff07   Size (GiB):5       Path: /var/lib/heketi/mounts/vg_4d81fdb2e784633d56a5dadf74cdb2df/brick_c737738b593a82793e8695572f7cff07/brick
                Id:6f060e69953f5b26fb025c174b535505   Name:/dev/sdc            State:online    Size (GiB):199     Used (GiB):0       Free (GiB):199     
                        Bricks:

        Node Id: 6a2b14ba0b802a9a0cd6981639a314e2
        State: online
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Zone: 1
        Management Hostnames: 192.168.9.97
        Storage Hostnames: 192.168.9.97
        Devices:
                Id:0c1e296976ac6bc2b1dadf58fb36cce3   Name:/dev/sdc            State:online    Size (GiB):199     Used (GiB):0       Free (GiB):199     
                        Bricks:
                Id:da3a8e59a6667fc54cea25d5f32bdb73   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       
                        Bricks:
                                Id:2d8465c9a80faf1353b511d0da5651d2   Size (GiB):5       Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_2d8465c9a80faf1353b511d0da5651d2/brick
                                Id:aeec9456e6b171d5f8f839f4fb47dad2   Size (GiB):90      Path: /var/lib/heketi/mounts/vg_da3a8e59a6667fc54cea25d5f32bdb73/brick_aeec9456e6b171d5f8f839f4fb47dad2/brick

        Node Id: 7acfa91bd0fd96a2f13aef3ff816a75e
        State: online
        Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
        Zone: 1
        Management Hostnames: 192.168.9.96
        Storage Hostnames: 192.168.9.96
        Devices:
                Id:7b1154491d4139e3749a3a4b145fac5c   Name:/dev/sdc            State:online    Size (GiB):199     Used (GiB):0       Free (GiB):199     
                        Bricks:
                Id:ad739f5de43e2139d95677cda807a71f   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       
                        Bricks:
                                Id:abb90d3dd295f6ce4b9f559d0592cbef   Size (GiB):5       Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_abb90d3dd295f6ce4b9f559d0592cbef/brick
                                Id:c9a9234da08003a5b23aeba06ed1c39b   Size (GiB):90      Path: /var/lib/heketi/mounts/vg_ad739f5de43e2139d95677cda807a71f/brick_c9a9234da08003a5b23aeba06ed1c39b/brick
```

### 4.3. 查看更新后的 Node 信息

```shell
[root@glusterfs-node-0 heketi]# heketi-cli node info 0ece0d8cc9e3b69dd6d1940107cee0ef
Node Id: 0ece0d8cc9e3b69dd6d1940107cee0ef
State: online
Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
Zone: 1
Management Hostname: 192.168.9.95
Storage Hostname: 192.168.9.95
Devices:
Id:4d81fdb2e784633d56a5dadf74cdb2df   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       Bricks:2       
Id:6f060e69953f5b26fb025c174b535505   Name:/dev/sdc            State:online    Size (GiB):199     Used (GiB):0       Free (GiB):199     Bricks:0  
```

### 4.4. 查看更新后的 VG 信息

```shell
[root@glusterfs-node-0 heketi]# vgs
  VG                                  #PV #LV #SN Attr   VSize   VFree  
  centos                                1   2   0 wz--n- <39.00g   4.00m
  vg_4d81fdb2e784633d56a5dadf74cdb2df   1   4   0 wz--n-  99.87g  <3.94g
  vg_6f060e69953f5b26fb025c174b535505   1   0   0 wz--n- 199.87g 199.87g
```

### 4.5.  查看更新后的 LV 信息

**自行在 k8s 创建存储卷后再查看**

```shell
[root@glusterfs-node-0 heketi]# lvs
  LV                                     VG                                  Attr       LSize  Pool                                Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root                                   centos                              -wi-ao---- 36.99g                                                                                   
  swap                                   centos                              -wi-ao----  2.00g                                                                                   
  brick_a95e3ef850140a5561b075b9a3cb9728 vg_4d81fdb2e784633d56a5dadf74cdb2df Vwi-aotz-- 90.00g tp_a95e3ef850140a5561b075b9a3cb9728        0.29                                   
  brick_c737738b593a82793e8695572f7cff07 vg_4d81fdb2e784633d56a5dadf74cdb2df Vwi-aotz--  5.00g tp_c737738b593a82793e8695572f7cff07        4.38                                   
  tp_a95e3ef850140a5561b075b9a3cb9728    vg_4d81fdb2e784633d56a5dadf74cdb2df twi-aotz-- 90.00g                                            0.29   3.49                            
  tp_c737738b593a82793e8695572f7cff07    vg_4d81fdb2e784633d56a5dadf74cdb2df twi-aotz--  5.00g                                            4.38   10.23                           
  brick_8b2110452f07729db3ecd8064a8c74c3 vg_6f060e69953f5b26fb025c174b535505 Vwi-aotz-- 20.00g tp_8b2110452f07729db3ecd8064a8c74c3        0.68                                   
  tp_8b2110452f07729db3ecd8064a8c74c3    vg_6f060e69953f5b26fb025c174b535505 twi-aotz-- 20.00g                                            0.68   10.07  
```

---

## 5. 其他常用操作

### 5.1. Device 移除

```shell
# 查找 Node 信息
[root@glusterfs-node-0 heketi]# heketi-cli node list
Id:0ece0d8cc9e3b69dd6d1940107cee0ef     Cluster:94fd8a9991e4d7fd6792d0408d28033f
Id:6a2b14ba0b802a9a0cd6981639a314e2     Cluster:94fd8a9991e4d7fd6792d0408d28033f
Id:7acfa91bd0fd96a2f13aef3ff816a75e     Cluster:94fd8a9991e4d7fd6792d0408d28033f

# 通过 Node ID 获取 Device ID
[root@glusterfs-node-0 heketi]# heketi-cli node info 7acfa91bd0fd96a2f13aef3ff816a75e 
Node Id: 7acfa91bd0fd96a2f13aef3ff816a75e
State: online
Cluster Id: 94fd8a9991e4d7fd6792d0408d28033f
Zone: 1
Management Hostname: 192.168.9.96
Storage Hostname: 192.168.9.96
Devices:
Id:806243bc466118e9eff0d8b08133cafc   Name:/dev/sdc            State:failed    Size (GiB):199     Used (GiB):0       Free (GiB):199     Bricks:0       
Id:ad739f5de43e2139d95677cda807a71f   Name:/dev/sdb            State:online    Size (GiB):99      Used (GiB):95      Free (GiB):4       Bricks:2       

# Disable Device
[root@glusterfs-node-0 heketi]# heketi-cli device disable 806243bc466118e9eff0d8b08133cafc
Device 806243bc466118e9eff0d8b08133cafc is now offline

# Remove Device
[root@glusterfs-node-0 heketi]# heketi-cli device remove 806243bc466118e9eff0d8b08133cafc
Device 806243bc466118e9eff0d8b08133cafc is now removed

# Delete Device
[root@glusterfs-node-0 heketi]# heketi-cli device delete  806243bc466118e9eff0d8b08133cafc
Device 806243bc466118e9eff0d8b08133cafc deleted
```

---

## 6. 总结

> **参考文档**

- [Heketi Github](https://github.com/heketi/heketi/blob/master/docs/admin/maintenance.md)

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
