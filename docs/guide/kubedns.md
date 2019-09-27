## 部署集群 DNS

DNS 是 k8s 集群首先需要部署的，集群中的其他 pods 使用它提供域名解析服务；主要可以解析 `集群服务名 SVC` 和 `Pod hostname`；目前 k8s v1.9+ 版本可以有两个选择：`kube-dns` 和 `coredns`（推荐），可以选择其中一个部署安装。
> _`kube-dns` 已经停止更新！_

_**特别说明：以下内容只针对 coredns。**_

### 部署 coredns 前言

+ 配置文件参考：
    - https://github.com/kubernetes/kubernetes
        - 重点参考内容 `cluster/addons/dns/coredns`
    - https://github.com/coredns/deployment
        - 重点参考内容 `kubernetes/`

+ 目前 kubeasz 已经自动集成安装 dns 组件，也可单独部署(`ansible-playbook /etc/ansible/07.cluster-addon-coredns.yaml`)，配置模板位于`roles/coredns/templates/`目录。

+ 集群 pod默认继承 node的dns 解析，修改kubelet服务启动参数 --resolv-conf=""，可以更改这个特性，详见 [kubelet](/roles/kube-node/templates/kubelet-config.yaml.j2) 启动参数。


> 注意：Default 不是默认的 DNS 策略。如果没有显式地指定dnsPolicy，将会使用 ClusterFirst
>
> + 如果 dnsPolicy 被设置为 “Default”，则名字解析配置会继承自 Pod 运行所在的节点。自定义上游域名服务器和存根域不能够与这个策略一起使用
> + 如果 dnsPolicy 被设置为 “ClusterFirst”，这就要依赖于是否配置了存根域和上游 DNS 服务器
>    + 未进行自定义配置：没有匹配上配置的集群域名后缀的任何请求，例如 “www.kubernetes.io”，将会被转发到继承自节点的上游域名服务器。
>    + 进行自定义配置：如果配置了存根域和上游 DNS 服务器（类似于 前面示例 配置的内容），DNS 查询将基于下面的流程对请求进行路由：
>        + 查询首先被发送到 kube-dns 中的 DNS 缓存层。
>        + 从缓存层，检查请求的后缀，并根据下面的情况转发到对应的 DNS 上：
>            + 具有集群后缀的名字（例如 “.cluster.local”）：请求被发送到 kubedns。
>            + 具有存根域后缀的名字（例如 “.acme.local”）：请求被发送到配置的自定义 DNS 解析器（例如：监听在 1.2.3.4）。
>            + 未能匹配上后缀的名字（例如 “widget.com”）：请求被转发到上游 DNS（例如：Google 公共 DNS 服务器，8.8.8.8 和 8.8.4.4）。

### 验证 dns服务
新建一个测试nginx服务
#### 方式一
`kubectl run nginx --image=nginx --expose --port=80`

```ini
# 多master集群服务 on 2019.09.08
[root@wfh_deploy ansible]# kubectl run nginx --image=nginx --expose --port=80
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
service/nginx created
deployment.apps/nginx created
```

确认nginx服务

```{.python .input}
kubectl get pod|grep nginx
nginx-7cbc4b4d9c-fl46v   1/1       Running   0          1m
kubectl get svc|grep nginx
nginx        ClusterIP   10.68.33.167   <none>        80/TCP    1m
```

```{.python .input}
# 多master集群服务 on 2019.09.08
[root@wfh_deploy ansible]# kubectl get pod|grep nginx
nginx-7c45b84548-67224   1/1     Running   0          3m57s
[root@wfh_deploy ansible]# kubectl get svc|grep nginx
nginx        ClusterIP   10.68.164.65   <none>        80/TCP    5m17s
```

测试 pod alpine

