# 基于KubeSphere玩转k8s-GlusterFS安装手记

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

本文接着上篇 **<<基于KubeSphere玩转k8s-KubeSphere安装手记>>** ，继续玩转KubeSphere，k8s，本期会讲解分布式存储GluterFS的安装部署以及与KubeSphere安装的k8s集群的对接。

> **本文知识点**

- 定级：**入门级**
- 使用Ansible安装部署GlusterFS服务
- 使用Ansible安装部署Heketi服务
- 在k8s上命令行的方式对接GlusterFS
- 在KubeSphere上图形化方式对接GlusterFS

> **演示服务器配置**

| 主机名              | IP           | CPU | 内存  | 系统盘 | 数据盘 | 用途                               |
|:----------------:|:------------:|:---:|:---:|:---:|:---:|:--------------------------------:|
| zdevops-master   | 192.168.9.9  | 2   | 4   | 40  | 200 | Ansible运维控制节点                    |
| ks-k8s-master-0  | 192.168.9.91 | 8   | 32  | 40  | 200 | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-1  | 192.168.9.92 | 8   | 32  | 40  | 200 | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-2  | 192.168.9.93 | 8   | 32  | 40  | 200 | KubeSphere/k8s-master/k8s-worker |
| glusterfs-node-0 | 192.168.9.95 | 4   | 8   | 40  | 200 | GlusterFS                        |
| glusterfs-node-1 | 192.168.9.96 | 4   | 8   | 40  | 200 | GlusterFS                        |
| glusterfs-node-2 | 192.168.9.97 | 4   | 8   | 40  | 200 | GlusterFS                        |

## 2. Ansible配置

> **01-增加hosts配置**

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

[servers:children]
k8s
glusterfs

[servers:vars]
ansible_connection=paramiko
ansible_ssh_user=root
ansible_ssh_pass=password
```

## 3. GlusterFS安装配置

> **01-检测服务器连通性**

```shell
# 利用ansible检测服务器的连通性

(ansible2.8) [root@zdevops-master dev]# ansible glusterfs -m ping
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
glusterfs-node-2 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
glusterfs-node-1 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
glusterfs-node-0 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

> **02-初始化服务器配置**

```shell
# 利用ansible-playbook初始化服务器配置

(ansible2.8) [root@zdevops-master dev]# ansible-playbook ../../playbooks/init-base.yaml -l glusterfs
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature

PLAY [初始化服务器.] ********************************************************************************************************************************************

TASK [01-停止并禁用firewalld服务.] *******************************************************************************************************************************
changed: [glusterfs-node-2]
changed: [glusterfs-node-0]
changed: [glusterfs-node-1]

TASK [02-配置主机名.] ******************************************************************************************************************************************
changed: [glusterfs-node-1]
changed: [glusterfs-node-2]
changed: [glusterfs-node-0]

TASK [03-配置时区.] *******************************************************************************************************************************************
ok: [glusterfs-node-2]
ok: [glusterfs-node-0]
ok: [glusterfs-node-1]

PLAY RECAP ************************************************************************************************************************************************
glusterfs-node-0           : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
glusterfs-node-1           : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
glusterfs-node-2           : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

**重点注意：ansible-playbook的 -l 参数，需要指定为glusterfs，因为init-base.yaml文件默认指定的是所有服务器都执行，不加-l就会把hosts文件里指定的所有服务器都初始化了。**

> **03-安装GlusterFS服务**

```shell
# 利用ansible-playbook安装GlusterFS服务

(ansible2.8) [root@zdevops-master dev]# ansible-playbook ../../playbooks/deploy-glusterfs-server.yaml
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature

PLAY [安装配置GlusterFS服务.] ***********************************************************************************************************************************

TASK [01-安装glusterfs的repo配置.] *****************************************************************************************************************************
changed: [glusterfs-node-2]
changed: [glusterfs-node-1]
changed: [glusterfs-node-0]

TASK [02-安装glusterfs-server.] *****************************************************************************************************************************
changed: [glusterfs-node-0]
changed: [glusterfs-node-1]
changed: [glusterfs-node-2]

TASK [03-启动并设置开机自动启动glusterd服务.] **************************************************************************************************************************
changed: [glusterfs-node-1]
changed: [glusterfs-node-2]
changed: [glusterfs-node-0]

PLAY RECAP ************************************************************************************************************************************************
glusterfs-node-0           : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
glusterfs-node-1           : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
glusterfs-node-2           : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

> **04-验证GlusterFS服务状态**

