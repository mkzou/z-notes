# 基于KubeSphere玩转k8s-KubeSphere安装手记

大家好，我是老Z！

基于KubeSphere玩转k8s系列文档是我在云原生技术领域的学习和运维实践手记，内容涵盖但不限于以下技术领域：

- **KubeSphere**
- **Kubernetes**
- **Ansible**
- **自动化运维**
- **CNCF技术栈**

> **本期知识点**

- 定级：**入门级**

- 使用Ansible进行k8s服务器初始化配置

- 使用KubeKey部署KubeSphere和Kubernetes

  

> **演示服务器配置**

|      主机名      |      IP      | CPU  | 内存 | 系统盘 | 数据盘 |               用途               |
| :--------------: | :----------: | :--: | :--: | :----: | :----: | :------------------------------: |
| ks-k8s-master-0  | 192.168.9.91 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-1  | 192.168.9.92 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-2  | 192.168.9.93 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| glusterfs-node-0 | 192.168.9.95 |  4   |  8   |   40   |  200   |            GlusterFS             |
| glusterfs-node-1 | 192.168.9.96 |  4   |  8   |   40   |  200   |            GlusterFS             |
| glusterfs-node-2 | 192.168.9.97 |  4   |  8   |   40   |  200   |            GlusterFS             |



## Ansible配置



> **01-切换到ansible代码目录**

  ```shell
[root@zdevops-master dev]# cd /data/ansible/ansible-zdevops/inventories/dev
[root@zdevops-master dev]# source /opt/ansible2.8/bin/activate
  ```



> **02-初始hosts配置**

```yaml
[k8s]
ks-k8s-master-0 ansible_ssh_host=192.168.9.91  host_name=ks-k8s-master-0
ks-k8s-master-1 ansible_ssh_host=192.168.9.92  host_name=ks-k8s-master-1
ks-k8s-master-2 ansible_ssh_host=192.168.9.93  host_name=ks-k8s-master-2

[servers:children]
k8s

[servers:vars]
ansible_connection=paramiko
ansible_ssh_user=root
ansible_ssh_pass=password
```



## 服务器初始化

> **01-检测服务器连通性**

```shell
# 利用ansible检测服务器的连通性

(ansible2.8) [root@zdevops-master dev]# ansible servers -m ping
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
ks-k8s-master-2 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
ks-k8s-master-0 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
ks-k8s-master-1 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```



> **02-初始化服务器配置**

```shell
# 利用ansible-playbook初始化服务器配置
(ansible2.8) [root@zdevops-master dev]# ansible-playbook ../../playbooks/init-base.yaml
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature

PLAY [初始化服务器配置.] ********************************************************************************************************************************

TASK [01-停止并禁用firewalld服务.] *******************************************************************************************************************
changed: [ks-k8s-master-0]
changed: [ks-k8s-master-1]
changed: [ks-k8s-master-2]

TASK [02-配置主机名.] ******************************************************************************************************************************
changed: [ks-k8s-master-0]
changed: [ks-k8s-master-1]
changed: [ks-k8s-master-2]

TASK [03-配置时区.] *******************************************************************************************************************************
ok: [ks-k8s-master-0]
ok: [ks-k8s-master-1]
ok: [ks-k8s-master-2]

TASK [04-配置/etc/hosts.] ***********************************************************************************************************************
changed: [ks-k8s-master-0]
changed: [ks-k8s-master-1]
changed: [ks-k8s-master-2]

TASK [05-升级操作系统.] *****************************************************************************************************************************
skipping: [ks-k8s-master-0]
skipping: [ks-k8s-master-1]
skipping: [ks-k8s-master-2]

TASK [05-升级操作系统后如果需要重启，则重启服务器.] ***************************************************************************************************************
skipping: [ks-k8s-master-0]
skipping: [ks-k8s-master-1]
skipping: [ks-k8s-master-2]

TASK [05-等待服务器完成重启.] **************************************************************************************************************************
skipping: [ks-k8s-master-0]
skipping: [ks-k8s-master-1]
skipping: [ks-k8s-master-2]

PLAY RECAP ************************************************************************************************************************************
ks-k8s-master-0            : ok=4    changed=3    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
ks-k8s-master-1            : ok=4    changed=3    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
ks-k8s-master-2            : ok=4    changed=3    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   


# 利用ansible同步服务器时间
(ansible2.8) [root@zdevops-master dev]# ansible servers -a 'yum install ntpdate -y'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
[WARNING]: Consider using the yum module rather than running 'yum'.  If you need to use command because yum is
insufficient you can add 'warn: false' to this command task or set 'command_warnings=False' in ansible.cfg to get
rid of this message.

ks-k8s-master-0 | CHANGED | rc=0 >>
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.cn99.com
 * centos-gluster9: mirrors.cn99.com
 * extras: mirrors.ustc.edu.cn
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package ntpdate.x86_64 0:4.2.6p5-29.el7.centos.2 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package        Arch          Version                         Repository   Size
================================================================================
Installing:
 ntpdate        x86_64        4.2.6p5-29.el7.centos.2         base         87 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 87 k
Installed size: 121 k
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : ntpdate-4.2.6p5-29.el7.centos.2.x86_64                       1/1 
  Verifying  : ntpdate-4.2.6p5-29.el7.centos.2.x86_64                       1/1 

Installed:
  ntpdate.x86_64 0:4.2.6p5-29.el7.centos.2                                      

Complete!

ks-k8s-master-1 | CHANGED | rc=0 >>
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.ustc.edu.cn
 * centos-gluster9: mirrors.ustc.edu.cn
 * extras: mirrors.ustc.edu.cn
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package ntpdate.x86_64 0:4.2.6p5-29.el7.centos.2 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package        Arch          Version                         Repository   Size
================================================================================
Installing:
 ntpdate        x86_64        4.2.6p5-29.el7.centos.2         base         87 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 87 k
Installed size: 121 k
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : ntpdate-4.2.6p5-29.el7.centos.2.x86_64                       1/1 
  Verifying  : ntpdate-4.2.6p5-29.el7.centos.2.x86_64                       1/1 

Installed:
  ntpdate.x86_64 0:4.2.6p5-29.el7.centos.2                                      

Complete!

ks-k8s-master-2 | CHANGED | rc=0 >>
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.ustc.edu.cn
 * centos-gluster9: mirrors.ustc.edu.cn
 * extras: mirrors.ustc.edu.cn
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package ntpdate.x86_64 0:4.2.6p5-29.el7.centos.2 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package        Arch          Version                         Repository   Size
================================================================================
Installing:
 ntpdate        x86_64        4.2.6p5-29.el7.centos.2         base         87 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 87 k
Installed size: 121 k
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : ntpdate-4.2.6p5-29.el7.centos.2.x86_64                       1/1 
  Verifying  : ntpdate-4.2.6p5-29.el7.centos.2.x86_64                       1/1 

Installed:
  ntpdate.x86_64 0:4.2.6p5-29.el7.centos.2                                      

Complete!

# 利用ansible同步服务器时间
(ansible2.8) [root@zdevops-master dev]# ansible servers -a 'ntpdate ntp.api.bz'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
ks-k8s-master-2 | CHANGED | rc=0 >>
 3 Apr 15:45:03 ntpdate[16375]: step time server 114.118.7.161 offset 54.841204 sec

ks-k8s-master-0 | CHANGED | rc=0 >>
 3 Apr 15:45:04 ntpdate[9559]: step time server 114.118.7.161 offset 53.892187 sec

ks-k8s-master-1 | CHANGED | rc=0 >>
 3 Apr 15:45:05 ntpdate[44869]: step time server 114.118.7.161 offset 54.899937 sec

(ansible2.8) [root@zdevops-master dev]# ansible servers -a 'date'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
ks-k8s-master-1 | CHANGED | rc=0 >>
Sun Apr  3 15:45:19 CST 2022

ks-k8s-master-2 | CHANGED | rc=0 >>
Sun Apr  3 15:45:19 CST 2022

ks-k8s-master-0 | CHANGED | rc=0 >>
Sun Apr  3 15:45:21 CST 2022

```



> **03-挂载数据盘**

**因为Kubekey安装的容器运行时，数据目录默认会在/var/lib/docker,且目前无法自定义更改，因此提前规划将该目录挂载独立的数据盘**

```shell
# 利用ansible-playbook初始化主机数据盘

(ansible2.8) [root@zdevops-master dev]# ansible-playbook ../../playbooks/init-disk.yaml
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature

PLAY [初始化磁盘.] *********************************************************************************************************************************

TASK [01-数据磁盘分区.] *****************************************************************************************************************************
changed: [ks-k8s-master-1]
changed: [ks-k8s-master-2]
changed: [ks-k8s-master-0]

TASK [02-格式化数据磁盘.] ****************************************************************************************************************************
changed: [ks-k8s-master-0]
changed: [ks-k8s-master-2]
changed: [ks-k8s-master-1]

TASK [03-挂载数据盘.] ******************************************************************************************************************************
changed: [ks-k8s-master-0]
changed: [ks-k8s-master-1]
changed: [ks-k8s-master-2]

PLAY RECAP ************************************************************************************************************************************
ks-k8s-master-0            : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ks-k8s-master-1            : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ks-k8s-master-2            : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```



> **04-验证数据盘的挂载**