```bash
kubectl run test --rm -it --image=alpine /bin/sh
#If you don't see a command prompt, try pressing enter.

# 进入容器后，执行下面的指令
cat /etc/resolv.conf
# nameserver 10.68.0.2
# search default.svc.cluster.local. svc.cluster.local. cluster.local.
# options ndots:5

# 测试集群内部服务解析
nslookup nginx.default.svc.cluster.local
```
```ini
# 输出结果
Server:    10.68.0.2
Address 1: 10.68.0.2 kube-dns.kube-system.svc.cluster.local

Name:      nginx
Address 1: 10.68.33.167 nginx.default.svc.cluster.local
/ # nslookup kubernetes.default.svc.cluster.local
Server:    10.68.0.2
Address 1: 10.68.0.2 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.68.0.1 kubernetes.default.svc.cluster.local
```
```bash
# 测试外部域名的解析，默认集成node的dns解析(提示符'/ #')
nslookup www.baidu.com
```
输出结果：
```ini
Server:    10.68.0.2
Address 1: 10.68.0.2 kube-dns.kube-system.svc.cluster.local

Name:      www.baidu.com
Address 1: 180.97.33.108
Address 2: 180.97.33.107
/ #
```

```bash
# 多master集群服务 on 2019.09.08
# 主控端执行([root@wfh_deploy ansible]#)
kubectl run test --generator=run-pod/v1 --rm -it --image=alpine /bin/sh
```