```shell
# 利用ansible验证GlusterFS服务状态

(ansible2.8) [root@zdevops-master dev]# ansible glusterfs -m shell -a 'systemctl status glusterd'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
glusterfs-node-0 | CHANGED | rc=0 >>
● glusterd.service - GlusterFS, a clustered file-system server
   Loaded: loaded (/usr/lib/systemd/system/glusterd.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2022-04-02 14:09:12 CST; 2min 26s ago
     Docs: man:glusterd(8)
  Process: 9760 ExecStart=/usr/sbin/glusterd -p /var/run/glusterd.pid --log-level $LOG_LEVEL $GLUSTERD_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 9761 (glusterd)
   CGroup: /system.slice/glusterd.service
           └─9761 /usr/sbin/glusterd -p /var/run/glusterd.pid --log-level INFO

Apr 02 14:09:12 localhost.localdomain systemd[1]: Starting GlusterFS, a clustered file-system server...
Apr 02 14:09:12 localhost.localdomain systemd[1]: Started GlusterFS, a clustered file-system server.

glusterfs-node-1 | CHANGED | rc=0 >>
● glusterd.service - GlusterFS, a clustered file-system server
   Loaded: loaded (/usr/lib/systemd/system/glusterd.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2022-04-02 14:09:11 CST; 2min 26s ago
     Docs: man:glusterd(8)
  Process: 9747 ExecStart=/usr/sbin/glusterd -p /var/run/glusterd.pid --log-level $LOG_LEVEL $GLUSTERD_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 9748 (glusterd)
   CGroup: /system.slice/glusterd.service
           └─9748 /usr/sbin/glusterd -p /var/run/glusterd.pid --log-level INFO

Apr 02 14:09:11 localhost.localdomain systemd[1]: Starting GlusterFS, a clustered file-system server...
Apr 02 14:09:11 localhost.localdomain systemd[1]: Started GlusterFS, a clustered file-system server.

glusterfs-node-2 | CHANGED | rc=0 >>
● glusterd.service - GlusterFS, a clustered file-system server
   Loaded: loaded (/usr/lib/systemd/system/glusterd.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2022-04-02 14:09:12 CST; 2min 26s ago
     Docs: man:glusterd(8)
  Process: 9867 ExecStart=/usr/sbin/glusterd -p /var/run/glusterd.pid --log-level $LOG_LEVEL $GLUSTERD_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 9868 (glusterd)
   CGroup: /system.slice/glusterd.service
           └─9868 /usr/sbin/glusterd -p /var/run/glusterd.pid --log-level INFO

Apr 02 14:09:12 localhost.localdomain systemd[1]: Starting GlusterFS, a clustered file-system server...
Apr 02 14:09:12 localhost.localdomain systemd[1]: Started GlusterFS, a clustered file-system server.


# 利用ansible验证GlusterFS服务端口状态
(ansible2.8) [root@zdevops-master dev]# ansible glusterfs -m shell -a 'ss -ntlup | grep glusterd'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
glusterfs-node-0 | CHANGED | rc=0 >>
tcp    LISTEN     0      128       *:24007                 *:*                   users:(("glusterd",pid=9761,fd=10))

glusterfs-node-1 | CHANGED | rc=0 >>
tcp    LISTEN     0      128       *:24007                 *:*                   users:(("glusterd",pid=9748,fd=10))

glusterfs-node-2 | CHANGED | rc=0 >>
tcp    LISTEN     0      128       *:24007                 *:*                   users:(("glusterd",pid=9868,fd=10))
```

> **05-检测GlusterFS集群节点之间的连通性**

```shell
# 利用ansible验证GlusterFS集群节点之间的连通性
# 选择利用节点1 ping测节点2和节点3

(ansible2.8) [root@zdevops-master dev]# ansible glusterfs-node-0 -m shell -a 'ping glusterfs-node-1 -c 4'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
glusterfs-node-0 | CHANGED | rc=0 >>
PING glusterfs-node-1 (192.168.9.96) 56(84) bytes of data.
64 bytes from glusterfs-node-1 (192.168.9.96): icmp_seq=1 ttl=64 time=0.289 ms
64 bytes from glusterfs-node-1 (192.168.9.96): icmp_seq=2 ttl=64 time=0.237 ms
64 bytes from glusterfs-node-1 (192.168.9.96): icmp_seq=3 ttl=64 time=0.236 ms
64 bytes from glusterfs-node-1 (192.168.9.96): icmp_seq=4 ttl=64 time=0.209 ms

--- glusterfs-node-1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3000ms
rtt min/avg/max/mdev = 0.209/0.242/0.289/0.034 ms

(ansible2.8) [root@zdevops-master dev]# ansible glusterfs-node-0 -m shell -a 'ping glusterfs-node-2 -c 4'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
glusterfs-node-0 | CHANGED | rc=0 >>
PING glusterfs-node-2 (192.168.9.97) 56(84) bytes of data.
64 bytes from glusterfs-node-2 (192.168.9.97): icmp_seq=1 ttl=64 time=0.223 ms
64 bytes from glusterfs-node-2 (192.168.9.97): icmp_seq=2 ttl=64 time=0.297 ms
64 bytes from glusterfs-node-2 (192.168.9.97): icmp_seq=3 ttl=64 time=0.269 ms
64 bytes from glusterfs-node-2 (192.168.9.97): icmp_seq=4 ttl=64 time=0.268 ms

--- glusterfs-node-2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3000ms
rtt min/avg/max/mdev = 0.223/0.264/0.297/0.028 ms
```

> **06-配置可信池(Configure the trusted pool)**

```shell
# 利用ansible配置可信池(Configure the trusted pool)

(ansible2.8) [root@zdevops-master dev]# ansible glusterfs-node-0 -m shell -a 'gluster peer probe glusterfs-node-1'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
glusterfs-node-0 | CHANGED | rc=0 >>
peer probe: success

(ansible2.8) [root@zdevops-master dev]# ansible glusterfs-node-0 -m shell -a 'gluster peer probe glusterfs-node-2'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
glusterfs-node-0 | CHANGED | rc=0 >>
peer probe: success

# 利用ansible检查Peer状态
(ansible2.8) [root@zdevops-master dev]# ansible glusterfs-node-0 -m shell -a 'gluster peer status'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
glusterfs-node-0 | CHANGED | rc=0 >>
Number of Peers: 2

Hostname: glusterfs-node-1
Uuid: 1759a580-d17f-4427-935c-50ad53f78c39
State: Peer in Cluster (Connected)

Hostname: glusterfs-node-2
Uuid: cd56bacf-f62b-4310-b193-5c600cf75a6b
State: Peer in Cluster (Connected)
```

## 4. 安装配置Heketi

**(在GlusterFs服务器中任选一个节点,这里选择节点1，glusterfs-node-0，也可以将Heketi独立部署)**

> **01-安装配置heketi服务**

- **01.配置软件源**
- **02.安装heketi和heketi-client**
- **03.生成heketi管理用ssh-key，并配置服务器免密**
- **04.创建heketi配置文件heketi.json**
- **05.启动heketi服务并设置开机自启**
- **06.创建topology.json配置文件**
- **07.利用topoly.json配置文件创建集群**
- **08.配置heketi管理用环境变量**

