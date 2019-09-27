
## 第一部分：EFK

`EFK` 插件是`k8s`项目的一个日志解决方案，它包括三个组件：[Elasticsearch](), [Fluentd](), [Kibana]()；Elasticsearch 是日志存储和日志搜索引擎，Fluentd 负责把`k8s`集群的日志发送给 Elasticsearch, Kibana 则是可视化界面查看和检索存储在 ES 中的数据。

### 准备
参考官方[部署文档](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch) 的基础上使用本项目`manifests/efk/`部署，以下为几点主要的修改：

+ 修改 fluentd-es-configmap.yaml 中的部分 journald 日志源（增加集群组件服务日志搜集）
+ 修改官方docker镜像，方便国内下载加速
+ 修改 es-statefulset.yaml 支持日志存储持久化等
+ 增加自动清理日志，见后文`第四部分`

### 安装

#### for elk

输入指令：
```bash
#[root@wfh_deploy ~]#
kubectl apply -f /etc/ansible/manifests/efk/
```

输出结果：
```{.python .input}
service/elasticsearch-logging
created
configmap/fluentd-es-config-v0.2.0 created
serviceaccount/fluentd-es
created
clusterrole.rbac.authorization.k8s.io/fluentd-es created
clusterrolebinding.rbac.authorization.k8s.io/fluentd-es created
daemonset.apps/fluentd-es-v2.4.0 created
deployment.apps/kibana-logging created
service/kibana-logging created
```

#### for es

输入指令：

```{.python .input}
[root@wfh_deploy ~]#
kubectl apply -f /etc/ansible/manifests/efk/es-without-pv/
```

输出结果：

```{.python .input}
serviceaccount/elasticsearch-logging created
clusterrole.rbac.authorization.k8s.io/elasticsearch-logging created
clusterrolebinding.rbac.authorization.k8s.io/elasticsearch-logging created
statefulset.apps/elasticsearch-logging created
```

### 验证

```bash
# on 2019.09.08
#[root@wfh_deploy ~]#
kubectl get pods -n kube-system |grep -E 'elasticsearch|fluentd|kibana'
#输出结果
elasticsearch-logging-0                       1/1     Running   0          16m
elasticsearch-logging-1                       1/1     Running   0          8m48s
fluentd-es-v2.4.0-5jxsr                       1/1     Running   0          17m
fluentd-es-v2.4.0-5pv9j                       1/1     Running   0          17m
fluentd-es-v2.4.0-6qwxn                       1/1     Running   0          17m
fluentd-es-v2.4.0-gnwgt                       1/1     Running   0          17m
fluentd-es-v2.4.0-sfh6g                       1/1     Running   0          17m
kibana-logging-ffd866674-7fjtr                1/1     Running   0          17m
```