- Note1: 如果你使用`calico`网络组件，通过命令`ansible-playbook
90.setup.yml`安装完集群后，直接安装dns组件，可能会出现如下BUG，分析是因为calico分配pod地址时候会从网段的第一个地址（网络地址）开始，详见提交的[ISSUE#1710](https://github.com/projectcalico/calico/issues/1710)，临时解决办法为手动删除POD，重新创建后获取后面的IP地址。

```bash
# BUG出现现象
$ kubectl get pod --all-namespaces -o wide
```
输出结果：
```ini
NAMESPACE     NAME                                       READY     STATUS             RESTARTS   AGE       IP              NODE
default       busy-5cc98488d4-s894w                      1/1       Running            0          28m       172.20.24.193   192.168.97.24
kube-system   calico-kube-controllers-6597d9c664-nq9hn   1/1       Running            0          1h        192.168.97.24   192.168.97.24
kube-system   calico-node-f8gnf                          2/2       Running            0          1h        192.168.97.24   192.168.97.24
kube-system   kube-dns-69bf9d5cc9-c68mw                  0/3       CrashLoopBackOff   27         31m       172.20.24.192   192.168.97.24
```
```bash
# 解决办法，删除pod，自动重建
kubectl delete pod -n kube-system kube-dns-69bf9d5cc9-c68mw
```

```bash
# 多master集群服务 on 2019.09.08
# 主控端 [root@wfh_deploy ansible]#
kubectl get pod --all-namespaces -o wide
```
输出结果：
```ini
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE     IP               NODE             NOMINATED NODE   READINESS GATES
default       nginx-7c45b84548-67224                        1/1     Running   0          22m     172.17.2.4       192.168.20.206   <none>           <none>
default       test                                          1/1     Running   1          9m35s   172.17.2.5       192.168.20.206   <none>           <none>
kube-system   coredns-797455887b-ntwq5                      1/1     Running   0          21h     172.17.3.2       192.168.20.205   <none>           <none>
kube-system   coredns-797455887b-ssdb5                      1/1     Running   0          21h     172.17.4.2       192.168.20.204   <none>           <none>
kube-system   heapster-5f848f54bc-d969p                     1/1     Running   0          21h     172.17.2.2       192.168.20.206   <none>           <none>
kube-system   kube-flannel-ds-amd64-424sr                   1/1     Running   0          21h     192.168.20.203   192.168.20.203   <none>           <none>
kube-system   kube-flannel-ds-amd64-659sj                   1/1     Running   0          21h     192.168.20.204   192.168.20.204   <none>           <none>
kube-system   kube-flannel-ds-amd64-ch8lw                   1/1     Running   0          21h     192.168.20.205   192.168.20.205   <none>           <none>
kube-system   kube-flannel-ds-amd64-fs578                   1/1     Running   0          21h     192.168.20.206   192.168.20.206   <none>           <none>
kube-system   kube-flannel-ds-amd64-g6rms                   1/1     Running   0          21h     192.168.20.201   192.168.20.201   <none>           <none>
kube-system   kubernetes-dashboard-5c7687cf8-pgw77          1/1     Running   0          21h     172.17.3.3       192.168.20.205   <none>           <none>
kube-system   metrics-server-85c7b8c8c4-gkgmb               1/1     Running   0          21h     172.17.4.3       192.168.20.204   <none>           <none>
kube-system   traefik-ingress-controller-766dbfdddd-mk7b5   1/1     Running   0          21h     172.17.2.3       192.168.20.206   <none>           <none>

# 删除测试(test) pod
[root@wfh_deploy ansible]# kubectl delete pod -n default test
pod "test" deleted
```

- Note2: 使用``` kubectl run test -it --rm --image=busybox /bin/sh``` 进行解析测试可能会失败,
busybox内的nslookup程序有bug, 详见 https://github.com/kubernetes/dns/issues/109

### 使用 dig 进行验证

coreDNS使用buybox无法验证，可以使用dig进行验证

参考：[K8S coreDNS部署及简单验证](https://blog.csdn.net/u013760072/article/details/88665073)

#### dig 配置文件 dns-dig.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name:
dig
  namespace: default
spec:
  containers:
    - name: dig
      image: docker.io/azukiapp/dig
      command:
        - sleep
        - "3600"
imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

> 我的保存位置 /etc/ansible/roles/cluster-addon/templates/dns-dig.yaml


#### 创建 dig 应用

创建指令：
```bash
# 主控端 [root@wfh_deploy ~]#
kubectl apply -f /etc/ansible/roles/cluster-addon/templates/dns-dig.yaml
```

输出结果：

```{.python .input .ini}
pod/dig created
```

#### 验证
验证指令：

```{.python .input}
[root@wfh_deploy ~]# kubectl exec -ti dig -- nslookup kubernetes
```

验证结果：

```{.python .input .ini}
Server:		10.68.0.2
Address:	10.68.0.2#53

Name:
kubernetes.default.svc.cluster.local
Address: 10.68.0.1
```

### 使用 dnstools 进行验证
参考：[kubernetes（k8s）coredns1.5.0部署](https://www.jianshu.com/p/834eb6ff2a72)

创建
dnstools pod 指令

```{.python .input}
[root@wfh_deploy ~]# kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools
```

返回结果：

```ini
If you don't see a command prompt, try pressing enter.
dnstools#
```

在 dnstools
pod 中查询 `kubernetes` pod 服务 域名，指令：

```bash
# 镜像pod中 (dnstools#)
nslookup kubernetes
```

输出结果：

```ini
Server:		10.68.0.2
Address:	10.68.0.2#53

Name:
kubernetes.default.svc.cluster.local
Address: 10.68.0.1
```

查询 `nginx`
pod的域名，指令：

```bash
# dnstools#
nslookup nginx
```

输出结果：

```ini
Server:		10.68.0.2
Address:	10.68.0.2#53

Name:	nginx.default.svc.cluster.local
Address:
10.68.164.65
```
```bash
# 退出测试容器
dnstools# exit
# pod "dnstools" deleted
```

### yaml 方式验证

nginx-ds.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-dm
spec:
  replicas: 2
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    name: nginx
```
alpine.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: alpine
spec:
  containers:
  - name: alpine
    image: alpine
    command:
    - sh
    - -c
    - while true; do sleep 1; done
```

#### 创建pods和svc
```bash
kubectl create -f nginx-ds.yaml
kubectl create -f alpine.yaml
```

#### 测试
指令：
```bash
kubectl exec -it alpine nslookup nginx-svc
```
输出结果
```ini
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx-svc
Address 1: 10.254.245.129 nginx-svc.default.svc.dns.kubernetes
```
指令：
```bash
kubectl exec -it alpine nslookup kubernetes
```
```ini
nslookup: can't resolve '(null)': Name does not resolve

Name:      kubernetes
Address 1: 10.254.0.1 kubernetes.default.svc.dns.kubernetes
```
