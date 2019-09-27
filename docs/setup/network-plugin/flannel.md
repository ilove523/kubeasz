## 06-安装flannel网络组件.md

本项目提供多种网络插件可选，如果需要安装flannel，请在/etc/ansible/hosts文件中设置变量
`CLUSTER_NETWORK="flannel"`，更多设置请查看`roles/flannel/defaults/main.yml`
`Flannel`是最早应用到k8s集群的网络插件之一，简单高效，且提供多个后端`backend`模式供选择；本文介绍以`DaemonSet
Pod`方式集成到k8s集群，需要在所有master节点和node节点安装。

```{.python .input .text}
roles/flannel/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── kube-flannel.yaml.j2
```

请在另外窗口打开`roles/flannel/tasks/main.yml`文件，对照看以下讲解内容。

### 下载基础cni 插件

请到CNI插件最新[release](https://github.com/containernetworking/plugins/releases)页面下载[cni-v0.6.0.tgz](https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-v0.6.0.tgz)，解压后里面有很多插件，选择如下几个复制到项目`bin`目录下

- flannel用到的插件
  - bridge
  - flannel
  - host-local
  - loopback
  - portmap

Flannel CNI 插件的配置文件可以包含多个`plugin` 或由其调用其他`plugin`；`Flannel DaemonSet Pod`运行以后会生成`/run/flannel/subnet.env `文件，例如：

```{.python .input}
FLANNEL_NETWORK=10.1.0.0/16
FLANNEL_SUBNET=10.1.17.1/24
FLANNEL_MTU=1472
FLANNEL_IPMASQ=true
```

然后它利用这个文件信息去配置和调用`bridge`插件来生成容器网络，调用`host-local`来管理`IP`地址，例如：

```{.python .input}
{
	"name": "mynet",
	"type": "bridge",
	"mtu": 1472,
	"ipMasq": false,
	"isGateway": true,
	"ipam": {
		"type": "host-local",
		"subnet": "10.1.17.0/24"
	}
}
```

- 更多相关介绍请阅读：
  - [flannel kubernetes集成](https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md)
  - [flannel cni插件](https://github.com/containernetworking/plugins/tree/master/plugins/meta/flannel)
  - [更多 cni 插件](https://github.com/containernetworking/plugins)

### 准备`Flannel DaemonSet` yaml配置文件

请阅读 `roles/flannel/templates/kube-flannel.yaml.j2` 内容，注意：
+ 注意：本安装方式，flannel 通过 apiserver 接口读取 podCidr 信息，详见https://github.com/coreos/flannel/issues/847 ；因此想要修改节点pod网段掩码，请前往`roles/kube-
master/defaults/main.yml`设置
+ 配置相关RBAC 权限和 `service account`
+ 配置`ConfigMap`包含 CNI配置和 flannel配置(指定backend等)，和`hosts`文件中相关设置对应
+ `DaemonSet Pod`包含两个容器，一个容器运行flannel本身，另一个init容器部署cni 配置文件
+ 为方便国内加速使用镜像`jmgao1983/flannel:v0.10.0-amd64` (官方镜像在docker-hub上的转存)
+ 特别注意：如果服务器是多网卡（例如vagrant环境），则需要在`roles/flannel/templates/kube-flannel.yaml.j2
`中增加指定环境变量，详见 [kubernetes ISSUE 39701](https://github.com/kubernetes/kubernetes/issues/39701)

```yaml
      ...
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: KUBERNETES_SERVICE_HOST   # 指定apiserver的主机地址
          value: {{ MASTER_IP }}
        - name: KUBERNETES_SERVICE_PORT   # 指定apiserver的服务端口
          value: {{ KUBE_APISERVER.split(':')[2] }}
       ...
```

### 安装 flannel网络

+ 安装之前必须确保kube-master和kube-node节点已经成功部署
+ 只需要在任意装有kubectl客户端的节点运行 kubectl create安装即可
+ 等待15s后(视网络拉取相关镜像速度)，flannel网络插件安装完成，删除之前kube-node安装时默认cni网络配置

### 验证flannel网络
执行flannel安装成功后可以验证如下：(需要等待镜像下载完成，有时候即便上一步已经配置了docker国内加速，还是可能比较慢，请确认以下容器运行起来以后，再执行后续验证步骤)

```bash
kubectl get pod --all-namespaces
```
输出结果：
```ini
NAMESPACE     NAME                    READY     STATUS    RESTARTS   AGE
kube-system   kube-flannel-ds-m8mzm   1/1       Running   0          3m
kube-system   kube-flannel-ds-mnj6j   1/1       Running   0          3m
kube-system   kube-flannel-ds-mxn6k   1/1       Running   0          3m
```

在集群创建几个测试pod:  `kubectl run test --image=busybox --replicas=3 sleep 30000`

```{.python .input}
# kubectl get pod --all-namespaces -o wide|head -n 4
NAMESPACE     NAME                    READY     STATUS    RESTARTS   AGE       IP             NODE
default       busy-5956b54c8b-ld4gb   1/1       Running   0          9m        172.20.2.7     192.168.1.1
default       busy-5956b54c8b-lj9l9   1/1       Running   0          9m        172.20.1.5     192.168.1.2
default       busy-5956b54c8b-wwpkz   1/1       Running   0          9m        172.20.0.6     192.168.1.3

# 查看路由
# ip route
default via 192.168.1.254 dev ens3 onlink
192.168.1.0/24 dev ens3  proto kernel  scope link  src 192.168.1.1
172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1 linkdown
172.20.0.0/24 via 192.168.1.3 dev ens3
172.20.1.0/24 via 192.168.1.2 dev ens3
172.20.2.0/24 dev cni0  proto kernel  scope link  src 172.20.2.1
```

在各节点上分别 ping 这三个POD IP地址，确保能通：

```{.python .input}
ping 172.20.2.7
ping 172.20.1.5
ping 172.20.0.6
```

### 错误修复日志
#### 由于被控服务器缺少默认路由导致 flannel 不能正常运行
```ini
[root@wfh_node01 ~]# kubectl get pods -n kube-system -o wide
NAME                                          READY   STATUS             RESTARTS   AGE   IP               NODE             NOMINATED NODE   READINESS GATES
coredns-797455887b-5866s                      1/1     Running            0          59m   172.20.3.3       192.168.20.203   <none>           <none>
coredns-797455887b-vldkf                      1/1     Running            0          66m   172.20.4.2       192.168.20.204   <none>           <none>
heapster-5f848f54bc-fxkkw                     1/1     Running            0          65m   172.20.3.2       192.168.20.203   <none>           <none>
kube-flannel-ds-amd64-9qwhk                   1/1     Running            0          67m   192.168.20.202   192.168.20.202   <none>           <none>
kube-flannel-ds-amd64-dkj8x                   0/1     CrashLoopBackOff   15         33m   192.168.20.206   192.168.20.206   <none>           <none>
kube-flannel-ds-amd64-drhrq                   1/1     Running            0          67m   192.168.20.203   192.168.20.203   <none>           <none>
kube-flannel-ds-amd64-dvsj5                   1/1     Running            0          67m   192.168.20.204   192.168.20.204   <none>           <none>
kube-flannel-ds-amd64-h6psv                   1/1     Running            0          67m   192.168.20.205   192.168.20.205   <none>           <none>
kube-flannel-ds-amd64-vf5zx                   1/1     Running            0          67m   192.168.20.201   192.168.20.201   <none>           <none>
kubernetes-dashboard-5c7687cf8-mb5dl          1/1     Running            0          55m   172.20.3.4       192.168.20.203   <none>           <none>
metrics-server-85c7b8c8c4-pg4wf               1/1     Running            0          66m   172.20.4.3       192.168.20.204   <none>           <none>
traefik-ingress-controller-766dbfdddd-97sjn   1/1     Running            0          65m   172.20.5.2       192.168.20.205   <none>           <none>
[root@wfh_node01 ~]# kubectl logs -n kube-system pods/kube-flannel-ds-amd64-dkj8x
I0924 05:04:48.270577       1 main.go:514] Determining IP address of default interface
E0924 05:04:48.271181       1 main.go:202] Failed to find any valid interface to use: failed to get default interface: Unable to find default route
```

解决办法：
需在出现问题的被控服务端添加一条默认路由
```ini
; 增加一条默认路由 for CentOS 7：
[root@wfh_node06 ~]# cat /etc/sysconfig/network-scripts/route-ens33
# 0.0.0.0/0 via 103.123.213.9 dev ens33
ADDRESS0=0.0.0.0
NETMASK0=0.0.0.0
GATEWAY0=103.123.213.9
```
```ini
; 重启网络服务
[root@wfh_node06 ~]# systemctl restart network
```
```ini
; 查看默认路由是否成功
[root@wfh_node06 ~]# ip route
default via 103.123.213.9 dev ens33
103.123.213.8/29 dev ens33 proto kernel scope link src 103.123.213.13
169.254.0.0/16 dev ens33 scope link metric 1003
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
192.168.20.0/24 dev ens192 proto kernel scope link src 192.168.20.206 metric 100
```
> 看到`default via 103.123.213.9 dev ens33`表示操作成功。

```ini
; 通过日志也能看到修复成功：
[root@wfh_node01 ~]# kubectl logs -n kube-system pods/kube-flannel-ds-amd64-dkj8x
I0924 05:15:04.168683       1 main.go:514] Determining IP address of default interface
I0924 05:15:04.169749       1 main.go:527] Using interface with name ens33 and address 103.123.213.13
I0924 05:15:04.169815       1 main.go:544] Defaulting external address to interface address (103.123.213.13)
I0924 05:15:04.462688       1 kube.go:126] Waiting 10m0s for node controller to sync
I0924 05:15:04.462881       1 kube.go:309] Starting kube subnet manager
I0924 05:15:05.463844       1 kube.go:133] Node controller sync successful
I0924 05:15:05.463933       1 main.go:244] Created subnet manager: Kubernetes Subnet Manager - 192.168.20.206
I0924 05:15:05.463958       1 main.go:247] Installing signal handlers
I0924 05:15:05.464407       1 main.go:386] Found network config - Backend type: vxlan
I0924 05:15:05.464620       1 vxlan.go:120] VXLAN config: VNI=1 Port=0 GBP=false DirectRouting=false
I0924 05:15:05.601963       1 main.go:351] Current network or subnet (172.20.0.0/16, 172.20.2.0/24) is not equal to previous one (0.0.0.0/0, 0.0.0.0/0), trying to recycle old iptables rules
I0924 05:15:05.783287       1 iptables.go:167] Deleting iptables rule: -s 0.0.0.0/0 -d 0.0.0.0/0 -j RETURN
I0924 05:15:05.803995       1 iptables.go:167] Deleting iptables rule: -s 0.0.0.0/0 ! -d 224.0.0.0/4 -j MASQUERADE --random-fully
I0924 05:15:05.878966       1 iptables.go:167] Deleting iptables rule: ! -s 0.0.0.0/0 -d 0.0.0.0/0 -j RETURN
I0924 05:15:05.881299       1 iptables.go:167] Deleting iptables rule: ! -s 0.0.0.0/0 -d 0.0.0.0/0 -j MASQUERADE --random-fully
I0924 05:15:05.966259       1 main.go:317] Wrote subnet file to /run/flannel/subnet.env
I0924 05:15:05.966319       1 main.go:321] Running backend.
I0924 05:15:05.966408       1 main.go:339] Waiting for all goroutines to exit
I0924 05:15:05.966483       1 vxlan_network.go:60] watching for new subnet leases
I0924 05:15:06.162571       1 iptables.go:145] Some iptables rules are missing; deleting and recreating rules
I0924 05:15:06.162633       1 iptables.go:167] Deleting iptables rule: -s 172.20.0.0/16 -d 172.20.0.0/16 -j RETURN
I0924 05:15:06.164571       1 iptables.go:145] Some iptables rules are missing; deleting and recreating rules
I0924 05:15:06.164645       1 iptables.go:167] Deleting iptables rule: -s 172.20.0.0/16 -j ACCEPT
I0924 05:15:06.166939       1 iptables.go:167] Deleting iptables rule: -d 172.20.0.0/16 -j ACCEPT
I0924 05:15:06.262962       1 iptables.go:155] Adding iptables rule: -s 172.20.0.0/16 -j ACCEPT
I0924 05:15:06.263688       1 iptables.go:167] Deleting iptables rule: -s 172.20.0.0/16 ! -d 224.0.0.0/4 -j MASQUERADE --random-fully
I0924 05:15:06.364628       1 iptables.go:167] Deleting iptables rule: ! -s 172.20.0.0/16 -d 172.20.2.0/24 -j RETURN
I0924 05:15:06.462924       1 iptables.go:155] Adding iptables rule: -d 172.20.0.0/16 -j ACCEPT
I0924 05:15:06.463527       1 iptables.go:167] Deleting iptables rule: ! -s 172.20.0.0/16 -d 172.20.0.0/16 -j MASQUERADE --random-fully
I0924 05:15:06.563332       1 iptables.go:155] Adding iptables rule: -s 172.20.0.0/16 -d 172.20.0.0/16 -j RETURN
I0924 05:15:06.568339       1 iptables.go:155] Adding iptables rule: -s 172.20.0.0/16 ! -d 224.0.0.0/4 -j MASQUERADE --random-fully
I0924 05:15:06.666810       1 iptables.go:155] Adding iptables rule: ! -s 172.20.0.0/16 -d 172.20.2.0/24 -j RETURN
I0924 05:15:06.764449       1 iptables.go:155] Adding iptables rule: ! -s 172.20.0.0/16 -d 172.20.0.0/16 -j MASQUERADE --random-fully
```

```ini
[root@wfh_deploy ansible]# kubectl get pods -n kube-system -o wide
NAME                                          READY   STATUS    RESTARTS   AGE   IP               NODE             NOMINATED NODE   READINESS GATES
coredns-797455887b-5866s                      1/1     Running   0          71m   172.20.3.3       192.168.20.203   <none>           <none>
coredns-797455887b-vldkf                      1/1     Running   0          79m   172.20.4.2       192.168.20.204   <none>           <none>
heapster-5f848f54bc-fxkkw                     1/1     Running   0          78m   172.20.3.2       192.168.20.203   <none>           <none>
kube-flannel-ds-amd64-9qwhk                   1/1     Running   0          79m   192.168.20.202   192.168.20.202   <none>           <none>
kube-flannel-ds-amd64-dkj8x                   1/1     Running   17         46m   192.168.20.206   192.168.20.206   <none>           <none>
kube-flannel-ds-amd64-drhrq                   1/1     Running   0          79m   192.168.20.203   192.168.20.203   <none>           <none>
kube-flannel-ds-amd64-dvsj5                   1/1     Running   0          79m   192.168.20.204   192.168.20.204   <none>           <none>
kube-flannel-ds-amd64-h6psv                   1/1     Running   0          79m   192.168.20.205   192.168.20.205   <none>           <none>
kube-flannel-ds-amd64-vf5zx                   1/1     Running   0          79m   192.168.20.201   192.168.20.201   <none>           <none>
kubernetes-dashboard-5c7687cf8-mb5dl          1/1     Running   0          68m   172.20.3.4       192.168.20.203   <none>           <none>
metrics-server-85c7b8c8c4-pg4wf               1/1     Running   0          79m   172.20.4.3       192.168.20.204   <none>           <none>
traefik-ingress-controller-766dbfdddd-97sjn   1/1     Running   0          78m   172.20.5.2       192.168.20.205   <none>           <none>
```

### [kubernetes使用flannel网络插件服务状态显示CrashLoopBackOff](https://www.2cto.com/net/201908/815119.html)