```shell
# 利用ansible-playbook安装配置heketi服务

(ansible2.8) [root@zdevops-master dev]# ansible-playbook ../../playbooks/deploy-glusterfs-heketi.yaml 
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature

PLAY [生成heketi服务使用的ssh管理密钥.] ****************************************************************************************

TASK [生成ssh密钥] ******************************************************************************************************
[WARNING]: Consider using the file module with state=directory rather than running 'mkdir'.  If you need to use
command because file is insufficient you can add 'warn: false' to this command task or set 'command_warnings=False'
in ansible.cfg to get rid of this message.

changed: [localhost]

PLAY [配置heketi ssh密钥认证] *********************************************************************************************

TASK [传送heketi认证ssh密钥] **********************************************************************************************
changed: [glusterfs-node-1]
changed: [glusterfs-node-0]
changed: [glusterfs-node-2]

PLAY [安装部署heketi.] **************************************************************************************************

TASK [01-安装glusterfs的repo配置.] ***************************************************************************************
ok: [glusterfs-node-0]

TASK [02-安装heketi-server.] ******************************************************************************************
changed: [glusterfs-node-0]

TASK [03-复制ssh-key-pub] *********************************************************************************************
changed: [glusterfs-node-0] => (item=private_key)
changed: [glusterfs-node-0] => (item=private_key.pub)

TASK [04-创建heketi文件-heketi.json.] ***********************************************************************************
changed: [glusterfs-node-0]

TASK [05-启动并设置开机自动启动heketi服务.] **************************************************************************************
changed: [glusterfs-node-0]

TASK [06-创建heketi文件-topology.json.] *********************************************************************************
changed: [glusterfs-node-0]

TASK [07-根据topology.json创建集群] ***************************************************************************************
changed: [glusterfs-node-0]

TASK [08-配置heketi管理用环境变量] *******************************************************************************************
changed: [glusterfs-node-0]

PLAY RECAP **********************************************************************************************************
glusterfs-node-0           : ok=9    changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
glusterfs-node-1           : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
glusterfs-node-2           : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
localhost                  : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

> **02-检测服务状态**

```shell
# ssh到glusterfs-node-0节点检测服务状态
# (ansible2.8) [root@zdevops-master dev]# ssh glusterfs-node-0

[root@glusterfs-node-0 ~]# systemctl status heketi -l
● heketi.service - Heketi Server
   Loaded: loaded (/usr/lib/systemd/system/heketi.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-04-03 10:15:27 CST; 13min ago
 Main PID: 16350 (heketi)
   CGroup: /system.slice/heketi.service
           └─16350 /usr/bin/heketi --config=/etc/heketi/heketi.json

Apr 03 10:27:28 glusterfs-node-0 heketi[16350]: Main PID: 9748 (glusterd)
Apr 03 10:27:28 glusterfs-node-0 heketi[16350]: CGroup: /system.slice/glusterd.service
Apr 03 10:27:28 glusterfs-node-0 heketi[16350]: └─9748 /usr/sbin/glusterd -p /var/run/glusterd.pid --log-level INFO
Apr 03 10:27:28 glusterfs-node-0 heketi[16350]: Apr 02 14:09:11 localhost.localdomain systemd[1]: Starting GlusterFS, a clustered file-system server...
Apr 03 10:27:28 glusterfs-node-0 heketi[16350]: Apr 02 14:09:11 localhost.localdomain systemd[1]: Started GlusterFS, a clustered file-system server.
Apr 03 10:27:28 glusterfs-node-0 heketi[16350]: Apr 02 14:20:11 localhost.localdomain systemd[1]: [/usr/lib/systemd/system/glusterd.service:4] Unknown lvalue 'StartLimitBurst' in section 'Unit'
Apr 03 10:27:28 glusterfs-node-0 heketi[16350]: Apr 02 14:20:11 localhost.localdomain systemd[1]: [/usr/lib/systemd/system/glusterd.service:5] Unknown lvalue 'StartLimitIntervalSec' in section 'Unit'
Apr 03 10:27:28 glusterfs-node-0 heketi[16350]: ]: Stderr []
Apr 03 10:27:28 glusterfs-node-0 heketi[16350]: [heketi] INFO 2022/04/03 10:27:28 Periodic health check status: node da8f9ae974c342b5225265cd414ebe7f up=true
Apr 03 10:27:28 glusterfs-node-0 heketi[16350]: [heketi] INFO 2022/04/03 10:27:28 Cleaned 0 nodes from health cache

# 查看集群列表
[root@glusterfs-node-0 ~]# heketi-cli cluster list
Clusters:
Id:deb78837bdb066bd8adf51b59a7e6c7e [file][block]

# 查看集群详细信息
[root@glusterfs-node-0 ~]# heketi-cli cluster info deb78837bdb066bd8adf51b59a7e6c7e
Cluster id: deb78837bdb066bd8adf51b59a7e6c7e
Nodes:
98a1975ca166134245e273849f8d345a
c139bd449c952355a85a0e7d50754435
da8f9ae974c342b5225265cd414ebe7f
Volumes:

Block: true

File: true