```bash
# on 2019.09.11
#[root@wfh_deploy ~]#
kubectl get pods -n kube-system|grep -E 'elasticsearch|fluentd|kibana'
```
```ini
; 输出结果1
elasticsearch-logging-0                       0/1     Init:0/1            0          48s
fluentd-es-v2.4.0-4pvrc                       0/1     ContainerCreating   0          63s
fluentd-es-v2.4.0-5h978                       0/1     ContainerCreating   0          63s
fluentd-es-v2.4.0-d58fs                       0/1     ContainerCreating   0          63s
fluentd-es-v2.4.0-rv5rr                       0/1     ContainerCreating   0          63s
fluentd-es-v2.4.0-z8nbw                       0/1     ContainerCreating   0          63s
kibana-logging-ffd866674-9tdsb                0/1     ContainerCreating   0          63s
; 输出结果2
elasticsearch-logging-0                       0/1     PodInitializing     0          3m41s
fluentd-es-v2.4.0-4pvrc                       1/1     Running             0          3m56s
fluentd-es-v2.4.0-5h978                       1/1     Running             0          3m56s
fluentd-es-v2.4.0-d58fs                       0/1     ContainerCreating   0          3m56s
fluentd-es-v2.4.0-rv5rr                       1/1     Running             0          3m56s
fluentd-es-v2.4.0-z8nbw                       1/1     Running             0          3m56s
kibana-logging-ffd866674-9tdsb                0/1     ContainerCreating   0          3m56s
; 输出结果3
elasticsearch-logging-0                       0/1     PodInitializing   0          7m3s
fluentd-es-v2.4.0-4pvrc                       1/1     Running           0          7m18s
fluentd-es-v2.4.0-5h978                       1/1     Running           0          7m18s
fluentd-es-v2.4.0-d58fs                       1/1     Running           0          7m18s
fluentd-es-v2.4.0-rv5rr                       1/1     Running           0          7m18s
fluentd-es-v2.4.0-z8nbw                       1/1     Running           0          7m18s
kibana-logging-ffd866674-9tdsb                1/1     Running           0          7m18s
; 输出结果4
elasticsearch-logging-0                       1/1     Running           0          10m
elasticsearch-logging-1                       0/1     PodInitializing   0          101s
fluentd-es-v2.4.0-4pvrc                       1/1     Running           0          10m
fluentd-es-v2.4.0-5h978                       1/1     Running           0          10m
fluentd-es-v2.4.0-d58fs                       1/1     Running           0          10m
fluentd-es-v2.4.0-rv5rr                       1/1     Running           0          10m
fluentd-es-v2.4.0-z8nbw                       1/1     Running           0          10m
kibana-logging-ffd866674-9tdsb                1/1     Running           0          10m
; 输出结果5
elasticsearch-logging-0                       1/1     Running   1          16m
elasticsearch-logging-1                       1/1     Running   0          7m44s
fluentd-es-v2.4.0-4pvrc                       1/1     Running   0          16m
fluentd-es-v2.4.0-5h978                       1/1     Running   0          16m
fluentd-es-v2.4.0-d58fs                       1/1     Running   0          16m
fluentd-es-v2.4.0-rv5rr                       1/1     Running   0          16m
fluentd-es-v2.4.0-z8nbw                       1/1     Running   0          16m
kibana-logging-ffd866674-9tdsb                1/1     Running   0          16m
```

> kibana Pod 第一次启动时会用较长时间(10-20分钟)来优化和 Cache 状态页面，可以查看 Pod 的日志观察进度，如下等待 `Ready`状态。

```bash
$ kubectl logs -n kube-system kibana-logging-d5cffd7c6-9lz2p -f
...
{"type":"log","@timestamp":"2018-03-13T07:33:00Z","tags":["listening","info"],"pid":1,"message":"Server running at http://0:5601"}
{"type":"log","@timestamp":"2018-03-13T07:33:00Z","tags":["status","ui settings","info"],"pid":1,"state":"green","message":"Status changed from uninitialized to green - Ready","prevState":"uninitialized","prevMsg":"uninitialized"}
```

```bash
# on 2019.09.08
#[root@wfh_deploy ~]#
kubectl logs -n kube-system kibana-logging-ffd866674-7fjtr -f
```

输出结果：
```ini
...
{"type":"log","@timestamp":"2019-09-08T11:16:43Z","tags":["warning","elasticsearch","admin"],"pid":1,"message":"No living connections"}
{"type":"log","@timestamp":"2019-09-08T11:16:45Z","tags":["warning","elasticsearch","admin"],"pid":1,"message":"Unable to revive connection: http://elasticsearch-logging:9200/"}
{"type":"log","@timestamp":"2019-09-08T11:16:45Z","tags":["warning","elasticsearch","admin"],"pid":1,"message":"No living connections"}
{"type":"log","@timestamp":"2019-09-08T11:16:48Z","tags":["status","plugin:elasticsearch@6.6.1","error"],"pid":1,"state":"red","message":"Status changed from red to red - Service Unavailable","prevState":"red","prevMsg":"Unable to connect to Elasticsearch."}
{"type":"log","@timestamp":"2019-09-08T11:16:59Z","tags":["status","plugin:elasticsearch@6.6.1","info"],"pid":1,"state":"green","message":"Status changed from red to green - Ready","prevState":"red","prevMsg":"Service Unavailable"}
{"type":"log","@timestamp":"2019-09-08T11:16:59Z","tags":["info","migrations"],"pid":1,"message":"Creating index .kibana_1."}
{"type":"log","@timestamp":"2019-09-08T11:17:02Z","tags":["info","migrations"],"pid":1,"message":"Pointing alias .kibana to .kibana_1."}
{"type":"log","@timestamp":"2019-09-08T11:17:03Z","tags":["info","migrations"],"pid":1,"message":"Finished in 4263ms."}
{"type":"log","@timestamp":"2019-09-08T11:17:03Z","tags":["listening","info"],"pid":1,"message":"Server running at http://0:5601"}
```

