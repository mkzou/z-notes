# 基于KubeSphere玩转k8s-ElasticSearch安装手记

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

本文接着上篇 **<<基于KubeSphere玩转k8s-KubeSphere初始化手记>>** ，继续玩转KubeSphere、玩转k8s。本文会介绍KubeSphere启用可插拔组件日志系统的安装和配置过程，由于采用了Kubernetes集群外部的ElasticSearch集群作为日志收集系统，因此，还会本文还会涉及ElasticSearch的安装配置的实践。为了让读者不仅能掌握ElasticSearch手工安装部署的技能，同时也能get到如何将手工安装部署文档转化成Ansible的Plaobooks，因此本文同时介绍了纯手工安装配置ElasticSearch和利用Ansible自动化安装配置ElasticSearch。

> **本文知识点**

- 定级：**入门级**
- ElasticSearch-手工安装配置
- ElasticSearch-Ansible自动安装配置
- ElasticSearch-启用http认证配置
- KubeSphere启用可插拔日志组件
- KubeSphere对接外部不开启认证的ElasticSearch
- KubeSphere对接外部开启认证的ElasticSearch

> **演示服务器配置**

|      主机名      |      IP      | CPU  | 内存 | 系统盘 | 数据盘  |               用途               |
| :--------------: | :----------: | :--: | :--: | :----: | :-----: | :------------------------------: |
|  zdeops-master   | 192.168.9.9  |  2   |  4   |   40   |   200   |       Ansible运维控制节点        |
| ks-k8s-master-0  | 192.168.9.91 |  8   |  32  |   40   |   200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-1  | 192.168.9.92 |  8   |  32  |   40   |   200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-2  | 192.168.9.93 |  8   |  32  |   40   |   200   | KubeSphere/k8s-master/k8s-worker |
| glusterfs-node-0 | 192.168.9.95 |  4   |  8   |   40   | 200+100 |     GlusterFS/ElasticSearch      |
| glusterfs-node-1 | 192.168.9.96 |  4   |  8   |   40   | 200+100 |     GlusterFS/ElasticSearch      |
| glusterfs-node-2 | 192.168.9.97 |  4   |  8   |   40   | 200+100 |     GlusterFS/ElasticSearch      |

## 2. Ansible配置

### 01. 增加hosts配置

**本系列文档中演示用的ElasticSearch和GlusterFS服务器复用，有条件的用户可以分开部署，注意调整hosts配置**

```yaml
# hosts文件配置

# 主要增加glusterfs
[k8s]
ks-k8s-master-0 ansible_ssh_host=192.168.9.91  host_name=ks-k8s-master-0
ks-k8s-master-1 ansible_ssh_host=192.168.9.92  host_name=ks-k8s-master-1
ks-k8s-master-2 ansible_ssh_host=192.168.9.93  host_name=ks-k8s-master-2

[glusterfs]
glusterfs-node-0 ansible_ssh_host=192.168.9.95 host_name=glusterfs-node-0
glusterfs-node-1 ansible_ssh_host=192.168.9.96 host_name=glusterfs-node-1
glusterfs-node-2 ansible_ssh_host=192.168.9.97 host_name=glusterfs-node-2

[es]
es-node-0 ansible_ssh_host=192.168.9.95 host_name=es-node-0
es-node-1 ansible_ssh_host=192.168.9.96 host_name=es-node-1
es-node-2 ansible_ssh_host=192.168.9.97 host_name=es-node-2

[servers:children]
k8s
glusterfs
es

[servers:vars]
ansible_connection=paramiko
ansible_ssh_user=root
ansible_ssh_pass=password
```



## 3. ElasticSearch安装配置

### 01. 初始化配置

> **01-检测服务器连通性**

```shell
# 利用ansible检测服务器的连通性
[root@zdevops-master /]# cd /data/ansible/ansible-zdevops/inventories/dev/
[root@zdevops-master dev]# source /opt/ansible2.8/bin/activate
(ansible2.8) [root@zdevops-master dev]# ansible -m ping es 
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
es-node-1 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
es-node-2 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
es-node-0 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

> **02-初始化服务器配置**

**由于是复用的服务器，在GlusterFS安装配置时已经配置，因此本文忽略，实际中可根据需要执行以下命令**

```shell
# 利用ansible-playbook初始化服务器配置
# (ansible2.8) [root@zdevops-master dev]# ansible-playbook -l es ../../playbooks/init-base.yaml
```

> **03-挂载数据盘**

**本文新增一块硬盘/dev/sdc, 格式化后挂载点为/data，作为ElasticSearch的数据存储目录，实际中请注意修改ansible-playbook的变量配置**

```shell
# 由于是采用的虚拟机，在虚拟化上挂载磁盘后，先检查一下磁盘是否被操作系统识别
# 利用ansible查看磁盘列表

(ansible2.8) [root@zdevops-master dev]# ansible es -m shell -a 'fdisk -l'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
es-node-1 | CHANGED | rc=0 >>

Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00019df2

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    83886079    40893440   8e  Linux LVM

Disk /dev/sdb: 214.7 GB, 214748364800 bytes, 419430400 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-root: 39.7 GB, 39720058880 bytes, 77578240 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

es-node-0 | CHANGED | rc=0 >>

Disk /dev/sdb: 214.7 GB, 214748364800 bytes, 419430400 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00019df2

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    83886079    40893440   8e  Linux LVM

Disk /dev/mapper/centos-root: 39.7 GB, 39720058880 bytes, 77578240 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

es-node-2 | CHANGED | rc=0 >>

Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00019df2

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    83886079    40893440   8e  Linux LVM

Disk /dev/sdb: 214.7 GB, 214748364800 bytes, 419430400 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-root: 39.7 GB, 39720058880 bytes, 77578240 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

# 上面的执行结果，发现并没有识别新磁盘
# 利用scsi的机制，触发磁盘扫描，识别新磁盘
# 先查看现有的scsi磁盘信息

(ansible2.8) [root@zdevops-master dev]# ansible es -m shell -a 'cat /proc/scsi/scsi'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
es-node-1 | CHANGED | rc=0 >>
Attached devices:
Host: scsi0 Channel: 00 Id: 00 Lun: 00
  Vendor: NECVMWar Model: VMware IDE CDR00 Rev: 1.00
  Type:   CD-ROM                           ANSI  SCSI revision: 05
Host: scsi2 Channel: 00 Id: 00 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
Host: scsi2 Channel: 00 Id: 01 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02

es-node-0 | CHANGED | rc=0 >>
Attached devices:
Host: scsi0 Channel: 00 Id: 00 Lun: 00
  Vendor: NECVMWar Model: VMware IDE CDR00 Rev: 1.00
  Type:   CD-ROM                           ANSI  SCSI revision: 05
Host: scsi2 Channel: 00 Id: 00 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
Host: scsi2 Channel: 00 Id: 01 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02

es-node-2 | CHANGED | rc=0 >>
Attached devices:
Host: scsi0 Channel: 00 Id: 00 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
Host: scsi0 Channel: 00 Id: 01 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
Host: scsi1 Channel: 00 Id: 00 Lun: 00
  Vendor: NECVMWar Model: VMware IDE CDR00 Rev: 1.00
  Type:   CD-ROM                           ANSI  SCSI revision: 05

# 手动添加新磁盘，这里重点注意一下，es-node-2的scsi的顺序跟其他两个节点不一样，需要单独执行
(ansible2.8) [root@zdevops-master dev]# ansible es-node-2 -m shell -a "echo 'scsi add-single-device 0 0 2 0' >/proc/scsi/scsi"
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
es-node-2 | CHANGED | rc=0 >>

(ansible2.8) [root@zdevops-master dev]# ansible es-node-0,es-node-1 -m shell -a "echo 'scsi add-single-device 2 0 2 0' >/proc/scsi/scsi"
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
es-node-1 | CHANGED | rc=0 >>


es-node-0 | CHANGED | rc=0 >>

# 查看新磁盘是否添加
# 查看scsi信息
(ansible2.8) [root@zdevops-master dev]# ansible es -m shell -a 'cat /proc/scsi/scsi'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
es-node-1 | CHANGED | rc=0 >>
Attached devices:
Host: scsi0 Channel: 00 Id: 00 Lun: 00
  Vendor: NECVMWar Model: VMware IDE CDR00 Rev: 1.00
  Type:   CD-ROM                           ANSI  SCSI revision: 05
Host: scsi2 Channel: 00 Id: 00 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
Host: scsi2 Channel: 00 Id: 01 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
Host: scsi2 Channel: 00 Id: 02 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02

es-node-2 | CHANGED | rc=0 >>
Attached devices:
Host: scsi0 Channel: 00 Id: 00 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
Host: scsi0 Channel: 00 Id: 01 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
Host: scsi1 Channel: 00 Id: 00 Lun: 00
  Vendor: NECVMWar Model: VMware IDE CDR00 Rev: 1.00
  Type:   CD-ROM                           ANSI  SCSI revision: 05
Host: scsi0 Channel: 00 Id: 02 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02

es-node-0 | CHANGED | rc=0 >>
Attached devices:
Host: scsi0 Channel: 00 Id: 00 Lun: 00
  Vendor: NECVMWar Model: VMware IDE CDR00 Rev: 1.00
  Type:   CD-ROM                           ANSI  SCSI revision: 05
Host: scsi2 Channel: 00 Id: 00 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
Host: scsi2 Channel: 00 Id: 01 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
Host: scsi2 Channel: 00 Id: 02 Lun: 00
  Vendor: VMware   Model: Virtual disk     Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
  
# 查看fdisk信息
(ansible2.8) [root@zdevops-master dev]# ansible es -m shell -a 'fdisk -l'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
es-node-1 | CHANGED | rc=0 >>

Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00019df2

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    83886079    40893440   8e  Linux LVM

Disk /dev/sdb: 214.7 GB, 214748364800 bytes, 419430400 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-root: 39.7 GB, 39720058880 bytes, 77578240 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdc: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

es-node-0 | CHANGED | rc=0 >>

Disk /dev/sdb: 214.7 GB, 214748364800 bytes, 419430400 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00019df2

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    83886079    40893440   8e  Linux LVM

Disk /dev/mapper/centos-root: 39.7 GB, 39720058880 bytes, 77578240 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdc: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

es-node-2 | CHANGED | rc=0 >>

Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00019df2

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    83886079    40893440   8e  Linux LVM

Disk /dev/sdb: 214.7 GB, 214748364800 bytes, 419430400 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-root: 39.7 GB, 39720058880 bytes, 77578240 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdc: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

# 利用ansible-playbook格式化数据盘
# 重点注意，一定要加-l参数指定es主机 利用-e参数替换默认的playbook的盘符和挂载点

(ansible2.8) [root@zdevops-master dev]# ansible-playbook -l es -e data_disk=sdc -e data_disk_path="/data" ../../playbooks/init-disk.yaml
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature

PLAY [初始化磁盘.] *************************************************************************************************

TASK [01-数据磁盘分区.] *********************************************************************************************
changed: [es-node-1]
changed: [es-node-2]
changed: [es-node-0]

TASK [02-格式化数据磁盘.] ********************************************************************************************
changed: [es-node-0]
changed: [es-node-1]
changed: [es-node-2]

TASK [03-挂载数据盘.] **********************************************************************************************
changed: [es-node-0]
changed: [es-node-1]
changed: [es-node-2]

PLAY RECAP ****************************************************************************************************
es-node-0                  : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
es-node-1                  : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
es-node-2                  : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```

- **扩展说明-虚拟化上磁盘插槽详情**
- ![esxi-scsi](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/esxi-scsi.png)

> **04-验证数据盘的挂载**

```shell
# 利用ansible验证数据盘是否格式化并挂载
(ansible2.8) [root@zdevops-master dev]# ansible es -m shell -a 'df -h'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
es-node-1 | CHANGED | rc=0 >>
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 2.0G     0  2.0G   0% /dev
tmpfs                    2.0G     0  2.0G   0% /dev/shm
tmpfs                    2.0G   73M  1.9G   4% /run
tmpfs                    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/centos-root   37G  1.6G   36G   5% /
/dev/sda1               1014M  168M  847M  17% /boot
tmpfs                    396M     0  396M   0% /run/user/0
/dev/sdc1                100G   33M  100G   1% /data

es-node-2 | CHANGED | rc=0 >>
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 2.0G     0  2.0G   0% /dev
tmpfs                    2.0G     0  2.0G   0% /dev/shm
tmpfs                    2.0G   89M  1.9G   5% /run
tmpfs                    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/centos-root   37G  1.6G   36G   5% /
/dev/sda1               1014M  168M  847M  17% /boot
/dev/sdc1                100G   33M  100G   1% /data
tmpfs                    396M     0  396M   0% /run/user/0

es-node-0 | CHANGED | rc=0 >>
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 2.0G     0  2.0G   0% /dev
tmpfs                    2.0G     0  2.0G   0% /dev/shm
tmpfs                    2.0G  215M  1.8G  11% /run
tmpfs                    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/centos-root   37G  1.7G   36G   5% /
/dev/sda1               1014M  168M  847M  17% /boot
tmpfs                    396M     0  396M   0% /run/user/0
/dev/sdc1                100G   33M  100G   1% /data

# 利用ansible验证数据盘是否配置自动挂载
(ansible2.8) [root@zdevops-master dev]# ansible es -m shell -a 'tail -1  /etc/fstab'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
es-node-1 | CHANGED | rc=0 >>
/dev/sdc1 /data xfs defaults 0 0

es-node-2 | CHANGED | rc=0 >>
/dev/sdc1 /data xfs defaults 0 0

es-node-0 | CHANGED | rc=0 >>
/dev/sdc1 /data xfs defaults 0 0

```



### 02. ElasticSearch安装配置-手工安装配置

**以下配置如无特殊说明，所有节点都要执行**

> **01-配置yum源**

```shell
[root@es-node-0 ~]# vi /etc/yum.repos.d/elasticsearch.repo

[elasticsearch]
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
gpgcheck=1
enable=1
```

> **02-安装ElasticSearch**

```shell
[root@es-node-0 ~]# yum install  elasticsearch -y
```

>  **03-创建elasticsearch配置文件**

```shell
[root@es-node-0 ~]# cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml.bak

[root@es-node-0 ~]# vi /etc/elasticsearch/elasticsearch.yml

cluster.name: es-lstack
node.name: es
node.master: true
path.data: /data01/elasticsearch/data
path.logs: /data01/elasticsearch/logs
network.host: 192.168.9.95		#注意修改每个节点对应的的ip地址
http.port: 9200
discovery.seed_hosts: ["192.168.9.95","192.168.9.96","192.168.9.97"]		#集群的每个节点地址
cluster.initial_master_nodes: ["es-node-0", "es-node-1", "es-node-2"]		#集群的每个节点名称

xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.keystore.path: cert/es-node-0.p12	#证书路径
xpack.security.transport.ssl.truststore.path: cert/es-node-0.p12	#证书路径
```

> **04-创建elasticsearch数据文件目录**

```shell
[root@es-node-0 ~]# mkdir -p /data01/elasticsearch
[root@es-node-0 ~]# chown -R elasticsearch.elasticsearch /data01/elasticsearch

# 证书目录
[root@es-node-0 ~]# mkdir -p /etc/elasticsearch/cert/
[root@es-node-0 ~]# chown -R elasticsearch.elasticsearch /etc/elasticsearch/cert/
```

>  **05-生成 instances文件用来配置证书-开启认证可选配置(需要在节点1中执行)**

```shell
# 在节点1中执行
[root@es-node-0 ~]# vi /etc/elasticsearch/elasticsearch-instances.yml

instances:
  - name: "es-node-0"  #注意修改每个节点的ip地址
    ip: 
      - "192.168.2.163"
  - name: "es-node-1"
    ip:
      - "192.168.2.124"
  - name: "es-node-2"
    ip:
      - "192.168.2.141"
```

>  **06-生成证书-开启认证可选配置(在节点1中执行)**

```shell
[root@es-node-0 ~]# /usr/share/elasticsearch/bin/elasticsearch-certutil cert --silent --in /etc/elasticsearch/elasticsearch-instances.yml --out /etc/elasticsearch/elasticsearch-instances.zip --pass VD41fXmOvp5hGlPc  #这是密码（在接下来是需要使用的）

# 解压证书压缩包
[root@es-node-0 ~]# cd  /etc/elasticsearch/
[root@es-node-0 ~]# unzip elasticsearch-instances.zip

# 将证书放到本机指定目录

[root@es-node-0 ~]# cp es-node-0/es-node-0.p12 /etc/elasticsearch/cert/

