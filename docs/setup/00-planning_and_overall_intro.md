## 00-集群规划和基础参数设定

### HA architecture

![ha-2x](../../pics/ha-2x.gif)

- 注意1：请确保各节点时区设置一致、时间同步。 如果你的环境没有提供NTP 时间同步，推荐集成安装[chrony](../guide/chrony.md)
- 注意2：在公有云上创建多主集群，请结合阅读[在公有云上部署 kubeasz](kubeasz_on_public_cloud.md)
- 注意3：建议操作系统升级到新的稳定内核，请结合阅读[内核升级文档](../guide/kernel_upgrade.md)

## 高可用集群所需节点配置如下
|角色|数量|描述|
|:-|:-|:-|
|管理节点|1|运行ansible/easzctl脚本，可以复用master，建议使用独立节点（1c1g）|
|etcd节点|3|注意etcd集群需要1,3,5,7...奇数个节点，一般复用master节点|
|master节点|2|高可用集群至少2个master节点|
|node节点|3|运行应用负载的节点，可根据需要提升机器配置/增加节点数|

在 kubeasz 2x 版本，多节点高可用集群安装可以使用2种方式

- 1.先部署单节点集群 [AllinOne部署](quickStart.md)，然后通过 [节点添加](../op/op-index.md) 扩容成高可用集群
- 2.按照如下步骤先规划准备，直接安装多节点高可用集群

## 部署步骤

按照`example/hosts.multi-
node`示例的节点配置，准备4台虚拟机，搭建一个多主高可用集群。

### 1.基础系统配置

+ 推荐内存2G/硬盘30G以上
+ 最小化安装`Ubuntu
16.04 server`或者`CentOS 7 Minimal`
+ 配置基础网络、更新源、SSH登陆等

### 2.在每个节点安装依赖工具

Ubuntu
16.04 请执行以下脚本:

```{.python .input}
# 文档中脚本默认均以root用户执行
apt-get update && apt-get upgrade -y && apt-get dist-upgrade -y
# 安装python2
apt-get install python2.7
# Ubuntu16.04可能需要配置以下软连接
ln -s /usr/bin/python2.7 /usr/bin/python
```

CentOS 7 请执行以下脚本：

```{.python .input}
# 文档中脚本默认均以root用户执行
yum update
# 安装python
yum install python -y
```

### 3.在ansible控制端安装及准备ansible

