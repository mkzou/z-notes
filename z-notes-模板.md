# z-notes-模板

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

Xxx

> **本文知识量**

- 阅读时长：x分
- 行：x
- 单词：x
- 字符：x
- 图片：x张

> **本文知识点**

- 定级：**入门级**
- xxxx

> **演示服务器配置**

|      主机名      |      IP      | CPU  | 内存 | 系统盘 | 数据盘  |                 用途                  |
| :--------------: | :----------: | :--: | :--: | :----: | :-----: | :-----------------------------------: |
|  zdeops-master   | 192.168.9.9  |  2   |  4   |   40   |   200   |          Ansible运维控制节点          |
| ks-k8s-master-0  | 192.168.9.91 |  4   |  16  |   40   | 200+200 | KubeSphere/k8s-master/k8s-worker/Ceph |
| ks-k8s-master-1  | 192.168.9.92 |  4   |  16  |   40   | 200+200 | KubeSphere/k8s-master/k8s-worker/Ceph |
| ks-k8s-master-2  | 192.168.9.93 |  4   |  16  |   40   | 200+200 | KubeSphere/k8s-master/k8s-worker/Ceph |
| glusterfs-node-0 | 192.168.9.95 |  2   |  8   |   40   |   200   |               GlusterFS               |
| glusterfs-node-1 | 192.168.9.96 |  2   |  8   |   40   |   200   |               GlusterFS               |
| glusterfs-node-2 | 192.168.9.97 |  2   |  8   |   40   |   200   |               GlusterFS               |
|      harbor      | 192.168.9.89 |  2   |  8   |   40   |   200   |                Harbor                 |
|       合计       |      8       |  22  |  84  |  320   |  2200   |                                       |

> **演示环境涉及软件版本信息**

- 操作系统：**CentOS-7.9-x86_64**

- Ansible：**2.8.20**

- Harbor：**2.5.1**

  

## 2. Ansible配置

## 3. 主题正文

## 4. 常见问题

## 5. 总结

> **参考文档**

> **Get文档**

- Github https://github.com/devops/z-notes
- Gitee https://gitee.com/zdevops/z-notes

> **Get代码**

- Github https://github.com/devops/ansible-zdevops
- Gitee https://gitee.com/zdevops/ansible-zdevops

> **B 站**

- [老 Z 手记](https://space.bilibili.com/1039301316) https://space.bilibili.com/1039301316

> **版权声明** 

- 所有内容均属于原创，整理不易，感谢收藏，转载请标明出处。

> **About Me**

- 昵称：老Z
- 坐标：山东济南
- 职业：运维架构师/高级运维工程师=**运维**
- 微信：zdevops
- 关注的领域：云计算/云原生技术运维，自动化运维
- 技能标签：OpenStack、Ansible、K8S、Python、Go、CNCF