# 复制证书到其他的机器上
[root@es-node-0 ~]# scp -r es-node-1/es-node-1.p12 root@192.168.9.96:/etc/elasticsearch/cert/
[root@es-node-0 ~]# scp -r es-node-2/es-node-2.p12 root@192.168.9.97:/etc/elasticsearch/cert/
```

> **07-配置keystore-开启认证可选配置**

```shell
# 生成 keystore 文件
[root@es-node-0 ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore create

# 添加配置项到keystore文件
# 下面两个命令，均需要输入生成证书命令中使用的的密码
[root@es-node-0 ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
[root@es-node-0 ~]# /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password

# 创建用户
# -r指定权限superuser超级权限
[root@es-node-0 ~]# /usr/share/elasticsearch/bin/elasticsearch-users useradd lstack -p P@88w0rd -r superuser   
```

> **08-启动并设置开机自动启动elasticsearch服务**

```shell
[root@es-node-0 ~]# systemctl enable elasticsearch && systemctl start  elasticsearch
```

> **09-测试访问**

```shell
# ElasticSearch不开启认证
(ansible2.8) [root@zdevops-master dev]#  curl 192.168.9.95:9200/_cat/nodes?
192.168.9.97 49 69 6 0.03 0.29 0.21 cdfhilmrstw * es-node-2
192.168.9.95 26 73 3 0.04 0.13 0.14 cdfhilmrstw - es-node-0
192.168.9.96 37 66 4 0.01 0.10 0.11 cdfhilmrstw - es-node-1

# ElasticSearch开启认证
(ansible2.8) [root@zdevops-master dev]#  curl -ulstack:'P@88w0rd' 192.168.9.95:9200/_cat/nodes?
192.168.9.97 49 69 6 0.03 0.29 0.21 cdfhilmrstw * es-node-2
192.168.9.95 26 73 3 0.04 0.13 0.14 cdfhilmrstw - es-node-0
192.168.9.96 37 66 4 0.01 0.10 0.11 cdfhilmrstw - es-node-1
```



### 03. ElasticSearch安装配置-Ansible自动安装配置

```shell
# 利用ansible-playbook自动化安装配置ElasticSearch
(ansible2.8) [root@zdevops-master dev]# ansible-playbook ../../playbooks/deploy-elasticsearch.yaml 
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature

PLAY [安装配置ElasticSearch.] *************************************************************************************

TASK [01-配置yum源-配置elasticsearch软件源.] **************************************************************************
changed: [es-node-1]
changed: [es-node-0]
changed: [es-node-2]

TASK [02-安装elasticsearch.] ************************************************************************************
changed: [es-node-2]
changed: [es-node-1]
changed: [es-node-0]

TASK [03-创建elasticsearch配置文件.] ********************************************************************************
changed: [es-node-0]
changed: [es-node-2]
changed: [es-node-1]

TASK [04-创建elasticsearch数据文件目录.] ******************************************************************************
changed: [es-node-1]
changed: [es-node-0]
changed: [es-node-2]

PLAY [安装配置ElasticSearch SSL认证.] *******************************************************************************

TASK [01-生成ElasticSearch Instance文件用来配置ssl证书.] ****************************************************************
changed: [es-node-0]

TASK [02-生成证书.] ***********************************************************************************************
changed: [es-node-0]

TASK [03-安装基本工具包.] ********************************************************************************************
ok: [es-node-0] => (item=[u'unzip'])

TASK [04-解压cert文件.] *******************************************************************************************
changed: [es-node-0]

TASK [05-从服务器获取cert配置文件.] *************************************************************************************
changed: [es-node-0] => (item=es-node-0)
changed: [es-node-0] => (item=es-node-1)
changed: [es-node-0] => (item=es-node-2)

PLAY [认证配置.] **************************************************************************************************

TASK [01-创建elasticsearch cert目录.] *****************************************************************************
changed: [es-node-0]
changed: [es-node-2]
changed: [es-node-1]

TASK [02-同步cert文件.] *******************************************************************************************
changed: [es-node-0]
changed: [es-node-1]
changed: [es-node-2]

TASK [03-生成keystore文件.] ***************************************************************************************
ok: [es-node-0]
ok: [es-node-1]
ok: [es-node-2]

TASK [04-添加配置项到keystore文件.] ***********************************************************************************
changed: [es-node-1]
changed: [es-node-0]
changed: [es-node-2]

TASK [05-创建用户并指定superuser权限.] *********************************************************************************
changed: [es-node-1]
changed: [es-node-0]
changed: [es-node-2]

PLAY [终极配置.] **************************************************************************************************

TASK [01-启动并设置开机自动启动elasticsearch服务.] *************************************************************************
changed: [es-node-1]
changed: [es-node-0]
changed: [es-node-2]

PLAY RECAP ****************************************************************************************************
es-node-0                  : ok=15   changed=13   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
es-node-1                  : ok=10   changed=9    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
es-node-2                  : ok=10   changed=9    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```



### 04. ElasticSearch常用运维命令

```shell
# 查看服务状态
systemctl status elasticsearch.service
# 启动服务
systemctl start elasticsearch.service

# 停止服务
systemctl stop elasticsearch.service

# 查看简略的服务日志
journalctl -u elasticsearch.service

# 查看详细的服务日志
/data/elasticsearch/logs

# 查看gc日志
/var/log/elasticsearch
```

### 05. ElasticSearch调优

**暂未调优，后续补齐**



## 4. KubeSphere配置

本文后面的内容是一篇略显冗余的长文，之所以出现这个情况，是因为我自己配置疏忽导致的，具体疏忽在哪里下面到了具体环节的时候有重点标注。为了保留故事的完整性我后面就将错就错了，给大家展示了我排查故障、发现错误、改正错误的全流程。

### 01. 开启KubeSphere日志系统可插拔组件

1. 编辑**CRD**中的**ks-installer**的YAML配置文件

   - 在YAML文件中，搜索 **logging,auditing,events**，并将**enabled**的**false** 改为 **true**。

   - ```yaml
     logging:
       containerruntime: docker
       enabled: true
       logsidecar:
         enabled: true
         replicas: 2
     
     events:
       enabled: true
     
     auditing:
       enabled: true
     ```

   - 在YAML文件中，搜索 **es**，将**enabled**的**false** 改为 **true**。并修改外部的es服务器的信息

   - ```yaml
     es:
       basicAuth:
         enabled: true
         password: 'P@88w0rd'
         username: 'lstack'
       elkPrefix: logstash
       externalElasticsearchHost: '192.168.9.65'
       externalElasticsearchPort: '9200'
       logMaxAge: 7
     ```

2. 所有配置完成后，点击右下角的确定，保存配置。

3. 在 kubectl 中执行以下命令检查安装过程

   - ```shell
     kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
     ```

   - 看到如下状态，说明安装成功。

   - ```yaml
     TASK [ks-core/prepare : KubeSphere | Generating kubeconfig-admin] **************
     skipping: [localhost]
     
     PLAY RECAP *********************************************************************
     localhost                  : ok=26   changed=14   unreachable=0    failed=0    skipped=12   rescued=0    ignored=0
     
     Start installing monitoring
     Start installing multicluster
     Start installing openpitrix
     Start installing network
     Start installing alerting
     Start installing auditing
     Start installing devops
     Start installing events
     Start installing logging
     **************************************************
     Waiting for all tasks to be completed ...
     task alerting status is successful  (1/9)
     task network status is successful  (2/9)
     task multicluster status is successful  (3/9)
     task openpitrix status is successful  (4/9)
     task auditing status is successful  (5/9)
     task logging status is successful  (6/9)
     task events status is successful  (7/9)
     task devops status is successful  (8/9)
     task monitoring status is successful  (9/9)
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
     https://kubesphere.io             2022-04-15 11:54:50
     #####################################################
     ```

4. 验证安装结果

   > **01-日志系统组件**

   - 登录控制台，**平台管理**->**集群管理**->**系统组件**，检查是否 **日志** 标签页中的所有组件都处于**健康**状态。如果是，表明组件安装成功。
   - ![kubesphere-clusters-components-logging](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-clusters-components-logging.png)

   - 点击右下角的**工具箱**按钮，多出几个分析工具的菜单
   - ![kubesphere-clusters-components-logging-2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-clusters-components-logging-2.png)

   - 在kubectl工具中查看

   - ```shell
     / # kubectl get pod -n kubesphere-logging-system
     NAME                                            READY   STATUS    RESTARTS   AGE
     fluent-bit-j5rvq                                1/1     Running   0          136m
     fluent-bit-tqrqs                                1/1     Running   0          136m
     fluent-bit-wv5hn                                1/1     Running   0          136m
     fluentbit-operator-745bf5559f-lgq9p             1/1     Running   0          137m
     ks-events-exporter-59d48f6777-ncgpc             2/2     Running   0          133m
     ks-events-operator-5944645757-kqt65             1/1     Running   0          134m
     ks-events-ruler-575669b4-lnpxc                  2/2     Running   0          133m
     ks-events-ruler-575669b4-mh2fn                  2/2     Running   0          133m
     kube-auditing-operator-84857bf967-kg7bs         1/1     Running   0          135m
     kube-auditing-webhook-deploy-64cfb8c9f8-c65cb   1/1     Running   0          134m
     kube-auditing-webhook-deploy-64cfb8c9f8-j8w5d   1/1     Running   0          134m
     logsidecar-injector-deploy-5fb6fdc6dd-fv8vg     2/2     Running   0          134m
     logsidecar-injector-deploy-5fb6fdc6dd-fxsbf     2/2     Running   0          134m
     ```

   > **02-问题说明**

   - 上面的验证，表面上看日志系统是配置成功了，但是实际上还是存在问题的，接下来我们继续深入验证。

   - 点击**工具箱**中的**容器日志查询**，会出现如下报错，说明我们的es对接并没有成功。

   - ![kubesphere-logging-error-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-logging-error-1.png)

   - 执行以下命令查看Output的配置，发现几个重要参数并没有变成我们想要的结果

   - ```shell
     / # kubectl get output -n kubesphere-logging-system es -oyaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"logging","logging.kubesphere.io/enabled":"true"},"name":"es","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","logstashFormat":true,"logstashPrefix":"ks-logstash-log","port":9200,"timeKey":"@timestamp"},"matchRegex":"(?:kube|service)\\.(.*)"}}
       creationTimestamp: "2022-04-15T03:51:11Z"
       generation: 1
       labels:
         logging.kubesphere.io/component: logging
         logging.kubesphere.io/enabled: "true"
       name: es
       namespace: kubesphere-logging-system
       resourceVersion: "1503226"
       uid: 1a60036f-9ad0-4a3c-bab1-e077102a2724
     spec:
       es:
         generateID: true
         host: elasticsearch-logging-data.kubesphere-logging-system.svc
         logstashFormat: true
         logstashPrefix: ks-logstash-log
         port: 9200
         timeKey: '@timestamp'
       matchRegex: (?:kube|service)\.(.*)
     ```

   - 也可以在KubeSphere控制台中查看Output的配置

   - 登录控制台，在**集群管理**->**CRD**中, 搜索output
   
   - ![kubesphere-crd-output](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-crd-output.png)
   
   - 点击**Output**，进入Output详情页，会发现有**es**、**es-auditing**、**es-events**三种资源
   
   - ![kubesphere-crd-output-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-crd-output-1.png)
   
   - 编辑**es**的YAML配置（点击右侧的三个竖点的图标，会弹出编辑YAML的选项），可以看到配置文件跟我们命令行输出的结果一致。
   
   - ![kubesphere-crd-output-es-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-crd-output-es-1.png)
   
   - 查看fluent-bit容器
   
   - ```shell
     / # kubectl get pods -n kubesphere-logging-system 
     NAME                                            READY   STATUS    RESTARTS   AGE
     fluent-bit-j5rvq                                1/1     Running   0          3h34m
     fluent-bit-vv8pb                                1/1     Running   0          103s
     fluent-bit-wv5hn                                1/1     Running   0          3h34m
     fluentbit-operator-745bf5559f-lgq9p             1/1     Running   0          3h36m
     ks-events-exporter-59d48f6777-ncgpc             2/2     Running   0          3h32m
     ks-events-operator-5944645757-kqt65             1/1     Running   0          3h32m
     ks-events-ruler-575669b4-lnpxc                  2/2     Running   0          3h32m
     ks-events-ruler-575669b4-mh2fn                  2/2     Running   0          3h32m
     kube-auditing-operator-84857bf967-kg7bs         1/1     Running   0          3h33m
     kube-auditing-webhook-deploy-64cfb8c9f8-c65cb   1/1     Running   0          3h32m
     kube-auditing-webhook-deploy-64cfb8c9f8-j8w5d   1/1     Running   0          3h32m
     logsidecar-injector-deploy-5fb6fdc6dd-fv8vg     2/2     Running   0          3h32m
     logsidecar-injector-deploy-5fb6fdc6dd-fxsbf     2/2     Running   0          3h32m
     ```
   
   - 查看pod的log，也会发现大量的错误日志。
   
   - ```yaml
     / # kubectl logs  fluent-bit-j5rvq -n kubesphere-logging-system 
     
     ...
      [2022/04/15 07:25:47] [ warn] [net] getaddrinfo(host='elasticsearch-logging-data.kubesphere-logging-system.svc', err=4): Domain name not found
     
      [2022/04/15 07:25:47] [ warn] [engine] failed to flush chunk '12-1650007543.816834850.flb', retry in 6 seconds: task_id=2, input=tail.2 > output=es.0 (out_id=0)
     ```
   
     

### 02. 外部ElasticSearch配置失败的解决过程

1. 查找官方论坛，关键词使用**外部es**找到了以下一篇看着比较接近的文档，打开来看看。

   - > [3.2.1版本日志系统对接外部ES无法正常收集日志](https://kubesphere.com.cn/forum/d/6455-321es)

   - ![elasticsearch-error-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/elasticsearch-error-1.png)

   - 看了一遍文档，现象跟我的一摸一样，文档中也提到了解决方案，貌似提问的人按着配置文档配置成功了，那我们去看看文档中的解决方案

   - ![image-20220420095405948](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220420095405948.png)

   - 跳转到[第一个参考链接](https://kubesphere.io/zh/docs/faq/observability/logging/#%E5%A6%82%E4%BD%95%E5%B0%86%E6%97%A5%E5%BF%97%E5%AD%98%E5%82%A8%E6%94%B9%E4%B8%BA%E5%A4%96%E9%83%A8-elasticsearch-%E5%B9%B6%E5%85%B3%E9%97%AD%E5%86%85%E9%83%A8-elasticsearch)

   - 第一个参考链接里，有一节**如何将日志存储改为外部 Elasticsearch 并关闭内部 Elasticsearch**,看了一遍就是介绍如何开启外部es的配置，跟我们之前在界面上编辑**ClusterConfiguration**是一样的。

   - 但是第一个他这一步介绍里手工重新执行**ks-installer**的方法，我就假设我界面修改了配置，但是**ks-installer**没重新执行（实际上肯定已经执行了），我在手工执行一次。

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl rollout restart deploy -n kubesphere-system ks-installer
     deployment.apps/ks-installer restarted
     [root@ks-k8s-master-0 ~]# kubectl rollout status deploy -n kubesphere-system ks-installer
     deployment "ks-installer" successfully rolled out
     ```

   - 然后发现并没有什么改变，问题依旧。

   - 跳转到[第二个参考链接](https://github.com/wenchajun/ks-installer/blob/master/roles/ks-logging/templates/custom-output-elasticsearch-logging.yaml.j2)，这里是告诉我们手工修改的配置方式，暂时先不考虑，总感觉很多人遇到了，肯定有更好的方案，我们继续看论坛帖子

   - 然后我在刚才的搜索结果中找到了[外部es无法正常收集日志](https://kubesphere.com.cn/forum/d/6590-es/3)，这篇文档的最后那兄弟说参考[github的文档](https://github.com/kubesphere/kubesphere/issues/4640)解决了问题，那我们去试试.

   - ![image-20220420103039813](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220420103039813.png)

   - 该解决方案中也是修改配置文件，但是有一点，需要删掉配置文件中原来的es相关的status配置后，再次重新执行ks-installer

   - ![image-20220420103344968](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220420103344968.png)

   - 既然有这一段话 那么我们去看看我们实际环境的配置,但是我们的环境中并没有es的配置，因为我们最早的时候就没有启用内置的es服务。

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl edit cc -n kubesphere-system ks-installer
     
     events:
       enabledTime: 2022-04-15T16:22:59CST
       status: enabled
     fluentbit:
       enabledTime: 2022-04-15T16:19:46CST
       status: enabled
     logging:
       enabledTime: 2022-04-15T16:22:59CST
       status: enabled
     ```

   - 但是啊，本着宁杀错的原则，那我们删了这几段试试看看，重新执行**ks-installer**，看看效果。

   - 执行下面的命令，查看安装过程，确实开始了重新配置的过程（过程没截图,截取了几个相近的配置过程）

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
     
     ...
     
     TASK [common : KubeSphere | Deleting elasticsearch] ****************************
     skipping: [localhost]
     
     TASK [common : KubeSphere | Waiting for seconds] *******************************
     skipping: [localhost]
     
     TASK [common : KubeSphere | Deploying elasticsearch-logging] *******************
     skipping: [localhost]
     
     TASK [common : KubeSphere | Importing es status] *******************************
     skipping: [localhost]
     
     TASK [common : KubeSphere | Deploying elasticsearch-logging-curator] ***********
     changed: [localhost]
     
     TASK [common : KubeSphere | Getting fluentbit installation files] **************
     changed: [localhost]
     
     TASK [common : KubeSphere | Creating custom manifests] *************************
     changed: [localhost] => (item={'path': 'fluentbit', 'file': 'custom-fluentbit-fluentBit.yaml'})
     changed: [localhost] => (item={'path': 'init', 'file': 'custom-fluentbit-operator-deployment.yaml'})
     
     TASK [common : KubeSphere | Preparing fluentbit operator setup] ****************
     changed: [localhost]
     
     TASK [common : KubeSphere | Deploying new fluentbit operator] ******************
     changed: [localhost]
     
     TASK [common : KubeSphere | Importing fluentbit status] ************************
     changed: [localhost]
     
     ```

   - 等到彻底执行完成后，我发现我又失望了，问题依旧,看了一眼Output的配置，没啥变化。

   - ```shell
     [root@ks-k8s-master-0 ~]#  kubectl get output -n kubesphere-logging-system es -oyaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"logging","logging.kubesphere.io/enabled":"true"},"name":"es","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","logstashFormat":true,"logstashPrefix":"ks-logstash-log","port":9200,"timeKey":"@timestamp"},"matchRegex":"(?:kube|service)\\.(.*)"}}
       creationTimestamp: "2022-04-15T03:51:11Z"
       generation: 1
       labels:
         logging.kubesphere.io/component: logging
         logging.kubesphere.io/enabled: "true"
       name: es
       namespace: kubesphere-logging-system
       resourceVersion: "1503226"
       uid: 1a60036f-9ad0-4a3c-bab1-e077102a2724
     spec:
       es:
         generateID: true
         host: elasticsearch-logging-data.kubesphere-logging-system.svc
         logstashFormat: true
         logstashPrefix: ks-logstash-log
         port: 9200
         timeKey: '@timestamp'
       matchRegex: (?:kube|service)\.(.*)
     ```

   - 看来上面提到的方法，应该可能只适用于**原来开启过内置es，后来又想换到外部es的场景**。

   - 那我们继续回来看[外部es无法正常收集日志](https://kubesphere.com.cn/forum/d/6590-es/3)这篇文档，里面还提到了一个参考链接[在使用kusphere的操作审计功能时，总有弹窗提示：dial tcp: lookup http on 10.96.0.10:53: no such host](https://kubesphere.com.cn/forum/d/2450-kuspheredial-tcp-lookup-http-on-109601053-no-such-host/4).

   - ![image-20220420111124760](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220420111124760.png)

   - 该文档中的描述跟我们的想象类似，但不完全相同，文中提到的问题可以通过修改**configmap**来解决，文章最后那哥们也说解决成功了。

   - ![image-20220420111152382](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220420111152382.png)

   - 我们看看我们对应的configmap配置

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl get configmap kubesphere-config -n kubesphere-system -o yaml
     apiVersion: v1
     data:
       kubesphere.yaml: |
         authentication:
           authenticateRateLimiterMaxTries: 10
           authenticateRateLimiterDuration: 10m0s
           loginHistoryRetentionPeriod: 168h
           maximumClockSkew: 10s
           multipleLogin: True
           kubectlImage: kubesphere/kubectl:v1.21.0
           jwtSecret: "1xh3ldfKpJSzSmFCbi2HXXbw5dn4o4kv"
           oauthOptions:
             clients:
             - name: kubesphere
               secret: kubesphere
               redirectURIs:
               - '*'
     
         ldap:
           host: openldap.kubesphere-system.svc:389
           managerDN: cn=admin,dc=kubesphere,dc=io
           managerPassword: admin
           userSearchBase: ou=Users,dc=kubesphere,dc=io
           groupSearchBase: ou=Groups,dc=kubesphere,dc=io
     
         redis:
           host: redis.kubesphere-system.svc
           port: 6379
           password: KUBESPHERE_REDIS_PASSWORD
           db: 0
     
     
         s3:
           endpoint: http://minio.kubesphere-system.svc:9000
           region: us-east-1
           disableSSL: True
           forcePathStyle: True
           accessKeyID: openpitrixminioaccesskey
           secretAccessKey: openpitrixminiosecretkey
           bucket: s2i-binaries
     
         network:
           ippoolType: none
         devops:
           host: http://devops-jenkins.kubesphere-devops-system.svc/
           username: admin
           password: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImFkbWluQGt1YmVzcGhlcmUuaW8iLCJ1c2VybmFtZSI6ImFkbWluIiwidG9rZW5fdHlwZSI6InN0YXRpY190b2tlbiJ9.DVnt9FY7UNu2Mvshh_46UMKhZG7_X7NPC-ClQ68ynB0
           maxConnections: 100
           endpoint: http://devops-apiserver.kubesphere-devops-system:9090
         openpitrix:
           s3:
             endpoint: http://minio.kubesphere-system.svc:9000
             region: us-east-1
             disableSSL: True
             forcePathStyle: True
             accessKeyID: openpitrixminioaccesskey
             secretAccessKey: openpitrixminiosecretkey
             bucket: app-store
         monitoring:
           endpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090
           enableGPUMonitoring: false
         gpu:
           kinds:
           - resourceName: nvidia.com/gpu
             resourceType: GPU
             default: True
         notification:
           endpoint: http://notification-manager-svc.kubesphere-monitoring-system.svc:19093
         logging:
           host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200
           basicAuth: True
           username: "lstack"
           password: "P@88w0rd"
           indexPrefix: ks-logstash-log
         events:
           host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200
           basicAuth: True
           username: "lstack"
           password: "P@88w0rd"
           indexPrefix: ks-logstash-events
         auditing:
           enable: true
           webhookURL: https://kube-auditing-webhook-svc.kubesphere-logging-system.svc:6443/audit/webhook/event
           host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200
           basicAuth: True
           username: "lstack"
           password: "P@88w0rd"
           indexPrefix: ks-logstash-auditing
     
         alerting:
           prometheusEndpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090
           thanosRulerEndpoint: http://thanos-ruler-operated.kubesphere-monitoring-system.svc:10902
           thanosRuleResourceLabels: thanosruler=thanos-ruler,role=thanos-alerting-rules
     
     
         gateway:
           watchesPath: /var/helm-charts/watches.yaml
           repository: kubesphere/nginx-ingress-controller
           tag: v0.48.1
           namespace: kubesphere-controls-system
     kind: ConfigMap
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"v1","data":{"kubesphere.yaml":"authentication:\n  authenticateRateLimiterMaxTries: 10\n  authenticateRateLimiterDuration: 10m0s\n  loginHistoryRetentionPeriod: 168h\n  maximumClockSkew: 10s\n  multipleLogin: True\n  kubectlImage: kubesphere/kubectl:v1.21.0\n  jwtSecret: \"1xh3ldfKpJSzSmFCbi2HXXbw5dn4o4kv\"\n  oauthOptions:\n    clients:\n    - name: kubesphere\n      secret: kubesphere\n      redirectURIs:\n      - '*'\n\nldap:\n  host: openldap.kubesphere-system.svc:389\n  managerDN: cn=admin,dc=kubesphere,dc=io\n  managerPassword: admin\n  userSearchBase: ou=Users,dc=kubesphere,dc=io\n  groupSearchBase: ou=Groups,dc=kubesphere,dc=io\n\nredis:\n  host: redis.kubesphere-system.svc\n  port: 6379\n  password: KUBESPHERE_REDIS_PASSWORD\n  db: 0\n\n\ns3:\n  endpoint: http://minio.kubesphere-system.svc:9000\n  region: us-east-1\n  disableSSL: True\n  forcePathStyle: True\n  accessKeyID: openpitrixminioaccesskey\n  secretAccessKey: openpitrixminiosecretkey\n  bucket: s2i-binaries\n\nnetwork:\n  ippoolType: none\ndevops:\n  host: http://devops-jenkins.kubesphere-devops-system.svc/\n  username: admin\n  password: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImFkbWluQGt1YmVzcGhlcmUuaW8iLCJ1c2VybmFtZSI6ImFkbWluIiwidG9rZW5fdHlwZSI6InN0YXRpY190b2tlbiJ9.DVnt9FY7UNu2Mvshh_46UMKhZG7_X7NPC-ClQ68ynB0\n  maxConnections: 100\n  endpoint: http://devops-apiserver.kubesphere-devops-system:9090\nopenpitrix:\n  s3:\n    endpoint: http://minio.kubesphere-system.svc:9000\n    region: us-east-1\n    disableSSL: True\n    forcePathStyle: True\n    accessKeyID: openpitrixminioaccesskey\n    secretAccessKey: openpitrixminiosecretkey\n    bucket: app-store\nmonitoring:\n  endpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090\n  enableGPUMonitoring: false\ngpu:\n  kinds:\n  - resourceName: nvidia.com/gpu\n    resourceType: GPU\n    default: True\nnotification:\n  endpoint: http://notification-manager-svc.kubesphere-monitoring-system.svc:19093\nlogging:\n  host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200\n  basicAuth: True\n  username: \"lstack\"\n  password: \"P@88w0rd\"\n  indexPrefix: ks-logstash-log\nevents:\n  host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200\n  basicAuth: True\n  username: \"lstack\"\n  password: \"P@88w0rd\"\n  indexPrefix: ks-logstash-events\nauditing:\n  enable: true\n  webhookURL: https://kube-auditing-webhook-svc.kubesphere-logging-system.svc:6443/audit/webhook/event\n  host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200\n  basicAuth: True\n  username: \"lstack\"\n  password: \"P@88w0rd\"\n  indexPrefix: ks-logstash-auditing\n\nalerting:\n  prometheusEndpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090\n  thanosRulerEndpoint: http://thanos-ruler-operated.kubesphere-monitoring-system.svc:10902\n  thanosRuleResourceLabels: thanosruler=thanos-ruler,role=thanos-alerting-rules\n\n\ngateway:\n  watchesPath: /var/helm-charts/watches.yaml\n  repository: kubesphere/nginx-ingress-controller\n  tag: v0.48.1\n  namespace: kubesphere-controls-system\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"kubesphere-config","namespace":"kubesphere-system"}}
       creationTimestamp: "2022-04-09T14:42:47Z"
       name: kubesphere-config
       namespace: kubesphere-system
       resourceVersion: "1557079"
       uid: 2d45c34b-bc9d-43bd-a3d3-37b294393531
     ```

   - 我们发现几个相关配置的host地址确实是错的跟我们之前发现的一样，而且这里并没有明确的es配置。对于技术细节了解不深的我，直觉告诉我这个方向应该是错的，暂时不选。

   - 排查到此时，我的心里已经烦躁不安了，搞了两个小时的我此时不想再找下去了，直接放出最后一招，去改Output文件看看。

2. 问题排查第二环节，修改Output文件

   - 先看看参考模板**custom-output-elasticsearch-logging.yaml.j2**的样子

   - ```yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       name: es
       namespace: kubesphere-logging-system
       labels:
         logging.kubesphere.io/enabled: "true"
         logging.kubesphere.io/component: "logging"
     spec:
       matchRegex: (?:kube|service)\.(.*)
       es:
         host: "{% if common.es.externalElasticsearchHost is defined and common.es.externalElasticsearchHost != "" %}{{ common.es.externalElasticsearchHost }}{% else %}elasticsearch-logging-data.kubesphere-logging-system.svc{% endif %}"
         port: {% if common.es.externalElasticsearchPort is defined and common.es.externalElasticsearchPort != "" %}{{ common.es.externalElasticsearchPort }}{% else %}9200{% endif %}
     
     {% if common.es.basicAuth is defined and common.es.basicAuth.enabled is defined and common.es.basicAuth.enabled %}
     {% if common.es.basicAuth.username is defined and common.es.basicAuth.username != "" %}
         httpUser:
           valueFrom:
             secretKeyRef:
               key: "username"
               name: "elasticsearch-credentials"
     {% endif %}
     {% if common.es.basicAuth.password is defined and common.es.basicAuth.password != "" %}
         httpPassword:
           valueFrom:
             secretKeyRef:
               key: "password"
               name: "elasticsearch-credentials"
     {% endif %}
     {% endif %}
         generateID: true
         logstashPrefix: "ks-{{ common.es.elkPrefix }}-log"
         logstashFormat: true
         timeKey: "@timestamp"
     {% if common.es.externalElasticsearchProtocol is defined and common.es.externalElasticsearchProtocol == "https" %}
         tls:
           verify: false
     {% endif %}
     ```

   - 小伙伴们你们看懂了么，好在我有Ansible的使用经验，之前也写过Jinja2的语法，否则暴脾气的我，又要拍桌子了。没看懂的也没关系，暂时先放着，我会在本文对应的直播视频中给大家讲解具体含义。

   - 再来贴一次我们线上的配置

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl get outputs -n kubesphere-logging-system es -o yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"logging","logging.kubesphere.io/enabled":"true"},"name":"es","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","logstashFormat":true,"logstashPrefix":"ks-logstash-log","port":9200,"timeKey":"@timestamp"},"matchRegex":"(?:kube|service)\\.(.*)"}}
       creationTimestamp: "2022-04-15T03:51:11Z"
       generation: 1
       labels:
         logging.kubesphere.io/component: logging
         logging.kubesphere.io/enabled: "true"
       name: es
       namespace: kubesphere-logging-system
       resourceVersion: "1503226"
       uid: 1a60036f-9ad0-4a3c-bab1-e077102a2724
     spec:
       es:
         generateID: true
         host: elasticsearch-logging-data.kubesphere-logging-system.svc
         logstashFormat: true
         logstashPrefix: ks-logstash-log
         port: 9200
         timeKey: '@timestamp'
       matchRegex: (?:kube|service)\.(.*)
     ```

   - 通过分析模板，发现如果开启认证的话，需要调用**elasticsearch-credentials**这个secrets，所以我们先检查一下，这个secret是否存在。

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl get secrets elasticsearch-credentials -n kubesphere-logging-system 
     NAME                        TYPE                       DATA   AGE
     elasticsearch-credentials   kubernetes.io/basic-auth   2      4d21h
     [root@ks-k8s-master-0 ~]# kubectl get secrets elasticsearch-credentials -n kubesphere-logging-system -o yaml
     apiVersion: v1
     data:
       password: UEA4OHcwcmQ=
       username: bHN0YWNr
     kind: Secret
     metadata:
       creationTimestamp: "2022-04-15T05:56:51Z"
       name: elasticsearch-credentials
       namespace: kubesphere-logging-system
       resourceVersion: "1528520"
       uid: d918b1e0-4562-490b-bed7-d1dce702589a
     type: kubernetes.io/basic-auth
     ```

   - 确认secrets存在，接下来我先把Outpt配置文件输出一份备份下来，然后再去修改配置文件。

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl get outputs -n kubesphere-logging-system es -o yaml > es.output.yaml
     [root@ks-k8s-master-0 ~]# kubectl edit outputs -n kubesphere-logging-system es
     ```

   - 修改后的内容如下：

   - ```yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"logging","logging.kubesphere.io/enabled":"true"},"name":"es","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","logstashFormat":true,"logstashPrefix":"ks-logstash-log","port":9200,"timeKey":"@timestamp"},"matchRegex":"(?:kube|service)\\.(.*)"}}
       creationTimestamp: "2022-04-15T03:51:11Z"
       generation: 1
       labels:
         logging.kubesphere.io/component: logging
         logging.kubesphere.io/enabled: "true"
       name: es
       namespace: kubesphere-logging-system
       resourceVersion: "1503226"
       uid: 1a60036f-9ad0-4a3c-bab1-e077102a2724
     spec:
       es:
         generateID: true
         host: 192.168.9.95
         port: 9200
         httpUser:
           valueFrom:
             secretKeyRef:
               key: "username"
               name: "elasticsearch-credentials"
         httpPassword:
           valueFrom:
             secretKeyRef:
               key: "password"
               name: "elasticsearch-credentials"
         logstashFormat: true
         logstashPrefix: ks-logstash-log
         tls:
           verify: false
         timeKey: '@timestamp'
       matchRegex: (?:kube|service)\.(.*)
     ```

   - 配置文件改造完成后，保存退出

     ```shell
     [root@ks-k8s-master-0 ~]# kubectl edit outputs -n kubesphere-logging-system es
     output.logging.kubesphere.io/es edited
     ```

   - 再次查看es的Output的资源配置,发现已经有变化了

     > **重点说明: 本文写到最后的时候我发现我这步就做错了,配置文件我不应该加下面的参数，不加的话到这步为止就都ok了不会再有后面的内容了，为了故事的完整性就继续按错的写了！！！**
     >
     > tls:
     >   verify: false

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl get outputs -n kubesphere-logging-system es -o yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"logging","logging.kubesphere.io/enabled":"true"},"name":"es","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","logstashFormat":true,"logstashPrefix":"ks-logstash-log","port":9200,"timeKey":"@timestamp"},"matchRegex":"(?:kube|service)\\.(.*)"}}
       creationTimestamp: "2022-04-15T03:51:11Z"
       generation: 2
       labels:
         logging.kubesphere.io/component: logging
         logging.kubesphere.io/enabled: "true"
       name: es
       namespace: kubesphere-logging-system
       resourceVersion: "2865406"
       uid: 1a60036f-9ad0-4a3c-bab1-e077102a2724
     spec:
       es:
         generateID: true
         host: 192.168.9.95
         httpPassword:
           valueFrom:
             secretKeyRef:
               key: password
               name: elasticsearch-credentials
         httpUser:
           valueFrom:
             secretKeyRef:
               key: username
               name: elasticsearch-credentials
         logstashFormat: true
         logstashPrefix: ks-logstash-log
         port: 9200
         timeKey: '@timestamp'
         tls:
           verify: false
       matchRegex: (?:kube|service)\.(.*)
     ```

   - 我们再看看fluent-bit的日志(截取部分关键日志)

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl logs  fluent-bit-j5rvq -n kubesphere-logging-system
     
     [2022/04/20 03:43:19] [ warn] [net] getaddrinfo(host='elasticsearch-logging-data.kubesphere-logging-system.svc', err=4): Domain name not found
     
      [2022/04/20 03:43:19] [ warn] [engine] failed to flush chunk '12-1650426197.168443930.flb', retry in 8 seconds: task_id=1, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 03:43:23] [ warn] [net] getaddrinfo(host='elasticsearch-logging-data.kubesphere-logging-system.svc', err=4): Domain name not found
     
      [2022/04/20 03:43:23] [ warn] [engine] chunk '12-1650426192.752749410.flb' cannot be retried: task_id=4, input=tail.2 > output=es.0
     
      [2022/04/20 03:43:23] [ warn] [net] getaddrinfo(host='elasticsearch-logging-data.kubesphere-logging-system.svc', err=4): Domain name not found
     
      [2022/04/20 03:43:23] [ warn] [engine] chunk '12-1650426187.71256663.flb' cannot be retried: task_id=0, input=tail.2 > output=es.0
     
      [2022/04/20 03:43:24] [ info] [engine] service stopped
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=416649 watch_fd=1
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=461549 watch_fd=2
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134582558 watch_fd=3
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=268955529 watch_fd=5
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=373423 watch_fd=6
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=373208 watch_fd=7
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403716654 watch_fd=8
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134583208 watch_fd=9
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=269010417 watch_fd=11
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403995814 watch_fd=13
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=962171 watch_fd=14
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134580575 watch_fd=16
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=268955289 watch_fd=17
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403715964 watch_fd=18
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403855647 watch_fd=19
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=413299 watch_fd=20
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=375870 watch_fd=21
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134581094 watch_fd=22
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=268956011 watch_fd=26
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=373969 watch_fd=27
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403792092 watch_fd=28
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=412431 watch_fd=29
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134738124 watch_fd=30
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=467559 watch_fd=31
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403860482 watch_fd=32
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=463110 watch_fd=160
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=135190584 watch_fd=161
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134738131 watch_fd=162
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134583424 watch_fd=173
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134583425 watch_fd=177
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=466327 watch_fd=183
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=269514179 watch_fd=184
     
      [2022/04/20 03:43:24] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=412659 watch_fd=186
     
      level=error msg="Fluent bit exited" error=null
     
      level=info msg=backoff delay=1s
     
      level=info msg="backoff timer done" actual=1.000475459s expected=1s
     
      level=info msg="Fluent bit started"
     
      Fluent Bit v1.8.3
     
      * Copyright (C) 2019-2021 The Fluent Bit Authors
     
      * Copyright (C) 2015-2018 Treasure Data
     
      * Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
     
      * https://fluentbit.io
     
      
     
      [2022/04/20 03:43:25] [ info] [engine] started (pid=18)
     
      [2022/04/20 03:43:25] [ info] [storage] version=1.1.1, initializing...
     
      [2022/04/20 03:43:25] [ info] [storage] in-memory
     
      [2022/04/20 03:43:25] [ info] [storage] normal synchronization mode, checksum disabled, max_chunks_up=128
     
      [2022/04/20 03:43:25] [ info] [cmetrics] version=0.1.6
     
      [2022/04/20 03:43:25] [ info] [input:systemd:systemd.0] seek_cursor=s=5d0022eeaf454ba8a40dde2836de2f4d;i=963... OK
     
      [2022/04/20 03:43:25] [ info] [input:systemd:systemd.1] seek_cursor=s=5d0022eeaf454ba8a40dde2836de2f4d;i=96c... OK
     
      [2022/04/20 03:43:25] [ info] [filter:kubernetes:kubernetes.4] https=1 host=kubernetes.default.svc port=443
     
      [2022/04/20 03:43:25] [ info] [filter:kubernetes:kubernetes.4] local POD info OK
     
      [2022/04/20 03:43:25] [ info] [filter:kubernetes:kubernetes.4] testing connectivity with API server...
     
      [2022/04/20 03:43:25] [ info] [filter:kubernetes:kubernetes.4] connectivity OK
     
      [2022/04/20 03:43:25] [ info] [sp] stream processor started
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=416649 watch_fd=1 name=/var/log/containers/alertmanager-main-2_kubesphere-monitoring-system_alertmanager-1a71f1d5b1136320644f3289e4b22544620db4a0d35a1ffec52bc534d729c358.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=461549 watch_fd=2 name=/var/log/containers/alertmanager-main-2_kubesphere-monitoring-system_config-reloader-bc14c53008f1e800d6240a5b912239a3d5682c6dcb719f48c69b7e8ba5892e35.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134582558 watch_fd=3 name=/var/log/containers/calico-kube-controllers-75ddb95444-8xvf4_kube-system_calico-kube-controllers-dc90044aa0ce62e61dfb0842c8f2e5aa8d021738adffca847b003a5c9621512c.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134583425 watch_fd=4 name=/var/log/containers/calico-node-pzvrj_kube-system_calico-node-f2f40fb9878e3328a4783ceb9c01eaed9edf24c85579ed87c7ffc2161779c3e2.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268955529 watch_fd=5 name=/var/log/containers/calico-node-pzvrj_kube-system_flexvol-driver-abf59d8a7af22c683dfb0776a0835c719f41ef23446e912f82901a4fd9cf166f.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373423 watch_fd=6 name=/var/log/containers/calico-node-pzvrj_kube-system_install-cni-b3d49f2e9c03e63c3863d50365d2ade01dead979c90285c526ac0029b58411cd.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373208 watch_fd=7 name=/var/log/containers/calico-node-pzvrj_kube-system_upgrade-ipam-6d4425d7ea5302511df1c278f2b113ac8fdfa0a372e07b025e0a9b45d30129ac.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403716654 watch_fd=8 name=/var/log/containers/coredns-5495dd7c88-68k9b_kube-system_coredns-879c087a4c8717d0886e6603a63e9503a406d215cfe77941cbcc1cc7c138d010.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134583208 watch_fd=9 name=/var/log/containers/coredns-5495dd7c88-z9q6j_kube-system_coredns-80cd4427410de02465b415683817a75674a95777ffc0929f93e87506b120ca59.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=466327 watch_fd=10 name=/var/log/containers/ks-apiserver-574966976-snfm4_kubesphere-system_ks-apiserver-8871a0cc18cfef4663a15680761e272efeafe061ea9a898319265a145b494b18.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269010417 watch_fd=11 name=/var/log/containers/ks-console-65f4d44d88-hzw9b_kubesphere-system_ks-console-f600455c3af064c53adc8eb7e5412505c4d0e50362ab11b8e59d2e71b552b0f1.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269514179 watch_fd=12 name=/var/log/containers/ks-controller-manager-79c7dc79f5-j9fz5_kubesphere-system_ks-controller-manager-a8c35247c7cda868d3c023616026515690d2fedf2abb0f9367d1dc338103f7af.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403995814 watch_fd=13 name=/var/log/containers/ks-events-ruler-575669b4-mh2fn_kubesphere-logging-system_config-reloader-2d51bab0babdd47864ec235efdbd69d2075cc8bf72c3642449be69f68aa1f0d5.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=962171 watch_fd=14 name=/var/log/containers/ks-events-ruler-575669b4-mh2fn_kubesphere-logging-system_events-ruler-da24092a12c4fc0d0aab8a02c7fc7cb1e0e82d15a924717222bdbadee7935c84.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134583424 watch_fd=15 name=/var/log/containers/kube-apiserver-ks-k8s-master-0_kube-system_kube-apiserver-b4e97a525442956fe7612d27367a184ee09d65a56464149ec913ea004db0f1ef.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134580575 watch_fd=16 name=/var/log/containers/kube-controller-manager-ks-k8s-master-0_kube-system_kube-controller-manager-a07d1c5ed4108d98de2ba322b47939d5007c9d266a4ff558d57ec87ab10b2d54.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268955289 watch_fd=17 name=/var/log/containers/kube-proxy-bc59k_kube-system_kube-proxy-79144efa6b42a0f9636132538cc4ade6665269fea5903f3e1e0be05f14376d7c.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403715964 watch_fd=18 name=/var/log/containers/kube-scheduler-ks-k8s-master-0_kube-system_kube-scheduler-1a2ffcf4000a9bb0fa526fcfa1970e69465dfe0bb4c635a4d95f50f2e27ef763.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403855647 watch_fd=19 name=/var/log/containers/minio-859cb4d777-mpsk9_kubesphere-system_minio-d76f9516fa485cd24851afccfb2c85fd74ce894ba580af98703704860e1c00ed.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=413299 watch_fd=20 name=/var/log/containers/node-exporter-wwrpf_kubesphere-monitoring-system_kube-rbac-proxy-e35229db1e46e8b7b0a7d5a3cd581107102a3531b31d2e32d7e787a118c82896.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=375870 watch_fd=21 name=/var/log/containers/node-exporter-wwrpf_kubesphere-monitoring-system_node-exporter-e5cc5a4cc937cf9de149dd25bd6e8441ca176bf26cb2a214dd0b47627e7d8861.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134581094 watch_fd=22 name=/var/log/containers/nodelocaldns-sp59h_kube-system_node-cache-8b5bdf5a116af255f979a4a349c168257b632d24805d5f257a642dd97b0d53a1.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=463110 watch_fd=23 name=/var/log/containers/prometheus-k8s-0_kubesphere-monitoring-system_config-reloader-1378a8d9cd7a96334cb15adeafb468497ece0a9a600a3193fb80b60bc5b7e9b5.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135190584 watch_fd=24 name=/var/log/containers/prometheus-k8s-0_kubesphere-monitoring-system_prometheus-26e94cf0828cd1bd9c5da45217b7f3e654a7a41ce0dc5943375b1644d1a6a002.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134738131 watch_fd=25 name=/var/log/containers/prometheus-k8s-0_kubesphere-monitoring-system_prometheus-97de8b484747bae4aeac54ffd9b75c1ddc9582a28c768ebc9be395390bd031c3.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268956011 watch_fd=26 name=/var/log/containers/redis-ha-haproxy-868fdbddd4-kqrl5_kubesphere-system_config-init-0fb775710352218a66aec02ea2a736892c33df43dd4206e155e0bce6af4aba63.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373969 watch_fd=27 name=/var/log/containers/redis-ha-haproxy-868fdbddd4-kqrl5_kubesphere-system_haproxy-6c15749c02904875f5c1a280a42a908baa074ccffd3365f40b2692b6fe60acf5.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403792092 watch_fd=28 name=/var/log/containers/redis-ha-server-2_kubesphere-system_config-init-fa7c66040a53bef1c3c9b1844a4b25870835a6147cd83abf914ebb129a036536.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=412431 watch_fd=29 name=/var/log/containers/redis-ha-server-2_kubesphere-system_redis-6b19bb18f79c55cca6022bd8caa13a33d8c57386897db47fc011fd9fbe589b8a.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134738124 watch_fd=30 name=/var/log/containers/redis-ha-server-2_kubesphere-system_sentinel-7b2cbde0a38f45b692dd1679912bb9ba4bdfc358667c32164592a0e7ab2a87a6.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=467559 watch_fd=31 name=/var/log/containers/thanos-ruler-kubesphere-1_kubesphere-monitoring-system_config-reloader-28aef566c79b3cc073a67787c6ad44902c88930f069ab26cf5657aa2d1235108.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403860482 watch_fd=32 name=/var/log/containers/thanos-ruler-kubesphere-1_kubesphere-monitoring-system_thanos-ruler-24cfbc7280d54f3289c4ae70d0bb25a4964dd2f759e8bddefad3cf92070b1903.log
     
      [2022/04/20 03:43:25] [ info] [input:tail:tail.2] inotify_fs_add(): inode=412659 watch_fd=33 name=/var/log/containers/fluent-bit-vv8pb_kubesphere-logging-system_fluent-bit-dfd9ef652ece992d5f6b9a57c84288d19ebae1d8dcbd8618758f956e35370182.log
     
      [2022/04/20 03:43:30] [ warn] [engine] failed to flush chunk '18-1650426205.971801521.flb', retry in 9 seconds: task_id=1, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 03:43:30] [ warn] [engine] failed to flush chunk '18-1650426205.162979514.flb', retry in 6 seconds: task_id=0, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 03:43:30] [ warn] [engine] failed to flush chunk '18-1650426206.629361363.flb', retry in 9 seconds: task_id=2, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 03:43:35] [ warn] [engine] failed to flush chunk '18-1650426210.70782057.flb', retry in 10 seconds: task_id=3, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 03:43:36] [ warn] [engine] chunk '18-1650426205.162979514.flb' cannot be retried: task_id=0, input=tail.2 > output=es.0
     
      [2022/04/20 03:43:39] [ warn] [engine] chunk '18-1650426206.629361363.flb' cannot be retried: task_id=2, input=tail.2 > output=es.0
     
      [2022/04/20 03:43:39] [ warn] [engine] chunk '18-1650426205.971801521.flb' cannot be retried: task_id=1, input=tail.2 > output=es.0
     
      [2022/04/20 03:43:40] [ warn] [engine] failed to flush chunk '18-1650426215.131578181.flb', retry in 11 seconds: task_id=0, input=tail.2 > output=es.0 (out_id=0)
     ```

   - 分析日志发现**Domain name not found**这个报错已经不存在了，说明es地址配置变更了，但是还有其他报错我们继续分析。

   - 我们发现现在运行的**fluent-bit**还是5天前创建的（好吧，可以看到这个事情已经困扰了我5天了），尝试使用**重启大法**，没准pod在初始创建的时候会做什么特殊操作呢。

   - ![kubesphere-daemonsets-flunet-bit](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-daemonsets-flunet-bit.png)

   - 点击**更多操作**，**重新创建**

   - 不知道为啥 只给我重建了一个，不过先不管了，先看日志

   - ![kubesphere-daemonsets-flunet-bit-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-daemonsets-flunet-bit-1.png)

   - 重建完成后，我们看看pod的新启动的日志,发现跟刚才差不多。

   - ```yaml
     容器日志
     
      level=info msg="Fluent bit started"
     
      Fluent Bit v1.8.3
     
      * Copyright (C) 2019-2021 The Fluent Bit Authors
     
      * Copyright (C) 2015-2018 Treasure Data
     
      * Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
     
      * https://fluentbit.io
     
      
     
      [2022/04/20 04:07:18] [ info] [engine] started (pid=15)
     
      [2022/04/20 04:07:18] [ info] [storage] version=1.1.1, initializing...
     
      [2022/04/20 04:07:18] [ info] [storage] in-memory
     
      [2022/04/20 04:07:18] [ info] [storage] normal synchronization mode, checksum disabled, max_chunks_up=128
     
      [2022/04/20 04:07:18] [ info] [cmetrics] version=0.1.6
     
      [2022/04/20 04:07:18] [ info] [filter:kubernetes:kubernetes.4] https=1 host=kubernetes.default.svc port=443
     
      [2022/04/20 04:07:18] [ info] [filter:kubernetes:kubernetes.4] local POD info OK
     
      [2022/04/20 04:07:18] [ info] [filter:kubernetes:kubernetes.4] testing connectivity with API server...
     
      [2022/04/20 04:07:18] [ info] [filter:kubernetes:kubernetes.4] connectivity OK
     
      [2022/04/20 04:07:18] [ info] [sp] stream processor started
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=438144 watch_fd=1 name=/var/log/containers/alertmanager-main-0_kubesphere-monitoring-system_alertmanager-da4bf030d6c508ccc7ae8b0d25e9a5d3f4167635b3b1e7c9132f43ca18e26759.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134775019 watch_fd=2 name=/var/log/containers/alertmanager-main-0_kubesphere-monitoring-system_config-reloader-0e9ea4d0750bec9cca7a04090fb8ead8d05101fa2fa86e36378c5305483895ad.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135308807 watch_fd=3 name=/var/log/containers/calico-node-grzs6_kube-system_calico-node-e110ff3989cef4695508721c54abe41c82b020e485a04d51ee067c3749b40ff0.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268955566 watch_fd=4 name=/var/log/containers/calico-node-grzs6_kube-system_flexvol-driver-bf0279ba75c245f871727d85546e8e135b9c1647ca3af318a13b961c9ab63fef.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373353 watch_fd=5 name=/var/log/containers/calico-node-grzs6_kube-system_install-cni-2020df6843b42253a7fe3cc71db6a8c380f76dd719985daec3217155a3b99fb9.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268955530 watch_fd=6 name=/var/log/containers/calico-node-grzs6_kube-system_upgrade-ipam-d31b5da3acd65ea3a1630576cf8a3527ce89f75a713331f0f79f173180ba2807.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134745703 watch_fd=7 name=/var/log/containers/default-http-backend-5bf68ff9b8-qmsqj_kubesphere-controls-system_default-http-backend-3024b370ba28e45f0d739157883e1c05b0450ca6dd6fac17ff86704c25e307df.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=404112903 watch_fd=8 name=/var/log/containers/devops-27507060-slntp_kubesphere-devops-system_pipeline-run-gc-67341c0a99432db725e08104b44a9d3b47ab96df5622d7fa42cc03bb51772360.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=404114497 watch_fd=9 name=/var/log/containers/devops-27507090-7b7n9_kubesphere-devops-system_pipeline-run-gc-23c2506ed4daa1f43fe43eb25a1b30a799afd4b1a41ead8f514197b9f0d321e1.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=404114514 watch_fd=10 name=/var/log/containers/devops-27507120-bwgkg_kubesphere-devops-system_pipeline-run-gc-af8382bfca6e901ec451e15bb1fac7d89619e8c3c15e780672bb4824af823c19.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135613174 watch_fd=11 name=/var/log/containers/devops-controller-98975d478-td5xx_kubesphere-devops-system_init-config-156a981f1398223c968105b4feb24dbbcee97403844ca623159bfed4cd647e85.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135675036 watch_fd=12 name=/var/log/containers/devops-controller-98975d478-td5xx_kubesphere-devops-system_ks-devops-953f7eff3a2cef2ca441be9b57024286c1e058af6033de0a0d9453649e79fe8f.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=404546115 watch_fd=13 name=/var/log/containers/devops-jenkins-5d744f66b9-cfnq5_kubesphere-devops-system_copy-default-config-8970ad7a6668683a6e73c80479b92e6cec3d8280add9983b6b911db708e2b46f.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135827943 watch_fd=14 name=/var/log/containers/devops-jenkins-5d744f66b9-cfnq5_kubesphere-devops-system_devops-jenkins-ebe951dce6b04d70e0982a50a27ea1c84c46f6fd5bb324f8d98a6d3c8d98f03d.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135304775 watch_fd=15 name=/var/log/containers/elasticsearch-logging-curator-elasticsearch-curator-275050tdf82_kubesphere-logging-system_elasticsearch-curator-150f0f2dfaf91b75c2f357d4a610c6ff435325bf7b6946d0972299d297c99aba.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=2740314 watch_fd=16 name=/var/log/containers/elasticsearch-logging-curator-elasticsearch-curator-275064gfsnt_kubesphere-logging-system_elasticsearch-curator-b057cc4fa0f64009cc4e8862b105275f5180dec25ffd85aafecd3e38b3fe8607.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=1448183 watch_fd=17 name=/var/log/containers/fluentbit-operator-745bf5559f-lgq9p_kubesphere-logging-system_fluentbit-operator-6e0515f3ec61269e573610598fb0fafa1f4b6c7b6c9737cc5149f4dce73f0726.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269964578 watch_fd=18 name=/var/log/containers/fluentbit-operator-745bf5559f-lgq9p_kubesphere-logging-system_setenv-160f20273aae37dead28b4ee01faaf8417e66c8679253218fa1d69f7f67d9986.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269092123 watch_fd=19 name=/var/log/containers/ks-apiserver-574966976-zlzdn_kubesphere-system_ks-apiserver-0f59c89e26d48f7e7a33434412136d6f2c6e36b3cb9a6c551b5921a9b3c648c3.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269001282 watch_fd=20 name=/var/log/containers/ks-console-65f4d44d88-6zh2n_kubesphere-system_ks-console-473625ee71e3c79256031c7e17b4c7e3cd8c01a32674dabcbc278b113759b86b.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403767194 watch_fd=21 name=/var/log/containers/ks-controller-manager-79c7dc79f5-b4887_kubesphere-system_ks-controller-manager-8c0f54653d9f6f3f5647cdf6c1491d7f3676a31ee06a61539d1ad22ac9a9f0e3.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=404597475 watch_fd=22 name=/var/log/containers/ks-events-operator-5944645757-kqt65_kubesphere-logging-system_events-operator-2d6a462d820e369d58f4add588f2d8798579505db12141cd711c0df6b8adf323.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=2531381 watch_fd=23 name=/var/log/containers/ks-events-ruler-575669b4-lnpxc_kubesphere-logging-system_config-reloader-135d523190a321b1c1b5164f065f10bccb803d1a9b01cfa6c59285f00175fc0e.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=270078583 watch_fd=24 name=/var/log/containers/ks-events-ruler-575669b4-lnpxc_kubesphere-logging-system_events-ruler-0698fb3748bfa98dc345dfabcbd44eb456b71c36bdffb56222643a25040cd878.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134910338 watch_fd=25 name=/var/log/containers/kube-apiserver-ks-k8s-master-2_kube-system_kube-apiserver-06d36c6647da955d83f369af80185dfa57801bd6397c54b833595425abd937d1.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269993224 watch_fd=26 name=/var/log/containers/kube-auditing-operator-84857bf967-kg7bs_kubesphere-logging-system_kube-auditing-operator-086ff221b47aa055daea0bbde34869f53a266129b11c90e8123fb1d43506f46d.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373019 watch_fd=27 name=/var/log/containers/kube-controller-manager-ks-k8s-master-2_kube-system_kube-controller-manager-dc6148b02773d71cd3ae73afc259d8989fc8fa3d5497a1edc5fb6c28561e4b93.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134581155 watch_fd=28 name=/var/log/containers/kube-proxy-bzx69_kube-system_kube-proxy-d130696f10d7ea91f909720c178b90b68b553dce873030fba8e7daf8fcd1b26b.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134580882 watch_fd=29 name=/var/log/containers/kube-scheduler-ks-k8s-master-2_kube-system_kube-scheduler-7fe7efb1fbd3f599ebc530f7ca4379b02aa0bc0e8b23c269cee79bec36c6e103.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=664327 watch_fd=30 name=/var/log/containers/kubectl-admin-6667774bb-mv9fm_kubesphere-controls-system_kubectl-a15f90842b782a08fe6c5f944974e2551a5995bca4116d741537b7d9b7531ef1.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135322058 watch_fd=31 name=/var/log/containers/logsidecar-injector-deploy-5fb6fdc6dd-fxsbf_kubesphere-logging-system_config-reloader-7e5fb4de2d6433568d612d3092d23b38beca28a5089106ad84a5f9b33258c3fd.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=404567916 watch_fd=32 name=/var/log/containers/logsidecar-injector-deploy-5fb6fdc6dd-fxsbf_kubesphere-logging-system_logsidecar-injector-5e0181bde98c6444d5de2e9aac6ab459bbf957c09edc1ba06be473d0e24ce21f.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134969612 watch_fd=33 name=/var/log/containers/node-exporter-8tldd_kubesphere-monitoring-system_kube-rbac-proxy-c6a4a0dc8ce6ef8f6b73a940ee8473e20c08c1b85fefb679003c4a08c1889c09.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134778346 watch_fd=34 name=/var/log/containers/node-exporter-8tldd_kubesphere-monitoring-system_node-exporter-c0458098c9d6d132a765c753b3a7edc23887ecd48e457be393fa359ca731eef2.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269110037 watch_fd=35 name=/var/log/containers/notification-manager-deployment-78664576cb-2zqhb_kubesphere-monitoring-system_notification-manager-8a6449e8a99c56341b5a196251b99c95893b49ccf7b97c239e1da6b5fe899aca.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135304736 watch_fd=36 name=/var/log/containers/notification-manager-deployment-78664576cb-2zqhb_kubesphere-monitoring-system_tenant-4851fb71b6d33755ea467a0745abb4f0c7757e840d18a3ab34b7c4d5bcb61fbb.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135203266 watch_fd=37 name=/var/log/containers/notification-manager-deployment-78664576cb-qdzg8_kubesphere-monitoring-system_notification-manager-26afc727ce8fff65033c5432d74fd8da02dc297de6d6f9013eca432a6ed14a67.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=404111997 watch_fd=38 name=/var/log/containers/notification-manager-deployment-78664576cb-qdzg8_kubesphere-monitoring-system_tenant-b7dec63c57fdb165777eddc9ac3f33a3bbb8d752c5508e9bd64b53f7fa7c1667.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403837581 watch_fd=39 name=/var/log/containers/notification-manager-operator-7d44854f54-5x8ml_kubesphere-monitoring-system_kube-rbac-proxy-e7e058ebc7a31bfe48f950473ce4e5945ba63117b970c4817aed6f341fc6a1c5.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269104667 watch_fd=40 name=/var/log/containers/notification-manager-operator-7d44854f54-5x8ml_kubesphere-monitoring-system_notification-manager-operator-6cbd966b72900f3e53460e0f599d6f29e2d12d81e480ae091e951f3a718e26e1.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=404140622 watch_fd=41 name=/var/log/containers/openldap-0_kubesphere-system_openldap-ha-34603befcd35f87ef4f343d4774193d968eb285669c68e617a6df561976b2c90.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=404112865 watch_fd=42 name=/var/log/containers/openldap-0_kubesphere-system_openldap-ha-8bb8c1a38b9a28adee5ec2b98c64c44987f3958d5f649a2c0f8572988812edba.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135305918 watch_fd=43 name=/var/log/containers/openpitrix-import-job-56b4v_kubesphere-system_import-6b9e87ef772f7e985b172f6da40a44c14e3ebede2cb726299b1260b2f254772a.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135305893 watch_fd=44 name=/var/log/containers/openpitrix-import-job-56b4v_kubesphere-system_wait-apiserver-9d2f65fc31598f6bd4a3097919149d24b0fa8b7a26e2b3892b0486a38f8264d0.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=437223 watch_fd=45 name=/var/log/containers/prometheus-operator-5c5db79546-fhz5t_kubesphere-monitoring-system_kube-rbac-proxy-57f49887b2934e7989c4bdf830eb7c1f963f8ad9da453897e63545e541cfa1b1.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269001982 watch_fd=46 name=/var/log/containers/prometheus-operator-5c5db79546-fhz5t_kubesphere-monitoring-system_prometheus-operator-95c8b3de232d26e297314b8e37859c4c65dd46748467ff2bbf7d7b85df90858a.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403724186 watch_fd=47 name=/var/log/containers/redis-ha-haproxy-868fdbddd4-jhxvc_kubesphere-system_config-init-10b5cff84fe2c3e566783fea86136e3f90d1e2fc87dcebd61cff74592d454e4b.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134685563 watch_fd=48 name=/var/log/containers/redis-ha-haproxy-868fdbddd4-jhxvc_kubesphere-system_haproxy-dc89c72b1e221d94b82ee32bcb0eec43bbbade1c1dfa6ae6b8272b339c1014bf.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268975519 watch_fd=49 name=/var/log/containers/redis-ha-server-0_kubesphere-system_config-init-a24eeaae3519f7a0be9cb25a58f28676e978dbdcb155d64de83c6d2c564bcb84.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403761176 watch_fd=50 name=/var/log/containers/redis-ha-server-0_kubesphere-system_redis-1952b51c7b5bf653b3b4f308d0447fca0395267f654b42ad936f5446a8f0ba11.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403763877 watch_fd=51 name=/var/log/containers/redis-ha-server-0_kubesphere-system_sentinel-93ec7fdfd37546554e9ee20cad99fc79eb1120658f041c3ac3f8ae790ba9f3e0.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135616217 watch_fd=52 name=/var/log/containers/s2ioperator-0_kubesphere-devops-system_manager-ec0eef93d54b446690c5afb45fc20c145e8555f0efeac367b317ca3747e58a15.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268956363 watch_fd=53 name=/var/log/containers/snapshot-controller-0_kube-system_snapshot-controller-8efdaf2c22056e15431868c285909845a9a50220a8599428f13827c2a9999605.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.3] inotify_fs_add(): inode=270078314 watch_fd=1 name=/var/log/containers/kube-auditing-webhook-deploy-64cfb8c9f8-c65cb_kubesphere-logging-system_kube-auditing-webhook-2315ba3f2e910546235bebb5c2cef37419d6bbb6568f0073f94b7cc9cec5882e.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.3] inotify_fs_add(): inode=404597510 watch_fd=2 name=/var/log/containers/kube-auditing-webhook-deploy-64cfb8c9f8-j8w5d_kubesphere-logging-system_kube-auditing-webhook-27248c5f2dc6b8e16f178cb6028731bd4d752a99b558558b8cbc38ebc872996d.log
     
      [2022/04/20 04:07:18] [ info] [input:tail:tail.2] inotify_fs_add(): inode=404114523 watch_fd=54 name=/var/log/containers/fluent-bit-79w92_kubesphere-logging-system_fluent-bit-6fca993aec2186244af2f1632d8fcd563be88aeddf0f892fe05908712f4143e1.log
     
      [2022/04/20 04:07:22] [ warn] [engine] failed to flush chunk '15-1650427638.306801183.flb', retry in 10 seconds: task_id=0, input=systemd.0 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:22] [ warn] [engine] failed to flush chunk '15-1650427638.921453506.flb', retry in 6 seconds: task_id=2, input=systemd.1 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:22] [ warn] [engine] failed to flush chunk '15-1650427638.395905124.flb', retry in 6 seconds: task_id=1, input=systemd.1 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:22] [ warn] [engine] failed to flush chunk '15-1650427639.998272919.flb', retry in 9 seconds: task_id=4, input=systemd.1 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:22] [ warn] [engine] failed to flush chunk '15-1650427639.430712081.flb', retry in 10 seconds: task_id=3, input=systemd.1 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:22] [ warn] [engine] failed to flush chunk '15-1650427638.433892561.flb', retry in 11 seconds: task_id=6, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:22] [ warn] [engine] failed to flush chunk '15-1650427640.508747628.flb', retry in 8 seconds: task_id=5, input=systemd.1 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:22] [ warn] [engine] failed to flush chunk '15-1650427638.591363944.flb', retry in 9 seconds: task_id=7, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:27] [ warn] [engine] failed to flush chunk '15-1650427642.916578491.flb', retry in 10 seconds: task_id=8, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:28] [ warn] [engine] chunk '15-1650427638.921453506.flb' cannot be retried: task_id=2, input=systemd.1 > output=es.0
     
      [2022/04/20 04:07:28] [ warn] [engine] chunk '15-1650427638.395905124.flb' cannot be retried: task_id=1, input=systemd.1 > output=es.0
     
      [2022/04/20 04:07:30] [ warn] [engine] chunk '15-1650427640.508747628.flb' cannot be retried: task_id=5, input=systemd.1 > output=es.0
     
      [2022/04/20 04:07:31] [ warn] [engine] chunk '15-1650427639.998272919.flb' cannot be retried: task_id=4, input=systemd.1 > output=es.0
     
      [2022/04/20 04:07:31] [ warn] [engine] chunk '15-1650427638.591363944.flb' cannot be retried: task_id=7, input=tail.2 > output=es.0
     
      [2022/04/20 04:07:32] [ warn] [engine] failed to flush chunk '15-1650427650.364418985.flb', retry in 7 seconds: task_id=2, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:32] [ warn] [engine] failed to flush chunk '15-1650427647.894424287.flb', retry in 11 seconds: task_id=1, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:32] [ warn] [engine] failed to flush chunk '15-1650427651.290039243.flb', retry in 11 seconds: task_id=4, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:32] [ warn] [engine] failed to flush chunk '15-1650427652.113363794.flb', retry in 10 seconds: task_id=5, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:32] [ warn] [engine] chunk '15-1650427638.306801183.flb' cannot be retried: task_id=0, input=systemd.0 > output=es.0
     
      [2022/04/20 04:07:32] [ warn] [engine] chunk '15-1650427639.430712081.flb' cannot be retried: task_id=3, input=systemd.1 > output=es.0
     
      [2022/04/20 04:07:33] [ warn] [engine] chunk '15-1650427638.433892561.flb' cannot be retried: task_id=6, input=tail.2 > output=es.0
     
      [2022/04/20 04:07:37] [ warn] [engine] chunk '15-1650427642.916578491.flb' cannot be retried: task_id=8, input=tail.2 > output=es.0
     
      [2022/04/20 04:07:37] [ warn] [engine] failed to flush chunk '15-1650427652.895622405.flb', retry in 10 seconds: task_id=0, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:39] [ warn] [engine] chunk '15-1650427650.364418985.flb' cannot be retried: task_id=2, input=tail.2 > output=es.0
     
      [2022/04/20 04:07:42] [ warn] [engine] chunk '15-1650427652.113363794.flb' cannot be retried: task_id=5, input=tail.2 > output=es.0
     
      [2022/04/20 04:07:42] [ warn] [engine] failed to flush chunk '15-1650427657.894452234.flb', retry in 8 seconds: task_id=2, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:42] [ warn] [engine] failed to flush chunk '15-1650427660.999669706.flb', retry in 8 seconds: task_id=3, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/20 04:07:43] [ warn] [engine] chunk '15-1650427647.894424287.flb' cannot be retried: task_id=1, input=tail.2 > output=es.0
     
      [2022/04/20 04:07:43] [ warn] [engine] chunk '15-1650427651.290039243.flb' cannot be retried: task_id=4, input=tail.2 > output=es.0
     ```

3. 问题排查第三环节，检查fluent-bit

   - 打开百度，搜索**failed to flush chunk**

   - ![image-20220420163534855](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220420163534855.png)

   - 搜索出来的结果，凭直觉没啥用处，真不想说啥了，直接换人

   - 打开**Bing**，搜索**failed to flush chunk**

   - ![image-20220420163607854](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220420163607854.png)

   - 点开[第一个参考链接](https://github.com/fluent/fluent-bit/issues/3301),看了一圈感觉就是再说需要开启一个参数**Trace_Error On**，才能看的更清楚。

   - 咱先不开启，先看看fluentd-bit是否跟ES服务器有通讯，通过tcp抓包可以看到是有数据推送的

   - ```shell
     [root@glusterfs-node-0 logs]# tcpdump -i any port 9200 -s 0
     tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
     listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
     14:42:31.713955 IP ks-k8s-master-1.57144 > glusterfs-node-0.wap-wsp: Flags [S], seq 4284553181, win 28000, options [mss 1400,sackOK,TS val 1006807002 ecr 0,nop,wscale 7], length 0
     14:42:31.713987 IP glusterfs-node-0.wap-wsp > ks-k8s-master-1.57144: Flags [S.], seq 3050954572, ack 4284553182, win 28960, options [mss 1460,sackOK,TS val 1446355813 ecr 1006807002,nop,wscale 7], length 0
     14:42:31.714478 IP ks-k8s-master-1.57144 > glusterfs-node-0.wap-wsp: Flags [.], ack 1, win 219, options [nop,nop,TS val 1006807002 ecr 1446355813], length 0
     14:42:31.714535 IP ks-k8s-master-1.57146 > glusterfs-node-0.wap-wsp: Flags [S], seq 763700519, win 28000, options [mss 1400,sackOK,TS val 1006807002 ecr 0,nop,wscale 7], length 0
     14:42:31.714562 IP glusterfs-node-0.wap-wsp > ks-k8s-master-1.57146: Flags [S.], seq 141769299, ack 763700520, win 28960, options [mss 1460,sackOK,TS val 1446355813 ecr 1006807002,nop,wscale 7], length 0
     14:42:31.714931 IP ks-k8s-master-1.57146 > glusterfs-node-0.wap-wsp: Flags [.], ack 1, win 219, options [nop,nop,TS val 1006807003 ecr 1446355813], length 0
     14:42:31.720484 IP ks-k8s-master-1.57144 > glusterfs-node-0.wap-wsp: Flags [P.], seq 1:305, ack 1, win 219, options [nop,nop,TS val 1006807008 ecr 1446355813], length 304
     14:42:31.720511 IP glusterfs-node-0.wap-wsp > ks-k8s-master-1.57144: Flags [.], ack 305, win 235, options [nop,nop,TS val 1446355819 ecr 1006807008], length 0
     14:42:31.720882 IP ks-k8s-master-1.57146 > glusterfs-node-0.wap-wsp: Flags [P.], seq 1:305, ack 1, win 219, options [nop,nop,TS val 1006807008 ecr 1446355813], length 304
     14:42:31.720915 IP glusterfs-node-0.wap-wsp > ks-k8s-master-1.57146: Flags [.], ack 305, win 235, options [nop,nop,TS val 1446355820 ecr 1006807008], length 0
     14:42:31.721259 IP glusterfs-node-0.wap-wsp > ks-k8s-master-1.57146: Flags [P.], seq 1:706, ack 305, win 235, options [nop,nop,TS val 1446355820 ecr 1006807008], length 705
     14:42:31.721272 IP glusterfs-node-0.wap-wsp > ks-k8s-master-1.57144: Flags [P.], seq 1:750, ack 305, win 235, options [nop,nop,TS val 1446355820 ecr 1006807008], length 749
     14:42:31.721327 IP glusterfs-node-0.wap-wsp > ks-k8s-master-1.57146: Flags [F.], seq 706, ack 305, win 235, options [nop,nop,TS val 1446355820 ecr 1006807008], length 0
     14:42:31.721356 IP glusterfs-node-0.wap-wsp > ks-k8s-master-1.57144: Flags [F.], seq 750, ack 305, win 235, options [nop,nop,TS val 1446355820 ecr 1006807008], length 0
     14:42:31.721666 IP ks-k8s-master-1.57146 > glusterfs-node-0.wap-wsp: Flags [.], ack 706, win 230, options [nop,nop,TS val 1006807009 ecr 1446355820], length 0
     14:42:31.721707 IP ks-k8s-master-1.57144 > glusterfs-node-0.wap-wsp: Flags [.], ack 750, win 231, options [nop,nop,TS val 1006807009 ecr 1446355820], length 0
     14:42:31.721951 IP ks-k8s-master-1.57146 > glusterfs-node-0.wap-wsp: Flags [R.], seq 305, ack 707, win 230, options [nop,nop,TS val 1006807009 ecr 1446355820], length 0
     14:42:31.722011 IP ks-k8s-master-1.57144 > glusterfs-node-0.wap-wsp: Flags [R.], seq 305, ack 751, win 231, options [nop,nop,TS val 1006807010 ecr 1446355820], length 0
     14:42:31.846578 IP ks-k8s-master-2.34902 > glusterfs-node-0.wap-wsp: Flags [S], seq 2846110148, win 28000, options [mss 1400,sackOK,TS val 1006745001 ecr 0,nop,wscale 7], length 0
     14:42:31.846621 IP glusterfs-node-0.wap-wsp > ks-k8s-master-2.34902: Flags [S.], seq 2264454988, ack 2846110149, win 28960, options [mss 1460,sackOK,TS val 1446355945 ecr 1006745001,nop,wscale 7], length 0
     14:42:31.847064 IP ks-k8s-master-2.34902 > glusterfs-node-0.wap-wsp: Flags [.], ack 1, win 219, options [nop,nop,TS val 1006745002 ecr 1446355945], length 0
     14:42:31.851910 IP ks-k8s-master-2.34902 > glusterfs-node-0.wap-wsp: Flags [P.], seq 1:305, ack 1, win 219, options [nop,nop,TS val 1006745007 ecr 1446355945], length 304
     14:42:31.851934 IP glusterfs-node-0.wap-wsp > ks-k8s-master-2.34902: Flags [.], ack 305, win 235, options [nop,nop,TS val 1446355951 ecr 1006745007], length 0
     14:42:31.852261 IP glusterfs-node-0.wap-wsp > ks-k8s-master-2.34902: Flags [P.], seq 1:334, ack 305, win 235, options [nop,nop,TS val 1446355951 ecr 1006745007], length 333
     14:42:31.852307 IP glusterfs-node-0.wap-wsp > ks-k8s-master-2.34902: Flags [F.], seq 334, ack 305, win 235, options [nop,nop,TS val 1446355951 ecr 1006745007], length 0
     14:42:31.852575 IP ks-k8s-master-2.34902 > glusterfs-node-0.wap-wsp: Flags [.], ack 334, win 228, options [nop,nop,TS val 1006745007 ecr 1446355951], length 0
     14:42:31.852595 IP ks-k8s-master-2.34902 > glusterfs-node-0.wap-wsp: Flags [R.], seq 305, ack 335, win 228, options [nop,nop,TS val 1006745007 ecr 1446355951], length 0
     14:42:32.025492 IP ks-k8s-master-0.40940 > glusterfs-node-0.wap-wsp: Flags [S], seq 2794210096, win 28000, options [mss 1400,sackOK,TS val 1006811001 ecr 0,nop,wscale 7], length 0
     14:42:32.025534 IP glusterfs-node-0.wap-wsp > ks-k8s-master-0.40940: Flags [S.], seq 3740534993, ack 2794210097, win 28960, options [mss 1460,sackOK,TS val 1446356124 ecr 1006811001,nop,wscale 7], length 0
     14:42:32.026129 IP ks-k8s-master-0.40940 > glusterfs-node-0.wap-wsp: Flags [.], ack 1, win 219, options [nop,nop,TS val 1006811002 ecr 1446356124], length 0
     14:42:32.032593 IP ks-k8s-master-0.40940 > glusterfs-node-0.wap-wsp: Flags [P.], seq 1:305, ack 1, win 219, options [nop,nop,TS val 1006811008 ecr 1446356124], length 304
     14:42:32.032615 IP glusterfs-node-0.wap-wsp > ks-k8s-master-0.40940: Flags [.], ack 305, win 235, options [nop,nop,TS val 1446356131 ecr 1006811008], length 0
     14:42:32.033329 IP glusterfs-node-0.wap-wsp > ks-k8s-master-0.40940: Flags [P.], seq 1:386, ack 305, win 235, options [nop,nop,TS val 1446356132 ecr 1006811008], length 385
     14:42:32.033396 IP glusterfs-node-0.wap-wsp > ks-k8s-master-0.40940: Flags [F.], seq 386, ack 305, win 235, options [nop,nop,TS val 1446356132 ecr 1006811008], length 0
     14:42:32.033643 IP ks-k8s-master-0.40940 > glusterfs-node-0.wap-wsp: Flags [.], ack 386, win 228, options [nop,nop,TS val 1006811010 ecr 1446356132], length 0
     14:42:32.033718 IP ks-k8s-master-0.40940 > glusterfs-node-0.wap-wsp: Flags [R.], seq 305, ack 387, win 228, options [nop,nop,TS val 1006811010 ecr 1446356132], length 0
     14:42:33.025645 IP ks-k8s-master-0.40970 > glusterfs-node-0.wap-wsp: Flags [S], seq 556017134, win 28000, options [mss 1400,sackOK,TS val 1006812001 ecr 0,nop,wscale 7], length 0
     14:42:33.025700 IP glusterfs-node-0.wap-wsp > ks-k8s-master-0.40970: Flags [S.], seq 3957625474, ack 556017135, win 28960, options [mss 1460,sackOK,TS val 1446357124 ecr 1006812001,nop,wscale 7], length 0
     14:42:33.026255 IP ks-k8s-master-0.40970 > glusterfs-node-0.wap-wsp: Flags [.], ack 1, win 219, options [nop,nop,TS val 1006812002 ecr 1446357124], length 0
     14:42:33.031462 IP ks-k8s-master-0.40970 > glusterfs-node-0.wap-wsp: Flags [P.], seq 1:305, ack 1, win 219, options [nop,nop,TS val 1006812007 ecr 1446357124], length 304
     14:42:33.031493 IP glusterfs-node-0.wap-wsp > ks-k8s-master-0.40970: Flags [.], ack 305, win 235, options [nop,nop,TS val 1446357130 ecr 1006812007], length 0
     14:42:33.032411 IP glusterfs-node-0.wap-wsp > ks-k8s-master-0.40970: Flags [P.], seq 1:676, ack 305, win 235, options [nop,nop,TS val 1446357131 ecr 1006812007], length 675
     14:42:33.032547 IP glusterfs-node-0.wap-wsp > ks-k8s-master-0.40970: Flags [F.], seq 676, ack 305, win 235, options [nop,nop,TS val 1446357131 ecr 1006812007], length 0
     14:42:33.032789 IP ks-k8s-master-0.40970 > glusterfs-node-0.wap-wsp: Flags [.], ack 676, win 230, options [nop,nop,TS val 1006812009 ecr 1446357131], length 0
     14:42:33.032906 IP ks-k8s-master-0.40970 > glusterfs-node-0.wap-wsp: Flags [R.], seq 305, ack 677, win 230, options [nop,nop,TS val 1006812009 ecr 1446357131], length 0
     ^C
     45 packets captured
     45 packets received by filter
     0 packets dropped by kernel
     ```

   - 查看es集群中的index是否创建,结果发现索引并没有创建。

   - ```shell
     [root@glusterfs-node-0 logs]# curl -ulstack:'P@88w0rd' 192.168.9.95:9200/_cat/indices?v
     health status index            uuid                   pri rep docs.count docs.deleted store.size pri.store.size
     green  open   .geoip_databases jOKPAu5-S3yv0rZdZx4ufw   1   1         40           38    107.2mb         69.3mb
     ```

   - 那为啥没索引呢，默认日志是看不出来的，只能开启elasticsearch日志的debug模式了

   - ```shell
     [root@glusterfs-node-0 logs]# curl -ulstack:'P@88w0rd' -H "Content-Type: application/json" -X PUT  192.168.9.95:9200/_cluster/settings -d '{"transient" : {"logger._root":"DEBUG"} }'
     ```

   - 开启以后日志就丰富了很多，我们找找有用的信息

   - ```yaml
     [2022-04-20T15:23:25,633][DEBUG][r.suppressed             ] [es-node-0] path: /bad-request, params: {}
     java.lang.IllegalArgumentException: invalid version format: <8a>-»>¸Â×YÄÌ^Y^^HR·<84>ºÞ^@ÊËD!Ô^SÄÜA<9c>EÛº^@>^S^B^S^C^S^AÀ,À0^@<9f>Ì©Ì¨ÌªÀ+À/^@<9e>À$À(^@KÀ#À'^@GÀ
     
     [2022-04-20T15:23:25,630][DEBUG][r.suppressed             ] [es-node-0] path: /bad-request, params: {}
     java.lang.IllegalArgumentException: text is empty (possibly HTTP/0.9)
     
     ```

   - **DEBUG**级别的日志也就能看到这些了，再上个更狠的TRACE级别的，要是还不行就去开启**Fluent-bit**的参数。

   - ```shell
     [root@glusterfs-node-0 logs]# curl -ulstack:'P@88w0rd' -H "Content-Type: application/json" -X PUT  192.168.9.95:9200/_cluster/settings -d '{"transient" : {"logger._root":"TRACE"} }'
     ```

   - 分析日志，发现还是没有实质性的错误说明，也只是看到了错误的信息哪来的。

   - ```yaml
     [2022-04-20T16:50:18,577][TRACE][o.e.h.HttpTracer         ] [es-node-0] [40714][null][GET][/bad-request] received request from [Netty4HttpChannel{localAddress=/192.168.9.95:9200, remoteAddress=/192.168.9.93:34784}]
     java.lang.IllegalArgumentException: invalid version format: ¸Ñ^AÐ^E1<8b>HU^R0<88>^[)¶ËQT=<87>"}Þ¥AËÉ³ZY^@>^S^B^S^C^S^AÀ,À0^@<9f>Ì©Ì¨ÌªÀ+À/^@<9e>À$À(^@KÀ#À'^@GÀ
     
     [2022-04-20T16:50:18,549][TRACE][o.e.h.HttpTracer         ] [es-node-0] [40713][null][GET][/bad-request] received request from [Netty4HttpChannel{localAddress=/192.168.9.95:9200, remoteAddress=/192.168.9.92:49734}]
     java.lang.IllegalArgumentException: text is empty (possibly HTTP/0.9)
     
     ```

   - 先把ElasticSearch的日志级别改回默认的INFO，我们回去研究**Fluent-bit**

   - ```shell
     [root@glusterfs-node-0 logs]# curl -ulstack:'P@88w0rd' -H "Content-Type: application/json" -X PUT  192.168.9.95:9200/_cluster/settings -d '{"transient" : {"logger._root":"INFO"} }'
     ```

   - 返回KubeSphere的论坛，继续以关键字**外部es**搜索，发现一篇跟fluent bit有关的文档,[安装 日志插件后， fluent bit 启动失败，查询日志一直是0， 外部es也没有创建索引](https://kubesphere.com.cn/forum/d/5961-fluent-bit-0-es)

   - ![elasticsearch-error-2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/elasticsearch-error-2.png)

   - 该文中的fluent的报错信息跟我的不一样，先不管了，先看看他的配置跟我是否一样。https://github.com/kubesphere/kubesphere/issues/4425

   - ![image-20220420173658602](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220420173658602.png)

   - ![image-20220420173919350](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220420173919350.png)

     

   - 先看看我们有几个inputs

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl get inputs -n kubesphere-logging-system 
     NAME            AGE
     docker          5d5h
     kubelet         5d5h
     tail            5d5h
     tail-auditing   5d5h
     tail-events     5d5h
     ```

   - 先看看我们现在的**tail**配置

   - ```yaml
     [root@ks-k8s-master-0 ~]# kubectl get inputs -n kubesphere-logging-system tail -o yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Input
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Input","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"logging","logging.kubesphere.io/enabled":"true"},"name":"tail","namespace":"kubesphere-logging-system"},"spec":{"tail":{"db":"/fluent-bit/tail/pos.db","dbSync":"Normal","excludePath":"/var/log/containers/*_kubesphere-logging-system_events-exporter*.log,/var/log/containers/kube-auditing-webhook*_kubesphere-logging-system_kube-auditing-webhook*.log","memBufLimit":"5MB","parser":"docker","path":"/var/log/containers/*.log","refreshIntervalSeconds":10,"skipLongLines":true,"tag":"kube.*"}}}
       creationTimestamp: "2022-04-15T03:51:11Z"
       generation: 1
       labels:
         logging.kubesphere.io/component: logging
         logging.kubesphere.io/enabled: "true"
       name: tail
       namespace: kubesphere-logging-system
       resourceVersion: "1503222"
       uid: fdf165ed-5a03-42ef-bb0b-6f625b194a3f
     spec:
       tail:
         db: /fluent-bit/tail/pos.db
         dbSync: Normal
         excludePath: /var/log/containers/*_kubesphere-logging-system_events-exporter*.log,/var/log/containers/kube-auditing-webhook*_kubesphere-logging-system_kube-auditing-webhook*.log
         memBufLimit: 5MB
         parser: docker
         path: /var/log/containers/*.log
         refreshIntervalSeconds: 10
         skipLongLines: true
         tag: kube.*
     ```

   - 看现在的配置，跟github上的对比，发现配置存在无需调整。

   - 再看看我们现在的**filter-auditing**配置

   - ```yaml
     [root@ks-k8s-master-0 ~]# kubectl get  filters -n kubesphere-logging-system filter-auditing -o yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Filter
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Filter","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"auditing","logging.kubesphere.io/enabled":"true"},"name":"filter-auditing","namespace":"kubesphere-logging-system"},"spec":{"filters":[{"parser":{"keyName":"log","parser":"json"}},{"modify":{"conditions":[{"keyDoesNotExist":{"AuditID":""}}],"rules":[{"add":{"ignore":"true"}}]}},{"grep":{"exclude":"ignore true"}}],"match":"kube_auditing"}}
       creationTimestamp: "2022-04-15T03:51:23Z"
       generation: 1
       labels:
         logging.kubesphere.io/component: auditing
         logging.kubesphere.io/enabled: "true"
       name: filter-auditing
       namespace: kubesphere-logging-system
       resourceVersion: "1503307"
       uid: f0e88853-dc14-41f9-bcbf-f1a49fd121cf
     spec:
       filters:
       - parser:
           keyName: log
           parser: json
       - modify:
           conditions:
           - keyDoesNotExist:
               AuditID: ""
           rules:
           - add:
               ignore: "true"
       - grep:
           exclude: ignore true
       match: kube_auditing
     ```

   - 看现在的配置，跟github上的对比，发现配置存在无需调整。

   - 又无路可走了。。。清醒一下脑袋，进入下一环节

4. 问题排查第四环节，再探Fluent-bit

   - 没其他思路了，把Fluent-bit的Trace开开吧

   - 找到我们的fluent-bit-config的ConfigMap

   - ```yaml
     [root@ks-k8s-master-0 ~]# kubectl get secrets fluent-bit-config -n kubesphere-logging-system -o yaml
     apiVersion: v1
     data:
       fluent-bit.conf: W1NlcnZpY2VdCiAgICBQYXJzZXJzX0ZpbGUgICAgcGFyc2Vycy5jb25mCltJbnB1dF0KICAgIE5hbWUgICAgc3lzdGVtZAogICAgUGF0aCAgICAvdmFyL2xvZy9qb3VybmFsCiAgICBEQiAgICAvZmx1ZW50LWJpdC90YWlsL2RvY2tlci5kYgogICAgREIuU3luYyAgICBOb3JtYWwKICAgIFRhZyAgICBzZXJ2aWNlLmRvY2tlcgogICAgU3lzdGVtZF9GaWx0ZXIgICAgX1NZU1RFTURfVU5JVD1kb2NrZXIuc2VydmljZQpbSW5wdXRdCiAgICBOYW1lICAgIHN5c3RlbWQKICAgIFBhdGggICAgL3Zhci9sb2cvam91cm5hbAogICAgREIgICAgL2ZsdWVudC1iaXQvdGFpbC9rdWJlbGV0LmRiCiAgICBEQi5TeW5jICAgIE5vcm1hbAogICAgVGFnICAgIHNlcnZpY2Uua3ViZWxldAogICAgU3lzdGVtZF9GaWx0ZXIgICAgX1NZU1RFTURfVU5JVD1rdWJlbGV0LnNlcnZpY2UKW0lucHV0XQogICAgTmFtZSAgICB0YWlsCiAgICBQYXRoICAgIC92YXIvbG9nL2NvbnRhaW5lcnMvKi5sb2cKICAgIEV4Y2x1ZGVfUGF0aCAgICAvdmFyL2xvZy9jb250YWluZXJzLypfa3ViZXNwaGVyZS1sb2dnaW5nLXN5c3RlbV9ldmVudHMtZXhwb3J0ZXIqLmxvZywvdmFyL2xvZy9jb250YWluZXJzL2t1YmUtYXVkaXRpbmctd2ViaG9vaypfa3ViZXNwaGVyZS1sb2dnaW5nLXN5c3RlbV9rdWJlLWF1ZGl0aW5nLXdlYmhvb2sqLmxvZwogICAgUmVmcmVzaF9JbnRlcnZhbCAgICAxMAogICAgU2tpcF9Mb25nX0xpbmVzICAgIHRydWUKICAgIERCICAgIC9mbHVlbnQtYml0L3RhaWwvcG9zLmRiCiAgICBEQi5TeW5jICAgIE5vcm1hbAogICAgTWVtX0J1Zl9MaW1pdCAgICA1TUIKICAgIFBhcnNlciAgICBkb2NrZXIKICAgIFRhZyAgICBrdWJlLioKW0lucHV0XQogICAgTmFtZSAgICB0YWlsCiAgICBQYXRoICAgIC92YXIvbG9nL2NvbnRhaW5lcnMva3ViZS1hdWRpdGluZy13ZWJob29rKl9rdWJlc3BoZXJlLWxvZ2dpbmctc3lzdGVtX2t1YmUtYXVkaXRpbmctd2ViaG9vayoubG9nCiAgICBSZWZyZXNoX0ludGVydmFsICAgIDEwCiAgICBTa2lwX0xvbmdfTGluZXMgICAgdHJ1ZQogICAgREIgICAgL2ZsdWVudC1iaXQvdGFpbC9wb3MtYXVkaXRpbmcuZGIKICAgIERCLlN5bmMgICAgTm9ybWFsCiAgICBNZW1fQnVmX0xpbWl0ICAgIDVNQgogICAgUGFyc2VyICAgIGRvY2tlcgogICAgVGFnICAgIGt1YmVfYXVkaXRpbmcKW0lucHV0XQogICAgTmFtZSAgICB0YWlsCiAgICBQYXRoICAgIC92YXIvbG9nL2NvbnRhaW5lcnMvKl9rdWJlc3BoZXJlLWxvZ2dpbmctc3lzdGVtX2V2ZW50cy1leHBvcnRlcioubG9nCiAgICBSZWZyZXNoX0ludGVydmFsICAgIDEwCiAgICBTa2lwX0xvbmdfTGluZXMgICAgdHJ1ZQogICAgREIgICAgL2ZsdWVudC1iaXQvdGFpbC9wb3MtZXZlbnRzLmRiCiAgICBEQi5TeW5jICAgIE5vcm1hbAogICAgTWVtX0J1Zl9MaW1pdCAgICA1TUIKICAgIFBhcnNlciAgICBkb2NrZXIKICAgIFRhZyAgICBrdWJlX2V2ZW50cwpbRmlsdGVyXQogICAgTmFtZSAgICBwYXJzZXIKICAgIE1hdGNoICAgIGt1YmVfYXVkaXRpbmcKICAgIEtleV9OYW1lICAgIGxvZwogICAgUGFyc2VyICAgIGpzb24KW0ZpbHRlcl0KICAgIE5hbWUgICAgbW9kaWZ5CiAgICBNYXRjaCAgICBrdWJlX2F1ZGl0aW5nCiAgICBDb25kaXRpb24gICAgS2V5X2RvZXNfbm90X2V4aXN0ICAgIEF1ZGl0SUQgICAgCiAgICBBZGQgICAgaWdub3JlICAgIHRydWUKW0ZpbHRlcl0KICAgIE5hbWUgICAgZ3JlcAogICAgTWF0Y2ggICAga3ViZV9hdWRpdGluZwogICAgRXhjbHVkZSAgICBpZ25vcmUgdHJ1ZQpbRmlsdGVyXQogICAgTmFtZSAgICBwYXJzZXIKICAgIE1hdGNoICAgIGt1YmVfZXZlbnRzCiAgICBLZXlfTmFtZSAgICBsb2cKICAgIFBhcnNlciAgICBqc29uCltGaWx0ZXJdCiAgICBOYW1lICAgIGt1YmVybmV0ZXMKICAgIE1hdGNoICAgIGt1YmUuKgogICAgS3ViZV9VUkwgICAgaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjOjQ0MwogICAgS3ViZV9DQV9GaWxlICAgIC92YXIvcnVuL3NlY3JldHMva3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9jYS5jcnQKICAgIEt1YmVfVG9rZW5fRmlsZSAgICAvdmFyL3J1bi9zZWNyZXRzL2t1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvdG9rZW4KICAgIExhYmVscyAgICBmYWxzZQogICAgQW5ub3RhdGlvbnMgICAgZmFsc2UKW0ZpbHRlcl0KICAgIE5hbWUgICAgbmVzdAogICAgTWF0Y2ggICAga3ViZS4qCiAgICBPcGVyYXRpb24gICAgbGlmdAogICAgTmVzdGVkX3VuZGVyICAgIGt1YmVybmV0ZXMKICAgIEFkZF9wcmVmaXggICAga3ViZXJuZXRlc18KW0ZpbHRlcl0KICAgIE5hbWUgICAgbW9kaWZ5CiAgICBNYXRjaCAgICBrdWJlLioKICAgIFJlbW92ZSAgICBzdHJlYW0KICAgIFJlbW92ZSAgICBrdWJlcm5ldGVzX3BvZF9pZAogICAgUmVtb3ZlICAgIGt1YmVybmV0ZXNfaG9zdAogICAgUmVtb3ZlICAgIGt1YmVybmV0ZXNfY29udGFpbmVyX2hhc2gKW0ZpbHRlcl0KICAgIE5hbWUgICAgbmVzdAogICAgTWF0Y2ggICAga3ViZS4qCiAgICBPcGVyYXRpb24gICAgbmVzdAogICAgV2lsZGNhcmQgICAga3ViZXJuZXRlc18qCiAgICBOZXN0X3VuZGVyICAgIGt1YmVybmV0ZXMKICAgIFJlbW92ZV9wcmVmaXggICAga3ViZXJuZXRlc18KW0ZpbHRlcl0KICAgIE5hbWUgICAgbHVhCiAgICBNYXRjaCAgICBzZXJ2aWNlLioKICAgIHNjcmlwdCAgICAvZmx1ZW50LWJpdC9jb25maWcvc3lzdGVtZC5sdWEKICAgIGNhbGwgICAgYWRkX3RpbWUKICAgIHRpbWVfYXNfdGFibGUgICAgdHJ1ZQpbT3V0cHV0XQogICAgTmFtZSAgICBlcwogICAgTWF0Y2hfUmVnZXggICAgKD86a3ViZXxzZXJ2aWNlKVwuKC4qKQogICAgSG9zdCAgICAxOTIuMTY4LjkuOTUKICAgIFBvcnQgICAgOTIwMAogICAgSFRUUF9Vc2VyICAgIGxzdGFjawogICAgSFRUUF9QYXNzd2QgICAgUEA4OHcwcmQKICAgIExvZ3N0YXNoX0Zvcm1hdCAgICB0cnVlCiAgICBMb2dzdGFzaF9QcmVmaXggICAga3MtbG9nc3Rhc2gtbG9nCiAgICBUaW1lX0tleSAgICBAdGltZXN0YW1wCiAgICBHZW5lcmF0ZV9JRCAgICB0cnVlCiAgICB0bHMgICAgT24KICAgIHRscy52ZXJpZnkgICAgZmFsc2UKW091dHB1dF0KICAgIE5hbWUgICAgZXMKICAgIE1hdGNoICAgIGt1YmVfYXVkaXRpbmcKICAgIEhvc3QgICAgZWxhc3RpY3NlYXJjaC1sb2dnaW5nLWRhdGEua3ViZXNwaGVyZS1sb2dnaW5nLXN5c3RlbS5zdmMKICAgIFBvcnQgICAgOTIwMAogICAgSFRUUF9Vc2VyICAgIGxzdGFjawogICAgSFRUUF9QYXNzd2QgICAgUEA4OHcwcmQKICAgIExvZ3N0YXNoX0Zvcm1hdCAgICB0cnVlCiAgICBMb2dzdGFzaF9QcmVmaXggICAga3MtbG9nc3Rhc2gtYXVkaXRpbmcKICAgIEdlbmVyYXRlX0lEICAgIHRydWUKICAgIHRscyAgICBPbgogICAgdGxzLnZlcmlmeSAgICBmYWxzZQpbT3V0cHV0XQogICAgTmFtZSAgICBlcwogICAgTWF0Y2ggICAga3ViZV9ldmVudHMKICAgIEhvc3QgICAgZWxhc3RpY3NlYXJjaC1sb2dnaW5nLWRhdGEua3ViZXNwaGVyZS1sb2dnaW5nLXN5c3RlbS5zdmMKICAgIFBvcnQgICAgOTIwMAogICAgSFRUUF9Vc2VyICAgIGxzdGFjawogICAgSFRUUF9QYXNzd2QgICAgUEA4OHcwcmQKICAgIExvZ3N0YXNoX0Zvcm1hdCAgICB0cnVlCiAgICBMb2dzdGFzaF9QcmVmaXggICAga3MtbG9nc3Rhc2gtZXZlbnRzCiAgICBHZW5lcmF0ZV9JRCAgICB0cnVlCiAgICB0bHMgICAgT24KICAgIHRscy52ZXJpZnkgICAgZmFsc2UK
       parsers.conf: ""
       systemd.lua: ZnVuY3Rpb24gYWRkX3RpbWUodGFnLCB0aW1lc3RhbXAsIHJlY29yZCkKICBuZXdfcmVjb3JkID0ge30KICB0aW1lU3RyID0gb3MuZGF0ZSgiISp0IiwgdGltZXN0YW1wWyJzZWMiXSkKICB0ID0gc3RyaW5nLmZvcm1hdCgiJTRkLSUwMmQtJTAyZFQlMDJkOiUwMmQ6JTAyZC4lc1oiLAoJCXRpbWVTdHJbInllYXIiXSwgdGltZVN0clsibW9udGgiXSwgdGltZVN0clsiZGF5Il0sCgkJdGltZVN0clsiaG91ciJdLCB0aW1lU3RyWyJtaW4iXSwgdGltZVN0clsic2VjIl0sCgkJdGltZXN0YW1wWyJuc2VjIl0pCiAga3ViZXJuZXRlcyA9IHt9CiAga3ViZXJuZXRlc1sicG9kX25hbWUiXSA9IHJlY29yZFsiX0hPU1ROQU1FIl0KICBrdWJlcm5ldGVzWyJjb250YWluZXJfbmFtZSJdID0gcmVjb3JkWyJTWVNMT0dfSURFTlRJRklFUiJdCiAga3ViZXJuZXRlc1sibmFtZXNwYWNlX25hbWUiXSA9ICJrdWJlLXN5c3RlbSIKICBuZXdfcmVjb3JkWyJ0aW1lIl0gPSB0CiAgbmV3X3JlY29yZFsibG9nIl0gPSByZWNvcmRbIk1FU1NBR0UiXQogIG5ld19yZWNvcmRbImt1YmVybmV0ZXMiXSA9IGt1YmVybmV0ZXMKICByZXR1cm4gMSwgdGltZXN0YW1wLCBuZXdfcmVjb3JkCmVuZA==
     kind: Secret
     metadata:
       creationTimestamp: "2022-04-15T03:49:33Z"
       name: fluent-bit-config
       namespace: kubesphere-logging-system
       ownerReferences:
       - apiVersion: logging.kubesphere.io/v1alpha2
         blockOwnerDeletion: true
         controller: true
         kind: FluentBitConfig
         name: fluent-bit-config
         uid: 8637b012-ed24-404f-b1bb-6404b0a18537
       resourceVersion: "2865407"
       uid: 66b56299-fb2c-435a-ae83-f21904656863
     type: Opaque
     ```

   - 居然没有明文，居然是加密的，这看个鬼啊。。。

   - 不过，好在我们知道k8s里的好多字典都是base64加密的，而base64又是可以反解密的。我们解开试试

   - 先把base64加密过的字符串复制下来，新建一个文件**fluent-bit-config**

   - ```yaml
     [root@ks-k8s-master-0 ~]# vi fluent-bit-config
     写入以下内容
     
     W1NlcnZpY2VdCiAgICBQYXJzZXJzX0ZpbGUgICAgcGFyc2Vycy5jb25mCltJbnB1dF0KICAgIE5hbWUgICAgc3lzdGVtZAogICAgUGF0aCAgICAvdmFyL2xvZy9qb3VybmFsCiAgICBEQiAgICAvZmx1ZW50LWJpdC90YWlsL2RvY2tlci5kYgogICAgREIuU3luYyAgICBOb3JtYWwKICAgIFRhZyAgICBzZXJ2aWNlLmRvY2tlcgogICAgU3lzdGVtZF9GaWx0ZXIgICAgX1NZU1RFTURfVU5JVD1kb2NrZXIuc2VydmljZQpbSW5wdXRdCiAgICBOYW1lICAgIHN5c3RlbWQKICAgIFBhdGggICAgL3Zhci9sb2cvam91cm5hbAogICAgREIgICAgL2ZsdWVudC1iaXQvdGFpbC9rdWJlbGV0LmRiCiAgICBEQi5TeW5jICAgIE5vcm1hbAogICAgVGFnICAgIHNlcnZpY2Uua3ViZWxldAogICAgU3lzdGVtZF9GaWx0ZXIgICAgX1NZU1RFTURfVU5JVD1rdWJlbGV0LnNlcnZpY2UKW0lucHV0XQogICAgTmFtZSAgICB0YWlsCiAgICBQYXRoICAgIC92YXIvbG9nL2NvbnRhaW5lcnMvKi5sb2cKICAgIEV4Y2x1ZGVfUGF0aCAgICAvdmFyL2xvZy9jb250YWluZXJzLypfa3ViZXNwaGVyZS1sb2dnaW5nLXN5c3RlbV9ldmVudHMtZXhwb3J0ZXIqLmxvZywvdmFyL2xvZy9jb250YWluZXJzL2t1YmUtYXVkaXRpbmctd2ViaG9vaypfa3ViZXNwaGVyZS1sb2dnaW5nLXN5c3RlbV9rdWJlLWF1ZGl0aW5nLXdlYmhvb2sqLmxvZwogICAgUmVmcmVzaF9JbnRlcnZhbCAgICAxMAogICAgU2tpcF9Mb25nX0xpbmVzICAgIHRydWUKICAgIERCICAgIC9mbHVlbnQtYml0L3RhaWwvcG9zLmRiCiAgICBEQi5TeW5jICAgIE5vcm1hbAogICAgTWVtX0J1Zl9MaW1pdCAgICA1TUIKICAgIFBhcnNlciAgICBkb2NrZXIKICAgIFRhZyAgICBrdWJlLioKW0lucHV0XQogICAgTmFtZSAgICB0YWlsCiAgICBQYXRoICAgIC92YXIvbG9nL2NvbnRhaW5lcnMva3ViZS1hdWRpdGluZy13ZWJob29rKl9rdWJlc3BoZXJlLWxvZ2dpbmctc3lzdGVtX2t1YmUtYXVkaXRpbmctd2ViaG9vayoubG9nCiAgICBSZWZyZXNoX0ludGVydmFsICAgIDEwCiAgICBTa2lwX0xvbmdfTGluZXMgICAgdHJ1ZQogICAgREIgICAgL2ZsdWVudC1iaXQvdGFpbC9wb3MtYXVkaXRpbmcuZGIKICAgIERCLlN5bmMgICAgTm9ybWFsCiAgICBNZW1fQnVmX0xpbWl0ICAgIDVNQgogICAgUGFyc2VyICAgIGRvY2tlcgogICAgVGFnICAgIGt1YmVfYXVkaXRpbmcKW0lucHV0XQogICAgTmFtZSAgICB0YWlsCiAgICBQYXRoICAgIC92YXIvbG9nL2NvbnRhaW5lcnMvKl9rdWJlc3BoZXJlLWxvZ2dpbmctc3lzdGVtX2V2ZW50cy1leHBvcnRlcioubG9nCiAgICBSZWZyZXNoX0ludGVydmFsICAgIDEwCiAgICBTa2lwX0xvbmdfTGluZXMgICAgdHJ1ZQogICAgREIgICAgL2ZsdWVudC1iaXQvdGFpbC9wb3MtZXZlbnRzLmRiCiAgICBEQi5TeW5jICAgIE5vcm1hbAogICAgTWVtX0J1Zl9MaW1pdCAgICA1TUIKICAgIFBhcnNlciAgICBkb2NrZXIKICAgIFRhZyAgICBrdWJlX2V2ZW50cwpbRmlsdGVyXQogICAgTmFtZSAgICBwYXJzZXIKICAgIE1hdGNoICAgIGt1YmVfYXVkaXRpbmcKICAgIEtleV9OYW1lICAgIGxvZwogICAgUGFyc2VyICAgIGpzb24KW0ZpbHRlcl0KICAgIE5hbWUgICAgbW9kaWZ5CiAgICBNYXRjaCAgICBrdWJlX2F1ZGl0aW5nCiAgICBDb25kaXRpb24gICAgS2V5X2RvZXNfbm90X2V4aXN0ICAgIEF1ZGl0SUQgICAgCiAgICBBZGQgICAgaWdub3JlICAgIHRydWUKW0ZpbHRlcl0KICAgIE5hbWUgICAgZ3JlcAogICAgTWF0Y2ggICAga3ViZV9hdWRpdGluZwogICAgRXhjbHVkZSAgICBpZ25vcmUgdHJ1ZQpbRmlsdGVyXQogICAgTmFtZSAgICBwYXJzZXIKICAgIE1hdGNoICAgIGt1YmVfZXZlbnRzCiAgICBLZXlfTmFtZSAgICBsb2cKICAgIFBhcnNlciAgICBqc29uCltGaWx0ZXJdCiAgICBOYW1lICAgIGt1YmVybmV0ZXMKICAgIE1hdGNoICAgIGt1YmUuKgogICAgS3ViZV9VUkwgICAgaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjOjQ0MwogICAgS3ViZV9DQV9GaWxlICAgIC92YXIvcnVuL3NlY3JldHMva3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9jYS5jcnQKICAgIEt1YmVfVG9rZW5fRmlsZSAgICAvdmFyL3J1bi9zZWNyZXRzL2t1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvdG9rZW4KICAgIExhYmVscyAgICBmYWxzZQogICAgQW5ub3RhdGlvbnMgICAgZmFsc2UKW0ZpbHRlcl0KICAgIE5hbWUgICAgbmVzdAogICAgTWF0Y2ggICAga3ViZS4qCiAgICBPcGVyYXRpb24gICAgbGlmdAogICAgTmVzdGVkX3VuZGVyICAgIGt1YmVybmV0ZXMKICAgIEFkZF9wcmVmaXggICAga3ViZXJuZXRlc18KW0ZpbHRlcl0KICAgIE5hbWUgICAgbW9kaWZ5CiAgICBNYXRjaCAgICBrdWJlLioKICAgIFJlbW92ZSAgICBzdHJlYW0KICAgIFJlbW92ZSAgICBrdWJlcm5ldGVzX3BvZF9pZAogICAgUmVtb3ZlICAgIGt1YmVybmV0ZXNfaG9zdAogICAgUmVtb3ZlICAgIGt1YmVybmV0ZXNfY29udGFpbmVyX2hhc2gKW0ZpbHRlcl0KICAgIE5hbWUgICAgbmVzdAogICAgTWF0Y2ggICAga3ViZS4qCiAgICBPcGVyYXRpb24gICAgbmVzdAogICAgV2lsZGNhcmQgICAga3ViZXJuZXRlc18qCiAgICBOZXN0X3VuZGVyICAgIGt1YmVybmV0ZXMKICAgIFJlbW92ZV9wcmVmaXggICAga3ViZXJuZXRlc18KW0ZpbHRlcl0KICAgIE5hbWUgICAgbHVhCiAgICBNYXRjaCAgICBzZXJ2aWNlLioKICAgIHNjcmlwdCAgICAvZmx1ZW50LWJpdC9jb25maWcvc3lzdGVtZC5sdWEKICAgIGNhbGwgICAgYWRkX3RpbWUKICAgIHRpbWVfYXNfdGFibGUgICAgdHJ1ZQpbT3V0cHV0XQogICAgTmFtZSAgICBlcwogICAgTWF0Y2hfUmVnZXggICAgKD86a3ViZXxzZXJ2aWNlKVwuKC4qKQogICAgSG9zdCAgICAxOTIuMTY4LjkuOTUKICAgIFBvcnQgICAgOTIwMAogICAgSFRUUF9Vc2VyICAgIGxzdGFjawogICAgSFRUUF9QYXNzd2QgICAgUEA4OHcwcmQKICAgIExvZ3N0YXNoX0Zvcm1hdCAgICB0cnVlCiAgICBMb2dzdGFzaF9QcmVmaXggICAga3MtbG9nc3Rhc2gtbG9nCiAgICBUaW1lX0tleSAgICBAdGltZXN0YW1wCiAgICBHZW5lcmF0ZV9JRCAgICB0cnVlCiAgICB0bHMgICAgT24KICAgIHRscy52ZXJpZnkgICAgZmFsc2UKW091dHB1dF0KICAgIE5hbWUgICAgZXMKICAgIE1hdGNoICAgIGt1YmVfYXVkaXRpbmcKICAgIEhvc3QgICAgZWxhc3RpY3NlYXJjaC1sb2dnaW5nLWRhdGEua3ViZXNwaGVyZS1sb2dnaW5nLXN5c3RlbS5zdmMKICAgIFBvcnQgICAgOTIwMAogICAgSFRUUF9Vc2VyICAgIGxzdGFjawogICAgSFRUUF9QYXNzd2QgICAgUEA4OHcwcmQKICAgIExvZ3N0YXNoX0Zvcm1hdCAgICB0cnVlCiAgICBMb2dzdGFzaF9QcmVmaXggICAga3MtbG9nc3Rhc2gtYXVkaXRpbmcKICAgIEdlbmVyYXRlX0lEICAgIHRydWUKICAgIHRscyAgICBPbgogICAgdGxzLnZlcmlmeSAgICBmYWxzZQpbT3V0cHV0XQogICAgTmFtZSAgICBlcwogICAgTWF0Y2ggICAga3ViZV9ldmVudHMKICAgIEhvc3QgICAgZWxhc3RpY3NlYXJjaC1sb2dnaW5nLWRhdGEua3ViZXNwaGVyZS1sb2dnaW5nLXN5c3RlbS5zdmMKICAgIFBvcnQgICAgOTIwMAogICAgSFRUUF9Vc2VyICAgIGxzdGFjawogICAgSFRUUF9QYXNzd2QgICAgUEA4OHcwcmQKICAgIExvZ3N0YXNoX0Zvcm1hdCAgICB0cnVlCiAgICBMb2dzdGFzaF9QcmVmaXggICAga3MtbG9nc3Rhc2gtZXZlbnRzCiAgICBHZW5lcmF0ZV9JRCAgICB0cnVlCiAgICB0bHMgICAgT24KICAgIHRscy52ZXJpZnkgICAgZmFsc2UK
     ```

   - 执行下面的命令讲解密后的文件输出到**fluent-bit-config.conf**

   - ```shell
     [root@ks-k8s-master-0 ~]# base64 -d fluent-bit-config > fluent-bit-config.conf
     ```

   - 看看输出的文件都有啥

   - ```yaml
     [root@ks-k8s-master-0 ~]# cat fluent-bit-config.conf 
     [Service]
         Parsers_File    parsers.conf
     [Input]
         Name    systemd
         Path    /var/log/journal
         DB    /fluent-bit/tail/docker.db
         DB.Sync    Normal
         Tag    service.docker
         Systemd_Filter    _SYSTEMD_UNIT=docker.service
     [Input]
         Name    systemd
         Path    /var/log/journal
         DB    /fluent-bit/tail/kubelet.db
         DB.Sync    Normal
         Tag    service.kubelet
         Systemd_Filter    _SYSTEMD_UNIT=kubelet.service
     [Input]
         Name    tail
         Path    /var/log/containers/*.log
         Exclude_Path    /var/log/containers/*_kubesphere-logging-system_events-exporter*.log,/var/log/containers/kube-auditing-webhook*_kubesphere-logging-system_kube-auditing-webhook*.log
         Refresh_Interval    10
         Skip_Long_Lines    true
         DB    /fluent-bit/tail/pos.db
         DB.Sync    Normal
         Mem_Buf_Limit    5MB
         Parser    docker
         Tag    kube.*
     [Input]
         Name    tail
         Path    /var/log/containers/kube-auditing-webhook*_kubesphere-logging-system_kube-auditing-webhook*.log
         Refresh_Interval    10
         Skip_Long_Lines    true
         DB    /fluent-bit/tail/pos-auditing.db
         DB.Sync    Normal
         Mem_Buf_Limit    5MB
         Parser    docker
         Tag    kube_auditing
     [Input]
         Name    tail
         Path    /var/log/containers/*_kubesphere-logging-system_events-exporter*.log
         Refresh_Interval    10
         Skip_Long_Lines    true
         DB    /fluent-bit/tail/pos-events.db
         DB.Sync    Normal
         Mem_Buf_Limit    5MB
         Parser    docker
         Tag    kube_events
     [Filter]
         Name    parser
         Match    kube_auditing
         Key_Name    log
         Parser    json
     [Filter]
         Name    modify
         Match    kube_auditing
         Condition    Key_does_not_exist    AuditID    
         Add    ignore    true
     [Filter]
         Name    grep
         Match    kube_auditing
         Exclude    ignore true
     [Filter]
         Name    parser
         Match    kube_events
         Key_Name    log
         Parser    json
     [Filter]
         Name    kubernetes
         Match    kube.*
         Kube_URL    https://kubernetes.default.svc:443
         Kube_CA_File    /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
         Kube_Token_File    /var/run/secrets/kubernetes.io/serviceaccount/token
         Labels    false
         Annotations    false
     [Filter]
         Name    nest
         Match    kube.*
         Operation    lift
         Nested_under    kubernetes
         Add_prefix    kubernetes_
     [Filter]
         Name    modify
         Match    kube.*
         Remove    stream
         Remove    kubernetes_pod_id
         Remove    kubernetes_host
         Remove    kubernetes_container_hash
     [Filter]
         Name    nest
         Match    kube.*
         Operation    nest
         Wildcard    kubernetes_*
         Nest_under    kubernetes
         Remove_prefix    kubernetes_
     [Filter]
         Name    lua
         Match    service.*
         script    /fluent-bit/config/systemd.lua
         call    add_time
         time_as_table    true
     [Output]
         Name    es
         Match_Regex    (?:kube|service)\.(.*)
         Host    192.168.9.95
         Port    9200
         HTTP_User    lstack
         HTTP_Passwd    P@88w0rd
         Logstash_Format    true
         Logstash_Prefix    ks-logstash-log
         Time_Key    @timestamp
         Generate_ID    true
         tls    On
         tls.verify    false
     [Output]
         Name    es
         Match    kube_auditing
         Host    elasticsearch-logging-data.kubesphere-logging-system.svc
         Port    9200
         HTTP_User    lstack
         HTTP_Passwd    P@88w0rd
         Logstash_Format    true
         Logstash_Prefix    ks-logstash-auditing
         Generate_ID    true
         tls    On
         tls.verify    false
     [Output]
         Name    es
         Match    kube_events
         Host    elasticsearch-logging-data.kubesphere-logging-system.svc
         Port    9200
         HTTP_User    lstack
         HTTP_Passwd    P@88w0rd
         Logstash_Format    true
         Logstash_Prefix    ks-logstash-events
         Generate_ID    true
         tls    On
         tls.verify    false
     ```

   - 细节我们先不深究(其实我也不知道啥意思)，我们直奔我们的目标，找到**[Output] Name为es**且标有**Match_Regex**的配置，添加**Trace_Error     On**

   - 效果如下

   - ```yaml
     [Output]
         Name    es
         Match_Regex    (?:kube|service)\.(.*)
         Host    192.168.9.95
         Port    9200
         HTTP_User    lstack
         HTTP_Passwd    P@88w0rd
         Logstash_Format    true
         Logstash_Prefix    ks-logstash-log
         Time_Key    @timestamp
         Generate_ID    true
         tls    On
         tls.verify    false
         Trace_Error     On
     ```

   - 保存文件并退出，再将该文件中的内容进行base64编码

   - ```she
     [root@ks-k8s-master-0 ~]# base64 -w 0 fluent-bit-config.conf 
     W1NlcnZpY2VdCiAgICBQYXJzZXJzX0ZpbGUgICAgcGFyc2Vycy5jb25mCltJbnB1dF0KICAgIE5hbWUgICAgc3lzdGVtZAogICAgUGF0aCAgICAvdmFyL2xvZy9qb3VybmFsCiAgICBEQiAgICAvZmx1ZW50LWJpdC90YWlsL2RvY2tlci5kYgogICAgREIuU3luYyAgICBOb3JtYWwKICAgIFRhZyAgICBzZXJ2aWNlLmRvY2tlcgogICAgU3lzdGVtZF9GaWx0ZXIgICAgX1NZU1RFTURfVU5JVD1kb2NrZXIuc2VydmljZQpbSW5wdXRdCiAgICBOYW1lICAgIHN5c3RlbWQKICAgIFBhdGggICAgL3Zhci9sb2cvam91cm5hbAogICAgREIgICAgL2ZsdWVudC1iaXQvdGFpbC9rdWJlbGV0LmRiCiAgICBEQi5TeW5jICAgIE5vcm1hbAogICAgVGFnICAgIHNlcnZpY2Uua3ViZWxldAogICAgU3lzdGVtZF9GaWx0ZXIgICAgX1NZU1RFTURfVU5JVD1rdWJlbGV0LnNlcnZpY2UKW0lucHV0XQogICAgTmFtZSAgICB0YWlsCiAgICBQYXRoICAgIC92YXIvbG9nL2NvbnRhaW5lcnMvKi5sb2cKICAgIEV4Y2x1ZGVfUGF0aCAgICAvdmFyL2xvZy9jb250YWluZXJzLypfa3ViZXNwaGVyZS1sb2dnaW5nLXN5c3RlbV9ldmVudHMtZXhwb3J0ZXIqLmxvZywvdmFyL2xvZy9jb250YWluZXJzL2t1YmUtYXVkaXRpbmctd2ViaG9vaypfa3ViZXNwaGVyZS1sb2dnaW5nLXN5c3RlbV9rdWJlLWF1ZGl0aW5nLXdlYmhvb2sqLmxvZwogICAgUmVmcmVzaF9JbnRlcnZhbCAgICAxMAogICAgU2tpcF9Mb25nX0xpbmVzICAgIHRydWUKICAgIERCICAgIC9mbHVlbnQtYml0L3RhaWwvcG9zLmRiCiAgICBEQi5TeW5jICAgIE5vcm1hbAogICAgTWVtX0J1Zl9MaW1pdCAgICA1TUIKICAgIFBhcnNlciAgICBkb2NrZXIKICAgIFRhZyAgICBrdWJlLioKW0lucHV0XQogICAgTmFtZSAgICB0YWlsCiAgICBQYXRoICAgIC92YXIvbG9nL2NvbnRhaW5lcnMva3ViZS1hdWRpdGluZy13ZWJob29rKl9rdWJlc3BoZXJlLWxvZ2dpbmctc3lzdGVtX2t1YmUtYXVkaXRpbmctd2ViaG9vayoubG9nCiAgICBSZWZyZXNoX0ludGVydmFsICAgIDEwCiAgICBTa2lwX0xvbmdfTGluZXMgICAgdHJ1ZQogICAgREIgICAgL2ZsdWVudC1iaXQvdGFpbC9wb3MtYXVkaXRpbmcuZGIKICAgIERCLlN5bmMgICAgTm9ybWFsCiAgICBNZW1fQnVmX0xpbWl0ICAgIDVNQgogICAgUGFyc2VyICAgIGRvY2tlcgogICAgVGFnICAgIGt1YmVfYXVkaXRpbmcKW0lucHV0XQogICAgTmFtZSAgICB0YWlsCiAgICBQYXRoICAgIC92YXIvbG9nL2NvbnRhaW5lcnMvKl9rdWJlc3BoZXJlLWxvZ2dpbmctc3lzdGVtX2V2ZW50cy1leHBvcnRlcioubG9nCiAgICBSZWZyZXNoX0ludGVydmFsICAgIDEwCiAgICBTa2lwX0xvbmdfTGluZXMgICAgdHJ1ZQogICAgREIgICAgL2ZsdWVudC1iaXQvdGFpbC9wb3MtZXZlbnRzLmRiCiAgICBEQi5TeW5jICAgIE5vcm1hbAogICAgTWVtX0J1Zl9MaW1pdCAgICA1TUIKICAgIFBhcnNlciAgICBkb2NrZXIKICAgIFRhZyAgICBrdWJlX2V2ZW50cwpbRmlsdGVyXQogICAgTmFtZSAgICBwYXJzZXIKICAgIE1hdGNoICAgIGt1YmVfYXVkaXRpbmcKICAgIEtleV9OYW1lICAgIGxvZwogICAgUGFyc2VyICAgIGpzb24KW0ZpbHRlcl0KICAgIE5hbWUgICAgbW9kaWZ5CiAgICBNYXRjaCAgICBrdWJlX2F1ZGl0aW5nCiAgICBDb25kaXRpb24gICAgS2V5X2RvZXNfbm90X2V4aXN0ICAgIEF1ZGl0SUQgICAgCiAgICBBZGQgICAgaWdub3JlICAgIHRydWUKW0ZpbHRlcl0KICAgIE5hbWUgICAgZ3JlcAogICAgTWF0Y2ggICAga3ViZV9hdWRpdGluZwogICAgRXhjbHVkZSAgICBpZ25vcmUgdHJ1ZQpbRmlsdGVyXQogICAgTmFtZSAgICBwYXJzZXIKICAgIE1hdGNoICAgIGt1YmVfZXZlbnRzCiAgICBLZXlfTmFtZSAgICBsb2cKICAgIFBhcnNlciAgICBqc29uCltGaWx0ZXJdCiAgICBOYW1lICAgIGt1YmVybmV0ZXMKICAgIE1hdGNoICAgIGt1YmUuKgogICAgS3ViZV9VUkwgICAgaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjOjQ0MwogICAgS3ViZV9DQV9GaWxlICAgIC92YXIvcnVuL3NlY3JldHMva3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9jYS5jcnQKICAgIEt1YmVfVG9rZW5fRmlsZSAgICAvdmFyL3J1bi9zZWNyZXRzL2t1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvdG9rZW4KICAgIExhYmVscyAgICBmYWxzZQogICAgQW5ub3RhdGlvbnMgICAgZmFsc2UKW0ZpbHRlcl0KICAgIE5hbWUgICAgbmVzdAogICAgTWF0Y2ggICAga3ViZS4qCiAgICBPcGVyYXRpb24gICAgbGlmdAogICAgTmVzdGVkX3VuZGVyICAgIGt1YmVybmV0ZXMKICAgIEFkZF9wcmVmaXggICAga3ViZXJuZXRlc18KW0ZpbHRlcl0KICAgIE5hbWUgICAgbW9kaWZ5CiAgICBNYXRjaCAgICBrdWJlLioKICAgIFJlbW92ZSAgICBzdHJlYW0KICAgIFJlbW92ZSAgICBrdWJlcm5ldGVzX3BvZF9pZAogICAgUmVtb3ZlICAgIGt1YmVybmV0ZXNfaG9zdAogICAgUmVtb3ZlICAgIGt1YmVybmV0ZXNfY29udGFpbmVyX2hhc2gKW0ZpbHRlcl0KICAgIE5hbWUgICAgbmVzdAogICAgTWF0Y2ggICAga3ViZS4qCiAgICBPcGVyYXRpb24gICAgbmVzdAogICAgV2lsZGNhcmQgICAga3ViZXJuZXRlc18qCiAgICBOZXN0X3VuZGVyICAgIGt1YmVybmV0ZXMKICAgIFJlbW92ZV9wcmVmaXggICAga3ViZXJuZXRlc18KW0ZpbHRlcl0KICAgIE5hbWUgICAgbHVhCiAgICBNYXRjaCAgICBzZXJ2aWNlLioKICAgIHNjcmlwdCAgICAvZmx1ZW50LWJpdC9jb25maWcvc3lzdGVtZC5sdWEKICAgIGNhbGwgICAgYWRkX3RpbWUKICAgIHRpbWVfYXNfdGFibGUgICAgdHJ1ZQpbT3V0cHV0XQogICAgTmFtZSAgICBlcwogICAgTWF0Y2hfUmVnZXggICAgKD86a3ViZXxzZXJ2aWNlKVwuKC4qKQogICAgSG9zdCAgICAxOTIuMTY4LjkuOTUKICAgIFBvcnQgICAgOTIwMAogICAgSFRUUF9Vc2VyICAgIGxzdGFjawogICAgSFRUUF9QYXNzd2QgICAgUEA4OHcwcmQKICAgIExvZ3N0YXNoX0Zvcm1hdCAgICB0cnVlCiAgICBMb2dzdGFzaF9QcmVmaXggICAga3MtbG9nc3Rhc2gtbG9nCiAgICBUaW1lX0tleSAgICBAdGltZXN0YW1wCiAgICBHZW5lcmF0ZV9JRCAgICB0cnVlCiAgICB0bHMgICAgT24KICAgIHRscy52ZXJpZnkgICAgZmFsc2UKICAgIFRyYWNlX0Vycm9yICAgICBPbgpbT3V0cHV0XQogICAgTmFtZSAgICBlcwogICAgTWF0Y2ggICAga3ViZV9hdWRpdGluZwogICAgSG9zdCAgICBlbGFzdGljc2VhcmNoLWxvZ2dpbmctZGF0YS5rdWJlc3BoZXJlLWxvZ2dpbmctc3lzdGVtLnN2YwogICAgUG9ydCAgICA5MjAwCiAgICBIVFRQX1VzZXIgICAgbHN0YWNrCiAgICBIVFRQX1Bhc3N3ZCAgICBQQDg4dzByZAogICAgTG9nc3Rhc2hfRm9ybWF0ICAgIHRydWUKICAgIExvZ3N0YXNoX1ByZWZpeCAgICBrcy1sb2dzdGFzaC1hdWRpdGluZwogICAgR2VuZXJhdGVfSUQgICAgdHJ1ZQogICAgdGxzICAgIE9uCiAgICB0bHMudmVyaWZ5ICAgIGZhbHNlCltPdXRwdXRdCiAgICBOYW1lICAgIGVzCiAgICBNYXRjaCAgICBrdWJlX2V2ZW50cwogICAgSG9zdCAgICBlbGFzdGljc2VhcmNoLWxvZ2dpbmctZGF0YS5rdWJlc3BoZXJlLWxvZ2dpbmctc3lzdGVtLnN2YwogICAgUG9ydCAgICA5MjAwCiAgICBIVFRQX1VzZXIgICAgbHN0YWNrCiAgICBIVFRQX1Bhc3N3ZCAgICBQQDg4dzByZAogICAgTG9nc3Rhc2hfRm9ybWF0ICAgIHRydWUKICAgIExvZ3N0YXNoX1ByZWZpeCAgICBrcy1sb2dzdGFzaC1ldmVudHMKICAgIEdlbmVyYXRlX0lEICAgIHRydWUKICAgIHRscyAgICBPbgogICAgdGxzLnZlcmlmeSAgICBmYWxzZQo=
     ```

   - 先把输出结果复制下来

   - 再来修改我们的Fluent-bit的配置文件的secrets，修改**fluent-bit.conf**的值

   - ```shell
     k8s-master-0 ~]# kubectl kubectl edit secrets fluent-bit-config -n kubesphere-logging-system
     secret/fluent-bit-config edited
     ```

   - 然后我又崩溃了，修改完的配置又改回原来的样子了，看样子又是使用了**fluent-operator**不能直接修改conf文件，唉。。。

   - 先折回Output的配置，把**es-auditing**和**es-events**的配置先改对了

   - 有了上面改造es的经验，细节我就不多说了，忘记的可以翻翻上面的文档

   - 备份**es-auditing**和**es-events**的Output配置

   - ```shell
     [root@ks-k8s-master-0 ~]#  kubectl get outputs -n kubesphere-logging-system es-auditing -o yaml > es-auditing.output.yaml
     [root@ks-k8s-master-0 ~]#  kubectl get outputs -n kubesphere-logging-system es-events -o yaml > es-events.output.yaml
     ```

   - 先看看**es-auditing**的现状

   - ```yaml
     [root@ks-k8s-master-0 ~]# kubectl get outputs -n kubesphere-logging-system es-auditing -o yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"auditing","logging.kubesphere.io/enabled":"true"},"name":"es-auditing","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","httpPassword":{"valueFrom":{"secretKeyRef":{"key":"password","name":"elasticsearch-credentials"}}},"httpUser":{"valueFrom":{"secretKeyRef":{"key":"username","name":"elasticsearch-credentials"}}},"logstashFormat":true,"logstashPrefix":"ks-logstash-auditing","port":9200,"tls":{"verify":false}},"match":"kube_auditing"}}
       creationTimestamp: "2022-04-15T03:51:23Z"
       generation: 3
       labels:
         logging.kubesphere.io/component: auditing
         logging.kubesphere.io/enabled: "true"
       name: es-auditing
       namespace: kubesphere-logging-system
       resourceVersion: "1537731"
       uid: 42e31e0d-d641-48c0-8423-165a505b8124
     spec:
       es:
         generateID: true
         host: elasticsearch-logging-data.kubesphere-logging-system.svc
         httpPassword:
           valueFrom:
             secretKeyRef:
               key: password
               name: elasticsearch-credentials
         httpUser:
           valueFrom:
             secretKeyRef:
               key: username
               name: elasticsearch-credentials
         logstashFormat: true
         logstashPrefix: ks-logstash-auditing
         port: 9200
         tls:
           verify: false
       match: kube_auditing
     ```

   - 分析上面的文件，发现只需要修改**host**地址就可以了，其他的都不需要，这可比上面修改**es**的配置舒服多了

   - 修改**es-auditing**的配置

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl edit outputs -n kubesphere-logging-system es-auditing 
     output.logging.kubesphere.io/es-auditing edited
     ```

   - 修改后,再次查看，发现资源定义的内容如下：

   - ```yaml
     [root@ks-k8s-master-0 ~]# kubectl get outputs -n kubesphere-logging-system es-auditing -o yaml
     
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"auditing","logging.kubesphere.io/enabled":"true"},"name":"es-auditing","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","httpPassword":{"valueFrom":{"secretKeyRef":{"key":"password","name":"elasticsearch-credentials"}}},"httpUser":{"valueFrom":{"secretKeyRef":{"key":"username","name":"elasticsearch-credentials"}}},"logstashFormat":true,"logstashPrefix":"ks-logstash-auditing","port":9200,"tls":{"verify":false}},"match":"kube_auditing"}}
       creationTimestamp: "2022-04-15T03:51:23Z"
       generation: 4
       labels:
         logging.kubesphere.io/component: auditing
         logging.kubesphere.io/enabled: "true"
       name: es-auditing
       namespace: kubesphere-logging-system
       resourceVersion: "3183473"
       uid: 42e31e0d-d641-48c0-8423-165a505b8124
     spec:
       es:
         generateID: true
         host: 192.168.9.95
         httpPassword:
           valueFrom:
             secretKeyRef:
               key: password
               name: elasticsearch-credentials
         httpUser:
           valueFrom:
             secretKeyRef:
               key: username
               name: elasticsearch-credentials
         logstashFormat: true
         logstashPrefix: ks-logstash-auditing
         port: 9200
         tls:
           verify: false
       match: kube_auditing
     ```

   - 接下来修改**es-events**的配置，先看看**es-events**的现状

   - ```yaml
     [root@ks-k8s-master-0 ~]#  kubectl get outputs -n kubesphere-logging-system es-events -o yaml
     
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"events","logging.kubesphere.io/enabled":"true"},"name":"es-events","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","httpPassword":{"valueFrom":{"secretKeyRef":{"key":"password","name":"elasticsearch-credentials"}}},"httpUser":{"valueFrom":{"secretKeyRef":{"key":"username","name":"elasticsearch-credentials"}}},"logstashFormat":true,"logstashPrefix":"ks-logstash-events","port":9200,"tls":{"verify":false}},"match":"kube_events"}}
       creationTimestamp: "2022-04-15T03:51:42Z"
       generation: 3
       labels:
         logging.kubesphere.io/component: events
         logging.kubesphere.io/enabled: "true"
       name: es-events
       namespace: kubesphere-logging-system
       resourceVersion: "1537778"
       uid: 02127aef-c69f-4578-b92a-8ecf73685240
     spec:
       es:
         generateID: true
         host: elasticsearch-logging-data.kubesphere-logging-system.svc
         httpPassword:
           valueFrom:
             secretKeyRef:
               key: password
               name: elasticsearch-credentials
         httpUser:
           valueFrom:
             secretKeyRef:
               key: username
               name: elasticsearch-credentials
         logstashFormat: true
         logstashPrefix: ks-logstash-events
         port: 9200
         tls:
           verify: false
       match: kube_events
     ```

   - 分析上面的文件，发现一样只需要修改**host**地址就可以

   - 修改**es-auditing**的配置

   - ```shell
     [root@ks-k8s-master-0 ~]#  kubectl edit outputs -n kubesphere-logging-system es-events
     output.logging.kubesphere.io/es-events edited
     ```

   - 修改后,再次查看，发现资源定义的内容如下：

   - ```yaml
     [root@ks-k8s-master-0 ~]#  kubectl get outputs -n kubesphere-logging-system es-events -o yaml
     
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"events","logging.kubesphere.io/enabled":"true"},"name":"es-events","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","httpPassword":{"valueFrom":{"secretKeyRef":{"key":"password","name":"elasticsearch-credentials"}}},"httpUser":{"valueFrom":{"secretKeyRef":{"key":"username","name":"elasticsearch-credentials"}}},"logstashFormat":true,"logstashPrefix":"ks-logstash-events","port":9200,"tls":{"verify":false}},"match":"kube_events"}}
       creationTimestamp: "2022-04-15T03:51:42Z"
       generation: 4
       labels:
         logging.kubesphere.io/component: events
         logging.kubesphere.io/enabled: "true"
       name: es-events
       namespace: kubesphere-logging-system
       resourceVersion: "3184391"
       uid: 02127aef-c69f-4578-b92a-8ecf73685240
     spec:
       es:
         generateID: true
         host: 192.168.9.95
         httpPassword:
           valueFrom:
             secretKeyRef:
               key: password
               name: elasticsearch-credentials
         httpUser:
           valueFrom:
             secretKeyRef:
               key: username
               name: elasticsearch-credentials
         logstashFormat: true
         logstashPrefix: ks-logstash-events
         port: 9200
         tls:
           verify: false
       match: kube_events
     ```

   - 我们再次重建Fluent-bit，看看是否有奇迹

   - 其中一个pod重启后的日志如下

   - ```yaml
     容器日志
     
      level=info msg="Fluent bit started"
     
      Fluent Bit v1.8.3
     
      * Copyright (C) 2019-2021 The Fluent Bit Authors
     
      * Copyright (C) 2015-2018 Treasure Data
     
      * Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
     
      * https://fluentbit.io
     
      
     
      [2022/04/21 08:02:30] [ info] [engine] started (pid=16)
     
      [2022/04/21 08:02:30] [ info] [storage] version=1.1.1, initializing...
     
      [2022/04/21 08:02:30] [ info] [storage] in-memory
     
      [2022/04/21 08:02:30] [ info] [storage] normal synchronization mode, checksum disabled, max_chunks_up=128
     
      [2022/04/21 08:02:30] [ info] [cmetrics] version=0.1.6
     
      [2022/04/21 08:02:30] [ info] [filter:kubernetes:kubernetes.4] https=1 host=kubernetes.default.svc port=443
     
      [2022/04/21 08:02:30] [ info] [filter:kubernetes:kubernetes.4] local POD info OK
     
      [2022/04/21 08:02:30] [ info] [filter:kubernetes:kubernetes.4] testing connectivity with API server...
     
      [2022/04/21 08:02:30] [ info] [filter:kubernetes:kubernetes.4] connectivity OK
     
      [2022/04/21 08:02:30] [ info] [sp] stream processor started
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=416649 watch_fd=1 name=/var/log/containers/alertmanager-main-2_kubesphere-monitoring-system_alertmanager-1a71f1d5b1136320644f3289e4b22544620db4a0d35a1ffec52bc534d729c358.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=461549 watch_fd=2 name=/var/log/containers/alertmanager-main-2_kubesphere-monitoring-system_config-reloader-bc14c53008f1e800d6240a5b912239a3d5682c6dcb719f48c69b7e8ba5892e35.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134582558 watch_fd=3 name=/var/log/containers/calico-kube-controllers-75ddb95444-8xvf4_kube-system_calico-kube-controllers-dc90044aa0ce62e61dfb0842c8f2e5aa8d021738adffca847b003a5c9621512c.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268955529 watch_fd=4 name=/var/log/containers/calico-node-pzvrj_kube-system_flexvol-driver-abf59d8a7af22c683dfb0776a0835c719f41ef23446e912f82901a4fd9cf166f.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373423 watch_fd=5 name=/var/log/containers/calico-node-pzvrj_kube-system_install-cni-b3d49f2e9c03e63c3863d50365d2ade01dead979c90285c526ac0029b58411cd.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373208 watch_fd=6 name=/var/log/containers/calico-node-pzvrj_kube-system_upgrade-ipam-6d4425d7ea5302511df1c278f2b113ac8fdfa0a372e07b025e0a9b45d30129ac.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403716654 watch_fd=7 name=/var/log/containers/coredns-5495dd7c88-68k9b_kube-system_coredns-879c087a4c8717d0886e6603a63e9503a406d215cfe77941cbcc1cc7c138d010.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134583208 watch_fd=8 name=/var/log/containers/coredns-5495dd7c88-z9q6j_kube-system_coredns-80cd4427410de02465b415683817a75674a95777ffc0929f93e87506b120ca59.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=466327 watch_fd=9 name=/var/log/containers/ks-apiserver-574966976-snfm4_kubesphere-system_ks-apiserver-8871a0cc18cfef4663a15680761e272efeafe061ea9a898319265a145b494b18.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269010417 watch_fd=10 name=/var/log/containers/ks-console-65f4d44d88-hzw9b_kubesphere-system_ks-console-f600455c3af064c53adc8eb7e5412505c4d0e50362ab11b8e59d2e71b552b0f1.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269514179 watch_fd=11 name=/var/log/containers/ks-controller-manager-79c7dc79f5-j9fz5_kubesphere-system_ks-controller-manager-a8c35247c7cda868d3c023616026515690d2fedf2abb0f9367d1dc338103f7af.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403995814 watch_fd=12 name=/var/log/containers/ks-events-ruler-575669b4-mh2fn_kubesphere-logging-system_config-reloader-2d51bab0babdd47864ec235efdbd69d2075cc8bf72c3642449be69f68aa1f0d5.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=962171 watch_fd=13 name=/var/log/containers/ks-events-ruler-575669b4-mh2fn_kubesphere-logging-system_events-ruler-da24092a12c4fc0d0aab8a02c7fc7cb1e0e82d15a924717222bdbadee7935c84.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134580590 watch_fd=14 name=/var/log/containers/kube-apiserver-ks-k8s-master-0_kube-system_kube-apiserver-b4e97a525442956fe7612d27367a184ee09d65a56464149ec913ea004db0f1ef.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134580575 watch_fd=15 name=/var/log/containers/kube-controller-manager-ks-k8s-master-0_kube-system_kube-controller-manager-a07d1c5ed4108d98de2ba322b47939d5007c9d266a4ff558d57ec87ab10b2d54.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268955289 watch_fd=16 name=/var/log/containers/kube-proxy-bc59k_kube-system_kube-proxy-79144efa6b42a0f9636132538cc4ade6665269fea5903f3e1e0be05f14376d7c.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403715964 watch_fd=17 name=/var/log/containers/kube-scheduler-ks-k8s-master-0_kube-system_kube-scheduler-1a2ffcf4000a9bb0fa526fcfa1970e69465dfe0bb4c635a4d95f50f2e27ef763.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403855647 watch_fd=18 name=/var/log/containers/minio-859cb4d777-mpsk9_kubesphere-system_minio-d76f9516fa485cd24851afccfb2c85fd74ce894ba580af98703704860e1c00ed.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=413299 watch_fd=19 name=/var/log/containers/node-exporter-wwrpf_kubesphere-monitoring-system_kube-rbac-proxy-e35229db1e46e8b7b0a7d5a3cd581107102a3531b31d2e32d7e787a118c82896.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=375870 watch_fd=20 name=/var/log/containers/node-exporter-wwrpf_kubesphere-monitoring-system_node-exporter-e5cc5a4cc937cf9de149dd25bd6e8441ca176bf26cb2a214dd0b47627e7d8861.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134581094 watch_fd=21 name=/var/log/containers/nodelocaldns-sp59h_kube-system_node-cache-8b5bdf5a116af255f979a4a349c168257b632d24805d5f257a642dd97b0d53a1.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=463110 watch_fd=22 name=/var/log/containers/prometheus-k8s-0_kubesphere-monitoring-system_config-reloader-1378a8d9cd7a96334cb15adeafb468497ece0a9a600a3193fb80b60bc5b7e9b5.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135190584 watch_fd=23 name=/var/log/containers/prometheus-k8s-0_kubesphere-monitoring-system_prometheus-26e94cf0828cd1bd9c5da45217b7f3e654a7a41ce0dc5943375b1644d1a6a002.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134738131 watch_fd=24 name=/var/log/containers/prometheus-k8s-0_kubesphere-monitoring-system_prometheus-97de8b484747bae4aeac54ffd9b75c1ddc9582a28c768ebc9be395390bd031c3.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268956011 watch_fd=25 name=/var/log/containers/redis-ha-haproxy-868fdbddd4-kqrl5_kubesphere-system_config-init-0fb775710352218a66aec02ea2a736892c33df43dd4206e155e0bce6af4aba63.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373969 watch_fd=26 name=/var/log/containers/redis-ha-haproxy-868fdbddd4-kqrl5_kubesphere-system_haproxy-6c15749c02904875f5c1a280a42a908baa074ccffd3365f40b2692b6fe60acf5.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403792092 watch_fd=27 name=/var/log/containers/redis-ha-server-2_kubesphere-system_config-init-fa7c66040a53bef1c3c9b1844a4b25870835a6147cd83abf914ebb129a036536.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=412431 watch_fd=28 name=/var/log/containers/redis-ha-server-2_kubesphere-system_redis-6b19bb18f79c55cca6022bd8caa13a33d8c57386897db47fc011fd9fbe589b8a.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134738124 watch_fd=29 name=/var/log/containers/redis-ha-server-2_kubesphere-system_sentinel-7b2cbde0a38f45b692dd1679912bb9ba4bdfc358667c32164592a0e7ab2a87a6.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=467559 watch_fd=30 name=/var/log/containers/thanos-ruler-kubesphere-1_kubesphere-monitoring-system_config-reloader-28aef566c79b3cc073a67787c6ad44902c88930f069ab26cf5657aa2d1235108.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403860482 watch_fd=31 name=/var/log/containers/thanos-ruler-kubesphere-1_kubesphere-monitoring-system_thanos-ruler-24cfbc7280d54f3289c4ae70d0bb25a4964dd2f759e8bddefad3cf92070b1903.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134583426 watch_fd=32 name=/var/log/containers/calico-node-pzvrj_kube-system_calico-node-f2f40fb9878e3328a4783ceb9c01eaed9edf24c85579ed87c7ffc2161779c3e2.log
     
      [2022/04/21 08:02:30] [ info] [input:tail:tail.2] inotify_fs_add(): inode=984530 watch_fd=33 name=/var/log/containers/fluent-bit-97khn_kubesphere-logging-system_fluent-bit-9cb8370106aed1e55ca9b9304662301a3ab4cbe6f543616cb5d1eec39d6f7b25.log
     
      [2022/04/21 08:02:34] [ warn] [engine] failed to flush chunk '16-1650528150.277308324.flb', retry in 8 seconds: task_id=0, input=systemd.0 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:34] [ warn] [engine] failed to flush chunk '16-1650528150.357492295.flb', retry in 9 seconds: task_id=2, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:34] [ warn] [engine] failed to flush chunk '16-1650528150.296176788.flb', retry in 6 seconds: task_id=1, input=systemd.1 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:34] [ warn] [engine] failed to flush chunk '16-1650528150.386272778.flb', retry in 7 seconds: task_id=3, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:39] [ warn] [engine] failed to flush chunk '16-1650528158.305010058.flb', retry in 10 seconds: task_id=4, input=systemd.1 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:39] [ warn] [engine] failed to flush chunk '16-1650528154.329523292.flb', retry in 6 seconds: task_id=5, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:39] [ warn] [engine] failed to flush chunk '16-1650528154.753657988.flb', retry in 9 seconds: task_id=6, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:39] [ warn] [engine] failed to flush chunk '16-1650528155.347169912.flb', retry in 7 seconds: task_id=7, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:40] [ warn] [engine] chunk '16-1650528150.296176788.flb' cannot be retried: task_id=1, input=systemd.1 > output=es.0
     
      [2022/04/21 08:02:41] [ warn] [engine] chunk '16-1650528150.386272778.flb' cannot be retried: task_id=3, input=tail.2 > output=es.0
     
      [2022/04/21 08:02:42] [ warn] [engine] chunk '16-1650528150.277308324.flb' cannot be retried: task_id=0, input=systemd.0 > output=es.0
     
      [2022/04/21 08:02:43] [ warn] [engine] chunk '16-1650528150.357492295.flb' cannot be retried: task_id=2, input=tail.2 > output=es.0
     
      [2022/04/21 08:02:44] [ warn] [engine] failed to flush chunk '16-1650528159.328332342.flb', retry in 7 seconds: task_id=0, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:45] [ warn] [engine] chunk '16-1650528154.329523292.flb' cannot be retried: task_id=5, input=tail.2 > output=es.0
     
      [2022/04/21 08:02:46] [ warn] [engine] chunk '16-1650528155.347169912.flb' cannot be retried: task_id=7, input=tail.2 > output=es.0
     
      [2022/04/21 08:02:48] [ warn] [engine] chunk '16-1650528154.753657988.flb' cannot be retried: task_id=6, input=tail.2 > output=es.0
     
      [2022/04/21 08:02:49] [ warn] [engine] chunk '16-1650528158.305010058.flb' cannot be retried: task_id=4, input=systemd.1 > output=es.0
     
      [2022/04/21 08:02:49] [ warn] [engine] failed to flush chunk '16-1650528168.802004767.flb', retry in 6 seconds: task_id=1, input=systemd.1 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:49] [ warn] [engine] failed to flush chunk '16-1650528164.324962334.flb', retry in 6 seconds: task_id=2, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:51] [ warn] [engine] chunk '16-1650528159.328332342.flb' cannot be retried: task_id=0, input=tail.2 > output=es.0
     
      [2022/04/21 08:02:54] [ warn] [engine] failed to flush chunk '16-1650528169.329511724.flb', retry in 8 seconds: task_id=0, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:55] [ warn] [engine] chunk '16-1650528168.802004767.flb' cannot be retried: task_id=1, input=systemd.1 > output=es.0
     
      [2022/04/21 08:02:55] [ warn] [engine] chunk '16-1650528164.324962334.flb' cannot be retried: task_id=2, input=tail.2 > output=es.0
     
      [2022/04/21 08:02:59] [ warn] [engine] failed to flush chunk '16-1650528179.232206869.flb', retry in 8 seconds: task_id=1, input=systemd.1 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:59] [ warn] [engine] failed to flush chunk '16-1650528174.324611868.flb', retry in 8 seconds: task_id=2, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/21 08:02:59] [ warn] [engine] failed to flush chunk '16-1650528174.430818290.flb', retry in 7 seconds: task_id=3, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/21 08:03:02] [ warn] [engine] chunk '16-1650528169.329511724.flb' cannot be retried: task_id=0, input=tail.2 > output=es.0
     
      [2022/04/21 08:03:04] [ warn] [engine] failed to flush chunk '16-1650528179.322293412.flb', retry in 7 seconds: task_id=0, input=tail.2 > output=es.0 (out_id=0)
     
     ```

   - 操作了一圈，又回到了原点，还是想办法开Fluent-bit的Trace日志吧，既然Oupt模板里没有对应参数，那么还是去底层解决吧。

   - 首先我们要知道KubeSphere里部署的Fluent-bit采用了**fluent-operator**，operator里又专门定义了Output的类别的资源去对应Fluent-bit配置文件中的output章节。

   - 那我们就去找到fluent-operator的官方网站，看看源码里[crds中对于output资源如何定义的](https://github.com/fluent/fluent-operator/blob/master/charts/fluent-operator/crds/fluentbit.fluent.io_clusteroutputs.yaml)（这里面就涉及operator的知识了，这个技能请自己get吧，细节我目前也不懂，所以也讲不出来）

   - 打开资源定义文件后，在里面搜索关键词**traceError**，注意一定是在**es**章节里的

   - ![image-20220422085824266](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220422085824266.png)

   - 有两个相关字段**traceError**和**traceOutput**，这里看说明感觉**traceOutput**会更猛，但是前面有人说了开启**traceError**，那咱就先试试**traceError**，不行再来搞**traceOutput**。

   - 来吧，接着修改es的output文件,主要是在现有的配置文件中加入**traceError: true**

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl edit outputs -n kubesphere-logging-system es
     output.logging.kubesphere.io/es edited
     ```

   - 再用base64解码secrets，看看配置文件是否有变化，这次我们换个简单的方式**一把输出**

     > 你是不是要怼我了，既然有这么优雅的方式，之前为啥不用，呵呵，之前我也不会，是在前面搜索解决办法时无意中get到的，[出处在这](https://banzaicloud.com/docs/one-eye/logging-operator/operation/troubleshooting/fluentbit/))

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl get secret -n kubesphere-logging-system fluent-bit-config -o jsonpath="{.data['fluent-bit\.conf']}" | base64 --decode
     [Service]
         Parsers_File    parsers.conf
     [Input]
         Name    systemd
         Path    /var/log/journal
         DB    /fluent-bit/tail/docker.db
         DB.Sync    Normal
         Tag    service.docker
         Systemd_Filter    _SYSTEMD_UNIT=docker.service
     [Input]
         Name    systemd
         Path    /var/log/journal
         DB    /fluent-bit/tail/kubelet.db
         DB.Sync    Normal
         Tag    service.kubelet
         Systemd_Filter    _SYSTEMD_UNIT=kubelet.service
     [Input]
         Name    tail
         Path    /var/log/containers/*.log
         Exclude_Path    /var/log/containers/*_kubesphere-logging-system_events-exporter*.log,/var/log/containers/kube-auditing-webhook*_kubesphere-logging-system_kube-auditing-webhook*.log
         Refresh_Interval    10
         Skip_Long_Lines    true
         DB    /fluent-bit/tail/pos.db
         DB.Sync    Normal
         Mem_Buf_Limit    50MB
         Parser    docker
         Tag    kube.*
     [Input]
         Name    tail
         Path    /var/log/containers/kube-auditing-webhook*_kubesphere-logging-system_kube-auditing-webhook*.log
         Refresh_Interval    10
         Skip_Long_Lines    true
         DB    /fluent-bit/tail/pos-auditing.db
         DB.Sync    Normal
         Mem_Buf_Limit    50MB
         Parser    docker
         Tag    kube_auditing
     [Input]
         Name    tail
         Path    /var/log/containers/*_kubesphere-logging-system_events-exporter*.log
         Refresh_Interval    10
         Skip_Long_Lines    true
         DB    /fluent-bit/tail/pos-events.db
         DB.Sync    Normal
         Mem_Buf_Limit    50MB
         Parser    docker
         Tag    kube_events
     [Filter]
         Name    parser
         Match    kube_auditing
         Key_Name    log
         Parser    json
     [Filter]
         Name    modify
         Match    kube_auditing
         Condition    Key_does_not_exist    AuditID    
         Add    ignore    true
     [Filter]
         Name    grep
         Match    kube_auditing
         Exclude    ignore true
     [Filter]
         Name    parser
         Match    kube_events
         Key_Name    log
         Parser    json
     [Filter]
         Name    kubernetes
         Match    kube.*
         Kube_URL    https://kubernetes.default.svc:443
         Kube_CA_File    /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
         Kube_Token_File    /var/run/secrets/kubernetes.io/serviceaccount/token
         Labels    false
         Annotations    false
     [Filter]
         Name    nest
         Match    kube.*
         Operation    lift
         Nested_under    kubernetes
         Add_prefix    kubernetes_
     [Filter]
         Name    modify
         Match    kube.*
         Remove    stream
         Remove    kubernetes_pod_id
         Remove    kubernetes_host
         Remove    kubernetes_container_hash
     [Filter]
         Name    nest
         Match    kube.*
         Operation    nest
         Wildcard    kubernetes_*
         Nest_under    kubernetes
         Remove_prefix    kubernetes_
     [Filter]
         Name    lua
         Match    service.*
         script    /fluent-bit/config/systemd.lua
         call    add_time
         time_as_table    true
     [Output]
         Name    es
         Match_Regex    (?:kube|service)\.(.*)
         Host    192.168.9.95
         Port    9200
         HTTP_User    lstack
         HTTP_Passwd    P@88w0rd
         Logstash_Format    true
         Logstash_Prefix    ks-logstash-log
         Time_Key    @timestamp
         Generate_ID    true
         Trace_Error    true
         tls    On
         tls.verify    false
     [Output]
         Name    es
         Match    kube_auditing
         Host    192.168.9.95
         Port    9200
         HTTP_User    lstack
         HTTP_Passwd    P@88w0rd
         Logstash_Format    true
         Logstash_Prefix    ks-logstash-auditing
         Generate_ID    true
         Trace_Error    true
         tls    On
         tls.verify    false
     [Output]
         Name    es
         Match    kube_events
         Host    192.168.9.95
         Port    9200
         HTTP_User    lstack
         HTTP_Passwd    P@88w0rd
         Logstash_Format    true
         Logstash_Prefix    ks-logstash-events
         Generate_ID    true
         Trace_Error    true
         tls    On
         tls.verify    false
     ```

   - 可以看到配置文件里已经存在了**Trace_Error    true**，但是是否有效果、有变化呢？我们需要重建**Fluent-bit**的守护进程来看看

   - 登录控制台，**集群管理**->**应用负载**->**工作负载**->**守护进程集**，找到**fluent-bit**，点击**重新创建**

   - ![kubesphere-daemonsets-flunet-bit-2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-daemonsets-flunet-bit-2.png)

   - 然后点击**fluent-bit**，进入详情页面，可以看到有一个pod重建了（至于为什么其他的没变，咱先不说）

   - ![kubesphere-daemonsets-flunet-bit-3](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-daemonsets-flunet-bit-3.png)

   - 进入新创建的pod，咱看看日志输出有变化么！！！

   - ```yaml
      [2022/04/22 01:16:35] [ warn] [engine] failed to flush chunk '15-1650590190.748640950.flb', retry in 6 seconds: task_id=0, input=systemd.0 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:35] [ warn] [engine] failed to flush chunk '15-1650590190.792218570.flb', retry in 6 seconds: task_id=1, input=systemd.1 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:35] [ warn] [engine] failed to flush chunk '15-1650590191.359045217.flb', retry in 7 seconds: task_id=2, input=systemd.1 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:35] [ warn] [engine] failed to flush chunk '15-1650590191.862711494.flb', retry in 8 seconds: task_id=3, input=systemd.1 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:35] [ warn] [engine] failed to flush chunk '15-1650590192.369644921.flb', retry in 11 seconds: task_id=4, input=systemd.1 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:35] [ warn] [engine] failed to flush chunk '15-1650590192.873445035.flb', retry in 7 seconds: task_id=5, input=systemd.1 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:35] [ warn] [engine] failed to flush chunk '15-1650590193.381872747.flb', retry in 8 seconds: task_id=6, input=systemd.1 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:35] [ warn] [engine] failed to flush chunk '15-1650590193.897774507.flb', retry in 9 seconds: task_id=7, input=systemd.1 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:35] [ warn] [engine] failed to flush chunk '15-1650590190.874190363.flb', retry in 11 seconds: task_id=8, input=tail.2 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:35] [ warn] [engine] failed to flush chunk '15-1650590190.980708345.flb', retry in 8 seconds: task_id=9, input=tail.2 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:35] [ warn] [engine] failed to flush chunk '15-1650590194.547452615.flb', retry in 9 seconds: task_id=10, input=tail.2 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:40] [ warn] [engine] failed to flush chunk '15-1650590195.289567444.flb', retry in 11 seconds: task_id=11, input=systemd.1 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:40] [ warn] [engine] failed to flush chunk '15-1650590195.158774254.flb', retry in 8 seconds: task_id=12, input=tail.2 > output=es.0 (out_id=0)
       
      [2022/04/22 01:16:41] [ warn] [engine] chunk '15-1650590190.748640950.flb' cannot be retried: task_id=0, input=systemd.0 > output=es.0
       
      [2022/04/22 01:16:41] [ warn] [engine] chunk '15-1650590190.792218570.flb' cannot be retried: task_id=1, input=systemd.1 > output=es.0
       
      [2022/04/22 01:16:42] [ warn] [engine] chunk '15-1650590191.359045217.flb' cannot be retried: task_id=2, input=systemd.1 > output=es.0
       
      [2022/04/22 01:16:42] [ warn] [engine] chunk '15-1650590192.873445035.flb' cannot be retried: task_id=5, input=systemd.1 > output=es.0
       
      [2022/04/22 01:16:43] [ warn] [engine] chunk '15-1650590191.862711494.flb' cannot be retried: task_id=3, input=systemd.1 > output=es.0
       
      [2022/04/22 01:16:43] [ warn] [engine] chunk '15-1650590193.381872747.flb' cannot be retried: task_id=6, input=systemd.1 > output=es.0
       
      [2022/04/22 01:16:43] [ warn] [engine] chunk '15-1650590190.980708345.flb' cannot be retried: task_id=9, input=tail.2 > output=es.0
       
      [2022/04/22 01:16:44] [ warn] [engine] chunk '15-1650590193.897774507.flb' cannot be retried: task_id=7, input=systemd.1 > output=es.0
       
      [2022/04/22 01:16:44] [ warn] [engine] chunk '15-1650590194.547452615.flb' cannot be retried: task_id=10, input=tail.2 > output=es.0
     ```

   - 然而并没有什么x用，还是只有这些简单的输出，**说好的详细日志呢！！！**

   - 既然**traceError**不行，那我们再去看看**traceOutput**，细节不说了，总之配置完以后没啥变化。此时，我己经是**彻底崩溃**了，各种尝试都堵死了，到现在居然还没看到具体为啥。

     > **你们看文档这么点内容以为事情刚发生？？？实际上已经过去3天了！！！从这个事就能看出来，我其实很笨的）**

   - 深度怀疑前面的两个参数，虽然配置上了，但是并没有生效，但是我有没有证据。暂时没有其他处理思路了。

   - 放弃自查了，去各种群里问一圈，看看有没有现成的可以吃，然后我发现没朋友的可悲了，问了一圈居然没人搭理我，唉！！！

5. 放弃后的第一次站起，继续搞起

   - 既然群里没找到答案，继续自力更生吧

   - 分析一波现状

   - Fluent-bit日志报错关键词**failed to flush chunk**，ElasticSearch日志报错关键词**java.lang.IllegalArgumentException: invalid version format**和**java.lang.IllegalArgumentException: text is empty**。

   - ```yaml
     [2022-04-22T09:57:10,374][DEBUG][r.suppressed             ] [es-node-0] path: /bad-request, params: {}
     java.lang.IllegalArgumentException: invalid version format: C1®Þ\_¾L6Q>À,À0©Ì¨ÌªÀ+À/$À(KÀ#À'GÀ
             at io.netty.handler.codec.http.HttpVersion.<init>(HttpVersion.java:116) ~[netty-codec-http-4.1.66.Final.jar:4.1.66.Final]
             
     [2022-04-22T09:55:06,379][DEBUG][r.suppressed             ] [es-node-0] path: /bad-request, params: {}
     java.lang.IllegalArgumentException: text is empty (possibly HTTP/0.9)
             at io.netty.handler.codec.http.HttpVersion.valueOf(HttpVersion.java:65) ~[netty-codec-http-4.1.66.Final.jar:4.1.66.Final]
     ```

   - Fluent-bit报错**failed to flush chunk**说明连接elasticsearch失败。

   - ElasticSearch端的两个报错，说明Fluent-bit连接请求过来了，但是因为某种原因导致了ElasticSearch报错了。看报错应该是传递过来的数据不被ElasticSearch认可，也就是**invalid version format**和**text is empty**。

   - 我之前在两边开**Trace**，也是为了找到Fluent-bit传递了啥，ElasticSeach到底收了啥，但是两端都没有详细结果。

   - 排查到现在我一直是在es采用https**协议并配置了认证的场景中测试的，我感觉这个问题可能出在https认证上，为了验证我的想法，我们将es改为http的协议，并取消认证

6. 将es改为http协议，验证一下KubeSphere日志系统是否正常

   - 将kubesphere的日志系统停用，删除Fluent-bit的相关配置项

   - **平台管理**->**集群管理**->**CRD**,搜索**ClusterConfiguration**，编辑**ks-installer**,按下面的修改后，点击确定。

   - ```yaml
     auditing:
       enabled: false
      
     events:
       enabled: false
      
     logging:
       containerruntime: docker
       enabled: false
       logsidecar:
         enabled: true
         replicas: 2
     ```

   - 打开工具箱中的kubectl工具，执行下面的命令观察执行过程，等待任务完成，细节不说了，上面都有。

   - ```shell
     / # kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
     ```

   - 执行完了，我们看看KubeSphere有啥变化，我预期的是跟这几个日志组件有关的服务配置都应该删除了才对。

   - 但是，事与愿违，好像什么都没发生，之前的几个主角还在呢。至于为什么，暂时不要问我啊，我也不知道。。。

   - ![kubesphere-daemonsets-flunet-bit-4](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-daemonsets-flunet-bit-4-0619543.png)

   - ![kubesphere-daemonsets-flunet-bit-5](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-daemonsets-flunet-bit-5.png)

   - ![kubesphere-daemonsets-flunet-bit-6](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-daemonsets-flunet-bit-6.png)

   - ![kubesphere-daemonsets-flunet-bit-7](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-daemonsets-flunet-bit-7.png)

   - **工具箱**中的容器日志查询也都在呢，只是报错不一样了，这个问题据说要重启ks-apiserver解决，暂时咱不管他。

   - ![kubesphere-daemonsets-flunet-bit-8](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-daemonsets-flunet-bit-8.png)

   - 没办法了，为了实验的准确性我只能强制赶你们走了，手动的把Fluent-bit的operator和守护进程集删掉，直接界面执行操作，比较简单，不截图了

   - 再把**CRD**中的**Output**和**Input**资源也删掉。

   - ![kubesphere-crd-delete-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-crd-delete-1.png)

   - ![kubesphere-crd-delete-2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-crd-delete-2.png)

   - ![kubesphere-crd-delete-3](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-crd-delete-3.png)

   - ![kubesphere-crd-delete-4](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-crd-delete-4.png)

   - 执行完上面的删除操作以后，我们会发现界面开始有了一些变化（也不排除是早期的操作延时导致的，先不深究了，可能以后也不会原因深究）

   - ![kubesphere-crd-delete-5](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-crd-delete-5.png)

   - ![kubesphere-crd-delete-6](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-crd-delete-6.png)

   - 其实，测试验证的环境不删也就不删了，改改配置也是可以的，只是突然强迫症犯了，必须干掉。

   - 确认全部删除后，我们开始重装ElasticSearch

   - 利用ansible重新配置elasticsearch（仅限于测试验证环境，有数据的生产环境就不要瞎搞了）

   - ```shell
     # 利用ansible卸载旧的elasticsearch并删除数据
     [root@zdevops-master ~]# cd /data/ansible/ansible-zdevops/inventories/dev
     [root@zdevops-master dev]# source /opt/ansible2.8/bin/activate
     (ansible2.8) [root@zdevops-master dev]# ansible es -m shell -a 'systemctl stop elasticsearch'
     (ansible2.8) [root@zdevops-master dev]# ansible es -m shell -a 'yum remove -y elasticsearch'
     (ansible2.8) [root@zdevops-master dev]# ansible es -m file -a 'path=/data/elasticsearch state=absent'
     (ansible2.8) [root@zdevops-master dev]# ansible es -m file -a 'path=/etc/elasticsearch state=absent'
     
     # 利用ansible-playbook安装http模式的elasticsearch，注意-e重新定义变量elasticsearch_xpack_security_enabled的值为false
     
     (ansible2.8) [root@zdevops-master dev]# ansible-playbook -e elasticsearch_xpack_security_enabled=false ../../playbooks/deploy-elasticsearch.yaml 
     /opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
       from cryptography.exceptions import InvalidSignature
     
     PLAY [安装配置ElasticSearch.] ********************************************************************************
     
     TASK [01-配置yum源-配置elasticsearch软件源.] *********************************************************************
     ok: [es-node-1]
     ok: [es-node-0]
     ok: [es-node-2]
     
     TASK [02-安装elasticsearch.] *******************************************************************************
     changed: [es-node-2]
     changed: [es-node-1]
     changed: [es-node-0]
     
     TASK [03-创建elasticsearch配置文件.] ***************************************************************************
     changed: [es-node-0]
     changed: [es-node-2]
     changed: [es-node-1]
     
     TASK [04-创建elasticsearch数据文件目录.] *************************************************************************
     changed: [es-node-0]
     changed: [es-node-2]
     changed: [es-node-1]
     
     PLAY [安装配置ElasticSearch SSL认证.] **************************************************************************
     
     TASK [01-生成ElasticSearch Instance文件用来配置ssl证书.] ***********************************************************
     skipping: [es-node-0]
     
     TASK [02-生成证书.] ******************************************************************************************
     skipping: [es-node-0]
     
     TASK [03-安装基本工具包.] ***************************************************************************************
     skipping: [es-node-0] => (item=[]) 
     
     TASK [04-解压cert文件.] **************************************************************************************
     skipping: [es-node-0]
     
     TASK [05-从服务器获取cert配置文件.] ********************************************************************************
     skipping: [es-node-0] => (item=es-node-0) 
     skipping: [es-node-0] => (item=es-node-1) 
     skipping: [es-node-0] => (item=es-node-2) 
     
     PLAY [认证配置.] *********************************************************************************************
     
     TASK [01-创建elasticsearch cert目录.] ************************************************************************
     skipping: [es-node-0]
     skipping: [es-node-1]
     skipping: [es-node-2]
     
     TASK [02-同步cert文件.] **************************************************************************************
     skipping: [es-node-0]
     skipping: [es-node-1]
     skipping: [es-node-2]
     
     TASK [03-生成keystore文件.] **********************************************************************************
     skipping: [es-node-0]
     skipping: [es-node-1]
     skipping: [es-node-2]
     
     TASK [04-添加配置项到keystore文件.] ******************************************************************************
     skipping: [es-node-0]
     skipping: [es-node-1]
     skipping: [es-node-2]
     
     TASK [05-创建用户并指定superuser权限.] ****************************************************************************
     skipping: [es-node-0]
     skipping: [es-node-1]
     skipping: [es-node-2]
     
     PLAY [终极配置.] *********************************************************************************************
     
     TASK [01-启动并设置开机自动启动elasticsearch服务.] ********************************************************************
     changed: [es-node-1]
     changed: [es-node-2]
     changed: [es-node-0]
     
     PLAY RECAP ***********************************************************************************************
     es-node-0                  : ok=5    changed=4    unreachable=0    failed=0    skipped=10   rescued=0    ignored=0   
     es-node-1                  : ok=5    changed=4    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0   
     es-node-2                  : ok=5    changed=4    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0   
     ```
     
   - 查看ElasticSearch集群初始分片信息

   - ```shell
     [root@glusterfs-node-0 ~]# curl  192.168.9.97:9200/_cat/indices?v
     health status index            uuid                   pri rep docs.count docs.deleted store.size pri.store.size
     green  open   .geoip_databases DUibpJGVR66YyMiMxiUYpw   1   1          8            0     14.1mb            7mb
     ```

   - **平台管理**->**集群管理**->**CRD**,搜索**ClusterConfiguration**，编辑**ks-installer**,按下面的修改后，点击确定。

   - ```yaml
     auditing:
       enabled: true
     es:
       basicAuth:
         enabled: false
         password: ''
         username: ''
       elkPrefix: logstash
       externalElasticsearchHost: 192.168.9.95
       externalElasticsearchPort: 9200
       logMaxAge: 7
     events:
       enabled: true
      
     logging:
       containerruntime: docker
       enabled: true
       logsidecar:
         enabled: true
         replicas: 2
     ```

   - 打开工具箱中的kubectl工具，执行下面的命令观察执行过程，等待任务完成，细节不说了，上面都有。

   - ```shell
     / # kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
     ```

   - 执行完以后，正常的话我们之前操作的资源应该都回归了，**Output**、**Input**、**Fluent-bit**，**工具箱里的工具**不截图了自己看一下吧。

   - 但是我们会发现**Fluent-bit**的pod中还有如下报错

   - ```yaml
     [2022/04/22 15:20:20] [ warn] [net] getaddrinfo(host='elasticsearch-logging-data.kubesphere-logging-system.svc', err=4): Domain name not found
     
     [2022/04/22 15:20:20] [ warn] [net] getaddrinfo(host='elasticsearch-logging-data.kubesphere-logging-system.svc', err=4): Domain name not found
     ```

   - 看到这个知道怎么搞了吧？不知道？？？去上面翻翻

   - 依次修改**Output**中的**es**、**es-auditing**、**es-events**，改后如下：

   - ```yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: >
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"logging","logging.kubesphere.io/enabled":"true"},"name":"es","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","logstashFormat":true,"logstashPrefix":"ks-logstash-log","port":9200,"timeKey":"@timestamp"},"matchRegex":"(?:kube|service)\\.(.*)"}}
       labels:
         logging.kubesphere.io/component: logging
         logging.kubesphere.io/enabled: 'true'
       name: es
       namespace: kubesphere-logging-system
     spec:
       es:
         generateID: true
         host: 192.168.9.95
         logstashFormat: true
         logstashPrefix: ks-logstash-log
         port: 9200
         timeKey: '@timestamp'
       matchRegex: '(?:kube|service)\.(.*)'
     ```

   - ```yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: >
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"auditing","logging.kubesphere.io/enabled":"true"},"name":"es-auditing","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","logstashFormat":true,"logstashPrefix":"ks-logstash-auditing","port":9200},"match":"kube_auditing"}}
       labels:
         logging.kubesphere.io/component: auditing
         logging.kubesphere.io/enabled: 'true'
       name: es-auditing
       namespace: kubesphere-logging-system
     spec:
       es:
         generateID: true
         host: 192.168.9.95
         logstashFormat: true
         logstashPrefix: ks-logstash-auditing
         port: 9200
       match: kube_auditing
     ```

   - ```yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: >
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"events","logging.kubesphere.io/enabled":"true"},"name":"es-events","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","logstashFormat":true,"logstashPrefix":"ks-logstash-events","port":9200},"match":"kube_events"}}
       labels:
         logging.kubesphere.io/component: events
         logging.kubesphere.io/enabled: 'true'
       name: es-events
       namespace: kubesphere-logging-system
     spec:
       es:
         generateID: true
         host: 192.168.9.95
         logstashFormat: true
         logstashPrefix: ks-logstash-events
         port: 9200
       match: kube_events
     ```

   - 再去查看Fluent-bit容器的日志，你会发现Fluent-bit内部发现了配置文件的变化，并重启了**Fluent Bit**服务，且容器并没有重建。

   - ```yaml
      [2022/04/22 15:27:04] [engine] caught signal (SIGTERM)
       
      [2022/04/22 15:27:04] [ info] [input] pausing systemd.0
       
      [2022/04/22 15:27:04] [ info] [input] pausing systemd.1
       
      [2022/04/22 15:27:04] [ info] [input] pausing tail.2
       
      [2022/04/22 15:27:04] [ info] [input] pausing tail.3
       
      [2022/04/22 15:27:04] [ info] [input] pausing tail.4
       
      level=info msg="Config file changed, stopping Fluent Bit"
       
      level=info msg="Killed Fluent Bit"
       
      level=info msg="Config file changed, stopped Fluent Bit"
       
      level=info msg="Config file changed, stopping Fluent Bit"
       
      [2022/04/22 15:27:04] [ warn] [engine] service will stop in 5 seconds
       
      level=info msg="Killed Fluent Bit"
       
      level=info msg="Config file changed, stopped Fluent Bit"
       
      level=info msg="Config file changed, stopping Fluent Bit"
       
      level=info msg="Killed Fluent Bit"
       
      level=info msg="Config file changed, stopped Fluent Bit"
       
      [2022/04/22 15:27:09] [ info] [engine] service stopped
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=416649 watch_fd=1
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=461549 watch_fd=2
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134582558 watch_fd=3
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134581715 watch_fd=4
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=268955529 watch_fd=5
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=373423 watch_fd=6
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=373208 watch_fd=7
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403716654 watch_fd=8
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134583208 watch_fd=9
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134583458 watch_fd=10
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=269010417 watch_fd=11
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=269628719 watch_fd=12
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403995814 watch_fd=13
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=962171 watch_fd=14
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134583441 watch_fd=15
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134580575 watch_fd=16
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=268955289 watch_fd=17
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403715964 watch_fd=18
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403855647 watch_fd=19
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=413299 watch_fd=20
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=375870 watch_fd=21
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134581094 watch_fd=22
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=463110 watch_fd=23
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=135190584 watch_fd=24
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134738131 watch_fd=25
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=268956011 watch_fd=26
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=373969 watch_fd=27
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403792092 watch_fd=28
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=412431 watch_fd=29
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134738124 watch_fd=30
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=467559 watch_fd=31
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403860482 watch_fd=32
       
      [2022/04/22 15:27:09] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=997586 watch_fd=33
       
      level=error msg="Fluent bit exited" error=null
       
      level=info msg=backoff delay=1s
       
      level=info msg="backoff timer done" actual=1.00029973s expected=1s
       
      level=info msg="Fluent bit started"
       
      Fluent Bit v1.8.3
       
      * Copyright (C) 2019-2021 The Fluent Bit Authors
       
      * Copyright (C) 2015-2018 Treasure Data
       
      * Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
       
      * https://fluentbit.io
       
      
       
      [2022/04/22 15:27:10] [ info] [engine] started (pid=30)
       
      [2022/04/22 15:27:10] [ info] [storage] version=1.1.1, initializing...
       
      [2022/04/22 15:27:10] [ info] [storage] in-memory
       
      [2022/04/22 15:27:10] [ info] [storage] normal synchronization mode, checksum disabled, max_chunks_up=128
       
      [2022/04/22 15:27:10] [ info] [cmetrics] version=0.1.6
       
      [2022/04/22 15:27:10] [ info] [input:systemd:systemd.0] seek_cursor=s=5d0022eeaf454ba8a40dde2836de2f4d;i=b56... OK
       
      [2022/04/22 15:27:10] [ info] [input:systemd:systemd.1] seek_cursor=s=5d0022eeaf454ba8a40dde2836de2f4d;i=b5b... OK
       
      [2022/04/22 15:27:10] [ info] [filter:kubernetes:kubernetes.4] https=1 host=kubernetes.default.svc port=443
       
      [2022/04/22 15:27:10] [ info] [filter:kubernetes:kubernetes.4] local POD info OK
       
      [2022/04/22 15:27:10] [ info] [filter:kubernetes:kubernetes.4] testing connectivity with API server...
       
      [2022/04/22 15:27:10] [ info] [filter:kubernetes:kubernetes.4] connectivity OK
       
      [2022/04/22 15:27:10] [ info] [sp] stream processor started
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=416649 watch_fd=1 name=/var/log/containers/alertmanager-main-2_kubesphere-monitoring-system_alertmanager-1a71f1d5b1136320644f3289e4b22544620db4a0d35a1ffec52bc534d729c358.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=461549 watch_fd=2 name=/var/log/containers/alertmanager-main-2_kubesphere-monitoring-system_config-reloader-bc14c53008f1e800d6240a5b912239a3d5682c6dcb719f48c69b7e8ba5892e35.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134582558 watch_fd=3 name=/var/log/containers/calico-kube-controllers-75ddb95444-8xvf4_kube-system_calico-kube-controllers-dc90044aa0ce62e61dfb0842c8f2e5aa8d021738adffca847b003a5c9621512c.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268955529 watch_fd=4 name=/var/log/containers/calico-node-pzvrj_kube-system_flexvol-driver-abf59d8a7af22c683dfb0776a0835c719f41ef23446e912f82901a4fd9cf166f.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373423 watch_fd=5 name=/var/log/containers/calico-node-pzvrj_kube-system_install-cni-b3d49f2e9c03e63c3863d50365d2ade01dead979c90285c526ac0029b58411cd.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373208 watch_fd=6 name=/var/log/containers/calico-node-pzvrj_kube-system_upgrade-ipam-6d4425d7ea5302511df1c278f2b113ac8fdfa0a372e07b025e0a9b45d30129ac.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403716654 watch_fd=7 name=/var/log/containers/coredns-5495dd7c88-68k9b_kube-system_coredns-879c087a4c8717d0886e6603a63e9503a406d215cfe77941cbcc1cc7c138d010.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134583208 watch_fd=8 name=/var/log/containers/coredns-5495dd7c88-z9q6j_kube-system_coredns-80cd4427410de02465b415683817a75674a95777ffc0929f93e87506b120ca59.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134583458 watch_fd=9 name=/var/log/containers/ks-apiserver-5c68568694-w8xs6_kubesphere-system_ks-apiserver-b76c824251f298a17d7594eea392602cea7291e1b3e35ae35cefb4607e6e4cdf.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269010417 watch_fd=10 name=/var/log/containers/ks-console-65f4d44d88-hzw9b_kubesphere-system_ks-console-f600455c3af064c53adc8eb7e5412505c4d0e50362ab11b8e59d2e71b552b0f1.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269628719 watch_fd=11 name=/var/log/containers/ks-controller-manager-bf6b9bfb5-xfts6_kubesphere-system_ks-controller-manager-9c24833ae6bce8c0ff956db38b40d9acf0224ec364c317ebcefc7802d4d97855.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403995814 watch_fd=12 name=/var/log/containers/ks-events-ruler-575669b4-mh2fn_kubesphere-logging-system_config-reloader-2d51bab0babdd47864ec235efdbd69d2075cc8bf72c3642449be69f68aa1f0d5.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=962171 watch_fd=13 name=/var/log/containers/ks-events-ruler-575669b4-mh2fn_kubesphere-logging-system_events-ruler-da24092a12c4fc0d0aab8a02c7fc7cb1e0e82d15a924717222bdbadee7935c84.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134580575 watch_fd=14 name=/var/log/containers/kube-controller-manager-ks-k8s-master-0_kube-system_kube-controller-manager-a07d1c5ed4108d98de2ba322b47939d5007c9d266a4ff558d57ec87ab10b2d54.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268955289 watch_fd=15 name=/var/log/containers/kube-proxy-bc59k_kube-system_kube-proxy-79144efa6b42a0f9636132538cc4ade6665269fea5903f3e1e0be05f14376d7c.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403715964 watch_fd=16 name=/var/log/containers/kube-scheduler-ks-k8s-master-0_kube-system_kube-scheduler-1a2ffcf4000a9bb0fa526fcfa1970e69465dfe0bb4c635a4d95f50f2e27ef763.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403855647 watch_fd=17 name=/var/log/containers/minio-859cb4d777-mpsk9_kubesphere-system_minio-d76f9516fa485cd24851afccfb2c85fd74ce894ba580af98703704860e1c00ed.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=413299 watch_fd=18 name=/var/log/containers/node-exporter-wwrpf_kubesphere-monitoring-system_kube-rbac-proxy-e35229db1e46e8b7b0a7d5a3cd581107102a3531b31d2e32d7e787a118c82896.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=375870 watch_fd=19 name=/var/log/containers/node-exporter-wwrpf_kubesphere-monitoring-system_node-exporter-e5cc5a4cc937cf9de149dd25bd6e8441ca176bf26cb2a214dd0b47627e7d8861.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134581094 watch_fd=20 name=/var/log/containers/nodelocaldns-sp59h_kube-system_node-cache-8b5bdf5a116af255f979a4a349c168257b632d24805d5f257a642dd97b0d53a1.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=463110 watch_fd=21 name=/var/log/containers/prometheus-k8s-0_kubesphere-monitoring-system_config-reloader-1378a8d9cd7a96334cb15adeafb468497ece0a9a600a3193fb80b60bc5b7e9b5.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=135190584 watch_fd=22 name=/var/log/containers/prometheus-k8s-0_kubesphere-monitoring-system_prometheus-26e94cf0828cd1bd9c5da45217b7f3e654a7a41ce0dc5943375b1644d1a6a002.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134738131 watch_fd=23 name=/var/log/containers/prometheus-k8s-0_kubesphere-monitoring-system_prometheus-97de8b484747bae4aeac54ffd9b75c1ddc9582a28c768ebc9be395390bd031c3.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268956011 watch_fd=24 name=/var/log/containers/redis-ha-haproxy-868fdbddd4-kqrl5_kubesphere-system_config-init-0fb775710352218a66aec02ea2a736892c33df43dd4206e155e0bce6af4aba63.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373969 watch_fd=25 name=/var/log/containers/redis-ha-haproxy-868fdbddd4-kqrl5_kubesphere-system_haproxy-6c15749c02904875f5c1a280a42a908baa074ccffd3365f40b2692b6fe60acf5.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403792092 watch_fd=26 name=/var/log/containers/redis-ha-server-2_kubesphere-system_config-init-fa7c66040a53bef1c3c9b1844a4b25870835a6147cd83abf914ebb129a036536.log
       
      [2022/04/22 15:27:10] [ info] [input:tail:tail.2] inotify_fs_add(): inode=412431 watch_fd=27 name=/var/log/containers/redis-ha-server-2_kubesphere-system_redis-6b19bb18f79c55c
       
     ```

   - ![kubesphere-daemonsets-flunet-bit-9](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-daemonsets-flunet-bit-9.png)

   - 分析日志发现，Fluent-bit端没有错误信息了，再去ElasticSearch中查看索引。

   - **她来了她真的来了**,真xxx不容易。

   - ```yaml
     [root@glusterfs-node-0 ~]# curl  192.168.9.97:9200/_cat/indices?v
     health status index                           uuid                   pri rep docs.count docs.deleted store.size pri.store.size
     green  open   .geoip_databases                DUibpJGVR66YyMiMxiUYpw   1   1         40            0     75.8mb         37.9mb
     green  open   ks-logstash-log-2022.04.22      sG7rEJ2OT-27Kv6k0kq-CQ   1   1        486            0        1mb        529.7kb
     green  open   ks-logstash-auditing-2022.04.22 x52Q_uEZRZ-yNYXPRmyWLg   1   1          1            0     37.4kb         18.7kb
     green  open   ks-logstash-events-2022.04.22   AafsMv0bRd-xz_Uj_HlOag   1   1         10            0    149.5kb         74.6kb
     ```

   - 再回来看看我们KubeSphere工具箱中的几个分析工具是否正常了？

   - ![kubesphere-toolbox](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-toolbox.png)

   - ![kubesphere-toolbox-analysis-tools-container](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-toolbox-analysis-tools-container.png)

   - ![kubesphere-toolbox-analysis-tools-audit](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-toolbox-analysis-tools-audit.png)

   - ![kubesphere-toolbox-analysis-tools-event](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-toolbox-analysis-tools-event.png)

   - 三个查询工具还是有异常弹窗，不过不用这个问题如何处理，我也是心中有数的。

   - 这几天可不是白混的，论坛文档没少看，来打开kubesphere官方论坛，需要找到两篇文档结合来处理。

   - 打开[在使用kusphere的操作审计功能时，总有弹窗提示：dial tcp: lookup http on 10.96.0.10:53: no such host](https://kubesphere.com.cn/forum/d/2450-kuspheredial-tcp-lookup-http-on-109601053-no-such-host/4)，找到修改配置文件的命令（其实吧你改界面也行）

   - ![image-20220422235234281](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220422235234281.png)

   - 先来看看文档中的配置现状

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl get configmap kubesphere-config -n kubesphere-system -o yaml
     apiVersion: v1
     data:
       kubesphere.yaml: |
         authentication:
           authenticateRateLimiterMaxTries: 10
           authenticateRateLimiterDuration: 10m0s
           loginHistoryRetentionPeriod: 168h
           maximumClockSkew: 10s
           multipleLogin: True
           kubectlImage: kubesphere/kubectl:v1.21.0
           jwtSecret: "1xh3ldfKpJSzSmFCbi2HXXbw5dn4o4kv"
           oauthOptions:
             clients:
             - name: kubesphere
               secret: kubesphere
               redirectURIs:
               - '*'
     
         ldap:
           host: openldap.kubesphere-system.svc:389
           managerDN: cn=admin,dc=kubesphere,dc=io
           managerPassword: admin
           userSearchBase: ou=Users,dc=kubesphere,dc=io
           groupSearchBase: ou=Groups,dc=kubesphere,dc=io
     
         redis:
           host: redis.kubesphere-system.svc
           port: 6379
           password: KUBESPHERE_REDIS_PASSWORD
           db: 0
     
     
         s3:
           endpoint: http://minio.kubesphere-system.svc:9000
           region: us-east-1
           disableSSL: True
           forcePathStyle: True
           accessKeyID: openpitrixminioaccesskey
           secretAccessKey: openpitrixminiosecretkey
           bucket: s2i-binaries
     
         network:
           ippoolType: none
         devops:
           host: http://devops-jenkins.kubesphere-devops-system.svc/
           username: admin
           password: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImFkbWluQGt1YmVzcGhlcmUuaW8iLCJ1c2VybmFtZSI6ImFkbWluIiwidG9rZW5fdHlwZSI6InN0YXRpY190b2tlbiJ9.DVnt9FY7UNu2Mvshh_46UMKhZG7_X7NPC-ClQ68ynB0
           maxConnections: 100
           endpoint: http://devops-apiserver.kubesphere-devops-system:9090
         openpitrix:
           s3:
             endpoint: http://minio.kubesphere-system.svc:9000
             region: us-east-1
             disableSSL: True
             forcePathStyle: True
             accessKeyID: openpitrixminioaccesskey
             secretAccessKey: openpitrixminiosecretkey
             bucket: app-store
         monitoring:
           endpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090
           enableGPUMonitoring: false
         gpu:
           kinds:
           - resourceName: nvidia.com/gpu
             resourceType: GPU
             default: True
         notification:
           endpoint: http://notification-manager-svc.kubesphere-monitoring-system.svc:19093
         logging:
           host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200
           indexPrefix: ks-logstash-log
         events:
           host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200
           indexPrefix: ks-logstash-events
         auditing:
           enable: true
           webhookURL: https://kube-auditing-webhook-svc.kubesphere-logging-system.svc:6443/audit/webhook/event
           host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200
           indexPrefix: ks-logstash-auditing
     
         alerting:
           prometheusEndpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090
           thanosRulerEndpoint: http://thanos-ruler-operated.kubesphere-monitoring-system.svc:10902
           thanosRuleResourceLabels: thanosruler=thanos-ruler,role=thanos-alerting-rules
     
     
         gateway:
           watchesPath: /var/helm-charts/watches.yaml
           repository: kubesphere/nginx-ingress-controller
           tag: v0.48.1
           namespace: kubesphere-controls-system
     kind: ConfigMap
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"v1","data":{"kubesphere.yaml":"authentication:\n  authenticateRateLimiterMaxTries: 10\n  authenticateRateLimiterDuration: 10m0s\n  loginHistoryRetentionPeriod: 168h\n  maximumClockSkew: 10s\n  multipleLogin: True\n  kubectlImage: kubesphere/kubectl:v1.21.0\n  jwtSecret: \"1xh3ldfKpJSzSmFCbi2HXXbw5dn4o4kv\"\n  oauthOptions:\n    clients:\n    - name: kubesphere\n      secret: kubesphere\n      redirectURIs:\n      - '*'\n\nldap:\n  host: openldap.kubesphere-system.svc:389\n  managerDN: cn=admin,dc=kubesphere,dc=io\n  managerPassword: admin\n  userSearchBase: ou=Users,dc=kubesphere,dc=io\n  groupSearchBase: ou=Groups,dc=kubesphere,dc=io\n\nredis:\n  host: redis.kubesphere-system.svc\n  port: 6379\n  password: KUBESPHERE_REDIS_PASSWORD\n  db: 0\n\n\ns3:\n  endpoint: http://minio.kubesphere-system.svc:9000\n  region: us-east-1\n  disableSSL: True\n  forcePathStyle: True\n  accessKeyID: openpitrixminioaccesskey\n  secretAccessKey: openpitrixminiosecretkey\n  bucket: s2i-binaries\n\nnetwork:\n  ippoolType: none\ndevops:\n  host: http://devops-jenkins.kubesphere-devops-system.svc/\n  username: admin\n  password: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImFkbWluQGt1YmVzcGhlcmUuaW8iLCJ1c2VybmFtZSI6ImFkbWluIiwidG9rZW5fdHlwZSI6InN0YXRpY190b2tlbiJ9.DVnt9FY7UNu2Mvshh_46UMKhZG7_X7NPC-ClQ68ynB0\n  maxConnections: 100\n  endpoint: http://devops-apiserver.kubesphere-devops-system:9090\nopenpitrix:\n  s3:\n    endpoint: http://minio.kubesphere-system.svc:9000\n    region: us-east-1\n    disableSSL: True\n    forcePathStyle: True\n    accessKeyID: openpitrixminioaccesskey\n    secretAccessKey: openpitrixminiosecretkey\n    bucket: app-store\nmonitoring:\n  endpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090\n  enableGPUMonitoring: false\ngpu:\n  kinds:\n  - resourceName: nvidia.com/gpu\n    resourceType: GPU\n    default: True\nnotification:\n  endpoint: http://notification-manager-svc.kubesphere-monitoring-system.svc:19093\nlogging:\n  host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200\n  indexPrefix: ks-logstash-log\nevents:\n  host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200\n  indexPrefix: ks-logstash-events\nauditing:\n  enable: true\n  webhookURL: https://kube-auditing-webhook-svc.kubesphere-logging-system.svc:6443/audit/webhook/event\n  host: http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200\n  indexPrefix: ks-logstash-auditing\n\nalerting:\n  prometheusEndpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090\n  thanosRulerEndpoint: http://thanos-ruler-operated.kubesphere-monitoring-system.svc:10902\n  thanosRuleResourceLabels: thanosruler=thanos-ruler,role=thanos-alerting-rules\n\n\ngateway:\n  watchesPath: /var/helm-charts/watches.yaml\n  repository: kubesphere/nginx-ingress-controller\n  tag: v0.48.1\n  namespace: kubesphere-controls-system\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"kubesphere-config","namespace":"kubesphere-system"}}
       creationTimestamp: "2022-04-09T14:42:47Z"
       name: kubesphere-config
       namespace: kubesphere-system
       resourceVersion: "3541275"
       uid: 2d45c34b-bc9d-43bd-a3d3-37b294393531
     ```

   - 知道重点是啥了吧，把它**http://elasticsearch-logging-data.kubesphere-logging-system.svc:9200**全改成实际的**ElasticSearch**的地址

   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl edit configmap kubesphere-config -n kubesphere-system
     configmap/kubesphere-config edited
     
     # 可以在编辑模式使用 %s/elasticsearch-logging-data.kubesphere-logging-system.svc/192.168.9.95/g 批量替换
     
     # 实际修改内容如下
     logging:
       host: http://192.168.9.95:9200
       indexPrefix: ks-logstash-log
     events:
       host: http://192.168.9.95:9200
       indexPrefix: ks-logstash-events
     auditing:
       enable: true
       webhookURL: https://kube-auditing-webhook-svc.kubesphere-logging-system.svc:6443/audit/webhook/event
       host: http://192.168.9.95:9200
       indexPrefix: ks-logstash-auditing
     ```

   - 改完以后我发现，好像没啥变化，报错依旧。不过此时我已经不慌了。

   - 继续打开第二篇文档，[使用外部es的问题](https://kubesphere.com.cn/forum/d/6614-es),重点在图片上，红色字体的描述，这里提到了需要重启ks-apiserver才可以。

   - ![image-20220423001514866](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220423001514866.png)

   - 我们再看看ks-apiserver的配置，确定也挂载了kubesphere-config。

   - ![kubesphere-ks-apiserver-2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-ks-apiserver-2.png)

   - 我来重启一下ks-apiserver控制台，验证一下真伪。

   - 控制台，**集群管理**->**应用负载**->**工作负载**->**部署**->**ks-apiserver**,重启。

   - ![kubesphere-ks-apiserver-0](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-ks-apiserver-0.png)

   - ![kubesphere-ks-apiserver-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-ks-apiserver-1.png)

   - 重启的时候，右上角会有报错，不用管它。全部重启完以后我们再看看工具箱中的几个日志分析工具。

   - ![kubesphere-toolbox-analysis-tools-containe-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-toolbox-analysis-tools-containe-1.png)

   - 输入一个关键词，查询一下看看具体效果，而且从搜索框中还可以看到kubesphere日志系统支持的查询粒度。

   - ![kubesphere-toolbox-analysis-tools-container-2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-toolbox-analysis-tools-container-2.png)

   - 接下来在看看审计和事件功能。

   - ![kubesphere-toolbox-analysis-tools-event-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-toolbox-analysis-tools-event-1.png)

   - ![kubesphere-toolbox-analysis-tools-audit-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-toolbox-analysis-tools-audit-1.png)

   - 至此，KubeSphere日志系统对接外部基于Http协议的ElasticSearch集群算是彻底完成了，该有的都有了，至于后面的使用效果，那只能线上持续观测了。

   - 但是你们以为这就完了，打完收工么，想啥呢，还有一个大坑没填呢，HTTPS的ElasticSearch还没搞定呢，继续搞它。

7. 继续我们的ElasticSearch的HTTPS配置之旅

   - 通过上面的过程我们应该可以断定，KubeSphere部署的fluent-bit对接http协议的ES是没有任何问题的，那问题就是出在https协议上。我们换个思路，先跑掉Kubesphere不说，先去看看原生的Fluent-bit对接https协议的es有啥套路
   
   - 搜索关键词**fluent-bit https elasticsearch**
   
   - 各种搜索引擎都尝试过，都是大同小异，无非就是关掉tls验证，有的文档还提到了加上ca的配置。并没有解决问题的思路和方案（此时我还没有醒悟，我一开始方向就错了，后面细说），也就懒得截图了。
   
   - 上面的关键词不行，我又换了一个思路**fluent-bit x-pack elasticsearch**，毕竟我们的安全验证是启用了x-pack插件。
   
   - 哈哈哈哈，别说我还真找到一篇有意义并帮我最终解决问题的文档，那就是[它](https://vqiu.cn/efk-kube/)
   
   - ![image-20220428110517048](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220428110517048.png)
   
   - 此文档介绍的都是在k8s安装es，但是这个不重要，配置文件都是有参考意义的，看看他的配置文件：
   
   - ![image-20220428110847858](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/image-20220428110847858.png)
   
   - 看到这个配置文件，我突然灵光一闪，这个配置里没有tls什么事啊。此时，我才幡然醒悟，其实我要的并不是https并不是tls，我要的其实就是个**http协议加上用户名和密码验证**，这才是我真正的需求。
   
   - 我一开始就走错了路，被**ClusterConfiguration**中的这个参数**externalElasticsearchProtocol: https**给坑了，此时我已经忘记从哪里看到的文档需要配置这个参数了，我是一个不记仇的人，我就不找后帐了。
   
   - 思路有了，来我们验证一波。
   
   - 先**ClusterConfiguration**->**ks-installer**,编辑YAML。确保配置文件中es的配置如下，如果存在**externalElasticsearchProtocol: https**一定要干掉它。
   
   - ```yaml
     es:
       basicAuth:
         enabled: false
         password: P@88w0rd
         username: lstack
       elkPrefix: logstash
       externalElasticsearchHost: 192.168.9.95
       externalElasticsearchPort: 9200
       logMaxAge: 7
   
   - 修改完成后，点击**确定**，等待后台重新配置完成。
   
   - 再去查看**Output**的几个配置文件，**es**、**es-auditing**、**es-events**具体如下
   
   - ```yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: >
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"logging","logging.kubesphere.io/enabled":"true"},"name":"es","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","logstashFormat":true,"logstashPrefix":"ks-logstash-log","port":9200,"timeKey":"@timestamp"},"matchRegex":"(?:kube|service)\\.(.*)"}}
       labels:
         logging.kubesphere.io/component: logging
         logging.kubesphere.io/enabled: 'true'
       name: es
       namespace: kubesphere-logging-system
     spec:
       es:
         generateID: true
         host: 192.168.9.95
         httpPassword:
           valueFrom:
             secretKeyRef:
               key: password
               name: elasticsearch-credentials
         httpUser:
           valueFrom:
             secretKeyRef:
               key: username
               name: elasticsearch-credentials
         logstashFormat: true
         logstashPrefix: ks-logstash-log
         port: 9200
         timeKey: '@timestamp'
       matchRegex: '(?:kube|service)\.(.*)'
     
     ```
   
   - ```yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: >
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"auditing","logging.kubesphere.io/enabled":"true"},"name":"es-auditing","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","logstashFormat":true,"logstashPrefix":"ks-logstash-auditing","port":9200,"tls":{"verify":false}},"match":"kube_auditing"}}
       labels:
         logging.kubesphere.io/component: auditing
         logging.kubesphere.io/enabled: 'true'
       name: es-auditing
       namespace: kubesphere-logging-system
     spec:
       es:
         generateID: true
         host: 192.168.9.95
         httpPassword:
           valueFrom:
             secretKeyRef:
               key: password
               name: elasticsearch-credentials
         httpUser:
           valueFrom:
             secretKeyRef:
               key: username
               name: elasticsearch-credentials
         logstashFormat: true
         logstashPrefix: ks-logstash-log
         port: 9200
       match: kube_auditing
     ```
   
   - ```yaml
     apiVersion: logging.kubesphere.io/v1alpha2
     kind: Output
     metadata:
       annotations:
         kubectl.kubernetes.io/last-applied-configuration: >
           {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Output","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/component":"events","logging.kubesphere.io/enabled":"true"},"name":"es-events","namespace":"kubesphere-logging-system"},"spec":{"es":{"generateID":true,"host":"elasticsearch-logging-data.kubesphere-logging-system.svc","logstashFormat":true,"logstashPrefix":"ks-logstash-events","port":9200,"tls":{"verify":false}},"match":"kube_events"}}
       labels:
         logging.kubesphere.io/component: events
         logging.kubesphere.io/enabled: 'true'
       name: es-events
       namespace: kubesphere-logging-system
     spec:
       es:
         generateID: true
         host: 192.168.9.95
         httpPassword:
           valueFrom:
             secretKeyRef:
               key: password
               name: elasticsearch-credentials
         httpUser:
           valueFrom:
             secretKeyRef:
               key: username
               name: elasticsearch-credentials
         logstashFormat: true
         logstashPrefix: ks-logstash-log
         port: 9200
       match: kube_events
     ```
   
   - Output配置文件修改完成后，查看fluent-bit的容器log，看看是否正常输出日志到es了，在日志输出里可以发现，修改配置文件之前，还是有大量的报错，修改配置文件后fluent-bit监测到配置文件发生变化，自动重启了服务，启动后，没有报错，说明日志可以正常输出es。
   
   - ```shell
     [root@ks-k8s-master-0 ~]# kubectl logs  fluent-bit-xq2jb -n kubesphere-logging-system
     
     [2022/04/28 02:58:19] [ warn] [engine] failed to flush chunk '39-1651114694.735704833.flb', retry in 9 seconds: task_id=4, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/28 02:58:19] [ warn] [engine] failed to flush chunk '39-1651114695.509379382.flb', retry in 10 seconds: task_id=5, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/28 02:58:20] [ warn] [engine] chunk '39-1651114693.743740772.flb' cannot be retried: task_id=2, input=tail.2 > output=es.0
     
      level=info msg="Config file changed, stopping Fluent Bit"
     
      level=info msg="Killed Fluent Bit"
     
      level=info msg="Config file changed, stopped Fluent Bit"
     
      level=info msg="Config file changed, stopping Fluent Bit"
     
      level=info msg="Killed Fluent Bit"
     
      level=info msg="Config file changed, stopped Fluent Bit"
     
      [2022/04/28 02:58:20] [engine] caught signal (SIGTERM)
     
      [2022/04/28 02:58:20] [ info] [input] pausing systemd.0
     
      [2022/04/28 02:58:20] [ info] [input] pausing systemd.1
     
      [2022/04/28 02:58:20] [ info] [input] pausing tail.2
     
      [2022/04/28 02:58:20] [ info] [input] pausing tail.3
     
      [2022/04/28 02:58:20] [ info] [input] pausing tail.4
     
      [2022/04/28 02:58:20] [ warn] [engine] service will stop in 5 seconds
     
      [2022/04/28 02:58:20] [ warn] [engine] failed to flush chunk '39-1651114699.328367642.flb', retry in 6 seconds: task_id=0, input=tail.2 > output=es.0 (out_id=0)
     
      [2022/04/28 02:58:23] [ warn] [engine] chunk '39-1651114689.327129623.flb' cannot be retried: task_id=1, input=tail.2 > output=es.0
     
      [2022/04/28 02:58:25] [ info] [engine] service stopped
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=416649 watch_fd=1
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=461549 watch_fd=2
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134582558 watch_fd=3
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134581715 watch_fd=4
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=268955529 watch_fd=5
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=373423 watch_fd=6
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=373208 watch_fd=7
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403716654 watch_fd=8
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134583208 watch_fd=9
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=997310 watch_fd=10
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=269010417 watch_fd=11
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=984532 watch_fd=12
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403995814 watch_fd=13
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=962171 watch_fd=14
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134583424 watch_fd=15
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134580575 watch_fd=16
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=268955289 watch_fd=17
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403715964 watch_fd=18
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403855647 watch_fd=19
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=413299 watch_fd=20
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=375870 watch_fd=21
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134581094 watch_fd=22
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=463110 watch_fd=23
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=135190584 watch_fd=24
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134738131 watch_fd=25
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=268956011 watch_fd=26
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=373969 watch_fd=27
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403792092 watch_fd=28
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=412431 watch_fd=29
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=134738124 watch_fd=30
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=467559 watch_fd=31
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=403860482 watch_fd=32
     
      [2022/04/28 02:58:25] [ info] [input:tail:tail.2] inotify_fs_remove(): inode=997586 watch_fd=33
     
      level=error msg="Fluent bit exited" error=null
     
      level=info msg=backoff delay=1s
     
      level=info msg="backoff timer done" actual=1.000167488s expected=1s
     
      level=info msg="Fluent bit started"
     
      Fluent Bit v1.8.3
     
      * Copyright (C) 2019-2021 The Fluent Bit Authors
     
      * Copyright (C) 2015-2018 Treasure Data
     
      * Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
     
      * https://fluentbit.io
     
      
     
      [2022/04/28 02:58:26] [ info] [engine] started (pid=42)
     
      [2022/04/28 02:58:26] [ info] [storage] version=1.1.1, initializing...
     
      [2022/04/28 02:58:26] [ info] [storage] in-memory
     
      [2022/04/28 02:58:26] [ info] [storage] normal synchronization mode, checksum disabled, max_chunks_up=128
     
      [2022/04/28 02:58:26] [ info] [cmetrics] version=0.1.6
     
      [2022/04/28 02:58:26] [ info] [input:systemd:systemd.0] seek_cursor=s=5d0022eeaf454ba8a40dde2836de2f4d;i=e3f... OK
     
      [2022/04/28 02:58:26] [ info] [input:systemd:systemd.1] seek_cursor=s=5d0022eeaf454ba8a40dde2836de2f4d;i=e47... OK
     
      [2022/04/28 02:58:26] [ info] [filter:kubernetes:kubernetes.4] https=1 host=kubernetes.default.svc port=443
     
      [2022/04/28 02:58:26] [ info] [filter:kubernetes:kubernetes.4] local POD info OK
     
      [2022/04/28 02:58:26] [ info] [filter:kubernetes:kubernetes.4] testing connectivity with API server...
     
      [2022/04/28 02:58:26] [ info] [filter:kubernetes:kubernetes.4] connectivity OK
     
      [2022/04/28 02:58:26] [ info] [sp] stream processor started
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=416649 watch_fd=1 name=/var/log/containers/alertmanager-main-2_kubesphere-monitoring-system_alertmanager-1a71f1d5b1136320644f3289e4b22544620db4a0d35a1ffec52bc534d729c358.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=461549 watch_fd=2 name=/var/log/containers/alertmanager-main-2_kubesphere-monitoring-system_config-reloader-bc14c53008f1e800d6240a5b912239a3d5682c6dcb719f48c69b7e8ba5892e35.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134582558 watch_fd=3 name=/var/log/containers/calico-kube-controllers-75ddb95444-8xvf4_kube-system_calico-kube-controllers-dc90044aa0ce62e61dfb0842c8f2e5aa8d021738adffca847b003a5c9621512c.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134581715 watch_fd=4 name=/var/log/containers/calico-node-pzvrj_kube-system_calico-node-f2f40fb9878e3328a4783ceb9c01eaed9edf24c85579ed87c7ffc2161779c3e2.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=268955529 watch_fd=5 name=/var/log/containers/calico-node-pzvrj_kube-system_flexvol-driver-abf59d8a7af22c683dfb0776a0835c719f41ef23446e912f82901a4fd9cf166f.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373423 watch_fd=6 name=/var/log/containers/calico-node-pzvrj_kube-system_install-cni-b3d49f2e9c03e63c3863d50365d2ade01dead979c90285c526ac0029b58411cd.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=373208 watch_fd=7 name=/var/log/containers/calico-node-pzvrj_kube-system_upgrade-ipam-6d4425d7ea5302511df1c278f2b113ac8fdfa0a372e07b025e0a9b45d30129ac.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403716654 watch_fd=8 name=/var/log/containers/coredns-5495dd7c88-68k9b_kube-system_coredns-879c087a4c8717d0886e6603a63e9503a406d215cfe77941cbcc1cc7c138d010.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134583208 watch_fd=9 name=/var/log/containers/coredns-5495dd7c88-z9q6j_kube-system_coredns-80cd4427410de02465b415683817a75674a95777ffc0929f93e87506b120ca59.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=997310 watch_fd=10 name=/var/log/containers/ks-apiserver-6554c5ddb4-tzvmf_kubesphere-system_ks-apiserver-9536d87a3fa484f71c1a22a8eb489ce67e5a072a4d3f2d38796220f192a32b4f.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=269010417 watch_fd=11 name=/var/log/containers/ks-console-65f4d44d88-hzw9b_kubesphere-system_ks-console-f600455c3af064c53adc8eb7e5412505c4d0e50362ab11b8e59d2e71b552b0f1.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=984532 watch_fd=12 name=/var/log/containers/ks-controller-manager-b575b54ff-8tx8m_kubesphere-system_ks-controller-manager-d11c9db1fc376410555fc4ac7de2fa46bb2aa963c2bcbd407fab46d017caf788.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=403995814 watch_fd=13 name=/var/log/containers/ks-events-ruler-575669b4-mh2fn_kubesphere-logging-system_config-reloader-2d51bab0babdd47864ec235efdbd69d2075cc8bf72c3642449be69f68aa1f0d5.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=962171 watch_fd=14 name=/var/log/containers/ks-events-ruler-575669b4-mh2fn_kubesphere-logging-system_events-ruler-da24092a12c4fc0d0aab8a02c7fc7cb1e0e82d15a924717222bdbadee7935c84.log
     
      [2022/04/28 02:58:26] [ info] [input:tail:tail.2] inotify_fs_add(): inode=134583424 watch_fd=15 name=/var/log/containers/kube-apiserver-ks-k8s-master-0_kube-system_kube-apiserver-b4e97a525442956fe7612d27367a184ee09d65a56464149ec913ea004db0f1ef.log
     ```
   
   - 再看看es里是否有索引了。
   
   - ```shell
     [root@glusterfs-node-0 ~]# curl -ulstack:'P@88w0rd' 192.168.9.95:9200/_cat/indices?v
     health status index                      uuid                   pri rep docs.count docs.deleted store.size pri.store.size
     green  open   .geoip_databases           VFDw1lkoQYiWeX-u7bMC1Q   1   1         40            0     75.5mb         37.7mb
     green  open   ks-logstash-log-2022.04.28 eu6citkMQ4aHpsHnLhj7IQ   1   1        407            0    394.6kb        164.4kb
     ```
   
   - 注意啊，此时工具箱里的分析工具还是会报错的。按着上面的操作方法，重启一下ks-apiserver就可以解决并看到相应的结果。
   
8. 至此，ElasticSearch采用http协议不开启认证和开启认证两种方式的对接全部实现了，本文其实没有介绍ElasticSearch采用HTTPS协议的方式，后续如果真有需求再补充。

## 5. 技术关键点梳理

### 01 技术关键点（留坑，待补充）

1. fluentbit-operator
   - Input
   - Output
2. ClusterConfiguration的技术细节
3. ks-installer的技术细节
4. ElasticSearch的优化配置 

## 6. 总结

本文详细讲解了ElasticSearch基于http协议启用认证和不启用认证两种方式的集群安装部署过程，同时在KubeSphere中开启了可插拔的日志组件，并演示了与两种模式下的ElasticSearch集群对接以及对接过程中的遇坑、填坑的过程。本文的配置经验可直接用于生产环境。到此为止，我们KubeSphere和Kubernetes的安装配置和初始化已经全部完成，下一章开始，我们将开启利用KubeSphere在Kubernetes上安装配置各种常用服务的实践之旅。


> **参考文档**

- 太多了，大部分都在文中直接链接了
- [fluent-operator官网](https://github.com/fluent/fluent-operator/tree/master/charts/fluent-operator/crds)
- https://ptran32.github.io/2020-08-12-send-k8s-logs-with-fluentbit/
- https://banzaicloud.com/docs/one-eye/logging-operator/operation/troubleshooting/fluentbit/


> **Get文档**

- Github https://github.com/devops/z-notes**
- Gitee https://gitee.com/zdevops/z-notes

> **Get代码**

- Github https://github.com/devops/ansible-zdevops
- Gitee https://gitee.com/zdevops/ansible-zdevops

> **版权声明** 

- 所有内容均属于原创，整理不易，感谢收藏，转载请标明出处。

> About Me

- 昵称：老Z
- 坐标：山东济南
- 职业：运维架构师/高级运维工程师=**运维**
- 关注的领域：云计算/云原生技术运维，自动化运维
- 技能标签：OpenStack、Ansible、K8S、Python、Go、CNCF