```shell
# 利用ansible验证数据盘是否格式化并挂载

(ansible2.8) [root@zdevops-master dev]# ansible servers -m shell -a 'df -h'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
ks-k8s-master-0 | CHANGED | rc=0 >>
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                  16G     0   16G   0% /dev
tmpfs                     16G     0   16G   0% /dev/shm
tmpfs                     16G  8.8M   16G   1% /run
tmpfs                     16G     0   16G   0% /sys/fs/cgroup
/dev/mapper/centos-root   37G  1.4G   36G   4% /
/dev/sda1               1014M  168M  847M  17% /boot
/dev/sdb1                200G   33M  200G   1% /var/lib/docker
tmpfs                    3.2G     0  3.2G   0% /run/user/0

ks-k8s-master-2 | CHANGED | rc=0 >>
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                  16G     0   16G   0% /dev
tmpfs                     16G     0   16G   0% /dev/shm
tmpfs                     16G  8.8M   16G   1% /run
tmpfs                     16G     0   16G   0% /sys/fs/cgroup
/dev/mapper/centos-root   37G  1.4G   36G   4% /
/dev/sda1               1014M  168M  847M  17% /boot
/dev/sdb1                200G   33M  200G   1% /var/lib/docker
tmpfs                    3.2G     0  3.2G   0% /run/user/0

ks-k8s-master-1 | CHANGED | rc=0 >>
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                  16G     0   16G   0% /dev
tmpfs                     16G     0   16G   0% /dev/shm
tmpfs                     16G  8.8M   16G   1% /run
tmpfs                     16G     0   16G   0% /sys/fs/cgroup
/dev/mapper/centos-root   37G  1.4G   36G   4% /
/dev/sda1               1014M  168M  847M  17% /boot
/dev/sdb1                200G   33M  200G   1% /var/lib/docker
tmpfs                    3.2G     0  3.2G   0% /run/user/0


# 利用ansible验证数据盘是否配置自动挂载

(ansible2.8) [root@zdevops-master dev]# ansible servers -m shell -a 'tail -1  /etc/fstab'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
ks-k8s-master-1 | CHANGED | rc=0 >>
/dev/sdb1 /var/lib/docker xfs defaults 0 0

ks-k8s-master-2 | CHANGED | rc=0 >>
/dev/sdb1 /var/lib/docker xfs defaults 0 0

ks-k8s-master-0 | CHANGED | rc=0 >>
/dev/sdb1 /var/lib/docker xfs defaults 0 0

```



## Kubernetes基础依赖包安装

> **01-安装基础依赖包**

```shell
# 利用ansible-playbook安装kubernetes基础依赖包
# ansible-playbook中设置了启用GlusterFS存储的开关，默认开启,不需要的可以将参数设置为False

(ansible2.8) [root@zdevops-master dev]# ansible-playbook ../../playbooks/deploy-kubesphere.yaml
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature

PLAY [安装k8s基础环境.] *****************************************************************************************************************************

TASK [01-安装依赖包.] ******************************************************************************************************************************
changed: [ks-k8s-master-1]
changed: [ks-k8s-master-2]
changed: [ks-k8s-master-0]

TASK [02-安装GlusterFS软件仓库.] ********************************************************************************************************************
changed: [ks-k8s-master-0]
changed: [ks-k8s-master-1]
changed: [ks-k8s-master-2]

TASK [02-安装GlusterFS客户端.] *********************************************************************************************************************
changed: [ks-k8s-master-0]
changed: [ks-k8s-master-1]
changed: [ks-k8s-master-2]

PLAY RECAP ************************************************************************************************************************************
ks-k8s-master-0            : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ks-k8s-master-1            : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ks-k8s-master-2            : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

# 利用ansible验证GlusterFS软件是否安装
(ansible2.8) [root@zdevops-master dev]# ansible k8s -m shell -a 'rpm -qa | grep glusterfs'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
[WARNING]: Consider using the yum, dnf or zypper module rather than running 'rpm'.  If you need to use command because yum, dnf or zypper is
insufficient you can add 'warn: false' to this command task or set 'command_warnings=False' in ansible.cfg to get rid of this message.

ks-k8s-master-0 | CHANGED | rc=0 >>
glusterfs-9.5-1.el7.x86_64
glusterfs-client-xlators-9.5-1.el7.x86_64
glusterfs-fuse-9.5-1.el7.x86_64
libglusterfs0-9.5-1.el7.x86_64

ks-k8s-master-1 | CHANGED | rc=0 >>
glusterfs-9.5-1.el7.x86_64
glusterfs-client-xlators-9.5-1.el7.x86_64
glusterfs-fuse-9.5-1.el7.x86_64
libglusterfs0-9.5-1.el7.x86_64

ks-k8s-master-2 | CHANGED | rc=0 >>
glusterfs-9.5-1.el7.x86_64
glusterfs-client-xlators-9.5-1.el7.x86_64
glusterfs-fuse-9.5-1.el7.x86_64
libglusterfs0-9.5-1.el7.x86_64

```



## 使用KubeKey部署KubeSphere和Kubernetes

> **01-下载KubeKey**

```shell
# 登陆ks-k8s-master-0节点
[root@zdevops-master dev]# ssh root@ks-k8s-master-0

# 选择中文区下载(访问github受限时使用)
[root@ks-k8s-master-0 ~]# export KKZONE=cn

# 下载KubeKey
[root@ks-k8s-master-0 ~]# mkdir kubekey
[root@ks-k8s-master-0 ~]# cd kubekey/
[root@ks-k8s-master-0 kubekey]# curl -sfL https://get-kk.kubesphere.io | VERSION=v2.0.0 sh -

Downloading kubekey v2.0.0 from https://kubernetes.pek3b.qingstor.com/kubekey/releases/download/v2.0.0/kubekey-v2.0.0-linux-amd64.tar.gz ...


Kubekey v2.0.0 Download Complete!

# 看看下载了哪些文件
[root@ks-k8s-master-0 kubekey]# ls
kk  kubekey-v2.0.0-linux-amd64.tar.gz

# 添加可执行权限(可选，默认已经具有执行权限)
[root@ks-master-0 kubekey]# chmod +x kk
```



> **02-部署KubeSphere和Kubernetes**

```yaml
# 查看当前版本的KubeSphere支持的Kubernetes版本，从列表中选择一个需要的版本，不在列表中的版本不受支持
[root@ks-k8s-master-0 kubekey]# ./kk version --show-supported-k8s
v1.15.12
v1.16.8
v1.16.10
v1.16.12
v1.16.13
v1.17.0
v1.17.4
v1.17.5
v1.17.6
v1.17.7
v1.17.8
v1.17.9
v1.18.3
v1.18.5
v1.18.6
v1.18.8
v1.19.0
v1.19.8
v1.19.9
v1.20.4
v1.20.6
v1.20.10
v1.21.4
v1.21.5
v1.22.1
v1.23.0

# 创建集群配置文件，选择Kubernetesv 1.21.5
[root@ks-k8s-master-0 kubekey]# ./kk create config --with-kubesphere v3.2.1 --with-kubernetes v1.21.5

# 上面的操作，会在当前目录生成config-sample.yaml配置文件
[root@ks-k8s-master-0 kubekey]# ll
total 68480
-rw-r--r-- 1 root root     4777 Apr  3 12:59 config-sample.yaml
-rwxr-xr-x 1 1001  121 53764096 Mar  8 13:05 kk
-rw-r--r-- 1 root root 16348932 Apr  3 12:57 kubekey-v2.0.0-linux-amd64.tar.gz
```



> **03-编辑配置文件**

```yaml
# vi config-sample.yaml

apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: ks-k8s-master-0, address: 192.168.9.91, internalAddress: 192.168.9.91, user: root, password: "F@ywwpTj4bJtYwzpwCqD"}
  - {name: ks-k8s-master-1, address: 192.168.9.92, internalAddress: 192.168.9.92, user: root, password: "F@ywwpTj4bJtYwzpwCqD"}
  - {name: ks-k8s-master-2, address: 192.168.9.93, internalAddress: 192.168.9.93, user: root, password: "F@ywwpTj4bJtYwzpwCqD"}
  roleGroups:
    etcd:
    - ks-k8s-master-0
    - ks-k8s-master-1
    - ks-k8s-master-2
    control-plane: 
    - ks-k8s-master-0
    - ks-k8s-master-1
    - ks-k8s-master-2
    worker:
    - ks-k8s-master-0
    - ks-k8s-master-1
    - ks-k8s-master-2
  controlPlaneEndpoint:
    ## Internal loadbalancer for apiservers 
    internalLoadbalancer: haproxy

    domain: lb.kubesphere.local
    address: ""
    port: 6443
  kubernetes:
    version: v1.21.5
    clusterName: cluster.local
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
    ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni
    multusCNI:
      enabled: false
  registry:
    plainHTTP: false
    privateRegistry: ""
    namespaceOverride: ""
    registryMirrors: []
    insecureRegistries: []
  addons: []



---
apiVersion: installer.kubesphere.io/v1alpha1
kind: ClusterConfiguration
metadata:
  name: ks-installer
  namespace: kubesphere-system
  labels:
    version: v3.2.1
spec:
  persistence:
    storageClass: ""
  authentication:
    jwtSecret: ""
  local_registry: ""
  namespace_override: ""
  # dev_tag: ""
  etcd:
    monitoring: false
    endpointIps: localhost
    port: 2379
    tlsEnable: true
  common:
    core:
      console:
        enableMultiLogin: true
        port: 30880
        type: NodePort
    # apiserver:
    #  resources: {}
    # controllerManager:
    #  resources: {}
    redis:
      enabled: false
      volumeSize: 2Gi
    openldap:
      enabled: false
      volumeSize: 2Gi
    minio:
      volumeSize: 20Gi
    monitoring:
      # type: external
      endpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090
      GPUMonitoring:
        enabled: false
    gpu:
      kinds:         
      - resourceName: "nvidia.com/gpu"
        resourceType: "GPU"
        default: true
    es:
      # master:
      #   volumeSize: 4Gi
      #   replicas: 1
      #   resources: {}
      # data:
      #   volumeSize: 20Gi
      #   replicas: 1
      #   resources: {}
      logMaxAge: 7
      elkPrefix: logstash
      basicAuth:
        enabled: false
        username: ""
        password: ""
      externalElasticsearchHost: ""
      externalElasticsearchPort: ""
  alerting:
    enabled: false
    # thanosruler:
    #   replicas: 1
    #   resources: {}
  auditing:
    enabled: false
    # operator:
    #   resources: {}
    # webhook:
    #   resources: {}
  devops:
    enabled: false
    jenkinsMemoryLim: 2Gi
    jenkinsMemoryReq: 1500Mi
    jenkinsVolumeSize: 8Gi
    jenkinsJavaOpts_Xms: 512m
    jenkinsJavaOpts_Xmx: 512m
    jenkinsJavaOpts_MaxRAM: 2g
  events:
    enabled: false
    # operator:
    #   resources: {}
    # exporter:
    #   resources: {}
    # ruler:
    #   enabled: true
    #   replicas: 2
    #   resources: {}
  logging:
    enabled: false
    containerruntime: docker
    logsidecar:
      enabled: true
      replicas: 2
      # resources: {}
  metrics_server:
    enabled: false
  monitoring:
    storageClass: ""
    # kube_rbac_proxy:
    #   resources: {}
    # kube_state_metrics:
    #   resources: {}
    # prometheus:
    #   replicas: 1
    #   volumeSize: 20Gi
    #   resources: {}
    #   operator:
    #     resources: {}
    #   adapter:
    #     resources: {}
    # node_exporter:
    #   resources: {}
    # alertmanager:
    #   replicas: 1
    #   resources: {}
    # notification_manager:
    #   resources: {}
    #   operator:
    #     resources: {}
    #   proxy:
    #     resources: {}
    gpu:
      nvidia_dcgm_exporter:
        enabled: false
        # resources: {}
  multicluster:
    clusterRole: none 
  network:
    networkpolicy:
      enabled: false
    ippool:
      type: none
    topology:
      type: none
  openpitrix:
    store:
      enabled: false
  servicemesh:
    enabled: false
  kubeedge:
    enabled: false   
    cloudCore:
      nodeSelector: {"node-role.kubernetes.io/worker": ""}
      tolerations: []
      cloudhubPort: "10000"
      cloudhubQuicPort: "10001"
      cloudhubHttpsPort: "10002"
      cloudstreamPort: "10003"
      tunnelPort: "10004"
      cloudHub:
        advertiseAddress:
          - ""
        nodeLimit: "100"
      service:
        cloudhubNodePort: "30000"
        cloudhubQuicNodePort: "30001"
        cloudhubHttpsNodePort: "30002"
        cloudstreamNodePort: "30003"
        tunnelNodePort: "30004"
    edgeWatcher:
      nodeSelector: {"node-role.kubernetes.io/worker": ""}
      tolerations: []
      edgeWatcherAgent:
        nodeSelector: {"node-role.kubernetes.io/worker": ""}
        tolerations: []
```

