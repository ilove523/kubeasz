## 06-安装kube-ovn网络组件.md

由灵雀云开源的网络组件 kube-ovn，将已被 openstack 社区采用的成熟网络虚拟化技术 ovs/ovn 引入 kubernetes 平台；为 kubernetes 网络打开了新的大门，令人耳目一新；强烈推荐大家试用该网络组件，反馈建议以帮助项目早日走向成熟。

- 介绍 https://blog.csdn.net/alauda_andy/article/details/88886128
- 项目地址 https://github.com/alauda/kube-ovn

### 特性介绍

kube-ovn 提供了针对企业应用场景下容器网络实用功能，并为实现更高级的网络管理控制提供了可能性；现有主要功能:

- 1.Namespace 和子网的绑定，以及子网间的访问控制;
- 2.静态IP分配;
- 3.动态QoS;
- 4.分布式和集中式网关;
- 5.内嵌 LoadBalancer;

### kubeasz 集成安装 kube-ovn

kube-ovn 的安装十分简单，详见项目的安装文档；基于 kubeasz，以下两步将安装一个集成了 kube-ovn 网络的 k8s 集群；

- 在 ansible hosts 中设置变量 `CLUSTER_NETWORK="kube-ovn"`
- 执行安装 `ansible-playbook 90.setup.yml` 或者 `easzctl setup`

kubeasz 项目为`kube-ovn`网络生成的 ansible role 如下：

``` bash
roles/kube-ovn
├── defaults
│   └── main.yml		# kube-ovn 相关配置文件
├── tasks
│   └── main.yml		# 安装执行文件
└── templates
    ├── kube-ovn.yaml.j2	# kube-ovn yaml 模板
    └── ovn.yaml.j2		# ovn yaml 模板
```

安装成功后，可以验证所有 k8s 集群功能正常，查看集群的 pod 网络如下：

```
$ kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
kube-ovn      kube-ovn-cni-5php2                      1/1     Running   2          35h   192.168.1.43   192.168.1.43   <none>           <none>
kube-ovn      kube-ovn-cni-7dwmx                      1/1     Running   2          35h   192.168.1.42   192.168.1.42   <none>           <none>
kube-ovn      kube-ovn-cni-lhlvl                      1/1     Running   2          35h   192.168.1.41   192.168.1.41   <none>           <none>
kube-ovn      kube-ovn-controller-57955db7b4-6x6hd    1/1     Running   0          35h   192.168.1.43   192.168.1.43   <none>           <none>
kube-ovn      kube-ovn-controller-57955db7b4-chvz4    1/1     Running   0          35h   192.168.1.42   192.168.1.42   <none>           <none>
kube-ovn      ovn-central-bb8747d77-tr5nz             1/1     Running   0          35h   192.168.1.41   192.168.1.41   <none>           <none>
kube-ovn      ovs-ovn-2qhhr                           1/1     Running   0          35h   192.168.1.41   192.168.1.41   <none>           <none>
kube-ovn      ovs-ovn-np8rn                           1/1     Running   0          35h   192.168.1.43   192.168.1.43   <none>           <none>
kube-ovn      ovs-ovn-pkjw4                           1/1     Running   0          35h   192.168.1.42   192.168.1.42   <none>           <none>
kube-system   coredns-55f46dd959-76qb5                1/1     Running   0          35h   10.16.0.12     192.168.1.42   <none>           <none>
kube-system   coredns-55f46dd959-wn8kw                1/1     Running   0          35h   10.16.0.11     192.168.1.43   <none>           <none>
kube-system   heapster-fdb7596d6-xmmrx                1/1     Running   0          35h   10.16.0.15     192.168.1.42   <none>           <none>
kube-system   kubernetes-dashboard-68ddcc97fc-dwzbf   1/1     Running   0          35h   10.16.0.14     192.168.1.42   <none>           <none>
kube-system   metrics-server-6c898b5b8b-zvct2         1/1     Running   0          35h   10.16.0.13     192.168.1.43   <none>           <none>
```