### 访问 Kibana

推荐使用`kube-apiserver`方式访问（可以使用basic-auth、证书和rbac等方式进行认证授权），获取访问 URL

- 开启 apiserver basic-auth(用户名/密码认证)：`easzctl basic-auth -s -u admin -p
test1234`

```bash
kubectl cluster-info | grep Kibana
```
输出结果：
```ini
Kibana is running at https://192.168.1.10:8443/api/v1/namespaces/kube-system/services/kibana-logging/proxy
```

浏览器访问 URL：`https://192.168.1.10:8443/api/v1/namespaces/kube-
system/services/kibana-logging/proxy`，然后使用`basic-
auth`或者`证书`的方式认证后即可，关于认证可以参考[dashboard文档](dashboard.md)

首次登陆需要在`Management` -> `Index Patterns` 创建 `index pattern`，可以使用默认的 logstash-* pattern，点击下一步；在 Time Filter field name 下拉框选择 @timestamp; 点击创建Index Pattern后，稍等几分钟就可以在 Discover
菜单看到ElasticSearch logging 中汇聚的日志；

```bash
# on 2019.09.08
#[root@wfh_deploy ansible]#
kubectl cluster-info | grep Kibana
```
输出结果：
```ini
Kibana is running at https://192.168.20.201:6443/api/v1/namespaces/kube-system/services/kibana-logging/proxy
```

## 第二部分：日志持久化之静态PV
日志数据是存放于 `Elasticsearch POD`中，但是默认情况下它使用的是`emptyDir`存储类型，所以当
`POD`被删除或重新调度时，日志数据也就丢失了。以下讲解使用`NFS` 服务器手动（静态）创建`PV` 持久化保存日志数据的例子。

### 配置 NFS
+ 准备一个nfs服务器，如果没有可以参考[nfs-server](nfs-server.md)创建。
+ 配置nfs服务器的共享目录，即修改`/etc/exports`（根据实际网段替换`192.168.20.*`），修改后重启 `nfs`服务。

```bash
# 准备共享目录
mkdir -p /home/share/{es0,es1,es2}
chown -R nfsnobody.nfsnobody
/home/share
```

```bash
# 配置 nfs 服务
cat > /etc/exports <<EOF
/home/share
192.168.20.0/24(rw,sync,insecure,no_subtree_check,no_root_squash)
/home/share/es0
192.168.20.0/24(rw,sync,insecure,no_subtree_check,no_root_squash)
/home/share/es1
192.168.20.0/24(rw,sync,insecure,no_subtree_check,no_root_squash)
/home/share/es2
192.168.20.0/24(rw,sync,insecure,no_subtree_check,no_root_squash)
EOF
```

```bash
# 重启 nfs 服务
# for Centos7
systemctl enable nfs
systemctl restart nfs
```

### 使用静态 PV安装 EFK

- 请按实际日志容量需求修改 `es-static-pv/es-statefulset.yaml` 文件中 volumeClaimTemplates 设置的storage: 4Gi 大小。
- 请根据实际nfs服务器地址、共享目录、容量大小修改 `es-static-
pv/es-pv*.yaml` 文件中对应的设置。