- 3.1 pip 安装 ansible（如果 Ubuntu pip报错，请看[附录](00-planning_and_overall_intro.md#Appendix)）

```{.python .input}
# Ubuntu 16.04
apt-get install git python-pip -y
# CentOS 7
yum install git python-pip -y
# pip安装ansible(国内如果安装太慢可以直接用pip阿里云加速)
#pip install pip --upgrade
#pip install ansible==2.6.12
pip install pip --upgrade -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
pip install ansible==2.6.12 -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
```

- 3.2 在ansible控制端配置免密码登陆

```{.python .input}
# 更安全 Ed25519 算法
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
# 或者传统 RSA 算法
ssh-keygen -t rsa -b 2048 -N '' -f ~/.ssh/id_rsa

ssh-copy-id $IPs #$IPs为所有节点地址包括自身，按照提示输入yes 和root密码
```

### 4.在ansible控制端编排k8s安装

- 4.0 下载项目源码
- 4.1 下载二进制文件
- 4.2 下载离线docker镜像

推荐使用
easzup 脚本下载 4.0/4.1/4.2
所需文件；运行成功后，所有文件（kubeasz代码、二进制、离线镜像）均已整理好放入目录`/etc/ansilbe`

```{.python .input}
# 下载工具脚本easzup
$ curl -C- -fLO --retry 3 https://github.com/easzlab/kubeasz/releases/download/${release}/easzup
$ chmod +x ./easzup
# 使用工具脚本下载
$ ./easzup -D
```

- 4.3 配置集群参数
  - 4.3.1 必要配置：`cd /etc/ansible && cp example/hosts.multi-node
hosts`, 然后实际情况修改此hosts文件
  - 4.3.2 可选配置，初次使用可以不做修改，详见[配置指南](config_guide.md)
  -
4.3.3 验证ansible 安装：`ansible all -m ping` 正常能看到节点返回 SUCCESS

- 4.4 开始安装
如果你对集群安装流程不熟悉，请阅读项目首页 **安装步骤** 讲解后分步安装，并对 **每步都进行验证**

```bash
# 分步安装
ansible-playbook 01.prepare.yml
ansible-playbook 02.etcd.yml
ansible-playbook 03.docker.yml
ansible-playbook 04.kube-master.yml
ansible-playbook 05.kube-node.yml
ansible-playbook 06.network.yml # TODO:有错误
ansible-playbook 07.cluster-addon.yml
# 一步安装
#ansible-playbook 90.setup.yml
```

2019.07.12

执行指令`ansible-playbook 04.kube-master.yml`时，出现错误：

```bash
...
TASK
[kube-node : 轮询等待kubelet启动]
************************************************************************************************
changed: [192.168.20.3]
FAILED - RETRYING: 轮询等待kubelet启动 (8 retries left).
FAILED - RETRYING: 轮询等待kubelet启动 (7 retries left).
FAILED - RETRYING:
轮询等待kubelet启动 (6 retries left).
FAILED - RETRYING: 轮询等待kubelet启动 (5 retries
left).
FAILED - RETRYING: 轮询等待kubelet启动 (4 retries left).
FAILED - RETRYING:
轮询等待kubelet启动 (3 retries left).
FAILED - RETRYING: 轮询等待kubelet启动 (2 retries
left).
FAILED - RETRYING: 轮询等待kubelet启动 (1 retries left).
fatal:
[192.168.20.201]: FAILED! => {"attempts": 8, "changed": true, "cmd": "systemctl
status kubelet.service|grep Active", "delta": "0:00:00.004757", "end":
"2019-07-12 19:08:20.186398", "rc": 0, "start": "2019-07-12 19:08:20.181641",
"stderr": "", "stderr_lines": [], "stdout": "   Active: activating (auto-
restart) (Result: exit-code) since Fri 2019-07-12 19:08:18 CST; 1s ago",
"stdout_lines": ["   Active: activating (auto-restart) (Result: exit-code) since
Fri 2019-07-12 19:08:18 CST; 1s ago"]}

TASK [kube-node : 轮询等待node达到Ready状态]
********************************************************************************************
changed: [192.168.20.3]

TASK [kube-node : 设置node节点role]
*************************************************************************************************
changed: [192.168.20.3]

TASK [Making master nodes SchedulingDisabled]
***********************************************************************************
changed: [192.168.20.3]

TASK [Setting master role name]
*************************************************************************************************
changed: [192.168.20.3]

PLAY RECAP
**********************************************************************************************************************
192.168.20.201             : ok=35   changed=17   unreachable=0    failed=1
192.168.20.3               : ok=41   changed=23   unreachable=0    failed=0
```

2019.07.12

执行指令`[root@wfh_deploy ansible]# ansible-playbook 05.kube-
node.yml`出现下面错误：

```bash
TASK [kube-node : 开启kube-proxy 服务]
********************************************************************************************************
changed: [192.168.20.202]
changed: [192.168.20.203]
FAILED - RETRYING:
轮询等待kubelet启动 (8 retries left).

TASK [kube-node : 轮询等待kubelet启动]
**********************************************************************************************************
changed: [192.168.20.203]
FAILED - RETRYING: 轮询等待kubelet启动 (7 retries left).
FAILED - RETRYING: 轮询等待kubelet启动 (6 retries left).
FAILED - RETRYING:
轮询等待kubelet启动 (5 retries left).
FAILED - RETRYING: 轮询等待kubelet启动 (4 retries
left).
FAILED - RETRYING: 轮询等待kubelet启动 (3 retries left).
FAILED - RETRYING:
轮询等待kubelet启动 (2 retries left).
FAILED - RETRYING: 轮询等待kubelet启动 (1 retries
left).
fatal: [192.168.20.202]: FAILED! => {"attempts": 8, "changed": true,
"cmd": "systemctl status kubelet.service|grep Active", "delta":
"0:00:00.004670", "end": "2019-07-12 19:18:46.340741", "rc": 0, "start":
"2019-07-12 19:18:46.336071", "stderr": "", "stderr_lines": [], "stdout": "
Active: activating (auto-restart) (Result: exit-code) since Fri 2019-07-12
19:18:44 CST; 1s ago", "stdout_lines": ["   Active: activating (auto-restart)
(Result: exit-code) since Fri 2019-07-12 19:18:44 CST; 1s ago"]}

TASK [kube-node : 轮询等待node达到Ready状态]
******************************************************************************************************
changed: [192.168.20.203]

TASK [kube-node : 设置node节点role]
***********************************************************************************************************
changed: [192.168.20.203]

PLAY RECAP
********************************************************************************************************************************
192.168.20.202             : ok=33   changed=28   unreachable=0    failed=1
192.168.20.203             : ok=35   changed=30   unreachable=0    failed=0
```

**问题原因及解决办法：**

运行`journalctl -xefu kubelet` 命令查看systemd日志才发现，真正的错误是：

连不上unix://var/run/docker服务

根本原因是 docker服务没有启动，启动 docker服务后，恢复正常。
```bash
ssh <remote_ip> "systemctl enable docker;systemctl start docker"
```

+ [可选]对集群所有节点进行操作系统层面的安全加固 `ansible-playbook roles/os-harden/os-
harden.yml`，详情请参考[os-harden项目](https://github.com/dev-sec/ansible-os-hardening)
## Appendix

- Ubuntu 1604 安装 ansible 如果出现以下错误

```{.python .input}
Traceback (most recent call last):
  File "/usr/bin/pip", line 9, in <module>
    from pip import main
ImportError: cannot import name main
```

将`/usr/bin/pip`做以下修改即可

```{.python .input}
#原代码
from pip import main
if __name__ == '__main__':
    sys.exit(main())

#修改后
from pip import __main__
if __name__ == '__main__':
    sys.exit(__main__._main())
```

[后一篇](01-CA_and_prerequisite.md)