直观上 kube-ovn 与传统 k8s 网络（flannel/calico等）比较最大的不同是 pod 子网的分配：

- 传统网络插件下，集群中 pod 一般是不同 node 节点分配不同的子网；然后通过 overlay 等技术打通不同 node 节点的 pod 子网；
- kube-ovn 中 pod 网络根据其所在的 namespace 而定； namespace 在创建时可以根据 annotation 来配置它的子网/网关等参数；默认使用 10.16.0.0/16 的子网；

### 测试 namespace 子网分配

新建一个 namespace 测试分配一个新的 pod 子网

```
# 创建一个 namespace: test-ns
$ cat > test-ns.yaml << EOF
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    ovn.kubernetes.io/cidr: 10.17.0.0/24
    ovn.kubernetes.io/gateway: 10.17.0.1
    ovn.kubernetes.io/logical_switch: test-ns-subnet
    ovn.kubernetes.io/exclude_ips: "10.17.0.1..10.17.0.10"
  name: test-ns
EOF
$ kubectl apply -f test-ns.yaml

# 在 test-ns 中创建 nginx 部署
$ kubectl run -n test-ns nginx --image=nginx --replicas=2 --port=80 --expose

# 在 default 中创建 busy 客户端
$ kubectl run busy --image=busybox sleep 360000
```

创建成功后，查看 pod 地址的分配，可以看到确实 test-ns 中 pod 使用新的子网，而 default 中 pod 使用了默认子网，并验证 pod 之间的联通性（默认可通）

```
$ kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
default       busy-6c55ccddc5-qrm5j                   1/1     Running   0          31h   10.16.0.16     192.168.1.43   <none>           <none>
kube-ovn      kube-ovn-cni-5php2                      1/1     Running   2          35h   192.168.1.43   192.168.1.43   <none>           <none>
kube-ovn      kube-ovn-cni-7dwmx                      1/1     Running   2          35h   192.168.1.42   192.168.1.42   <none>           <none>
kube-ovn      kube-ovn-cni-lhlvl                      1/1     Running   2          35h   192.168.1.41   192.168.1.41   <none>           <none>
kube-ovn      kube-ovn-controller-57955db7b4-6x6hd    1/1     Running   0          35h   192.168.1.43   192.168.1.43   <none>           <none>
kube-ovn      kube-ovn-controller-57955db7b4-chvz4    1/1     Running   0          35h   192.168.1.42   192.168.1.42   <none>           <none>
kube-ovn      ovn-central-bb8747d77-tr5nz             1/1     Running   0          35h   192.168.1.41   192.168.1.41   <none>           <none>
kube-ovn      ovs-ovn-2qhhr                           1/1     Running   0          35h   192.168.1.41   192.168.1.41   <none>           <none>
kube-ovn      ovs-ovn-np8rn                           1/1     Running   0          35h   192.168.1.43   192.168.1.43   <none>           <none>
kube-ovn      ovs-ovn-pkjw4                           1/1     Running   0          35h   192.168.1.42   192.168.1.42   <none>           <none>
kube-system   coredns-55f46dd959-76qb5                1/1     Running   0          35h   10.16.0.12     192.168.1.42   <none>           <none>
kube-system   coredns-55f46dd959-wn8kw                1/1     Running   0          35h   10.16.0.11     192.168.1.43   <none>           <none>
kube-system   heapster-fdb7596d6-xmmrx                1/1     Running   0          35h   10.16.0.15     192.168.1.42   <none>           <none>
kube-system   kubernetes-dashboard-68ddcc97fc-dwzbf   1/1     Running   0          35h   10.16.0.14     192.168.1.42   <none>           <none>
kube-system   metrics-server-6c898b5b8b-zvct2         1/1     Running   0          35h   10.16.0.13     192.168.1.43   <none>           <none>
test-ns       nginx-755464dd6c-s6flj                  1/1     Running   0          31h   10.17.0.12     192.168.1.42   <none>           <none>
test-ns       nginx-755464dd6c-zct56                  1/1     Running   0          31h   10.17.0.11     192.168.1.43   <none>           <none>
```