```bash
# [root@wfh_deploy ~]#
# 如果之前已经安装了默认的EFK，请用以下两个命令先删除它
kubectl delete -f /etc/ansible/manifests/efk/
kubectl delete -f /etc/ansible/manifests/efk/es-without-pv/

# 安装静态PV 的 EFK
kubectl apply -f /etc/ansible/manifests/efk/
kubectl apply -f /etc/ansible/manifests/efk/es-static-pv/
```

+ 目录`es-static-pv` 下首先是利用 NFS服务预定义了三个 PV资源，然后在 `es-statefulset.yaml`定义中使用`volumeClaimTemplates` 去匹配使用预定义的 PV资源；注意 PV参数：`accessModes` `storageClassName` `storage`容量大小必须两边匹配。

### 验证安装

**1\) 集群中查看 `pod` `pv` `pvc` 等资源。**

输入pod查看指令：
```bash
kubectl get pods -n kube-system|grep -E 'elasticsearch|fluentd|kibana'
```
输出结果：
```ini
elasticsearch-logging-0                       1/1     Running   0          70s
elasticsearch-logging-1                       1/1     Running   0          65s
fluentd-es-v2.4.0-hqnmg                       1/1     Running   0          2m4s
fluentd-es-v2.4.0-lnqzx                       1/1     Running   0          2m4s
fluentd-es-v2.4.0-tdc69                       1/1     Running   0          2m4s
fluentd-es-v2.4.0-x8cql                       1/1     Running   0          2m4s
fluentd-es-v2.4.0-xbzhc                       1/1     Running   0          2m4s
kibana-logging-ffd866674-48ntq                1/1     Running   0          2m4s
```

输入pv查看指令：
```bash
kubectl get pv
```
输出结果：
```ini
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                                       STORAGECLASS       REASON   AGE
pv-es-0   4Gi        RWX            Recycle          Available                                                               es-storage-class            2m1s
pv-es-1   4Gi        RWX            Recycle          Bound       kube-system/elasticsearch-logging-elasticsearch-logging-0   es-storage-class            2m1s
pv-es-2   4Gi        RWX            Recycle          Bound       kube-system/elasticsearch-logging-elasticsearch-logging-1   es-storage-class            2m1s
```
输入pvc查看指令：
```bash
kubectl get pvc --all-namespaces
```
输出结果：
```ini
NAMESPACE     NAME                                            STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS       AGE
kube-system   elasticsearch-logging-elasticsearch-logging-0   Bound    pv-es-1   4Gi        RWX            es-storage-class   3m35s
kube-system   elasticsearch-logging-elasticsearch-logging-1   Bound    pv-es-2   4Gi        RWX            es-storage-class   3m29s
```

**2\) 网页访问 `kibana`查看具体的日志**
如上须等待（约15分钟） `kibana Pod`优化和 Cache 状态页面，达到 `Ready` 状态。

**3\) 登陆 NFS Server 查看对应目录和内部数据。**

```bash
$ ls /home/share
es0  es1  es2
```

## 第三部分：日志持久化之动态PV
`PV` 作为集群的存储资源，`StatefulSet` 依靠它实现 POD 的状态数据持久化，但是当`StatefulSet`动态伸缩时，它的 `PVC`请求也会变化，如果每次都需要管理员手动去创建对应的 `PV` 资源，那就很不方便；因此 K8S还提供了 `provisioner` 来动态创建 `PV`，不仅节省了管理员的时间，还可以根据不同的 `StorageClasses` 封装不同类型的存储供 PVC 选用。
+ 此功能需要 `API-SERVER` 参数 `--admission-control`字符串设置中包含`DefaultStorageClass`，本项目中已经开启。
+ `provisioner`指定 Volume 插件的类型，包括内置插件（如kubernetes.io/glusterfs）和外部插件（如 external-storage 提供的 ceph.com/cephfs，nfs-client 等），以下讲解使用 `nfs-client-provisioner`来动态创建 `PV`来持久化保存 `EFK`的日志数据。

