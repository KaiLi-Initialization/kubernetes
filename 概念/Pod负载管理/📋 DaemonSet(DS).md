### DaemonSet(DS)

**华为云参考文档：**https://support.huaweicloud.com/basics-cce/kubernetes_0017.html

DaemonSet类型的控制器可以保证在集群中的每一台（或指定）节点上都运行一个副本。**一般适用于日志收集、节点监控等场景**。也就是说，如果一个Pod提供的功能是**节点级别**的（每个节点都需要且只需要一个），那么这类Pod就适合使用DaemonSet类型的控制器创建。

![img](E:/GitHub/kubernetes/kubenetes-kubeadm笔记-旧/Kubenetes/pod详解-14)

DaemonSet控制器的特点：

- 每当向集群中添加一个节点时，指定的 Pod 副本也将添加到该节点上
- 当节点从集群中移除时，Pod 也就被垃圾回收了

下面先来看下DaemonSet的资源清单文件

```yaml
apiVersion: apps/v1 # 版本号
kind: DaemonSet # 类型       
metadata: # 元数据
  name: # ds名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: daemonset
spec: # 详情描述

  revisionHistoryLimit: 3 # 保留历史版本
  updateStrategy: # 更新策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxUnavailable: 1 # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
      
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

创建pc-daemonset.yaml，内容如下：

```yaml
apiVersion: apps/v1
kind: DaemonSet      
metadata:
  name: pc-daemonset
  namespace: dev
spec: 
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
```

```shell
# 创建daemonset
[root@k8s-master01 ~]# kubectl create -f  pc-daemonset.yaml
daemonset.apps/pc-daemonset created

# 查看daemonset
[root@k8s-master01 ~]#  kubectl get ds -n dev -o wide
NAME        DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE   AGE   CONTAINERS   IMAGES         
pc-daemonset   2        2        2      2           2        24s   nginx        nginx:1.17.1   

# 查看pod,发现在每个Node上都运行一个pod
[root@k8s-master01 ~]#  kubectl get pods -n dev -o wide
NAME                 READY   STATUS    RESTARTS   AGE   IP            NODE    
pc-daemonset-9bck8   1/1     Running   0          37s   10.244.1.43   node1     
pc-daemonset-k224w   1/1     Running   0          37s   10.244.2.74   node2      

# 删除daemonset
[root@k8s-master01 ~]# kubectl delete -f pc-daemonset.yaml
daemonset.apps "pc-daemonset" deleted
```



#### 关键点：

1. **没有 `nodeSelector` / `affinity` / `tolerations`**
    → DaemonSet 会默认调度到 **所有可调度的节点**。

2. **master 节点默认不调度**

   - 因为 master 节点带有 `node-role.kubernetes.io/master:NoSchedule` 污点。

   - 如果你也希望 Pod 运行在 master 节点，需要在 `template.spec` 里加上容忍规则：

     ```
     tolerations:
     - key: "node-role.kubernetes.io/master"
       effect: "NoSchedule"
     ```

3. **新加入的节点**

   - DaemonSet 会自动在新节点上创建 Pod，不需要手动操作。

#### 总结

DaemonSet 的本质是：
 👉 **在节点维度分发 Pod**，保证每个节点有一个副本，适合运行 **节点级守护进程**。

对比：

- **Deployment**：面向“应用副本数”，适合无状态应用。
- **StatefulSet**：面向“有状态应用”，适合有顺序和持久存储的应用。
- **DaemonSet**：面向“节点守护进程”，适合系统级服务。



### 管理集群守护进程

官方文档：https://kubernetes.io/zh-cn/docs/tasks/manage-daemon/



------

### 🔄 DaemonSet 的滚动更新机制

 在 **Kubernetes** 里，DaemonSet 也支持 **滚动更新**，原理和 Deployment 有点类似，但有一些特殊点。

1. **默认策略**

   - 在 `spec.updateStrategy.type` 中，默认是 `RollingUpdate`。
   - 逐个节点地删除旧 Pod 并创建新 Pod。

2. **更新方式**

   - 修改 DaemonSet 的 **Pod Template**（比如更换镜像版本），K8s 就会触发滚动更新。
   - 每次更新时，某个节点上的 Pod 会先被删除，再拉起新 Pod。

3. **可配置参数**（`RollingUpdate` 下才有用）：

   ```yaml
   updateStrategy:
     type: RollingUpdate
     rollingUpdate:
       maxUnavailable: 1
   ```

   - `maxUnavailable`：滚动更新时，**允许同时不可用的 Pod 数量**（可以是绝对值 `1`，也可以是百分比 `10%`）。
   - 默认是 `1`，意味着更新时只会“一个节点一个节点”地替换。

4. **OnDelete 策略**

   ```yaml
   updateStrategy:
     type: OnDelete
   ```

   - 表示 **不会自动滚动更新**。
   - 当 Pod 被手动删除时，才会用新的模板拉起新 Pod。
   - 适合对稳定性要求特别高的系统级组件。

------

##### ✅ 示例：滚动更新的 DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: log-agent
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2   # 一次最多允许2个节点的 Pod 不可用
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
      - name: log-agent
        image: myrepo/log-agent:v2.0   # 修改镜像版本 → 触发滚动更新
```

------

##### 🔍 查看滚动更新进度

```bash
kubectl rollout status daemonset log-agent -n kube-system
```

会显示每个节点 Pod 的更新状态。

------

##### 🚀 总结

- DaemonSet 默认支持 **滚动更新**，逐个节点替换 Pod。
- 可以用 `maxUnavailable` 控制并发更新数量。
- 如果不希望自动更新，可以改成 `OnDelete`。

------







####  DaemonSet vs Deployment 滚动更新对比

------

**📊 DaemonSet 滚动更新 vs Deployment 滚动更新**

| 特性                     | **DaemonSet**                                       | **Deployment**                                               |
| ------------------------ | --------------------------------------------------- | ------------------------------------------------------------ |
| **控制对象**             | 每个节点各 1 个 Pod（守护进程）                     | 全局副本数（应用实例）                                       |
| **默认更新策略**         | `RollingUpdate`（逐节点替换）                       | `RollingUpdate`（按副本比例替换）                            |
| **可选更新策略**         | `RollingUpdate` / `OnDelete`                        | `RollingUpdate` / `Recreate`                                 |
| **触发更新条件**         | 修改 Pod 模板（镜像、环境变量等）                   | 修改 Pod 模板（镜像、环境变量等）                            |
| **并发控制参数**         | `maxUnavailable`（允许同时下线的 Pod 数量，默认 1） | `maxUnavailable`（允许同时下线的 Pod 数量，默认 25%）和 `maxSurge`（可多起的 Pod 数量，默认 25%） |
| **更新顺序**             | 每个节点上的旧 Pod 被删除 → 新 Pod 拉起             | 先拉起新 Pod，再删旧 Pod（受 `maxUnavailable` 和 `maxSurge` 控制） |
| **适用场景**             | 节点级服务（日志收集、监控代理、CNI 插件等）        | 应用级服务（Web 服务、微服务等）                             |
| **回滚方式**             | `kubectl rollout undo daemonset <name>`             | `kubectl rollout undo deployment <name>`                     |
| **容忍 master 节点更新** | 需要显式 `tolerations`                              | 不涉及，按副本数调度                                         |

------

## 🔑 核心区别

- **Deployment**：保证“应用整体可用”，用副本数和流量控制更新节奏。
- **DaemonSet**：保证“每个节点有 Pod”，主要关心的是“逐节点替换”而不是“整体副本比例”。

------