# 查看集群节点列表
[root@glusterfs-node-0 ~]# heketi-cli node list
Id:98a1975ca166134245e273849f8d345a     Cluster:deb78837bdb066bd8adf51b59a7e6c7e
Id:c139bd449c952355a85a0e7d50754435     Cluster:deb78837bdb066bd8adf51b59a7e6c7e
Id:da8f9ae974c342b5225265cd414ebe7f     Cluster:deb78837bdb066bd8adf51b59a7e6c7e
```

> **03-验证测试**

1. **创建卷**
   
   ```shell
   # 创建一个2G大小3副本的卷
   [root@glusterfs-node-0 ~]# heketi-cli volume create --size=2 --replica=3
   Name: vol_4e6a0c0cdd8a4457cee30a2bb50c8dd5
   Size: 2
   Volume Id: 4e6a0c0cdd8a4457cee30a2bb50c8dd5
   Cluster Id: deb78837bdb066bd8adf51b59a7e6c7e
   Mount: 192.168.9.95:vol_4e6a0c0cdd8a4457cee30a2bb50c8dd5
   Mount Options: backup-volfile-servers=192.168.9.97,192.168.9.96
   Block: false
   Free Size: 0
   Reserved Size: 0
   Block Hosting Restriction: (none)
   Block Volumes: []
   Durability Type: replicate
   Distribute Count: 1
   Replica Count: 3
   
   # 查看创建的卷
   [root@glusterfs-node-0 ~]# heketi-cli volume list
   Id:4e6a0c0cdd8a4457cee30a2bb50c8dd5    Cluster:deb78837bdb066bd8adf51b59a7e6c7e    Name:vol_4e6a0c0cdd8a4457cee30a2bb50c8dd5
   
   # 查看服务器挂载点
   [root@glusterfs-node-0 ~]# df -h
   Filesystem                                                                              Size  Used Avail Use% Mounted on
   devtmpfs                                                                                989M     0  989M   0% /dev
   tmpfs                                                                                  1000M     0 1000M   0% /dev/shm
   tmpfs                                                                                  1000M  8.9M  991M   1% /run
   tmpfs                                                                                  1000M     0 1000M   0% /sys/fs/cgroup
   /dev/mapper/centos-root                                                                  37G  1.6G   36G   5% /
   /dev/sda1                                                                              1014M  168M  847M  17% /boot
   tmpfs                                                                                   200M     0  200M   0% /run/user/0
   /dev/mapper/vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4-brick_93655a762cc3c2750aab395d7bb348a9  2.0G   33M  2.0G   2% /var/lib/heketi/mounts/vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4/brick_93655a762cc3c2750aab395d7bb348a9
   ```

2. **挂载测试**
   
   ```shell
   # 查看卷的详细信息
   [root@glusterfs-node-0 ~]# heketi-cli volume info 4e6a0c0cdd8a4457cee30a2bb50c8dd5
   Name: vol_4e6a0c0cdd8a4457cee30a2bb50c8dd5
   Size: 2
   Volume Id: 4e6a0c0cdd8a4457cee30a2bb50c8dd5
   Cluster Id: deb78837bdb066bd8adf51b59a7e6c7e
   Mount: 192.168.9.95:vol_4e6a0c0cdd8a4457cee30a2bb50c8dd5
   Mount Options: backup-volfile-servers=192.168.9.97,192.168.9.96
   Block: false
   Free Size: 0
   Reserved Size: 0
   Block Hosting Restriction: (none)
   Block Volumes: []
   Durability Type: replicate
   Distribute Count: 1
   Replica Count: 3
   
   # 挂载卷，mount命令中的192.168.9.95:vol_4e6a0c0cdd8a4457cee30a2bb50c8dd5为上方heketi-cli volum info 卷id查看卷详细信息时Mount的返回值
   [root@glusterfs-node-0 ~]# mount -t glusterfs 192.168.9.95:vol_4e6a0c0cdd8a4457cee30a2bb50c8dd5 /mnt/
   
   # 查看挂载详情
   [root@glusterfs-node-0 ~]# df -h
   Filesystem                                                                              Size  Used Avail Use% Mounted on
   devtmpfs                                                                                989M     0  989M   0% /dev
   tmpfs                                                                                  1000M     0 1000M   0% /dev/shm
   tmpfs                                                                                  1000M  8.9M  991M   1% /run
   tmpfs                                                                                  1000M     0 1000M   0% /sys/fs/cgroup
   /dev/mapper/centos-root                                                                  37G  1.6G   36G   5% /
   /dev/sda1                                                                              1014M  168M  847M  17% /boot
   tmpfs                                                                                   200M     0  200M   0% /run/user/0
   /dev/mapper/vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4-brick_93655a762cc3c2750aab395d7bb348a9  2.0G   33M  2.0G   2% /var/lib/heketi/mounts/vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4/brick_93655a762cc3c2750aab395d7bb348a9
   192.168.9.95:vol_4e6a0c0cdd8a4457cee30a2bb50c8dd5                                       2.0G   54M  2.0G   3% /mnt
   
   # 写入文件测试
   [root@glusterfs-node-0 ~]# echo "`date` test write" >> /mnt/test.txt
   [root@glusterfs-node-0 ~]# cat /mnt/test.txt 
   Sun Apr  3 10:53:17 CST 2022 test write
   
   # 卸载卷
   [root@glusterfs-node-0 ~]# umount /mnt/
   
   # 查看卷卸载后服务器挂载情况
   [root@glusterfs-node-0 ~]# df -h
   Filesystem                                                                              Size  Used Avail Use% Mounted on
   devtmpfs                                                                                989M     0  989M   0% /dev
   tmpfs                                                                                  1000M     0 1000M   0% /dev/shm
   tmpfs                                                                                  1000M  8.9M  991M   1% /run
   tmpfs                                                                                  1000M     0 1000M   0% /sys/fs/cgroup
   /dev/mapper/centos-root                                                                  37G  1.6G   36G   5% /
   /dev/sda1                                                                              1014M  168M  847M  17% /boot
   tmpfs                                                                                   200M     0  200M   0% /run/user/0
   /dev/mapper/vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4-brick_93655a762cc3c2750aab395d7bb348a9  2.0G   33M  2.0G   2% /var/lib/heketi/mounts/vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4/brick_93655a762cc3c2750aab395d7bb348a9
   ```

3. **底层详细信息查看**
   
   **可以看出底层vg和lv的创建详情，了解底层的分配细节**
   
   ```shell
   ## 节点一 glusterfs-node-0
   
   # 查看节点创建出的pv
   [root@glusterfs-node-0 ~]# pvdisplay 
     --- Physical volume ---
     PV Name               /dev/sdb
     VG Name               vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4
     PV Size               200.00 GiB / not usable 132.00 MiB
     Allocatable           yes 
     PE Size               4.00 MiB
     Total PE              51167
     Free PE               50649
     Allocated PE          518
     PV UUID               6Wtn9q-pE1L-qNdS-wT2N-oiNy-6UEL-1pUa9G
   
   # 查看节点创建出的vg
   [root@glusterfs-node-0 ~]# vgdisplay 
     --- Volume group ---
     VG Name               vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4
     System ID             
     Format                lvm2
     Metadata Areas        1
     Metadata Sequence No  6
     VG Access             read/write
     VG Status             resizable
     MAX LV                0
     Cur LV                2
     Open LV               1
     Max PV                0
     Cur PV                1
     Act PV                1
     VG Size               199.87 GiB
     PE Size               4.00 MiB
     Total PE              51167
     Alloc PE / Size       518 / 2.02 GiB
     Free  PE / Size       50649 / <197.85 GiB
     VG UUID               JDyEl1-Rv2D-27zU-zDIo-DSD6-wq8z-23ypMI
   
   # 查看节点创建出的lv
   [root@glusterfs-node-0 ~]# lvdisplay 
     --- Logical volume ---
     LV Name                tp_93655a762cc3c2750aab395d7bb348a9
     VG Name                vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4
     LV UUID                09V7QE-CkNl-3MqO-hj92-ONTc-1OhH-Kp9gQL
     LV Write Access        read/write (activated read only)
     LV Creation host, time glusterfs-node-0, 2022-04-03 10:35:23 +0800
     LV Pool metadata       tp_93655a762cc3c2750aab395d7bb348a9_tmeta
     LV Pool data           tp_93655a762cc3c2750aab395d7bb348a9_tdata
     LV Status              available
     # open                 2
     LV Size                2.00 GiB
     Allocated pool data    0.70%
     Allocated metadata     10.32%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     8192
     Block device           253:4
   
     --- Logical volume ---
     LV Path                /dev/vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4/brick_93655a762cc3c2750aab395d7bb348a9
     LV Name                brick_93655a762cc3c2750aab395d7bb348a9
     VG Name                vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4
     LV UUID                8PzqIP-QeW7-LHQn-JEnf-v26v-sX02-yVtY80
     LV Write Access        read/write
     LV Creation host, time glusterfs-node-0, 2022-04-03 10:35:24 +0800
     LV Pool name           tp_93655a762cc3c2750aab395d7bb348a9
     LV Status              available
     # open                 1
     LV Size                2.00 GiB
     Mapped size            0.70%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     8192
     Block device           253:6
   
   ## 节点二 glusterfs-node-1
   # 查看节点创建出的pv
   [root@glusterfs-node-1 ~]# pvdisplay 
     --- Physical volume ---
     PV Name               /dev/sdb
     VG Name               vg_dd8b89e3ac92e364871ad1a288e089be
     PV Size               200.00 GiB / not usable 132.00 MiB
     Allocatable           yes 
     PE Size               4.00 MiB
     Total PE              51167
     Free PE               50649
     Allocated PE          518
     PV UUID               Xp02Bf-f3yP-ckuX-lwI3-hrYm-eDU8-4Ecn20
   
   # 查看节点创建出的vg
   [root@glusterfs-node-1 ~]# vgdisplay 
     --- Volume group ---
     VG Name               vg_dd8b89e3ac92e364871ad1a288e089be
     System ID             
     Format                lvm2
     Metadata Areas        1
     Metadata Sequence No  6
     VG Access             read/write
     VG Status             resizable
     MAX LV                0
     Cur LV                2
     Open LV               1
     Max PV                0
     Cur PV                1
     Act PV                1
     VG Size               199.87 GiB
     PE Size               4.00 MiB
     Total PE              51167
     Alloc PE / Size       518 / 2.02 GiB
     Free  PE / Size       50649 / <197.85 GiB
     VG UUID               kKhfyk-is0P-RiAk-N1Vo-C3Dl-7VcE-1sUhCo
   
   # 查看节点创建出的lv
   [root@glusterfs-node-1 ~]# lvdisplay 
     --- Logical volume ---
     LV Name                tp_14db6032ee2ed1e69f394af585d59480
     VG Name                vg_dd8b89e3ac92e364871ad1a288e089be
     LV UUID                DvKgju-KVA9-L4mI-VhSj-Dyx1-bnBW-OHUSYH
     LV Write Access        read/write (activated read only)
     LV Creation host, time glusterfs-node-1, 2022-04-03 10:35:23 +0800
     LV Pool metadata       tp_14db6032ee2ed1e69f394af585d59480_tmeta
     LV Pool data           tp_14db6032ee2ed1e69f394af585d59480_tdata
     LV Status              available
     # open                 2
     LV Size                2.00 GiB
     Allocated pool data    0.70%
     Allocated metadata     10.32%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     8192
     Block device           253:4
   
     --- Logical volume ---
     LV Path                /dev/vg_dd8b89e3ac92e364871ad1a288e089be/brick_b2c873001bcc768acecd63e970683e47
     LV Name                brick_b2c873001bcc768acecd63e970683e47
     VG Name                vg_dd8b89e3ac92e364871ad1a288e089be
     LV UUID                khYq7l-NETa-FR0r-6lXc-I9BH-Yrxe-WjKdvS
     LV Write Access        read/write
     LV Creation host, time glusterfs-node-1, 2022-04-03 10:35:23 +0800
     LV Pool name           tp_14db6032ee2ed1e69f394af585d59480
     LV Status              available
     # open                 1
     LV Size                2.00 GiB
     Mapped size            0.70%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     8192
     Block device           253:6
   
   ## 节点二 glusterfs-node-2
   # 查看节点创建出的pv
   [root@glusterfs-node-2 ~]# pvdisplay
     --- Physical volume ---
     PV Name               /dev/sdb
     VG Name               vg_7fbd09414aeebd8c7a2415172089ee6d
     PV Size               200.00 GiB / not usable 132.00 MiB
     Allocatable           yes 
     PE Size               4.00 MiB
     Total PE              51167
     Free PE               50649
     Allocated PE          518
     PV UUID               IrBlOc-zygh-aDec-qbuh-ESpp-FtYK-99jhIG
   
   # 查看节点创建出的vg
   [root@glusterfs-node-2 ~]# vgdisplay   
     --- Volume group ---
     VG Name               vg_7fbd09414aeebd8c7a2415172089ee6d
     System ID             
     Format                lvm2
     Metadata Areas        1
     Metadata Sequence No  6
     VG Access             read/write
     VG Status             resizable
     MAX LV                0
     Cur LV                2
     Open LV               1
     Max PV                0
     Cur PV                1
     Act PV                1
     VG Size               199.87 GiB
     PE Size               4.00 MiB
     Total PE              51167
     Alloc PE / Size       518 / 2.02 GiB
     Free  PE / Size       50649 / <197.85 GiB
     VG UUID               FuCeOh-IqNy-dkbf-sito-UADP-24TV-gPCUTp
   
   # 查看节点创建出的lv
   [root@glusterfs-node-2 ~]# lvdisplay   
     --- Logical volume ---
     LV Name                tp_f8b1b01bcd77125b7d17f2d78b744692
     VG Name                vg_7fbd09414aeebd8c7a2415172089ee6d
     LV UUID                aQjAyh-MHF3-s2Sp-vnZD-1A0f-u3BC-4vv82X
     LV Write Access        read/write (activated read only)
     LV Creation host, time glusterfs-node-2, 2022-04-03 10:35:23 +0800
     LV Pool metadata       tp_f8b1b01bcd77125b7d17f2d78b744692_tmeta
     LV Pool data           tp_f8b1b01bcd77125b7d17f2d78b744692_tdata
     LV Status              available
     # open                 2
     LV Size                2.00 GiB
     Allocated pool data    0.70%
     Allocated metadata     10.32%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     8192
     Block device           253:4
   
     --- Logical volume ---
     LV Path                /dev/vg_7fbd09414aeebd8c7a2415172089ee6d/brick_f8b1b01bcd77125b7d17f2d78b744692
     LV Name                brick_f8b1b01bcd77125b7d17f2d78b744692
     VG Name                vg_7fbd09414aeebd8c7a2415172089ee6d
     LV UUID                1FH1wP-Ktt6-JpJK-K3fD-b5uP-Lb5o-fk8odF
     LV Write Access        read/write
     LV Creation host, time glusterfs-node-2, 2022-04-03 10:35:24 +0800
     LV Pool name           tp_f8b1b01bcd77125b7d17f2d78b744692
     LV Status              available
     # open                 1
     LV Size                2.00 GiB
     Mapped size            0.70%
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     8192
     Block device           253:6
   ```

4. **删除测试卷**
   
   ```shell
   [root@glusterfs-node-0 ~]# heketi-cli volume delete 4e6a0c0cdd8a4457cee30a2bb50c8dd5
   Volume 4e6a0c0cdd8a4457cee30a2bb50c8dd5 deleted
   
   [root@glusterfs-node-0 ~]# df -h
   Filesystem               Size  Used Avail Use% Mounted on
   devtmpfs                 989M     0  989M   0% /dev
   tmpfs                   1000M     0 1000M   0% /dev/shm
   tmpfs                   1000M  8.8M  991M   1% /run
   tmpfs                   1000M     0 1000M   0% /sys/fs/cgroup
   /dev/mapper/centos-root   37G  1.6G   36G   5% /
   /dev/sda1               1014M  168M  847M  17% /boot
   tmpfs                    200M     0  200M   0% /run/user/0
   ```

## 5. k8s集群对接GlusterFs(命令行和KubeSphere图形化)

### 1. k8s命令行手动配置

**所有操作都在k8s的master节点上执行,操作根目录为/root/zdevops**

> **01-所有的k8s节点均安装glusterfs客户端**

**此步骤可以忽略，ansible初始化时已安装**

```shell
[root@ks-k8s-master-0 ~]#  yum install glusterfs-fuse -y
```

> **02-创建heketi使用的Secret的认证密码**

```shell
# 创建glusterfs目录,并切换到该目录
[root@ks-k8s-master-0 zdevops]# mkdir /root/zdevops/glusterfs
[root@ks-k8s-master-0 zdevops]# cd /root/zdevops/glusterfs/
[root@ks-k8s-master-0 glusterfs]# pwd
/root/zdevops/glusterfs