### 配置
NFS（同上）

确保 `/etc/exports` 配置如下共享目录，并确保`/home/share`目录可读可写权限，否则可能因为权限问题无法动态生成PV的对应目录。（根据实际情况替换IP段`192.168.20.*`）

```ini
/home/share          192.168.20.0/24(rw,sync,insecure,no_subtree_check,no_root_squash)
```

### 使用动态 PV安装 EFK

- 首先根据[集群存储](../setup/08-cluster-storage.ipynb)创建 `nfs-client-provisioner`。
- 然后按实际需求修改 `es-dynamic-pv/es-statefulset.yaml` 文件中volumeClaimTemplates 设置的 storage: 4Gi 大小。

```bash
# 如果之前已经安装了默认的EFK或者静态PV EFK，请用以下命令先删除它
#[root@wfh_deploy ~]#
kubectl delete -f /etc/ansible/manifests/efk/
kubectl delete -f /etc/ansible/manifests/efk/es-without-pv/
kubectl delete -f /etc/ansible/manifests/efk/es-static-pv/

kubectl get pv --all-namespaces
```
输出结果：
```ini
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                                                       STORAGECLASS       REASON   AGE
pv-es-1   4Gi        RWX            Recycle          Terminating   kube-system/elasticsearch-logging-elasticsearch-logging-0   es-storage-class            33m
pv-es-2   4Gi        RWX            Recycle          Terminating   kube-system/elasticsearch-logging-elasticsearch-logging-1   es-storage-class            33m
```

```bash
# 删除一直处于 Terminating 状态的pv
# 方法：kubectl patch pv <pv名> -p '{"metadata":{"finalizers":null}}'
#[root@wfh_deploy ~]#
# 删除 pv 指令
kubectl patch pv pv-es-1 -p '{"metadata":{"finalizers":null}}'
```
输出结果：
```ini
persistentvolume/pv-es-1 patched
```

输入修改pv配置参数指令：
```bash
#[root@wfh_deploy ~]#
kubectl patch pv pv-es-2 -p '{"metadata":{"finalizers":null}}'
```
输出结果：
```ini
persistentvolume/pv-es-2 patched
```

```bash
# 安装动态PV 的 EFK
$ kubectl apply -f /etc/ansible/manifests/efk/
$ kubectl apply -f /etc/ansible/manifests/efk/es-dynamic-pv/
```

+ 首先 `nfs-client-provisioner.yaml` 创建一个工作 POD，它监听集群的 PVC请求，并当 PVC请求来到时调用 `nfs-client` 去请求 `nfs-server`的存储资源，成功后即动态生成对应的 PV资源。
+ `nfs-dynamic-storageclass.yaml` 定义 NFS存储类型的类型名 `nfs-dynamic-class`，然后在 `es-statefulset.yaml`中必须使用这个类型名才能动态请求到资源。
+ 参考 [k8s pv无法删除问题](https://www.cnblogs.com/Dev0ps/p/10936496.html)

### 验证安装

1\) 集群中查看 `pod` `pv` `pvc` 等资源。

```bash
kubectl get pods -n kube-system|grep -E 'elasticsearch|fluentd|kibana'
```
输出结果：
```ini
elasticsearch-logging-0                    1/1       Running   0          10m
elasticsearch-logging-1                    1/1       Running   0          10m
fluentd-es-v2.0.2-6c95c                    1/1       Running   0          10m
fluentd-es-v2.0.2-f2xh8                    1/1       Running   0          10m
fluentd-es-v2.0.2-pv5q5                    1/1       Running   0          10m
kibana-logging-d5cffd7c6-9lz2p             1/1       Running   0          10m
```
输入查看pv指令：
```bash
kubectl get pv
```
输出结果：
```ini
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                                       STORAGECLASS        REASON    AGE
pvc-50644f36-358b-11e8-9edd-525400cecc16   4Gi        RWX            Delete           Bound     kube-system/elasticsearch-logging-elasticsearch-logging-0   nfs-dynamic-class             10m
pvc-5b105ee6-358b-11e8-9edd-525400cecc16   4Gi        RWX            Delete           Bound     kube-system/elasticsearch-logging-elasticsearch-logging-1   nfs-dynamic-class             10m
```
输入查看pvc指令：
```bash
kubectl get pvc --all-namespaces
```
```ini
NAMESPACE     NAME                                            STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
kube-system   elasticsearch-logging-elasticsearch-logging-0   Bound     pvc-50644f36-358b-11e8-9edd-525400cecc16   4Gi        RWX            nfs-dynamic-class   10m
kube-system   elasticsearch-logging-elasticsearch-logging-1   Bound     pvc-5b105ee6-358b-11e8-9edd-525400cecc16   4Gi        RWX            nfs-dynamic-class   10m
```

