## 创建 NFS 服务器

NFS 允许系统将其目录和文件共享给网络上的其他系统。通过
NFS，用户和应用程序可以访问远程系统上的文件，就象它们是本地文件一样。
参考：https://blog.csdn.net/dengyadeng/article/details/79549632

### 安装
安装 NFS 服务器：

```{.python .input}
# for Ubuntu 16.04
apt install nfs-kernel-server
# for Centos 7
yum -y install nfs-utils
```

### 配置
编辑`/etc/exports`文件添加需要共享目录，每个目录的设置独占一行，编写格式如下：

`NFS共享目录路径
客户机IP或者名称(参数1,参数2,...,参数n)`

例如：

```bash
# 准备共享目录
mkdir -p /home/share
chown -R nfsnobody.nfsnobody /home/share
```

```{.python .input}
#/home/share *(ro,sync,insecure,no_root_squash)
/home/share 192.168.20.0/24(rw,sync,insecure,no_subtree_check,no_root_squash)
```

| 参数 | 说明 |
| :- | :- |
| ro | 只读访问 |
| rw | 读写访问 |
| sync | 所有数据在请求时写入共享 |
|
async | nfs在写入数据前可以响应请求 |
| secure | nfs通过1024以下的安全TCP/IP端口发送 |
| insecure |
nfs通过1024以上的端口发送 |
| wdelay | 如果多个用户要写入nfs目录，则归组写入（默认） |
| no_wdelay |
如果多个用户要写入nfs目录，则立即写入，当使用async时，无需此设置 |
| hide | 在nfs共享目录中不共享其子目录 |
| no_hide |
共享nfs目录的子目录 |
| subtree_check | 如果共享/usr/bin之类的子目录时，强制nfs检查父目录的权限（默认） |
|
no_subtree_check | 不检查父目录权限 |
| all_squash | 共享文件的UID和GID映射匿名用户anonymous，适合公用目录
|
| no_all_squash | 保留共享文件的UID和GID（默认） |
| root_squash |
root用户的所有请求映射成如anonymous用户一样的权限（默认） |
| no_root_squash | root用户具有根目录的完全管理访问权限 |
| anonuid=xxx | 指定nfs服务器/etc/passwd文件中匿名用户的UID |
| anongid=xxx |
指定nfs服务器/etc/passwd文件中匿名用户的GID |

+ 注1：尽量指定主机名或IP或IP段最小化授权可以访问NFS 挂载的资源的客户端
+
注2：经测试参数insecure必须要加，否则客户端挂载出错mount.nfs: access denied by server while mounting
### 启动

配置完成后，您可以在终端提示符后运行以下命令来启动 NFS 服务器：

```{.python .input}
# for Ubuntu 16.04
systemctl enable nfs-kernel-server
systemctl start nfs-kernel-server
# for Centos 7.x
systemctl enable nfs
systemctl start nfs
```

### 客户端挂载

```bash
# for Ubuntu 16.04
apt install nfs-common

# for CentOS 7
yum install
nfs-utils
```

使用 mount 命令来挂载其他机器共享的 NFS 目录。可以在终端提示符后输入以下类似的命令：

```{.python .input}
mkdir -p /var/share
mount -t nfs 192.168.20.201:/home/share /var/share
```

挂载点 /local/ubuntu 目录必须已经存在。而且在 /local/ubuntu 目录中没有文件或子目录。

另一个挂载NFS 共享的方式就是在
/etc/fstab 文件中添加一行。该行必须指明 NFS 服务器的主机名、服务器输出的目录名以及挂载 NFS 共享的本机目录。

以下是在
/etc/fstab 中的常用语法：

```{.python .input}
192.168.20.201:/home/share /var/share nfs rsize=8192,wsize=8192,timeo=14,intr
```