- **spec.hosts:** 填写节点信息
-  **spec.roleGroups.etcd:** 填写etc节点的**hosts.name**字段的值
-  **spec.roleGroups.control-plane:** 填写k8s master节点的**hosts.name字段的值
-  **spec.roleGroups.worker:** 填写worker节点的**hosts.name字段的值
-  **spec.controlPlaneEndpoint.internalLoadbalancer:** 取消注释



> **04-开始安装KubeSphere和kubernetes**

```shell
# 利用kk开始安装
[root@ks-k8s-master-0 kubekey]# ./kk create cluster -f config-sample.yaml


 _   __      _          _   __           
| | / /     | |        | | / /           
| |/ / _   _| |__   ___| |/ /  ___ _   _ 
|    \| | | | '_ \ / _ \    \ / _ \ | | |
| |\  \ |_| | |_) |  __/ |\  \  __/ |_| |
\_| \_/\__,_|_.__/ \___\_| \_/\___|\__, |
                                    __/ |
                                   |___/

13:01:41 CST [NodePreCheckModule] A pre-check on nodes
13:01:42 CST success: [ks-k8s-master-1]
13:01:42 CST success: [ks-k8s-master-2]
13:01:42 CST success: [ks-k8s-master-0]
13:01:42 CST [ConfirmModule] Display confirmation form
+-----------------+------+------+---------+----------+-------+-------+-----------+--------+--------+------------+-------------+------------------+--------------+
| name            | sudo | curl | openssl | ebtables | socat | ipset | conntrack | chrony | docker | nfs client | ceph client | glusterfs client | time         |
+-----------------+------+------+---------+----------+-------+-------+-----------+--------+--------+------------+-------------+------------------+--------------+
| ks-k8s-master-0 | y    | y    | y       | y        | y     | y     | y         |        |        |            |             | y                | CST 13:01:42 |
| ks-k8s-master-1 | y    | y    | y       | y        | y     | y     | y         |        |        |            |             | y                | CST 13:01:41 |
| ks-k8s-master-2 | y    | y    | y       | y        | y     | y     | y         |        |        |            |             | y                | CST 13:01:41 |
+-----------------+------+------+---------+----------+-------+-------+-----------+--------+--------+------------+-------------+------------------+--------------+

This is a simple check of your environment.
Before installation, you should ensure that your machines meet all requirements specified at
https://github.com/kubesphere/kubekey#requirements-and-recommendations

Continue this installation? [yes/no]: yes
13:01:52 CST success: [LocalHost]
13:01:52 CST [NodeBinariesModule] Download installation binaries
13:01:52 CST message: [localhost]
downloading amd64 kubeadm v1.21.5 ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 42.7M  100 42.7M    0     0  1032k      0  0:00:42  0:00:42 --:--:-- 1199k
13:02:36 CST message: [localhost]
downloading amd64 kubelet v1.21.5 ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  112M  100  112M    0     0  1025k      0  0:01:52  0:01:52 --:--:-- 1201k
13:04:31 CST message: [localhost]
downloading amd64 kubectl v1.21.5 ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 44.4M  100 44.4M    0     0  1021k      0  0:00:44  0:00:44 --:--:-- 1067k
13:05:16 CST message: [localhost]
downloading amd64 helm v3.6.3 ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 43.0M  100 43.0M    0     0  1009k      0  0:00:43  0:00:43 --:--:--  999k
13:06:01 CST message: [localhost]
downloading amd64 kubecni v0.9.1 ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 37.9M  100 37.9M    0     0  1034k      0  0:00:37  0:00:37 --:--:-- 1190k
13:06:39 CST message: [localhost]
downloading amd64 docker 20.10.8 ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 58.1M  100 58.1M    0     0  37.5M      0  0:00:01  0:00:01 --:--:-- 37.5M
13:06:41 CST message: [localhost]
downloading amd64 crictl v1.22.0 ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 17.8M  100 17.8M    0     0  1046k      0  0:00:17  0:00:17 --:--:-- 1187k
13:06:59 CST message: [localhost]
downloading amd64 etcd v3.4.13 ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16.5M  100 16.5M    0     0  1039k      0  0:00:16  0:00:16 --:--:-- 1135k
13:07:16 CST success: [LocalHost]
13:07:16 CST [ConfigureOSModule] Prepare to init OS
13:07:18 CST success: [ks-k8s-master-0]
13:07:18 CST success: [ks-k8s-master-1]
13:07:18 CST success: [ks-k8s-master-2]
13:07:18 CST [ConfigureOSModule] Generate init os script
13:07:18 CST success: [ks-k8s-master-0]
13:07:18 CST success: [ks-k8s-master-1]
13:07:18 CST success: [ks-k8s-master-2]
13:07:18 CST [ConfigureOSModule] Exec init os script
13:07:21 CST stdout: [ks-k8s-master-1]
setenforce: SELinux is disabled
Disabled
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_local_reserved_ports = 30000-32767
vm.max_map_count = 262144
vm.swappiness = 1
fs.inotify.max_user_instances = 524288
kernel.pid_max = 65535
no crontab for root
13:07:22 CST stdout: [ks-k8s-master-0]
setenforce: SELinux is disabled
Disabled
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_local_reserved_ports = 30000-32767
vm.max_map_count = 262144
vm.swappiness = 1
fs.inotify.max_user_instances = 524288
kernel.pid_max = 65535
no crontab for root
13:07:22 CST stdout: [ks-k8s-master-2]
setenforce: SELinux is disabled
Disabled
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_local_reserved_ports = 30000-32767
vm.max_map_count = 262144
vm.swappiness = 1
fs.inotify.max_user_instances = 524288
kernel.pid_max = 65535
no crontab for root
13:07:22 CST success: [ks-k8s-master-1]
13:07:22 CST success: [ks-k8s-master-0]
13:07:22 CST success: [ks-k8s-master-2]
13:07:22 CST [ConfigureOSModule] configure the ntp server for each node
13:07:22 CST skipped: [ks-k8s-master-2]
13:07:22 CST skipped: [ks-k8s-master-0]
13:07:22 CST skipped: [ks-k8s-master-1]
13:07:22 CST [KubernetesStatusModule] Get kubernetes cluster status
13:07:25 CST success: [ks-k8s-master-0]
13:07:25 CST success: [ks-k8s-master-1]
13:07:25 CST success: [ks-k8s-master-2]
13:07:25 CST [InstallContainerModule] Sync docker binaries
13:07:31 CST success: [ks-k8s-master-0]
13:07:31 CST success: [ks-k8s-master-1]
13:07:31 CST success: [ks-k8s-master-2]
13:07:31 CST [InstallContainerModule] Generate containerd service
13:07:32 CST success: [ks-k8s-master-0]
13:07:32 CST success: [ks-k8s-master-1]
13:07:32 CST success: [ks-k8s-master-2]
13:07:32 CST [InstallContainerModule] Enable containerd
13:07:35 CST success: [ks-k8s-master-2]
13:07:35 CST success: [ks-k8s-master-1]
13:07:35 CST success: [ks-k8s-master-0]
13:07:35 CST [InstallContainerModule] Generate docker service
13:07:36 CST success: [ks-k8s-master-0]
13:07:36 CST success: [ks-k8s-master-1]
13:07:36 CST success: [ks-k8s-master-2]
13:07:36 CST [InstallContainerModule] Generate docker config
13:07:36 CST success: [ks-k8s-master-0]
13:07:36 CST success: [ks-k8s-master-1]
13:07:36 CST success: [ks-k8s-master-2]
13:07:36 CST [InstallContainerModule] Enable docker
13:07:38 CST success: [ks-k8s-master-0]
13:07:38 CST success: [ks-k8s-master-1]
13:07:38 CST success: [ks-k8s-master-2]
13:07:38 CST [InstallContainerModule] Add auths to container runtime
13:07:38 CST skipped: [ks-k8s-master-0]
13:07:38 CST skipped: [ks-k8s-master-1]
13:07:38 CST skipped: [ks-k8s-master-2]
13:07:38 CST [PullModule] Start to pull images on all nodes
13:07:38 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1
13:07:38 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1
13:07:38 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1
13:07:39 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-apiserver:v1.21.5
13:07:44 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-apiserver:v1.21.5
13:07:44 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-apiserver:v1.21.5
13:07:45 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-controller-manager:v1.21.5
13:07:51 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-controller-manager:v1.21.5
13:07:51 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-controller-manager:v1.21.5
13:07:52 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-scheduler:v1.21.5
13:08:00 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-proxy:v1.21.5
13:08:01 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-scheduler:v1.21.5
13:08:04 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-scheduler:v1.21.5
13:08:05 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-proxy:v1.21.5
13:08:07 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/coredns:1.8.0
13:08:07 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-proxy:v1.21.5
13:08:11 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/k8s-dns-node-cache:1.15.12
13:08:11 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/coredns:1.8.0
13:08:14 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/coredns:1.8.0
13:08:14 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/k8s-dns-node-cache:1.15.12
13:08:17 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/k8s-dns-node-cache:1.15.12
13:08:17 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-controllers:v3.20.0
13:08:21 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-controllers:v3.20.0
13:08:22 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/cni:v3.20.0
13:08:23 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/kube-controllers:v3.20.0
13:08:25 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/cni:v3.20.0
13:08:28 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/cni:v3.20.0
13:08:30 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/node:v3.20.0
13:08:34 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/node:v3.20.0
13:08:36 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/node:v3.20.0
13:08:41 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/pod2daemon-flexvol:v3.20.0
13:08:43 CST message: [ks-k8s-master-1]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/haproxy:2.3
13:08:46 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/pod2daemon-flexvol:v3.20.0
13:08:48 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/pod2daemon-flexvol:v3.20.0
13:08:49 CST message: [ks-k8s-master-0]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/haproxy:2.3
13:08:52 CST message: [ks-k8s-master-2]
downloading image: registry.cn-beijing.aliyuncs.com/kubesphereio/haproxy:2.3
13:09:01 CST success: [ks-k8s-master-1]
13:09:01 CST success: [ks-k8s-master-0]
13:09:01 CST success: [ks-k8s-master-2]
13:09:01 CST [ETCDPreCheckModule] Get etcd status
13:09:01 CST success: [ks-k8s-master-0]
13:09:01 CST success: [ks-k8s-master-1]
13:09:01 CST success: [ks-k8s-master-2]
13:09:01 CST [CertsModule] Fetcd etcd certs
13:09:01 CST success: [ks-k8s-master-0]
13:09:01 CST skipped: [ks-k8s-master-1]
13:09:01 CST skipped: [ks-k8s-master-2]
13:09:01 CST [CertsModule] Generate etcd Certs
[certs] Generating "ca" certificate and key
[certs] admin-ks-k8s-master-0 serving cert is signed for DNS names [etcd etcd.kube-system etcd.kube-system.svc etcd.kube-system.svc.cluster.local ks-k8s-master-0 ks-k8s-master-1 ks-k8s-master-2 lb.kubesphere.local localhost] and IPs [127.0.0.1 ::1 192.168.9.91 192.168.9.92 192.168.9.93]
[certs] member-ks-k8s-master-0 serving cert is signed for DNS names [etcd etcd.kube-system etcd.kube-system.svc etcd.kube-system.svc.cluster.local ks-k8s-master-0 ks-k8s-master-1 ks-k8s-master-2 lb.kubesphere.local localhost] and IPs [127.0.0.1 ::1 192.168.9.91 192.168.9.92 192.168.9.93]
[certs] node-ks-k8s-master-0 serving cert is signed for DNS names [etcd etcd.kube-system etcd.kube-system.svc etcd.kube-system.svc.cluster.local ks-k8s-master-0 ks-k8s-master-1 ks-k8s-master-2 lb.kubesphere.local localhost] and IPs [127.0.0.1 ::1 192.168.9.91 192.168.9.92 192.168.9.93]
[certs] admin-ks-k8s-master-1 serving cert is signed for DNS names [etcd etcd.kube-system etcd.kube-system.svc etcd.kube-system.svc.cluster.local ks-k8s-master-0 ks-k8s-master-1 ks-k8s-master-2 lb.kubesphere.local localhost] and IPs [127.0.0.1 ::1 192.168.9.91 192.168.9.92 192.168.9.93]
[certs] member-ks-k8s-master-1 serving cert is signed for DNS names [etcd etcd.kube-system etcd.kube-system.svc etcd.kube-system.svc.cluster.local ks-k8s-master-0 ks-k8s-master-1 ks-k8s-master-2 lb.kubesphere.local localhost] and IPs [127.0.0.1 ::1 192.168.9.91 192.168.9.92 192.168.9.93]
[certs] node-ks-k8s-master-1 serving cert is signed for DNS names [etcd etcd.kube-system etcd.kube-system.svc etcd.kube-system.svc.cluster.local ks-k8s-master-0 ks-k8s-master-1 ks-k8s-master-2 lb.kubesphere.local localhost] and IPs [127.0.0.1 ::1 192.168.9.91 192.168.9.92 192.168.9.93]
[certs] admin-ks-k8s-master-2 serving cert is signed for DNS names [etcd etcd.kube-system etcd.kube-system.svc etcd.kube-system.svc.cluster.local ks-k8s-master-0 ks-k8s-master-1 ks-k8s-master-2 lb.kubesphere.local localhost] and IPs [127.0.0.1 ::1 192.168.9.91 192.168.9.92 192.168.9.93]
[certs] member-ks-k8s-master-2 serving cert is signed for DNS names [etcd etcd.kube-system etcd.kube-system.svc etcd.kube-system.svc.cluster.local ks-k8s-master-0 ks-k8s-master-1 ks-k8s-master-2 lb.kubesphere.local localhost] and IPs [127.0.0.1 ::1 192.168.9.91 192.168.9.92 192.168.9.93]
[certs] node-ks-k8s-master-2 serving cert is signed for DNS names [etcd etcd.kube-system etcd.kube-system.svc etcd.kube-system.svc.cluster.local ks-k8s-master-0 ks-k8s-master-1 ks-k8s-master-2 lb.kubesphere.local localhost] and IPs [127.0.0.1 ::1 192.168.9.91 192.168.9.92 192.168.9.93]
13:09:06 CST success: [LocalHost]
13:09:06 CST [CertsModule] Synchronize certs file
13:09:17 CST success: [ks-k8s-master-0]
13:09:17 CST success: [ks-k8s-master-1]
13:09:17 CST success: [ks-k8s-master-2]
13:09:17 CST [CertsModule] Synchronize certs file to master
13:09:17 CST skipped: [ks-k8s-master-2]
13:09:17 CST skipped: [ks-k8s-master-0]
13:09:17 CST skipped: [ks-k8s-master-1]
13:09:17 CST [InstallETCDBinaryModule] Install etcd using binary
13:09:19 CST success: [ks-k8s-master-1]
13:09:19 CST success: [ks-k8s-master-0]
13:09:19 CST success: [ks-k8s-master-2]
13:09:19 CST [InstallETCDBinaryModule] Generate etcd service
13:09:19 CST success: [ks-k8s-master-0]
13:09:19 CST success: [ks-k8s-master-1]
13:09:19 CST success: [ks-k8s-master-2]
13:09:19 CST [InstallETCDBinaryModule] Generate access address
13:09:19 CST skipped: [ks-k8s-master-2]
13:09:19 CST skipped: [ks-k8s-master-1]
13:09:19 CST success: [ks-k8s-master-0]
13:09:19 CST [ETCDConfigureModule] Health check on exist etcd
13:09:19 CST skipped: [ks-k8s-master-2]
13:09:19 CST skipped: [ks-k8s-master-1]
13:09:19 CST skipped: [ks-k8s-master-0]
13:09:19 CST [ETCDConfigureModule] Generate etcd.env config on new etcd
13:09:21 CST success: [ks-k8s-master-0]
13:09:21 CST success: [ks-k8s-master-1]
13:09:21 CST success: [ks-k8s-master-2]
13:09:21 CST [ETCDConfigureModule] Refresh etcd.env config on all etcd
13:09:22 CST success: [ks-k8s-master-0]
13:09:22 CST success: [ks-k8s-master-1]
13:09:22 CST success: [ks-k8s-master-2]
13:09:22 CST [ETCDConfigureModule] Restart etcd
13:09:26 CST stdout: [ks-k8s-master-1]
Created symlink from /etc/systemd/system/multi-user.target.wants/etcd.service to /etc/systemd/system/etcd.service.
13:09:26 CST stdout: [ks-k8s-master-2]
Created symlink from /etc/systemd/system/multi-user.target.wants/etcd.service to /etc/systemd/system/etcd.service.
13:09:26 CST stdout: [ks-k8s-master-0]
Created symlink from /etc/systemd/system/multi-user.target.wants/etcd.service to /etc/systemd/system/etcd.service.
13:09:26 CST success: [ks-k8s-master-1]
13:09:26 CST success: [ks-k8s-master-2]
13:09:26 CST success: [ks-k8s-master-0]
13:09:26 CST [ETCDConfigureModule] Health check on all etcd
13:09:26 CST success: [ks-k8s-master-1]
13:09:26 CST success: [ks-k8s-master-2]
13:09:26 CST success: [ks-k8s-master-0]
13:09:26 CST [ETCDConfigureModule] Refresh etcd.env config to exist mode on all etcd
13:09:28 CST success: [ks-k8s-master-0]
13:09:28 CST success: [ks-k8s-master-1]
13:09:28 CST success: [ks-k8s-master-2]
13:09:28 CST [ETCDConfigureModule] Health check on all etcd
13:09:28 CST success: [ks-k8s-master-0]
13:09:28 CST success: [ks-k8s-master-2]
13:09:28 CST success: [ks-k8s-master-1]
13:09:28 CST [ETCDBackupModule] Backup etcd data regularly
13:09:35 CST success: [ks-k8s-master-0]
13:09:35 CST success: [ks-k8s-master-1]
13:09:35 CST success: [ks-k8s-master-2]
13:09:35 CST [InstallKubeBinariesModule] Synchronize kubernetes binaries
13:09:53 CST success: [ks-k8s-master-0]
13:09:53 CST success: [ks-k8s-master-1]
13:09:53 CST success: [ks-k8s-master-2]
13:09:53 CST [InstallKubeBinariesModule] Synchronize kubelet
13:09:53 CST success: [ks-k8s-master-0]
13:09:53 CST success: [ks-k8s-master-1]
13:09:53 CST success: [ks-k8s-master-2]
13:09:53 CST [InstallKubeBinariesModule] Generate kubelet service
13:09:54 CST success: [ks-k8s-master-0]
13:09:54 CST success: [ks-k8s-master-1]
13:09:54 CST success: [ks-k8s-master-2]
13:09:54 CST [InstallKubeBinariesModule] Enable kubelet service
13:09:54 CST success: [ks-k8s-master-2]
13:09:54 CST success: [ks-k8s-master-1]
13:09:54 CST success: [ks-k8s-master-0]
13:09:54 CST [InstallKubeBinariesModule] Generate kubelet env
13:09:55 CST success: [ks-k8s-master-0]
13:09:55 CST success: [ks-k8s-master-1]
13:09:55 CST success: [ks-k8s-master-2]
13:09:55 CST [InitKubernetesModule] Generate kubeadm config
13:09:55 CST skipped: [ks-k8s-master-2]
13:09:55 CST skipped: [ks-k8s-master-1]
13:09:55 CST success: [ks-k8s-master-0]
13:09:55 CST [InitKubernetesModule] Init cluster using kubeadm
13:10:28 CST stdout: [ks-k8s-master-0]
W0403 13:09:55.943937    7632 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]
[init] Using Kubernetes version: v1.21.5
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [ks-k8s-master-0 ks-k8s-master-0.cluster.local ks-k8s-master-1 ks-k8s-master-1.cluster.local ks-k8s-master-2 ks-k8s-master-2.cluster.local kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local lb.kubesphere.local localhost] and IPs [10.233.0.1 192.168.9.91 127.0.0.1 192.168.9.92 192.168.9.93]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] External etcd mode: Skipping etcd/ca certificate authority generation
[certs] External etcd mode: Skipping etcd/server certificate generation
[certs] External etcd mode: Skipping etcd/peer certificate generation
[certs] External etcd mode: Skipping etcd/healthcheck-client certificate generation
[certs] External etcd mode: Skipping apiserver-etcd-client certificate generation
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 21.505575 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.21" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node ks-k8s-master-0 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node ks-k8s-master-0 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: trplud.imfongtz95d70seh
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join lb.kubesphere.local:6443 --token trplud.imfongtz95d70seh \
        --discovery-token-ca-cert-hash sha256:952a8ef81ae3b4a1a05b5ef9f6b3772e5e9ffa3e8bd9881e189784d427fa085b \
        --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join lb.kubesphere.local:6443 --token trplud.imfongtz95d70seh \
        --discovery-token-ca-cert-hash sha256:952a8ef81ae3b4a1a05b5ef9f6b3772e5e9ffa3e8bd9881e189784d427fa085b
13:10:28 CST skipped: [ks-k8s-master-2]
13:10:28 CST skipped: [ks-k8s-master-1]
13:10:28 CST success: [ks-k8s-master-0]
13:10:28 CST [InitKubernetesModule] Copy admin.conf to ~/.kube/config
13:10:29 CST skipped: [ks-k8s-master-2]
13:10:29 CST skipped: [ks-k8s-master-1]
13:10:29 CST success: [ks-k8s-master-0]
13:10:29 CST [InitKubernetesModule] Remove master taint
13:10:29 CST stdout: [ks-k8s-master-0]
node/ks-k8s-master-0 untainted
13:10:29 CST skipped: [ks-k8s-master-2]
13:10:29 CST skipped: [ks-k8s-master-1]
13:10:29 CST success: [ks-k8s-master-0]
13:10:29 CST [InitKubernetesModule] Add worker label
13:10:29 CST stdout: [ks-k8s-master-0]
node/ks-k8s-master-0 labeled
13:10:29 CST skipped: [ks-k8s-master-2]
13:10:29 CST skipped: [ks-k8s-master-1]
13:10:29 CST success: [ks-k8s-master-0]
13:10:29 CST [ClusterDNSModule] Generate coredns service
13:10:30 CST skipped: [ks-k8s-master-2]
13:10:30 CST skipped: [ks-k8s-master-1]
13:10:30 CST success: [ks-k8s-master-0]
13:10:30 CST [ClusterDNSModule] Override coredns service
13:10:30 CST stdout: [ks-k8s-master-0]
service "kube-dns" deleted
13:10:31 CST stdout: [ks-k8s-master-0]
service/coredns created
Warning: resource clusterroles/system:coredns is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
clusterrole.rbac.authorization.k8s.io/system:coredns configured
13:10:31 CST skipped: [ks-k8s-master-2]
13:10:31 CST skipped: [ks-k8s-master-1]
13:10:31 CST success: [ks-k8s-master-0]
13:10:31 CST [ClusterDNSModule] Generate nodelocaldns
13:10:31 CST skipped: [ks-k8s-master-2]
13:10:31 CST skipped: [ks-k8s-master-1]
13:10:31 CST success: [ks-k8s-master-0]
13:10:31 CST [ClusterDNSModule] Deploy nodelocaldns
13:10:32 CST stdout: [ks-k8s-master-0]
serviceaccount/nodelocaldns created
daemonset.apps/nodelocaldns created
13:10:32 CST skipped: [ks-k8s-master-2]
13:10:32 CST skipped: [ks-k8s-master-1]
13:10:32 CST success: [ks-k8s-master-0]
13:10:32 CST [ClusterDNSModule] Generate nodelocaldns configmap
13:10:32 CST skipped: [ks-k8s-master-2]
13:10:32 CST skipped: [ks-k8s-master-1]
13:10:32 CST success: [ks-k8s-master-0]
13:10:32 CST [ClusterDNSModule] Apply nodelocaldns configmap
13:10:33 CST stdout: [ks-k8s-master-0]
configmap/nodelocaldns created
13:10:33 CST skipped: [ks-k8s-master-2]
13:10:33 CST skipped: [ks-k8s-master-1]
13:10:33 CST success: [ks-k8s-master-0]
13:10:33 CST [KubernetesStatusModule] Get kubernetes cluster status
13:10:33 CST stdout: [ks-k8s-master-0]
v1.21.5
13:10:33 CST stdout: [ks-k8s-master-0]
ks-k8s-master-0   v1.21.5   [map[address:192.168.9.91 type:InternalIP] map[address:ks-k8s-master-0 type:Hostname]]
13:10:35 CST stdout: [ks-k8s-master-0]
I0403 13:10:35.044308    9697 version.go:254] remote version is much newer: v1.23.5; falling back to: stable-1.21
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
7f336d1ef4ba900e254f5503dd7f5ab9f94333939564155e74635ba0c74050b1
13:10:36 CST stdout: [ks-k8s-master-0]
secret/kubeadm-certs patched
13:10:36 CST stdout: [ks-k8s-master-0]
secret/kubeadm-certs patched
13:10:36 CST stdout: [ks-k8s-master-0]
secret/kubeadm-certs patched
13:10:36 CST stdout: [ks-k8s-master-0]
zf886x.zo915d50drm7r76g
13:10:37 CST success: [ks-k8s-master-0]
13:10:37 CST success: [ks-k8s-master-1]
13:10:37 CST success: [ks-k8s-master-2]
13:10:37 CST [JoinNodesModule] Generate kubeadm config
13:10:37 CST skipped: [ks-k8s-master-0]
13:10:37 CST success: [ks-k8s-master-1]
13:10:37 CST success: [ks-k8s-master-2]
13:10:37 CST [JoinNodesModule] Join control-plane node
13:10:58 CST stdout: [ks-k8s-master-1]
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0403 13:10:43.690219    7522 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]
[preflight] Running pre-flight checks before initializing the new control plane instance
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [ks-k8s-master-0 ks-k8s-master-0.cluster.local ks-k8s-master-1 ks-k8s-master-1.cluster.local ks-k8s-master-2 ks-k8s-master-2.cluster.local kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local lb.kubesphere.local localhost] and IPs [10.233.0.1 192.168.9.92 127.0.0.1 192.168.9.91 192.168.9.93]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[certs] Using the existing "sa" key
[kubeconfig] Generating kubeconfig files
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[check-etcd] Skipping etcd check in external mode
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
[control-plane-join] using external etcd - no local stacked instance added
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[mark-control-plane] Marking the node ks-k8s-master-1 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node ks-k8s-master-1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.


To start administering your cluster from this node, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
13:10:59 CST stdout: [ks-k8s-master-2]
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0403 13:10:43.758537    7528 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]
[preflight] Running pre-flight checks before initializing the new control plane instance
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [ks-k8s-master-0 ks-k8s-master-0.cluster.local ks-k8s-master-1 ks-k8s-master-1.cluster.local ks-k8s-master-2 ks-k8s-master-2.cluster.local kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local lb.kubesphere.local localhost] and IPs [10.233.0.1 192.168.9.93 127.0.0.1 192.168.9.91 192.168.9.92]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[certs] Using the existing "sa" key
[kubeconfig] Generating kubeconfig files
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[check-etcd] Skipping etcd check in external mode
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
[control-plane-join] using external etcd - no local stacked instance added
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[mark-control-plane] Marking the node ks-k8s-master-2 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node ks-k8s-master-2 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.


To start administering your cluster from this node, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
13:10:59 CST skipped: [ks-k8s-master-0]
13:10:59 CST success: [ks-k8s-master-1]
13:10:59 CST success: [ks-k8s-master-2]
13:10:59 CST [JoinNodesModule] Join worker node
13:10:59 CST skipped: [ks-k8s-master-2]
13:10:59 CST skipped: [ks-k8s-master-0]
13:10:59 CST skipped: [ks-k8s-master-1]
13:10:59 CST [JoinNodesModule] Copy admin.conf to ~/.kube/config
13:11:00 CST skipped: [ks-k8s-master-0]
13:11:00 CST success: [ks-k8s-master-1]
13:11:00 CST success: [ks-k8s-master-2]
13:11:00 CST [JoinNodesModule] Remove master taint
13:11:00 CST stdout: [ks-k8s-master-1]
node/ks-k8s-master-1 untainted
13:11:01 CST stdout: [ks-k8s-master-2]
node/ks-k8s-master-2 untainted
13:11:01 CST skipped: [ks-k8s-master-0]
13:11:01 CST success: [ks-k8s-master-1]
13:11:01 CST success: [ks-k8s-master-2]
13:11:01 CST [JoinNodesModule] Add worker label to master
13:11:01 CST stdout: [ks-k8s-master-1]
node/ks-k8s-master-1 labeled
13:11:01 CST stdout: [ks-k8s-master-2]
node/ks-k8s-master-2 labeled
13:11:01 CST skipped: [ks-k8s-master-0]
13:11:01 CST success: [ks-k8s-master-1]
13:11:01 CST success: [ks-k8s-master-2]
13:11:01 CST [JoinNodesModule] Synchronize kube config to worker
13:11:01 CST skipped: [ks-k8s-master-2]
13:11:01 CST skipped: [ks-k8s-master-0]
13:11:01 CST skipped: [ks-k8s-master-1]
13:11:01 CST [JoinNodesModule] Add worker label to worker
13:11:01 CST skipped: [ks-k8s-master-2]
13:11:01 CST skipped: [ks-k8s-master-0]
13:11:01 CST skipped: [ks-k8s-master-1]
13:11:01 CST [InternalLoadbalancerModule] Generate haproxy.cfg
13:11:01 CST skipped: [ks-k8s-master-2]
13:11:01 CST skipped: [ks-k8s-master-0]
13:11:01 CST skipped: [ks-k8s-master-1]
13:11:01 CST [InternalLoadbalancerModule] Calculate the MD5 value according to haproxy.cfg
13:11:01 CST skipped: [ks-k8s-master-2]
13:11:01 CST skipped: [ks-k8s-master-1]
13:11:01 CST skipped: [ks-k8s-master-0]
13:11:01 CST [InternalLoadbalancerModule] Generate haproxy manifest
13:11:01 CST skipped: [ks-k8s-master-0]
13:11:01 CST skipped: [ks-k8s-master-2]
13:11:01 CST skipped: [ks-k8s-master-1]
13:11:01 CST [InternalLoadbalancerModule] Update kubelet config
13:11:01 CST stdout: [ks-k8s-master-0]
server: https://lb.kubesphere.local:6443
13:11:01 CST stdout: [ks-k8s-master-1]
server: https://lb.kubesphere.local:6443
13:11:01 CST stdout: [ks-k8s-master-2]
server: https://lb.kubesphere.local:6443
13:11:02 CST success: [ks-k8s-master-0]
13:11:02 CST success: [ks-k8s-master-1]
13:11:02 CST success: [ks-k8s-master-2]
13:11:02 CST [InternalLoadbalancerModule] Update kube-proxy configmap
13:11:03 CST success: [ks-k8s-master-0]
13:11:03 CST [InternalLoadbalancerModule] Update /etc/hosts
13:11:03 CST success: [ks-k8s-master-0]
13:11:03 CST success: [ks-k8s-master-1]
13:11:03 CST success: [ks-k8s-master-2]
13:11:03 CST [DeployNetworkPluginModule] Generate calico
13:11:04 CST skipped: [ks-k8s-master-2]
13:11:04 CST skipped: [ks-k8s-master-1]
13:11:04 CST success: [ks-k8s-master-0]
13:11:04 CST [DeployNetworkPluginModule] Deploy calico
13:11:05 CST stdout: [ks-k8s-master-0]
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
poddisruptionbudget.policy/calico-kube-controllers created
13:11:05 CST skipped: [ks-k8s-master-2]
13:11:05 CST skipped: [ks-k8s-master-1]
13:11:05 CST success: [ks-k8s-master-0]
13:11:05 CST [ConfigureKubernetesModule] Configure kubernetes
13:11:05 CST success: [ks-k8s-master-2]
13:11:05 CST success: [ks-k8s-master-1]
13:11:05 CST success: [ks-k8s-master-0]
13:11:05 CST [ChownModule] Chown user $HOME/.kube dir
13:11:06 CST success: [ks-k8s-master-1]
13:11:06 CST success: [ks-k8s-master-0]
13:11:06 CST success: [ks-k8s-master-2]
13:11:06 CST [AutoRenewCertsModule] Generate k8s certs renew script
13:11:06 CST success: [ks-k8s-master-0]
13:11:06 CST success: [ks-k8s-master-1]
13:11:06 CST success: [ks-k8s-master-2]
13:11:06 CST [AutoRenewCertsModule] Generate k8s certs renew service
13:11:07 CST success: [ks-k8s-master-0]
13:11:07 CST success: [ks-k8s-master-1]
13:11:07 CST success: [ks-k8s-master-2]
13:11:07 CST [AutoRenewCertsModule] Generate k8s certs renew timer
13:11:07 CST success: [ks-k8s-master-0]
13:11:07 CST success: [ks-k8s-master-1]
13:11:07 CST success: [ks-k8s-master-2]
13:11:07 CST [AutoRenewCertsModule] Enable k8s certs renew service
13:11:08 CST success: [ks-k8s-master-0]
13:11:08 CST success: [ks-k8s-master-1]
13:11:08 CST success: [ks-k8s-master-2]
13:11:08 CST [SaveKubeConfigModule] Save kube config as a configmap
13:11:08 CST success: [LocalHost]
13:11:08 CST [AddonsModule] Install addons
13:11:08 CST success: [LocalHost]
13:11:08 CST [DeployStorageClassModule] Generate OpenEBS manifest
13:11:08 CST skipped: [ks-k8s-master-2]
13:11:08 CST skipped: [ks-k8s-master-1]
13:11:08 CST success: [ks-k8s-master-0]
13:11:08 CST [DeployStorageClassModule] Deploy OpenEBS as cluster default StorageClass
13:11:09 CST skipped: [ks-k8s-master-1]
13:11:09 CST skipped: [ks-k8s-master-2]
13:11:09 CST success: [ks-k8s-master-0]
13:11:09 CST [DeployKubeSphereModule] Generate KubeSphere ks-installer crd manifests
13:11:10 CST skipped: [ks-k8s-master-2]
13:11:10 CST skipped: [ks-k8s-master-1]
13:11:10 CST success: [ks-k8s-master-0]
13:11:10 CST [DeployKubeSphereModule] Apply ks-installer
13:11:11 CST stdout: [ks-k8s-master-0]
namespace/kubesphere-system created
serviceaccount/ks-installer created
customresourcedefinition.apiextensions.k8s.io/clusterconfigurations.installer.kubesphere.io created
clusterrole.rbac.authorization.k8s.io/ks-installer created
clusterrolebinding.rbac.authorization.k8s.io/ks-installer created
deployment.apps/ks-installer created
13:11:11 CST skipped: [ks-k8s-master-2]
13:11:11 CST skipped: [ks-k8s-master-1]
13:11:11 CST success: [ks-k8s-master-0]
13:11:11 CST [DeployKubeSphereModule] Add config to ks-installer manifests
13:11:11 CST skipped: [ks-k8s-master-2]
13:11:11 CST skipped: [ks-k8s-master-1]
13:11:11 CST success: [ks-k8s-master-0]
13:11:11 CST [DeployKubeSphereModule] Create the kubesphere namespace
13:11:12 CST skipped: [ks-k8s-master-2]
13:11:12 CST skipped: [ks-k8s-master-1]
13:11:12 CST success: [ks-k8s-master-0]
13:11:12 CST [DeployKubeSphereModule] Setup ks-installer config
13:11:12 CST stdout: [ks-k8s-master-0]
secret/kube-etcd-client-certs created
13:11:12 CST skipped: [ks-k8s-master-2]
13:11:12 CST skipped: [ks-k8s-master-1]
13:11:12 CST success: [ks-k8s-master-0]
13:11:12 CST [DeployKubeSphereModule] Apply ks-installer
13:11:18 CST stdout: [ks-k8s-master-0]
namespace/kubesphere-system unchanged
serviceaccount/ks-installer unchanged
customresourcedefinition.apiextensions.k8s.io/clusterconfigurations.installer.kubesphere.io unchanged
clusterrole.rbac.authorization.k8s.io/ks-installer unchanged
clusterrolebinding.rbac.authorization.k8s.io/ks-installer unchanged
deployment.apps/ks-installer unchanged
clusterconfiguration.installer.kubesphere.io/ks-installer created
13:11:18 CST skipped: [ks-k8s-master-2]
13:11:18 CST skipped: [ks-k8s-master-1]
13:11:18 CST success: [ks-k8s-master-0]
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
https://kubesphere.io             2022-04-03 13:27:48
#####################################################
13:27:51 CST skipped: [ks-k8s-master-2]
13:27:51 CST skipped: [ks-k8s-master-1]
13:27:51 CST success: [ks-k8s-master-0]
13:27:51 CST Pipeline[CreateClusterPipeline] execute successful
Installation is complete.

Please check the result using the command:

        kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f

```