# 使用base64将密码转码生成Secret key使用的值,这里的密码为heketi配置文件中创建的用户密码
[root@ks-k8s-master-0 zdevops]# echo -n "admin@P@ssW0rd" | base64
YWRtaW5AUEBzc1cwcmQ=

# 创建secret定义文件
[root@ks-k8s-master-0 zdevops]# vi heketi-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
  namespace: kube-system
data:
  key: YWRtaW5AUEBzc1cwcmQ=
type: kubernetes.io/glusterfs

# 应用配置并查看
[root@ks-k8s-master-0 glusterfs]# kubectl apply -f heketi-secret.yaml 
secret/heketi-secret created
[root@ks-k8s-master-0 glusterfs]# kubectl get secrets heketi-secret -n kube-system 
NAME            TYPE                      DATA   AGE
heketi-secret   kubernetes.io/glusterfs   1      16s
```

> **03-创建StorageClass**

```shell
# 创建StorageClass定义文件
[root@ks-k8s-master-0 zdevops]# vi heketi-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs
  namespace: kube-system
parameters:
  resturl: "http://192.168.9.95:48080"
  clusterid: "deb78837bdb066bd8adf51b59a7e6c7e"
  restauthenabled: "true" 
  restuser: "admin"
  secretName: "heketi-secret"
  secretNamespace: "kube-system"
  volumetype: "replicate:3" 