- 更多的测试（pod网络QOS限速，namespace网络隔离等）请参考 kube-ovn 项目说明文档

### 错误记录

```ini
[root@wfh_deploy ansible]# kubectl get pods,svc  --all-namespaces -o wide
NAMESPACE     NAME                                              READY   STATUS             RESTARTS   AGE     IP               NODE             NOMINATED NODE   READINESS GATES
kube-ovn      pod/kube-ovn-cni-4rb5v                            1/1     Running            14         83m     192.168.20.206   192.168.20.206   <none>           <none>
kube-ovn      pod/kube-ovn-cni-r2pz8                            1/1     Running            8          83m     192.168.20.202   192.168.20.202   <none>           <none>
kube-ovn      pod/kube-ovn-cni-r5tnh                            1/1     Running            4          83m     192.168.20.201   192.168.20.201   <none>           <none>
kube-ovn      pod/kube-ovn-cni-s9vjx                            1/1     Running            7          83m     192.168.20.204   192.168.20.204   <none>           <none>
kube-ovn      pod/kube-ovn-cni-tpcz6                            1/1     Running            7          83m     192.168.20.203   192.168.20.203   <none>           <none>
kube-ovn      pod/kube-ovn-cni-xxr7g                            1/1     Running            3          83m     192.168.20.205   192.168.20.205   <none>           <none>
kube-ovn      pod/kube-ovn-controller-8547cf6b5-pf794           1/1     Running            2          83m     192.168.20.202   192.168.20.202   <none>           <none>
kube-ovn      pod/ovn-central-9ff49887f-drkgq                   1/1     Running            1          83m     192.168.20.201   192.168.20.201   <none>           <none>
kube-ovn      pod/ovs-ovn-2hr9n                                 1/1     Running            4          83m     192.168.20.203   192.168.20.203   <none>           <none>
kube-ovn      pod/ovs-ovn-5shrd                                 1/1     Running            1          83m     192.168.20.201   192.168.20.201   <none>           <none>
kube-ovn      pod/ovs-ovn-g292l                                 1/1     Running            4          83m     192.168.20.202   192.168.20.202   <none>           <none>
kube-ovn      pod/ovs-ovn-qn6ff                                 1/1     Running            2          83m     192.168.20.206   192.168.20.206   <none>           <none>
kube-ovn      pod/ovs-ovn-rmbb5                                 1/1     Running            1          83m     192.168.20.205   192.168.20.205   <none>           <none>
kube-ovn      pod/ovs-ovn-tx6gb                                 1/1     Running            4          83m     192.168.20.204   192.168.20.204   <none>           <none>
kube-system   pod/coredns-797455887b-g6n94                      0/1     Running            1          10m     172.20.0.3       192.168.20.203   <none>           <none>
kube-system   pod/coredns-797455887b-m7h7p                      0/1     Running            1          10m     172.20.0.4       192.168.20.206   <none>           <none>
kube-system   pod/heapster-5f848f54bc-xkpnt                     1/1     Running            1          83m     172.20.0.7       192.168.20.204   <none>           <none>
kube-system   pod/kubernetes-dashboard-5c7687cf8-w45f2          0/1     CrashLoopBackOff   7          9m51s   172.20.0.6       192.168.20.206   <none>           <none>
kube-system   pod/metrics-server-85c7b8c8c4-766vs               0/1     CrashLoopBackOff   7          10m     172.20.0.5       192.168.20.203   <none>           <none>
kube-system   pod/traefik-ingress-controller-766dbfdddd-2jb2j   1/1     Running            1          83m     172.20.0.2       192.168.20.204   <none>           <none>

# 删除不能正常运行pod
kubectl delete deployment,svc -n kube-system kube-dns coredns kubernetes-dashboard metrics-server
```

### 延伸阅读

- [kube-ovn 官方文档](https://github.com/alauda/kube-ovn/tree/master/docs)
- [从 Bridge 到 OVS，探索虚拟交换机](https://www.cnblogs.com/bakari/p/8097439.html)