- **安装开始时间：13:01:41**
- **安装完成时间： 13:27:51**
- **安装总用时: 26:10** 

> **05-验证安装**

```yaml
# 运行以下命令查看安装日志
[root@ks-k8s-master-1 ~]# kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f

# 只要看到"Welcome to KubeSphere!",说明安装成功
# 截取以下内容，看看详细效果

.......
TASK [ks-core/prepare : KubeSphere | Initing KubeSphere] ***********************
changed: [localhost] => (item=role-templates.yaml)

TASK [ks-core/prepare : KubeSphere | Generating kubeconfig-admin] **************
skipping: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=25   changed=18   unreachable=0    failed=0    skipped=13   rescued=0    ignored=0   

Start installing monitoring
Start installing multicluster
Start installing openpitrix
Start installing network
**************************************************
Waiting for all tasks to be completed ...
task network status is successful  (1/4)
task openpitrix status is successful  (2/4)
task multicluster status is successful  (3/4)
task monitoring status is successful  (4/4)
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
https://kubesphere.io             2022-04-03 13:27:48
#####################################################

```



> **06-查看k8s节点信息**

```shell
[root@ks-k8s-master-0 kubekey]# kubectl get nodes -o wide
NAME              STATUS   ROLES                         AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
ks-k8s-master-0   Ready    control-plane,master,worker   90m   v1.21.5   192.168.9.91   <none>        CentOS Linux 7 (Core)   3.10.0-1160.59.1.el7.x86_64   docker://20.10.8
ks-k8s-master-1   Ready    control-plane,master,worker   90m   v1.21.5   192.168.9.92   <none>        CentOS Linux 7 (Core)   3.10.0-1160.59.1.el7.x86_64   docker://20.10.8
ks-k8s-master-2   Ready    control-plane,master,worker   90m   v1.21.5   192.168.9.93   <none>        CentOS Linux 7 (Core)   3.10.0-1160.59.1.el7.x86_64   docker://20.10.8
```



