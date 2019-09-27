# Prometheus

随着`heapster`项目停止更新并慢慢被`metrics-
server`取代，集群监控这项任务也将最终转移。`prometheus`的监控理念、数据结构设计其实相当精简，包括其非常灵活的查询语言；但是对于初学者来说，想要在k8s集群中实践搭建一套相对可用的部署却比较麻烦，由此还产生了不少专门的项目（如：[prometheus-operator](https://github.com/coreos/prometheus-operator)），本文介绍使用`helm
chart`部署集群的prometheus监控。

- `helm`已成为`CNCF`独立托管项目，预计会更加流行起来

## 前提

- 安装 helm：以本项目[安全安装helm](helm.md)为例
- 安装 [kube-dns](kubedns.md)

## 准备

安装目录概览 `ll /etc/ansible/manifests/prometheus`

```{.python .input}
drwx------  3 root root  4096 Jun  3 22:42 grafana/
-rw-r-----  1 root root 67875 Jun  4 22:47 grafana-dashboards.yaml
-rw-r-----  1 root root   690 Jun  4 09:34 grafana-settings.yaml
-rw-r-----  1 root root  1105 May 30 16:54 prom-alertrules.yaml
-rw-r-----  1 root root   474 Jun  5 10:04 prom-alertsmanager.yaml
drwx------  3 root root  4096 Jun  2 21:39 prometheus/
-rw-r-----  1 root root   294 May 30 18:09 prom-settings.yaml
```

- 目录`prometheus/`和`grafana/`即官方的helm charts，可以使用`helm fetch --untar stable/prometheus` 和 `helm fetch --untar stable/grafana`下载，本安装不会修改任何官方charts里面的内容，这样方便以后跟踪charts版本的更新
- `prom-settings.yaml`：个性化prometheus安装参数，比如禁用PV，禁用pushgateway，设置nodePort等
- `prom-alertrules.yaml`：配置告警规则
- `prom-alertsmanager.yaml`：配置告警邮箱设置等
- `grafana-settings.yaml`：个性化grafana安装参数，比如用户名密码，datasources，dashboardProviders等
- `grafana-dashboards.yaml`：预设置dashboard

## 安装

```{.python .input}
$ source ~/.bashrc
$ cd /etc/ansible/manifests/prometheus
# 安装 prometheus chart，如果你的helm安装没有启用tls证书，请忽略--tls参数
$ helm install --tls \
    --name monitor \
    --namespace monitoring \
    -f prom-settings.yaml \
    -f prom-alertsmanager.yaml \
    -f prom-alertrules.yaml \
    prometheus
# 安装 grafana chart
$ helm install --tls \
    --name grafana \
    --namespace monitoring \
    -f grafana-settings.yaml \
    -f grafana-dashboards.yaml \
    grafana
```

### 通过　helm 部署　prometheus

```{.python .input}
[root@wfh_deploy prometheus]$ helm install
--tls \
    --name monitor \
    --namespace monitoring \
    -f prom-
settings.yaml \
    -f prom-alertsmanager.yaml \
    -f prom-alertrules.yaml \
prometheus
```

输出结果：

```{.python .input}
Error: context deadline exceeded
```

> **错误原因：**
我的部署主机与管理主机是分开的，而　helm
分客户端和服务端，默认情况下，部署主机上只有　`helm　客户端`，因此不能直接在部署主机上使用配置文件来部署prometheus。
>
> **解决办法：**
为了方便起见，我将　/etc/ansible/manifests/prometheus
发送到　第一台管理主机上（我这里是wfh_node01)，然后在执行上面的指令即可。

```{.python .input}
# 在部署主机(wfh_deploy)执行：
[root@wfh_deploy ~]$ scp -r /etc/ansible/manifests/prometheus wfh_node01:~/

#
切换到管理主机(wfh_node01)执行:
[root@wfh_node01 ~]$ cd prometheus
[root@wfh_node01 prometheus]$
helm install --tls \
    --name monitor \
    --namespace monitoring \
    -f prom-settings.yaml \
    -f prom-alertsmanager.yaml \
    -f prom-alertrules.yaml \
    prometheus
```

输出结果：

```ini
NAME:   monitor
LAST DEPLOYED: Sun Sep  8 17:34:28 2019
NAMESPACE:
monitoring
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME
DATA  AGE
monitor-prometheus-alertmanager  1     31s
monitor-prometheus-server
3     31s

==> v1/Pod(related)
NAME
READY  STATUS             RESTARTS  AGE
monitor-prometheus-
alertmanager-59cc5c4b6-vqlxf         0/2    ContainerCreating  0         31s
monitor-prometheus-kube-state-metrics-7556696ff7-s88rc  0/1    ContainerCreating
0         31s
monitor-prometheus-node-exporter-fhq5t                  0/1
ContainerCreating  0         30s
monitor-prometheus-node-exporter-kn4qc
1/1    Running            0         30s
monitor-prometheus-node-exporter-pfbwq
0/1    ContainerCreating  0         30s
monitor-prometheus-node-exporter-phpf4
1/1    Running            0         31s
monitor-prometheus-node-exporter-r9vmr
1/1    Running            0         30s
monitor-prometheus-
server-579954d46d-dpr5d              0/2    Init:0/1           0         30s
==> v1/Service
NAME                                   TYPE       CLUSTER-IP
EXTERNAL-IP  PORT(S)       AGE
monitor-prometheus-alertmanager        NodePort
10.68.93.6    <none>       80:39001/TCP  31s
monitor-prometheus-kube-state-
metrics  ClusterIP  None          <none>       80/TCP        31s
monitor-
prometheus-node-exporter       ClusterIP  None          <none>       9100/TCP
31s
monitor-prometheus-server              NodePort   10.68.60.193  <none>
80:39000/TCP  31s

==> v1/ServiceAccount
NAME
SECRETS  AGE
monitor-prometheus-alertmanager        1        31s
monitor-
prometheus-kube-state-metrics  1        31s
monitor-prometheus-node-exporter
1        31s
monitor-prometheus-pushgateway         1        31s
monitor-
prometheus-server              1        31s

==> v1beta1/ClusterRole
NAME
AGE
monitor-prometheus-kube-state-metrics  31s
monitor-prometheus-server
31s

==> v1beta1/ClusterRoleBinding
NAME                                   AGE
monitor-prometheus-kube-state-metrics  31s
monitor-prometheus-server
31s

==> v1beta1/DaemonSet
NAME                              DESIRED  CURRENT
READY  UP-TO-DATE  AVAILABLE  NODE SELECTOR  AGE
monitor-prometheus-node-
exporter  5        5        3      5           3          <none>         31s
==> v1beta1/Deployment
NAME                                   READY  UP-TO-DATE
AVAILABLE  AGE
monitor-prometheus-alertmanager        0/1    1           0
31s
monitor-prometheus-kube-state-metrics  0/1    1           0          31s
monitor-prometheus-server              0/1    1           0          31s
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS
name from within your cluster:
monitor-prometheus-
server.monitoring.svc.cluster.local


Get the Prometheus server URL by running
these commands in the same shell:
  export NODE_PORT=$(kubectl get --namespace
monitoring -o jsonpath="{.spec.ports[0].nodePort}" services monitor-prometheus-
server)
  export NODE_IP=$(kubectl get nodes --namespace monitoring -o
jsonpath="{.items[0].status.addresses[0].address}")
  echo
http://$NODE_IP:$NODE_PORT
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when
#####
######            the Server pod is terminated.
#####
#################################################################################
The Prometheus alertmanager can be accessed via port 80 on the following DNS
name from within your cluster:
monitor-prometheus-
alertmanager.monitoring.svc.cluster.local


Get the Alertmanager URL by running
these commands in the same shell:
  export NODE_PORT=$(kubectl get --namespace
monitoring -o jsonpath="{.spec.ports[0].nodePort}" services monitor-prometheus-
alertmanager)
  export NODE_IP=$(kubectl get nodes --namespace monitoring -o
jsonpath="{.items[0].status.addresses[0].address}")
  echo
http://$NODE_IP:$NODE_PORT
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when
#####
######            the AlertManager pod is terminated.
#####
#################################################################################
For more information on running Prometheus, visit:
https://prometheus.io/
```

根据提示，部署主机或管理主机上操作：

```bash
$ export NODE_PORT=$(kubectl get --namespace monitoring -o jsonpath="{.spec.ports[0].nodePort}" services monitor-prometheus-server)
$ export NODE_IP=$(kubectl get nodes --namespace monitoring -o jsonpath="{.items[0].status.addresses[0].address}")
$ echo $NODE_IP：$NODE_PORT
```

输出结果：

```ini
192.168.20.201：39000
```

### 通过　helm 部署　grafana

输入指令：

```bash
[root@wfh_node01 prometheus]$
helm install --tls \
     --name grafana \
     --namespace monitoring \
     -f grafana-settings.yaml \
     -f grafana-dashboards.yaml \
     grafana
```

输出结果：

```ini
NAME:   grafana
LAST DEPLOYED: Sun Sep  8 18:07:57 2019
NAMESPACE:
monitoring
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME
AGE
grafana-clusterrole  31s

==> v1/ClusterRoleBinding
NAME
AGE
grafana-clusterrolebinding  31s

==> v1/ConfigMap
NAME
DATA  AGE
grafana                     4     31s
grafana-dashboards-default  3
31s

==> v1/Pod(related)
NAME                    READY  STATUS
RESTARTS  AGE
grafana-78747c76-n9zqd  0/1    PodInitializing  0         31s

==>
v1/Secret
NAME     TYPE    DATA  AGE
grafana  Opaque  3     31s

==> v1/Service
NAME     TYPE      CLUSTER-IP     EXTERNAL-IP  PORT(S)       AGE
grafana
NodePort  10.68.179.177  <none>       80:39002/TCP  31s

==> v1/ServiceAccount
NAME     SECRETS  AGE
grafana  1        31s

==> v1beta1/PodSecurityPolicy
NAME
PRIV   CAPS      SELINUX   RUNASUSER  FSGROUP   SUPGROUP  READONLYROOTFS
VOLUMES
grafana  false  RunAsAny  RunAsAny  RunAsAny   RunAsAny  false
configMap,emptyDir,projected,secret,downwardAPI,persistentVolumeClaim

==>
v1beta1/Role
NAME     AGE
grafana  31s

==> v1beta1/RoleBinding
NAME     AGE
grafana  31s

==> v1beta2/Deployment
NAME     READY  UP-TO-DATE  AVAILABLE  AGE
grafana  0/1    1           0          31s


NOTES:
1. Get your 'admin' user
password by running:

   kubectl get secret --namespace monitoring grafana -o
jsonpath="{.data.admin-password}" | base64 --decode ; echo

2. The Grafana
server can be accessed via port 80 on the following DNS name from within your
cluster:

   grafana.monitoring.svc.cluster.local

   Get the Grafana URL to
visit by running these commands in the same shell:
export NODE_PORT=$(kubectl
get --namespace monitoring -o jsonpath="{.spec.ports[0].nodePort}" services
grafana)
     export NODE_IP=$(kubectl get nodes --namespace monitoring -o
jsonpath="{.items[0].status.addresses[0].address}")
     echo
http://$NODE_IP:$NODE_PORT


3. Login with the password from step 1 and the
username: admin
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when
#####
######            the Grafana pod is terminated.
#####
#################################################################################
```

## 验证安装

```{.python .input .bash}
# 查看相关pod和svc
$ kubectl get pod,svc -n monitoring
NAME                                                     READY     STATUS    RESTARTS   AGE
grafana-54dc76d47d-2mk55                                 1/1       Running   0          1m
monitor-prometheus-alertmanager-6d9d9b5b96-w57bk         2/2       Running   0          2m
monitor-prometheus-kube-state-metrics-69f5d56f49-fh9z7   1/1       Running   0          2m
monitor-prometheus-node-exporter-55bwx                   1/1       Running   0          2m
monitor-prometheus-node-exporter-k8sb2                   1/1       Running   0          2m
monitor-prometheus-node-exporter-kxlr9                   1/1       Running   0          2m
monitor-prometheus-node-exporter-r5dx8                   1/1       Running   0          2m
monitor-prometheus-server-5ccfc77dff-8h9k6               2/2       Running   0          2m

NAME                                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
grafana                                 NodePort    10.68.74.242   <none>        80:39002/TCP   1m
monitor-prometheus-alertmanager         NodePort    10.68.69.105   <none>        80:39001/TCP   2m
monitor-prometheus-kube-state-metrics   ClusterIP   None           <none>        80/TCP         2m
monitor-prometheus-node-exporter        ClusterIP   None           <none>        9100/TCP       2m
monitor-prometheus-server               NodePort    10.68.248.94   <none>        80:39000/TCP   2m
```

```{.python .input}
# on 2019.09.08

# 安装了　monitor　的情况
[root@wfh_deploy ~]$ kubectl get pod,svc -n monitoring
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/monitor-prometheus-alertmanager-59cc5c4b6-vqlxf          2/2     Running   0          25m
pod/monitor-prometheus-kube-state-metrics-7556696ff7-s88rc   1/1     Running   0          25m
pod/monitor-prometheus-node-exporter-fhq5t                   1/1     Running   0          25m
pod/monitor-prometheus-node-exporter-kn4qc                   1/1     Running   0          25m
pod/monitor-prometheus-node-exporter-pfbwq                   1/1     Running   0          25m
pod/monitor-prometheus-node-exporter-phpf4                   1/1     Running   0          25m
pod/monitor-prometheus-node-exporter-r9vmr                   1/1     Running   0          25m
pod/monitor-prometheus-server-579954d46d-dpr5d               2/2     Running   0          25m

NAME                                            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/monitor-prometheus-alertmanager         NodePort    10.68.93.6     <none>        80:39001/TCP   25m
service/monitor-prometheus-kube-state-metrics   ClusterIP   None           <none>        80/TCP         25m
service/monitor-prometheus-node-exporter        ClusterIP   None           <none>        9100/TCP       25m
service/monitor-prometheus-server               NodePort    10.68.60.193   <none>        80:39000/TCP   25m

# 安装了　grafana　的情况
[root@wfh_node01 prometheus]$ kubectl get pod,svc -n monitoring
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/grafana-78747c76-n9zqd                                   1/1     Running   0          38m
pod/monitor-prometheus-alertmanager-59cc5c4b6-vqlxf          2/2     Running   0          72m
pod/monitor-prometheus-kube-state-metrics-7556696ff7-s88rc   1/1     Running   0          72m
pod/monitor-prometheus-node-exporter-fhq5t                   1/1     Running   0          72m
pod/monitor-prometheus-node-exporter-kn4qc                   1/1     Running   0          72m
pod/monitor-prometheus-node-exporter-pfbwq                   1/1     Running   0          72m
pod/monitor-prometheus-node-exporter-phpf4                   1/1     Running   0          72m
pod/monitor-prometheus-node-exporter-r9vmr                   1/1     Running   0          72m
pod/monitor-prometheus-server-579954d46d-dpr5d               2/2     Running   0          72m

NAME                                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/grafana                                 NodePort    10.68.179.177   <none>        80:39002/TCP   38m
service/monitor-prometheus-alertmanager         NodePort    10.68.93.6      <none>        80:39001/TCP   72m
service/monitor-prometheus-kube-state-metrics   ClusterIP   None            <none>        80/TCP         72m
service/monitor-prometheus-node-exporter        ClusterIP   None            <none>        9100/TCP       72m
service/monitor-prometheus-server               NodePort    10.68.60.193    <none>        80:39000/TCP   72m
```

- 访问prometheus的web界面：`http://$NodeIP:39000`
- 访问alertmanager的web界面：`http://$NodeIP:39001`
- 访问grafana的web界面：`http://$NodeIP:39002` (默认用户密码 admin:admin，可在web界面修改)

## 管理操作
- 升级（修改配置）：修改配置请在`prom-settings.yaml` `prom-alertsmanager.yaml` 等文件中进行，保存后执行：

```bash
# 修改prometheus
$ helm upgrade --tls monitor -f prom-settings.yaml -f prom-alertsmanager.yaml -f prom-alertrules.yaml prometheus
# 修改grafana
$ helm upgrade --tls grafana -f grafana-settings.yaml -f grafana-dashboards.yaml grafana
```

- 回退：具体可以参考`helm help rollback`文档

```{.python .input}
$ helm rollback --tls monitor [REVISION]
```

- 删除

```{.python .input}
$ helm del --tls monitor --purge
$ helm del --tls grafana --purge
```

## 验证告警

- 修改`prom-alertsmanager.yaml`文件中邮件告警为有效的配置内容，并使用 helm upgrade更新安装
-
手动临时关闭 master 节点的 kubelet 服务，等待几分钟看是否有告警邮件发送

```{.python .input}
# 在 master 节点运行
$ systemctl stop kubelet
```

## [可选] 配置钉钉告警

- 创建钉钉群，获取群机器人 webhook 地址
使用钉钉创建群聊以后可以方便设置群机器人，【群设置】-【群机器人】-【添加】-【自定义】-【添加】，然后按提示操作即可，参考 https://open-
doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.666d4a97eCG7XA&treeId=257&articleId=105735&docType=1
上述配置好群机器人，获得这个机器人对应的Webhook地址，记录下来，后续配置钉钉告警插件要用，格式如下

```{.python .input}
https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxx
```

- 创建钉钉告警插件，参考 http://theo.im/blog/2017/10/16/release-prometheus-alertmanager-
webhook-for-dingtalk/

```{.python .input}
# 编辑修改文件中 access_token=xxxxxx 为上一步你获得的机器人认证 token
$ vi /etc/ansible/manifests/prometheus/dingtalk-webhook.yaml
# 运行插件
$ kubectl apply -f /etc/ansible/manifests/prometheus/dingtalk-webhook.yaml
```

- 修改 alertsmanager 告警配置后，更新 helm prometheus 部署，成功后如上节测试告警发送

```{.python .input}
# 修改 alertsmanager 告警配置
$ cd /etc/ansible/manifests/prometheus
$ vi prom-alertsmanager.yaml
# 增加 receiver dingtalk，然后在 route 配置使用 receiver: dingtalk
    receivers:
    - name: dingtalk
      webhook_configs:
      - send_resolved: false
        url: http://webhook-dingtalk.monitoring.svc.cluster.local:8060/dingtalk/webhook1/send
# ...
# 更新 helm prometheus 部署
$ helm upgrade --tls monitor -f prom-settings.yaml -f prom-alertsmanager.yaml -f prom-alertrules.yaml prometheus
```

## 下一步

- 继续了解prometheus查询语言和配置文件
- 继续了解prometheus告警规则，编写适合业务应用的告警规则
-
继续了解grafana的dashboard编写，本项目参考了部分[feisky的模板](https://grafana.com/orgs/feisky/dashboards)
如果对以上部分有心得总结，欢迎分享贡献在项目中。