provisioner: kubernetes.io/glusterfs
reclaimPolicy: Delete

# 应用配置
[root@ks-k8s-master-0 glusterfs]# kubectl apply -f heketi-storageclass.yaml 
storageclass.storage.k8s.io/glusterfs created

# 查看
[root@ks-k8s-master-0 glusterfs]# kubectl get sc
NAME              PROVISIONER               RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
glusterfs         kubernetes.io/glusterfs   Delete          Immediate              false                  12s
local (default)   openebs.io/local          Delete          WaitForFirstConsumer   false                  142m
```

- **parameters.resturl:** heketi服务的地址
- **parameters.clusterid:** 在heketi节点使用heketi-cli cluster list命令返回的集群id
- **parameters.restuser:**  heketi.json配置文件中创建的用户名，默认admin
- **parameters.secretName:** k8s中Secret资源定义中的metadata.name
- **parameters.secretNamespace:** k8s中Secret资源定义中的metadata.namespace
- **parameters.volumetype:** 创建的卷类型和副本数，这里是3副本复制卷

> **04-创建pvc测试**

```shell
# 创建pvc定义文件
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

# 应用配置
[root@ks-k8s-master-0 glusterfs]# kubectl apply -f heketi-pvc.yaml 
persistentvolumeclaim/heketi-pvc created

# 查看
[root@ks-k8s-master-0 glusterfs]# kubectl get pvc 
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
heketi-pvc   Bound    pvc-019107e4-8cb1-44a7-940e-2ab2aecb1eac   1Gi        RWO            glusterfs      10s
```

**注意创建的pvc的状态，如果是Pending，说明连接存储有问题**

- ```yaml
  [root@ks-k8s-master-0 glusterfs]# kubectl get pvc
  NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  heketi-pvc   Pending                                      glusterfs      6m38s
  ```

> **05-创建测试Pod挂载pvc**

```shell
# 创建pod定义文件
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