> **07-配置bash自动补全功能**

```shell
[root@ks-k8s-master-0 zdevops]# yum install bash-completion
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.cn99.com
 * centos-gluster9: mirrors.cn99.com
 * extras: mirrors.ustc.edu.cn
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package bash-completion.noarch 1:2.1-8.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=====================================================================================================================
 Package                          Arch                    Version                        Repository             Size
=====================================================================================================================
Installing:
 bash-completion                  noarch                  1:2.1-8.el7                    base                   87 k

Transaction Summary
=====================================================================================================================
Install  1 Package

Total download size: 87 k
Installed size: 263 k
Is this ok [y/d/N]: y
Downloading packages:
bash-completion-2.1-8.el7.noarch.rpm                                                          |  87 kB  00:00:00     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : 1:bash-completion-2.1-8.el7.noarch                                                                1/1 
  Verifying  : 1:bash-completion-2.1-8.el7.noarch                                                                1/1 

Installed:
  bash-completion.noarch 1:2.1-8.el7                                                                                 

Complete!


# kubectl 补全脚本导入（sourced）到 shell 会话中，两种方法二选一
# 方法一：当前用户
[root@ks-k8s-master-0 ~]# echo 'source <(kubectl completion bash)' >> ~/.bashrc

# 方法二：系统全局
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null

# 本文选则第一种，使sourced生效
[root@ks-k8s-master-0 ~]# source ~/.bashrc 
[root@ks-k8s-master-0 ~]# kubectl get n
namespacenetworkpolicies.network.kubesphere.io   networksets.crd.projectcalico.org
namespaces                                       nginxes.gateway.kubesphere.io
networkpolicies.crd.projectcalico.org            nodes
networkpolicies.networking.k8s.io                notificationmanagers.notification.kubesphere.io
```



