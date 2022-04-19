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

本文接着上篇 **<<基于KubeSphere玩转k8s-KubeSphere初始化手记>>** ，继续玩转KubeSphere，玩转k8s。本文会介绍KubeSphere启用可插拔组件日志系统的安装和配置过程，由于采用了Kubernetes集群外部的ElasticSearch集群作为日志收集系统，因此，还会本文还会涉及ElasticSearch的安装配置的实践。为了让读者不仅能掌握ElasticSearch手工安装部署的技能，同时也能get到如何将手工安装部署文档转化成Ansible的Plaobooks，因此本文同时介绍了纯手工安装配置ElasticSearch和利用Ansible自动化安装配置ElasticSearch。

> **本文知识点**

- 定级：**入门级**
- ElasticSearch-手工安装配置
- ElasticSearch-Ansible自动安装配置
- ElasticSearch-启用https配置
- KubeSphere启用可插拔日志组件
- KubeSphere对接外部的ElasticSearch

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

- **虚拟化上磁盘插槽详情**
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

>  **05-生成 instances文件用来配置证书(在节点1中执行)**

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

>  **06-生成证书(在节点1中执行)**

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

> **07-配置keystore**

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
   - ![kubesphere-clusters-components-logging](/Users/z/data-air/个人知识库/图床/kubesphere-clusters-components-logging.png)

   - 点击右下角的**工具箱**按钮，多出几个分析工具的菜单
   - ![kubesphere-clusters-components-logging-2](/Users/z/data-air/个人知识库/图床/kubesphere-clusters-components-logging-2.png)

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

   - 点击**工具箱**中的**容器日志查询**，会出现如下报错

   - ![kubesphere-logging-error-1](/Users/z/data-air/个人知识库/图床/kubesphere-logging-error-1.png)

   - 说明我们的es对接并没有成功。

   - 执行以下命令查看Out的配置

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

   - 查看pod的log

   - ```yaml
     / # kubectl logs  fluent-bit-j5rvq -n kubesphere-logging-system 
     
     ...
      [2022/04/15 07:25:47] [ warn] [net] getaddrinfo(host='elasticsearch-logging-data.kubesphere-logging-system.svc', err=4): Domain name not found
     
      [2022/04/15 07:25:47] [ warn] [engine] failed to flush chunk '12-1650007543.816834850.flb', retry in 6 seconds: task_id=2, input=tail.2 > output=es.0 (out_id=0)
     ```

   - 

### 问问

## 5. 总结

本文详细讲解了ElasticSearch的安装部署过程，同时在KubeSphere中开启了可插拔的日志组件，并演示了与ElasticSearch的对接。本文的配置经验可直接用于生产环境。到此为止，我们KubeSphere和Kubernetes的安装配置和初始化已经全部完成，下一章开始，我们将开启利用KubeSphere在Kubernetes安装配置各种常用服务的实践之旅。


> **参考文档**




> **Get文档**

- Github https://github.com/devops/z-notes
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