# 应用
[root@ks-k8s-master-0 glusterfs]# kubectl apply -f heketi-pod.yaml 
pod/heketi-pod created

# 查看pod状态，Running表示创建成功
[root@ks-k8s-master-0 glusterfs]# kubectl get pods -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP             NODE              NOMINATED NODE   READINESS GATES
heketi-pod   1/1     Running   0          22s   10.233.87.15   ks-k8s-master-1   <none>           <none>

# 查看pod中磁盘挂载情况
[root@ks-k8s-master-0 glusterfs]# kubectl exec heketi-pod -- df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                 199.9G      1.9G    198.0G   1% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                    15.7G         0     15.7G   0% /sys/fs/cgroup
192.168.9.97:vol_53ca44e0e2f84f723d7c814812648d5e
                       1014.0M     42.8M    971.2M   4% /pv-data
/dev/mapper/centos-root
                         37.0G      2.6G     34.4G   7% /dev/termination-log
/dev/sdb1               199.9G      1.9G    198.0G   1% /etc/resolv.conf
/dev/sdb1               199.9G      1.9G    198.0G   1% /etc/hostname
/dev/mapper/centos-root
                         37.0G      2.6G     34.4G   7% /etc/hosts
shm                      64.0M         0     64.0M   0% /dev/shm
tmpfs                    15.7G     12.0K     15.7G   0% /var/run/secrets/kubernetes.io/serviceaccount
tmpfs                    15.7G         0     15.7G   0% /proc/acpi
tmpfs                    64.0M         0     64.0M   0% /proc/kcore
tmpfs                    64.0M         0     64.0M   0% /proc/keys
tmpfs                    64.0M         0     64.0M   0% /proc/timer_list
tmpfs                    64.0M         0     64.0M   0% /proc/timer_stats
tmpfs                    64.0M         0     64.0M   0% /proc/sched_debug
tmpfs                    15.7G         0     15.7G   0% /proc/scsi
tmpfs                    15.7G         0     15.7G   0% /sys/firmware 
```

> **06-存储服务器测查看**

```shell
# 查看卷的信息
[root@glusterfs-node-0 heketi]# heketi-cli volume list
Id:53ca44e0e2f84f723d7c814812648d5e    Cluster:deb78837bdb066bd8adf51b59a7e6c7e    Name:vol_53ca44e0e2f84f723d7c814812648d5e

[root@glusterfs-node-0 heketi]# heketi-cli volume info 53ca44e0e2f84f723d7c814812648d5e
Name: vol_53ca44e0e2f84f723d7c814812648d5e
Size: 1
Volume Id: 53ca44e0e2f84f723d7c814812648d5e
Cluster Id: deb78837bdb066bd8adf51b59a7e6c7e
Mount: 192.168.9.95:vol_53ca44e0e2f84f723d7c814812648d5e
Mount Options: backup-volfile-servers=192.168.9.97,192.168.9.96
Block: false
Free Size: 0
Reserved Size: 0
Block Hosting Restriction: (none)
Block Volumes: []
Durability Type: replicate
Distribute Count: 1
Replica Count: 3
Snapshot Factor: 1.00

# 查看lvdisplay
[root@glusterfs-node-0 heketi]# lvdisplay 
  --- Logical volume ---
  LV Name                tp_7c2c26b78219f2fecaf97c728d25bd10
  VG Name                vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4
  LV UUID                vHkLw6-mtUl-IbgH-CRXp-WTh4-cG1A-1RbqQa
  LV Write Access        read/write (activated read only)
  LV Creation host, time glusterfs-node-0, 2022-04-03 15:59:55 +0800
  LV Pool metadata       tp_7c2c26b78219f2fecaf97c728d25bd10_tmeta
  LV Pool data           tp_7c2c26b78219f2fecaf97c728d25bd10_tdata
  LV Status              available
  # open                 2
  LV Size                1.00 GiB
  Allocated pool data    1.39%
  Allocated metadata     10.45%
  Current LE             256
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:4

  --- Logical volume ---
  LV Path                /dev/vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4/brick_f16c9f2207b3cd3db901f4a801823079
  LV Name                brick_f16c9f2207b3cd3db901f4a801823079
  VG Name                vg_f07e0b11c6c8e4da5bbb24b0b11d7fd4
  LV UUID                0Qurda-DDdZ-3SCo-7EKz-xSHc-mhgN-DDJyVr
  LV Write Access        read/write
  LV Creation host, time glusterfs-node-0, 2022-04-03 15:59:56 +0800
  LV Pool name           tp_7c2c26b78219f2fecaf97c728d25bd10
  LV Status              available
  # open                 1
  LV Size                1.00 GiB
  Mapped size            1.39%
  Current LE             256
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:6
```

> **07-清理测试资源**

```shell
# 清理测试的pod、pvc
[root@ks-k8s-master-0 glusterfs]# kubectl delete -f heketi-pod.yaml 
pod "heketi-pod" deleted
[root@ks-k8s-master-0 glusterfs]# kubectl delete -f heketi-pvc.yaml 
persistentvolumeclaim "heketi-pvc" deleted