## 深入探究

> **01-docker的安装方式及位置**

```yaml
# 二进制的方式安装

# 安装路径
[root@ks-k8s-master-1 ~]# ls -l /usr/bin/docker*
-rwxr-xr-x 1 1000 1000 52883072 Jul 31  2021 /usr/bin/docker
-rwxr-xr-x 1 1000 1000 64758664 Jul 31  2021 /usr/bin/dockerd
-rwxr-xr-x 1 1000 1000   708616 Jul 31  2021 /usr/bin/docker-init
-rwxr-xr-x 1 1000 1000  2784649 Jul 31  2021 /usr/bin/docker-proxy
```

> **02-kubenetes如何安装的？装了什么组件？**

```yaml
# 二进制的方式安装

# 安装路径及文件
[root@ks-k8s-master-0 kubekey]# ll -R /usr/local/bin/ 
/usr/local/bin/:
total 289488
-rwxr-xr-x 1 root root  23847904 Apr  1 22:42 etcd
-rwxr-xr-x 1 root root  17620576 Apr  1 22:42 etcdctl
-rwxr-xr-x 1 root root  45113344 Apr  1 22:42 helm
-rwxr-xr-x 1 root root  44851200 Apr  1 22:42 kubeadm
-rwxr-xr-x 1 root root  46645248 Apr  1 22:42 kubectl
-rwxr-xr-x 1 root root 118353264 Apr  1 22:42 kubelet
drwxr-xr-x 2 kube root        71 Apr  1 22:44 kube-scripts

/usr/local/bin/kube-scripts:
total 16
-rwxr-xr-x 1 root root 1410 Apr  1 22:42 etcd-backup.sh
-rwxr-xr-x 1 root root 4877 Apr  1 22:39 initOS.sh
-rwxr-xr-x 1 root root 1120 Apr  1 22:44 k8s-certs-renew.sh
```