2\) 网页访问 `kibana`查看具体的日志，如上须等待（约15分钟） `kibana Pod`优化和 Cache 状态页面，达到 `Ready` 状态。

3\) 登陆 NFS Server 查看对应目录和内部数据

在 NFS 服务端输入查看指令：
```bash
ls /home/share # 可以看到类似如下的目录生成
```
输出结果：
```ini
kube-system-elasticsearch-logging-elasticsearch-logging-0-pvc-50644f36-358b-11e8-9edd-525400cecc16
kube-system-elasticsearch-logging-elasticsearch-logging-1-pvc-5b105ee6-358b-11e8-9edd-525400cecc16
```

## 第四部分：日志自动清理

我们知道日志都存储在elastic集群中，且日志每天被分割成一个index，例如：

在容器中输入指令：
```bash
/ # curl elasticsearch-logging:9200/_cat/indices?v
```

输出结果：
```ini
health status index               uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   logstash-2019.04.29 ejMBlRcJQvqK76xIerenYg   5   1      69864            0     65.9mb         32.9mb
green  open   logstash-2019.04.28 hacNCuQVTQCUL62Sl8avOA   5   1      17558            0     21.3mb         10.6mb
green  open   .kibana_1           MVjF8lQeRDeKfoZcDhA93A   1   1          2            0     30.1kb           15kb
green  open   logstash-2019.05.05 m2aD8X9RQ3u48DvVq18x_Q   5   1      31218            0     34.4mb         17.2mb
green  open   logstash-2019.05.01 66OjwM5wT--DZaVfzUdXYQ   5   1      50610            0     54.6mb         27.1mb
green  open   logstash-2019.04.30 L3AH165jT6izjHHa5L5g0w   5   1      56401            0     55.5mb         27.8mb
...
```

因此 EFK 中的日志自动清理，只要定时去删除 es 中的 index 即可，如下命令

```bash
$ curl -X DELETE elasticsearch-logging:9200/logstash-xxxx.xx.xx
```

基于 alpine:3.8 创建镜像`es-index-rotator` [查看Dockerfile](../../dockerfiles/es-index-rotator/Dockerfile)，然后创建一个cronjob去完成清理任务

```bash
$ kubectl apply -f /etc/ansible/manifests/efk/es-index-rotator/
```

### 验证日志清理

#### 查看 cronjob

输入指令：
```bash
$ kubectl get cronjob -n kube-system
```

输出结果：
```ini
NAME               SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
es-index-rotator   3 1 */1 * *   False     0        19h             20h
```

#### 查看日志清理情况

输入指令：
```bash
kubectl get pod -n kube-system |grep es-index-rotator
```

输出结果：
```ini
es-index-rotator-1557507780-7xb89             0/1     Completed   0          19h
```

```bash
# 查看日志，可以了解日志清理情况
kubectl logs -n kube-system es-index-rotator-1557507780-7xb89 es-index-rotator
```

HAVE FUN!

## 参考

1. [EFK配置](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch)
1. [nfs-client-provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)
1. [persistent-volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
1. [storage-classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
1. [使用 ELK 查看 Ceph 读写速度](https://yq.aliyun.com/articles/584443)