# 注意，由于本文还需要演示图形化配置，因此将storageclass、secret也清除了，实际使用时不需要。
[root@ks-k8s-master-0 glusterfs]# kubectl delete -f heketi-storageclass.yaml 
storageclass.storage.k8s.io "glusterfs" deleted
[root@ks-k8s-master-0 glusterfs]# kubectl delete -f heketi-secret.yaml 
secret "heketi-secret" deleted
```

### 2. Kubesphere图形化配置

> **01-创建密钥**

- 平台管理->集群管理->配置->保密字典

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-plat.png" alt="kube-plat" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-cluster.png" alt="kube-cluster" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-conf.png" alt="kube-conf" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-secrets.png" alt="kube-secrets" style="zoom:50%;" />

- 创建
  
  - 名称：heketi-secret
  
  - 项目：kube-system
  
  <img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-secrets-0.png" alt="kube-glusterfs-secrets-0" style="zoom:50%;" />

- 数据设置，类型选择Opaque-> 添加数据->创建
  
  - 键：key
  - 值：YWRtaW5AUEBzc1cwcmQ=
  
  <img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-secrets-1.png" alt="kube-glusterfs-secrets-1" style="zoom:50%;" />
  
  <img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-secrets-2.png" alt="kube-glusterfs-secrets-2" style="zoom:50%;" />
  
  <img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-secrets-3.png" alt="kube-glusterfs-secrets-3" style="zoom:50%;" />

> **02-创建存储类型**

- 平台管理-> 存储->存储类型->创建
  
  <img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-storageclasses-0.png" alt="kube-glusterfs-storageclasses-0" style="zoom:50%;" />
  
  <img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-storageclasses-1.png" alt="kube-glusterfs-storageclasses-1" style="zoom:50%;" />

- 创建存储类型
  
  - 存储名称：glusterfs
  - 存储类型：glusterfs
  - 创建存储类型的详细信息
    - 存储卷扩容：是
    - 回收机制：Delete
    - 访问模式：ReadWriteOnce、ReadOnlyMany、ReadWriteMany
    - 存储系统：kubernetes.io/glusterfs
    - 存储卷绑定模式：立即绑定
    - REST URL：192.168.9.95:48080
    - 集群ID：deb78837bdb066bd8adf51b59a7e6c7e（heketi-cli cluster list命令返回的集群id ）
    - 开启REST认证：是
    - REST用户：admin (heketi服务的admin用户)
    - 密钥所属项目：kube-system
    - 密钥名称：heketi-secret （上面创建的secert名称）
    - GID最小值：留空使用默认值
    - GID最大值：留空使用默认值
    - 存储卷类型：replicate:3

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-storageclasses-2.png" alt="kube-glusterfs-storageclasses-2" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-storageclasses-3.png" alt="kube-glusterfs-storageclasses-3" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-storageclasses-4.png" alt="kube-glusterfs-storageclasses-4" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-storageclasses-5.png" alt="kube-glusterfs-storageclasses-5" style="zoom:50%;" />

- 查看存储类型是否创建成功

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-storageclasses-6.png" alt="kube-glusterfs-storageclasses-6" style="zoom:50%;" />

> **03-创建存储卷进行测试**

- 平台管理->存储->存储卷
  - 名称：glusterfs-test （可自定义）
  - 项目：default （可自定义）
  - 存储类型：glusterfs

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-pvc-0.png" alt="kube-glusterfs-pvc-0" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-pvc-1.png" alt="kube-glusterfs-pvc-1" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-pvc-2.png" alt="kube-glusterfs-pvc-2" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-pvc-3.png" alt="kube-glusterfs-pvc-3" style="zoom:50%;" />

> **04-验证测试卷是否创建成功状态为**

- **准备就绪**即为成功

<img src="https://gitee.com/zdevops/res/raw/main/cloudnative/glusterfs/kubesphere-glusterfs-9.jpg" style="zoom:80%;" />

- **但是写文档的时候创建失败了**

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-pvc-4.png" alt="kube-glusterfs-pvc-4" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-pvc-5.png" alt="kube-glusterfs-pvc-5" style="zoom:50%;" />

- 尝试解决
  
  - 查看存储卷配置，发现都是大写，怀疑是否是参数名称的问题，尝试按手工配置的参数修改，结果修改报错

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-pvc-6.png" alt="kube-glusterfs-pvc-6" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-pvc-7.png" alt="kube-glusterfs-pvc-7" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-pvc-8.png" alt="kube-glusterfs-pvc-8" style="zoom:50%;" />

<img src="https://gitee.com/zdevops/res/raw/main/z-notes/kube-glusterfs/kube-glusterfs-pvc-9.png" alt="kube-glusterfs-pvc-9" style="zoom:50%;" />

## 6. 常见问题

> **01-创建集群时报错解决办法**

**如果数据盘不是新盘曾经被使用过，创建集群时会报错，可按下面的方法处理**

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

> **02-创建pvc时，状态为pending**

- 现象
  
  ```shell
  [root@ks-k8s-master-0 glusterfs]# kubectl get pvc -o wide
  NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE     VOLUMEMODE
  heketi-pvc   Pending                                      glusterfs      6m10s   Filesystem
  [root@ks-k8s-master-0 glusterfs]# kubectl describe pvc heketi-pvc 
  Name:          heketi-pvc
  Namespace:     default
  StorageClass:  glusterfs
  Status:        Pending
  Volume:        
  Labels:        <none>
  Annotations:   volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/glusterfs
  Finalizers:    [kubernetes.io/pvc-protection]
  Capacity:      
  Access Modes:  
  VolumeMode:    Filesystem
  Used By:       heketi-pod
  Events:
    Type     Reason              Age                   From                         Message
    ----     ------              ----                  ----                         -------
    Warning  ProvisioningFailed  22s (x10 over 6m18s)  persistentvolume-controller  Failed to provision volume with StorageClass "glusterfs": failed to create volume: failed to create volume: see kube-controller-manager.log for details
  ```

```
- 解决方案

```shell
# 1.heketi服务器地址错误，可以使用curl http://192.168.9.95:48080/hello测试,如果不同检查配置文件heketi-storageclass.yaml，检查网络

[root@ks-k8s-master-0 ~]# curl http://192.168.9.95:48080/hello
Hello from Heketi

# 2.时间不同步,会有如下log，同步k8s和GlusterFS服务器的时间
Apr  3 15:18:21 glusterfs-node-0 heketi: [jwt] ERROR 2022/04/03 15:18:21 heketi/middleware/jwt.go:73:middleware.(*HeketiJwtClaims).Valid: iat validation failed: Token used before issued, time now: 2022-04-03 15:18:21 +0800 CST, time issued: 2022-04-03 15:35:02 +0800 CST
```

## 7. 总结

以上内容详细记录了GlusterFS安装部署过程以及KubeSphere对接GlusterFS的全过程，部署过程中k8s命令行对接GlusterFS成功，但是在KubeSphere图形化对接的过程中出现异常，后续解决后，我再更新文档。

> **01-遗留问题**

图形化对接GlusterFS失败，问题有待解决。

> **参考文档**

1. [GlusterFS](https://docs.gluster.org/en/latest/Quick-Start-Guide/Quickstart/)

2. https://github.com/heketi/heketi

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