> **03-Kubekey在/etc/hosts文件中写了啥?**

```shell
[root@ks-k8s-master-0 kubekey]# cat /etc/hosts
# Ansible managed: modified on 2022-04-02 15:47:07 by root on zdevops-master
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.9.91   ks-k8s-master-0
192.168.9.92   ks-k8s-master-1
192.168.9.93   ks-k8s-master-2
# kubekey hosts BEGIN
192.168.9.91  ks-k8s-master-0.cluster.local ks-k8s-master-0
192.168.9.92  ks-k8s-master-1.cluster.local ks-k8s-master-1
192.168.9.93  ks-k8s-master-2.cluster.local ks-k8s-master-2
127.0.0.1 lb.kubesphere.local
# kubekey hosts END
```



> **04-KubeSphere服务相关的容器有哪些？**

```shell
[root@ks-k8s-master-0 kubekey]# docker ps -a  | grep kube
56ddba114660   6d6859d1a42a                                                "/bin/prometheus --w…"   34 minutes ago      Up 34 minutes                            k8s_prometheus_prometheus-k8s-0_kubesphere-monitoring-system_549a26f4-ef11-4715-9d9a-f15e500cf0d0_1
7ceb9c282c6a   kubesphere/prometheus-config-reloader                       "/bin/prometheus-con…"   34 minutes ago      Up 34 minutes                            k8s_config-reloader_prometheus-k8s-0_kubesphere-monitoring-system_549a26f4-ef11-4715-9d9a-f15e500cf0d0_0
3bca88f44041   kubesphere/prometheus-config-reloader                       "/bin/prometheus-con…"   34 minutes ago      Up 34 minutes                            k8s_config-reloader_alertmanager-main-0_kubesphere-monitoring-system_a947da3d-a4ae-41ea-8c6a-50229f7f65da_0
0390dded2f54   kubesphere/ks-controller-manager                            "controller-manager …"   34 minutes ago      Up 34 minutes                            k8s_ks-controller-manager_ks-controller-manager-b79cb6bf7-m582v_kubesphere-system_19f804ff-699b-420d-b93e-c8acb431f500_0
d23fdaaf857c   kubesphere/ks-apiserver                                     "ks-apiserver --logt…"   34 minutes ago      Up 34 minutes                            k8s_ks-apiserver_ks-apiserver-86d89d546d-zfmnr_kubesphere-system_dddd8ec7-d98b-4e1d-9844-d93654ce9221_0
937a52985d89   prom/prometheus                                             "/bin/prometheus --w…"   38 minutes ago      Exited (2) 38 minutes ago                k8s_prometheus_prometheus-k8s-0_kubesphere-monitoring-system_549a26f4-ef11-4715-9d9a-f15e500cf0d0_0
72ef61f1dbfb   prom/alertmanager                                           "/bin/alertmanager -…"   39 minutes ago      Up 39 minutes                            k8s_alertmanager_alertmanager-main-0_kubesphere-monitoring-system_a947da3d-a4ae-41ea-8c6a-50229f7f65da_0
de84038cf90f   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 40 minutes ago      Up 40 minutes                            k8s_POD_ks-controller-manager-b79cb6bf7-m582v_kubesphere-system_19f804ff-699b-420d-b93e-c8acb431f500_0
035e8a1c6d1d   kubesphere/kubectl                                          "entrypoint.sh"          41 minutes ago      Up 41 minutes                            k8s_kubectl_kubectl-admin-6667774bb-wlqwz_kubesphere-controls-system_478f1f6a-45b9-49ef-b31f-a5a89b324ede_0
7ffcec7b74c5   kubesphere/notification-manager-operator                    "/notification-manag…"   41 minutes ago      Up 41 minutes                            k8s_notification-manager-operator_notification-manager-operator-7d44854f54-86h4h_kubesphere-monitoring-system_6877a826-b2f9-4036-b390-f06f253aa66d_0
02db40d5d3c0   kubesphere/kube-rbac-proxy                                  "/usr/local/bin/kube…"   48 minutes ago      Up 48 minutes                            k8s_kube-rbac-proxy_node-exporter-6ppnb_kubesphere-monitoring-system_ec7e1bf5-8a50-4121-9613-8ef371878520_0
335e3c541d4b   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 49 minutes ago      Up 49 minutes                            k8s_POD_prometheus-k8s-0_kubesphere-monitoring-system_549a26f4-ef11-4715-9d9a-f15e500cf0d0_0
04744197c675   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 49 minutes ago      Up 49 minutes                            k8s_POD_alertmanager-main-0_kubesphere-monitoring-system_a947da3d-a4ae-41ea-8c6a-50229f7f65da_0
231dba777bb3   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 51 minutes ago      Up 51 minutes                            k8s_POD_kubectl-admin-6667774bb-wlqwz_kubesphere-controls-system_478f1f6a-45b9-49ef-b31f-a5a89b324ede_0
a5e1e46a07a5   kubesphere/kube-rbac-proxy                                  "/usr/local/bin/kube…"   54 minutes ago      Up 54 minutes                            k8s_kube-rbac-proxy_notification-manager-operator-7d44854f54-86h4h_kubesphere-monitoring-system_6877a826-b2f9-4036-b390-f06f253aa66d_0
4badbd31f179   prom/node-exporter                                          "/bin/node_exporter …"   57 minutes ago      Up 57 minutes                            k8s_node-exporter_node-exporter-6ppnb_kubesphere-monitoring-system_ec7e1bf5-8a50-4121-9613-8ef371878520_0
cb7bd4bb9810   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 57 minutes ago      Up 57 minutes                            k8s_POD_ks-apiserver-86d89d546d-zfmnr_kubesphere-system_dddd8ec7-d98b-4e1d-9844-d93654ce9221_0
984b880a1f7f   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 58 minutes ago      Up 58 minutes                            k8s_POD_notification-manager-operator-7d44854f54-86h4h_kubesphere-monitoring-system_6877a826-b2f9-4036-b390-f06f253aa66d_0
93dff8857db9   kubesphere/ks-console                                       "docker-entrypoint.s…"   58 minutes ago      Up 58 minutes                            k8s_ks-console_ks-console-65f4d44d88-fwzfc_kubesphere-system_4ab1e0a4-1072-4092-a845-9aa3941553ee_0
c366e1f3a0e2   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 59 minutes ago      Up 59 minutes                            k8s_POD_node-exporter-6ppnb_kubesphere-monitoring-system_ec7e1bf5-8a50-4121-9613-8ef371878520_0
26699c22f650   d38a50cc542f                                                "redis-sentinel /dat…"   About an hour ago   Up About an hour                         k8s_sentinel_redis-ha-server-2_kubesphere-system_df05d5d6-630b-45a9-a015-97c30823de53_0
364c2c3d8823   d38a50cc542f                                                "redis-server /data/…"   About an hour ago   Up About an hour                         k8s_redis_redis-ha-server-2_kubesphere-system_df05d5d6-630b-45a9-a015-97c30823de53_0
436bf5c496fc   redis                                                       "sh /readonly-config…"   About an hour ago   Exited (0) About an hour ago             k8s_config-init_redis-ha-server-2_kubesphere-system_df05d5d6-630b-45a9-a015-97c30823de53_0
73012183fd42   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 About an hour ago   Up About an hour                         k8s_POD_ks-console-65f4d44d88-fwzfc_kubesphere-system_4ab1e0a4-1072-4092-a845-9aa3941553ee_0
ca75bf407ee9   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 About an hour ago   Up About an hour                         k8s_POD_redis-ha-server-2_kubesphere-system_df05d5d6-630b-45a9-a015-97c30823de53_0
a95f9a97136c   96ae905008ca                                                "docker-entrypoint.s…"   About an hour ago   Up About an hour                         k8s_haproxy_redis-ha-haproxy-868fdbddd4-dxvhm_kubesphere-system_ef25d617-6889-4034-94cb-252bb29060f2_0
0bb2f6953698   haproxy                                                     "sh /readonly/haprox…"   About an hour ago   Exited (0) About an hour ago             k8s_config-init_redis-ha-haproxy-868fdbddd4-dxvhm_kubesphere-system_ef25d617-6889-4034-94cb-252bb29060f2_0
0838d2f5444e   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 About an hour ago   Up About an hour                         k8s_POD_redis-ha-haproxy-868fdbddd4-dxvhm_kubesphere-system_ef25d617-6889-4034-94cb-252bb29060f2_0
3cd3b1e17b27   296a6d5035e2                                                "/coredns -conf /etc…"   About an hour ago   Up About an hour                         k8s_coredns_coredns-5495dd7c88-7vjlg_kube-system_4d096853-74fd-4663-8a21-c65890b68ba3_0
331ae9233ef5   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 About an hour ago   Up About an hour                         k8s_POD_coredns-5495dd7c88-7vjlg_kube-system_4d096853-74fd-4663-8a21-c65890b68ba3_0
3b054ee84a01   76ba70f4748f                                                "/usr/bin/kube-contr…"   About an hour ago   Up About an hour                         k8s_calico-kube-controllers_calico-kube-controllers-75ddb95444-66rw7_kube-system_0c1002c4-79c1-41d7-8568-a64480282f85_0
23f172aa5773   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 About an hour ago   Up About an hour                         k8s_POD_calico-kube-controllers-75ddb95444-66rw7_kube-system_0c1002c4-79c1-41d7-8568-a64480282f85_0
b97121f67a86   5ef66b403f4f                                                "start_runit"            About an hour ago   Up About an hour                         k8s_calico-node_calico-node-xbrv2_kube-system_8bec7710-5b02-4578-88e4-480b922f9f0b_0
6272abcdf59d   5991877ebc11                                                "/usr/local/bin/flex…"   About an hour ago   Exited (0) About an hour ago             k8s_flexvol-driver_calico-node-xbrv2_kube-system_8bec7710-5b02-4578-88e4-480b922f9f0b_0
a8bfccda4a11   4945b742b8e6                                                "/opt/cni/bin/install"   About an hour ago   Exited (0) About an hour ago             k8s_install-cni_calico-node-xbrv2_kube-system_8bec7710-5b02-4578-88e4-480b922f9f0b_0
e8cf1cc8bcb6   e08abd2be730                                                "/usr/local/bin/kube…"   About an hour ago   Up About an hour                         k8s_kube-proxy_kube-proxy-7hnl8_kube-system_c26e2c08-73e7-4c57-9517-d217a5f880f6_0
84ae94f60fbf   4945b742b8e6                                                "/opt/cni/bin/calico…"   About an hour ago   Exited (0) About an hour ago             k8s_upgrade-ipam_calico-node-xbrv2_kube-system_8bec7710-5b02-4578-88e4-480b922f9f0b_0
bfed559ac954   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 About an hour ago   Up About an hour                         k8s_POD_kube-proxy-7hnl8_kube-system_c26e2c08-73e7-4c57-9517-d217a5f880f6_0
bc425f4adf1d   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 About an hour ago   Up About an hour                         k8s_POD_calico-node-xbrv2_kube-system_8bec7710-5b02-4578-88e4-480b922f9f0b_0
260286005d9c   5340ba194ec9                                                "/node-cache -locali…"   About an hour ago   Up About an hour                         k8s_node-cache_nodelocaldns-w9v4t_kube-system_72389909-2262-4807-8adc-dc8a3326c7b9_0
90e2561dc91f   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 About an hour ago   Up About an hour                         k8s_POD_nodelocaldns-w9v4t_kube-system_72389909-2262-4807-8adc-dc8a3326c7b9_0
4a74d116f0c5   7b2ac941d4c3                                                "kube-apiserver --ad…"   About an hour ago   Up About an hour                         k8s_kube-apiserver_kube-apiserver-ks-k8s-master-0_kube-system_a3a95f029d9aa45f7c1c87b8542a6e7c_0
ecfca7a100a0   184ef4d127b4                                                "kube-controller-man…"   About an hour ago   Up About an hour                         k8s_kube-controller-manager_kube-controller-manager-ks-k8s-master-0_kube-system_b5a94fe882e6fcf23fa73eca1735fbd8_0
ff5a86654de9   8e60ea3644d6                                                "kube-scheduler --au…"   About an hour ago   Up About an hour                         k8s_kube-scheduler_kube-scheduler-ks-k8s-master-0_kube-system_86823db35b1fcd1d3c94e2622324bafa_0
1b5afb3c81c4   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 About an hour ago   Up About an hour                         k8s_POD_kube-apiserver-ks-k8s-master-0_kube-system_a3a95f029d9aa45f7c1c87b8542a6e7c_0
50693fd0c9d1   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 About an hour ago   Up About an hour                         k8s_POD_kube-scheduler-ks-k8s-master-0_kube-system_86823db35b1fcd1d3c94e2622324bafa_0
9d5c77fdae7e   registry.cn-beijing.aliyuncs.com/kubesphereio/pause:3.4.1   "/pause"                 About an hour ago   Up About an hour                         k8s_POD_kube-controller-manager-ks-k8s-master-0_kube-system_b5a94fe882e6fcf23fa73eca1735fbd8_0
[root@ks-k8s-master-0 kubekey]# docker ps -a  | grep kube | wc -l
47
[root@ks-k8s-master-0 kubekey]# docker ps -a  | grep kube | grep Up |wc -l
41
[root@ks-k8s-master-0 kubekey]# docker ps -a  | grep kube | grep Exited |wc -l
6
```



