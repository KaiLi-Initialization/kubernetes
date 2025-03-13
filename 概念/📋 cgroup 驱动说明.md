##  cgroup 驱动说明

1. 什么是cgroup

   Cgroup 是一个 Linux 内核特性，对一组进程的资源使用（CPU、内存、磁盘 I/O 和网络等）进行限制、审计和隔离。

2. cgroup 驱动分类

   可用的 cgroup 驱动有两个：

   - cgroupfs
   - systemd

3. cgroup在kubernetes里的应用

   - 容器运行时（docker或containerd）
   - kubelet（默认使用cgroupfs驱动）
   - kubeadm（1.22+版本默认值 systemd驱动）

   > kubelet 和底层容器运行时都需要对接控制组来强制执行 为 Pod 和容器管理资源 并为诸如 CPU、内存这类资源设置请求和限制。 故此 kubelet 和容器运行时需使用相同的 cgroup 驱动并且采用相同的配置。

   

4. 为什么选择systemd

   - ubuntu 系统，debian 系统，centos7 系统，都是使用 systemd 初始化系统的。

     > systemd 这边已经有一套 cgroup 管理器了，如果容器运行时和 kubelet 使用 cgroupfs，此时就会存在 cgroups 和 systemd 两种 cgroup 管理器。也就意味着操作系统里面存在两种资源分配的视图，当操作系统上存在 CPU，内存等等资源不足的时候，操作系统上的进程会变得不稳定。

   - kubeadm（1.22+版本默认值 systemd驱动）初始化集群时默认使用systemd驱动



## cgroup驱动的设置



### runtime

#### containerd

containerd文档：https://github.com/containerd/containerd/blob/main/docs/getting-started.md

containerd配置文件的路径 `/etc/containerd/config.toml` （默认不存在）

```shell
$ mkdir -p /etc/containerd/
$ containerd config default > /etc/containerd/config.toml 
```

在 Linux 上，containerd 的默认 CRI 套接字是 `/run/containerd/containerd.sock`。

### Cgroup 驱动程序

cri插件配置文档：https://github.com/containerd/containerd/blob/main/docs/cri/config.md

配置 containerd 以使用`systemd`驱动程序，请在中设置以下选项`/etc/containerd/config.toml`：

- 在 containerd 2.x 中

```
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
  SystemdCgroup = true
```

- 在 containerd 1.x 中

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

或使用命令

```shell
sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
```

除了 containerd 之外，您还必须配置`KubeletConfiguration`使用“systemd”cgroup 驱动程序。`KubeletConfiguration`通常位于`/var/lib/kubelet/config.yaml`：

```
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: "systemd"
```



### kubelet

参考文档：https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubelet-config-file/
参考文档：https://kubernetes.io/zh-cn/docs/reference/config-api/kubelet-config.v1beta1/

使用kubeadm安装集群时，kubelet会被kubeadm当成系统服务来管理，因为在版本 1.22 及更高版本中，如果用户没有在 KubeletConfiguration 中设置 cgroupDriver 字段， kubeadm 会将它设置为默认值 systemd。所以对基于 kubeadm 的安装， 我们推荐使用 systemd 驱动， 不推荐 kubelet 默认的 cgroupfs 驱动。



## 注意事项：

修改节点cgroup驱动需要将节点腾空再进行操作

清空一个节点（node）

参考文档：https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/safely-drain-node/

节点安全地逐出所有 Pod

kubectl drain 节点名称
命令文档：https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain

 kubectl drain 子命令自身实际上不清空节点上的 DaemonSet Pod 集合： DaemonSet 控制器（作为控制平面的一部分）会立即用新的等效 Pod 替换缺少的 Pod。 DaemonSet 控制器还会创建忽略不可调度污点的 Pod，这种污点允许在你正在清空的节点上启动新的 Pod。

kubectl drain --ignore-daemonsets <节点名称>

此时次节点已移除，后续操作不会影响集群



取消节点隔离


如果要在维护操作期间将节点留在集群中，则需要运行：

kubectl uncordon <node name>

然后告诉 Kubernetes，它可以继续在此节点上调度新的 Pod。