## 常见问题

> **问题1**

```yaml
# 报错信息
[root@ks-k8s-master-0 kubekey]# ./kk create cluster -f config-sample.yaml
You cannot set up the internal load balancer and the LB address at the same time.

# 解决方案
address: "192.168.9.90" 参数改为 address: ""
```

> **问题2**

```shell
#  报错信息
22:18:52 CST message: [LocalHost]
Failed to download kubeadm binary: curl -L -o /root/kubekey/kubekey/kube/v1.22.8/amd64/kubeadm https://storage.googleapis.com/kubernetes-release/release/v1.22.8/bin/linux/amd64/kubeadm error: No SHA256 found for kubeadm. v1.22.8 is not supported. 
22:18:52 CST failed: [LocalHost]
error: Pipeline[CreateClusterPipeline] execute failed: Module[NodeBinariesModule] exec failed: 
failed: [LocalHost] [DownloadBinaries] exec failed after 1 retires: Failed to download kubeadm binary: curl -L -o /root/kubekey/kubekey/kube/v1.22.8/amd64/kubeadm https://storage.googleapis.com/kubernetes-release/release/v1.22.8/bin/linux/amd64/kubeadm error: No SHA256 found for kubeadm. v1.22.8 is not supported. 

# 解决方案
目前3.2.1的版本不支持k8s v1.22.8，利用kk version --show-supported-k8s查看支持的版本，更换支持的版本，重新执行安装命令。
```



## 尾部



> **参考文档**

- [使用 KubeKey 内置 HAproxy 创建高可用集群]: https://kubesphere.com.cn/docs/installing-on-linux/high-availability-configurations/internal-ha-configuration/


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
