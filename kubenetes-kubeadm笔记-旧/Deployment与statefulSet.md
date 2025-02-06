## Deployment(Deploy)

**华为云参考文档：**https://support.huaweicloud.com/basics-cce/kubernetes_0014.html

**官网参考文档：**https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/

Deployment 是用来管理无状态应用的工作负载 API 对象。

为了更好的解决服务编排的问题，kubernetes在V1.2版本开始，引入了Deployment控制器。值得一提的是，这种控制器并不直接管理pod，而是通过管理ReplicaSet来间接管理Pod，即：Deployment管理ReplicaSet，ReplicaSet管理Pod。所以Deployment比ReplicaSet功能更加强大。

![img](E:/GitHub/kubernetes/kubenetes-kubeadm笔记-旧/Kubenetes/pod详解-10)

Deployment主要功能有下面几个：

- 支持ReplicaSet的所有功能
- 支持发布的停止、继续
- 支持滚动升级和回滚版本

Deployment的资源清单文件：

```yaml
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: deploy
spec: # 详情描述
  replicas: 3 # 副本数量
  
  revisionHistoryLimit: 3 # 保留历史版本(通过保留RS实现)，默认是10
  paused: false # 暂停部署，默认是false
  progressDeadlineSeconds: 600 # 部署超时时间（s），默认是600
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 30% # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
      
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

### 1 创建deployment

创建pc-deployment.yaml，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment      
metadata:
  name: deployment
  namespace: dev
spec: 
  replicas: 3
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
# 创建deployment
[root@k8s-master01 ~]# kubectl create -f pc-deployment.yaml --record=true
deployment.apps/pc-deployment created

# 查看deployment
# NAME 列出了名字空间中 Deployment 的名称。
# UP-TO-DATE 最新版本的pod的数量
# AVAILABLE  当前可用的pod的数量
# AGE 显示应用程序运行的时间
[root@k8s-master01 ~]# kubectl get deploy pc-deployment -n dev
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
pc-deployment   3/3     3            3           15s

# 查看rs
# 发现rs的名称是在原来deployment的名字后面添加了一个10位数的随机串
[root@k8s-master01 ~]# kubectl get rs -n dev
NAME                       DESIRED   CURRENT   READY   AGE
pc-deployment-6696798b78   3         3         3       23s

# 查看pod
[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6696798b78-d2c8n   1/1     Running   0          107s
pc-deployment-6696798b78-smpvp   1/1     Running   0          107s
pc-deployment-6696798b78-wvjd8   1/1     Running   0          107s
```

要查看 Deployment 上线状态，运行 **`kubectl rollout status deployment/nginx-deployment`**。

输出类似于：

```shell
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out
```



### 2 扩缩容

```shell
# 变更副本数量为5个
[root@k8s-master01 ~]# kubectl scale deploy pc-deployment --replicas=5  -n dev
deployment.apps/pc-deployment scaled

# 查看deployment
[root@k8s-master01 ~]# kubectl get deploy pc-deployment -n dev
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
pc-deployment   5/5     5            5           2m

# 查看pod
[root@k8s-master01 ~]#  kubectl get pods -n dev
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6696798b78-d2c8n   1/1     Running   0          4m19s
pc-deployment-6696798b78-jxmdq   1/1     Running   0          94s
pc-deployment-6696798b78-mktqv   1/1     Running   0          93s
pc-deployment-6696798b78-smpvp   1/1     Running   0          4m19s
pc-deployment-6696798b78-wvjd8   1/1     Running   0          4m19s

# 编辑deployment的副本数量，修改spec:replicas: 4即可
[root@k8s-master01 ~]# kubectl edit deploy pc-deployment -n dev
deployment.apps/pc-deployment edited

# 查看pod
[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6696798b78-d2c8n   1/1     Running   0          5m23s
pc-deployment-6696798b78-jxmdq   1/1     Running   0          2m38s
pc-deployment-6696798b78-smpvp   1/1     Running   0          5m23s
pc-deployment-6696798b78-wvjd8   1/1     Running   0          5m23s
```

**镜像更新**

deployment支持两种更新策略:**`重建更新`**和**`滚动更新`**,可以通过`strategy`指定策略类型,支持两个属性:

```markdown
strategy：指定新的Pod替换旧的Pod的策略， 支持两个属性：
  type：指定策略类型，支持两种策略
    Recreate：在创建出新的Pod之前会先杀掉所有已存在的Pod
    RollingUpdate：滚动更新，就是杀死一部分，就启动一部分，在更新过程中，存在两个版本Pod
  rollingUpdate：当type为RollingUpdate时生效，用于为RollingUpdate设置参数，支持两个属性：
    maxUnavailable：用来指定在升级过程中不可用Pod的最大数量，默认为25%。
    maxSurge： 用来指定在升级过程中可以超过期望的Pod的最大数量，默认为25%。
```

重建更新

1) 编辑pc-deployment.yaml,在spec节点下添加更新策略

```yaml
spec:
  strategy: # 策略
    type: Recreate # 重建更新
```

2) 创建deploy进行验证

```shell
# 变更镜像
[root@k8s-master01 ~]# kubectl set image deployment pc-deployment nginx=nginx:1.17.2 -n dev
deployment.apps/pc-deployment image updated

# 观察升级过程
[root@k8s-master01 ~]#  kubectl get pods -n dev -w
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-5d89bdfbf9-65qcw   1/1     Running   0          31s
pc-deployment-5d89bdfbf9-w5nzv   1/1     Running   0          31s
pc-deployment-5d89bdfbf9-xpt7w   1/1     Running   0          31s

pc-deployment-5d89bdfbf9-xpt7w   1/1     Terminating   0          41s
pc-deployment-5d89bdfbf9-65qcw   1/1     Terminating   0          41s
pc-deployment-5d89bdfbf9-w5nzv   1/1     Terminating   0          41s

pc-deployment-675d469f8b-grn8z   0/1     Pending       0          0s
pc-deployment-675d469f8b-hbl4v   0/1     Pending       0          0s
pc-deployment-675d469f8b-67nz2   0/1     Pending       0          0s

pc-deployment-675d469f8b-grn8z   0/1     ContainerCreating   0          0s
pc-deployment-675d469f8b-hbl4v   0/1     ContainerCreating   0          0s
pc-deployment-675d469f8b-67nz2   0/1     ContainerCreating   0          0s

pc-deployment-675d469f8b-grn8z   1/1     Running             0          1s
pc-deployment-675d469f8b-67nz2   1/1     Running             0          1s
pc-deployment-675d469f8b-hbl4v   1/1     Running             0          2s
```

滚动更新

1) 编辑pc-deployment.yaml,在spec节点下添加更新策略

```yaml
spec:
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate:
      maxSurge: 25% 
      maxUnavailable: 25%
```

2) 创建deploy进行验证

```shell
# 变更镜像
[root@k8s-master01 ~]# kubectl set image deployment pc-deployment nginx=nginx:1.17.3 -n dev 
deployment.apps/pc-deployment image updated

# 观察升级过程
[root@k8s-master01 ~]# kubectl get pods -n dev -w
NAME                           READY   STATUS    RESTARTS   AGE
pc-deployment-c848d767-8rbzt   1/1     Running   0          31m
pc-deployment-c848d767-h4p68   1/1     Running   0          31m
pc-deployment-c848d767-hlmz4   1/1     Running   0          31m
pc-deployment-c848d767-rrqcn   1/1     Running   0          31m

pc-deployment-966bf7f44-226rx   0/1     Pending             0          0s
pc-deployment-966bf7f44-226rx   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-226rx   1/1     Running             0          1s
pc-deployment-c848d767-h4p68    0/1     Terminating         0          34m

pc-deployment-966bf7f44-cnd44   0/1     Pending             0          0s
pc-deployment-966bf7f44-cnd44   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-cnd44   1/1     Running             0          2s
pc-deployment-c848d767-hlmz4    0/1     Terminating         0          34m

pc-deployment-966bf7f44-px48p   0/1     Pending             0          0s
pc-deployment-966bf7f44-px48p   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-px48p   1/1     Running             0          0s
pc-deployment-c848d767-8rbzt    0/1     Terminating         0          34m

pc-deployment-966bf7f44-dkmqp   0/1     Pending             0          0s
pc-deployment-966bf7f44-dkmqp   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-dkmqp   1/1     Running             0          2s
pc-deployment-c848d767-rrqcn    0/1     Terminating         0          34m

# 至此，新版本的pod创建完毕，就版本的pod销毁完毕
# 中间过程是滚动进行的，也就是边销毁边创建
```

滚动更新的过程：

![img](E:/GitHub/kubernetes/kubenetes-kubeadm笔记-旧/Kubenetes/pod详解-11)

镜像更新中rs的变化

```shell
# 查看rs,发现原来的rs的依旧存在，只是pod数量变为了0，而后又新产生了一个rs，pod数量为4
# 其实这就是deployment能够进行版本回退的奥妙所在，后面会详细解释
[root@k8s-master01 ~]# kubectl get rs -n dev
NAME                       DESIRED   CURRENT   READY   AGE
pc-deployment-6696798b78   0         0         0       7m37s
pc-deployment-6696798b11   0         0         0       5m37s
pc-deployment-c848d76789   4         4         4       72s
```

### 3 版本回退

deployment支持版本升级过程中的暂停、继续功能以及版本回退等诸多功能，下面具体来看.

kubectl rollout： 版本升级相关功能，支持下面的选项：

- status	显示当前升级状态
- history   显示 升级历史记录
- pause    暂停版本升级过程
- resume   继续已经暂停的版本升级过程
- restart    重启版本升级过程
- undo 回滚到上一级版本（可以使用--to-revision回滚到指定版本）

###### 版本确认

查看当前升级版本的状态

```shell
[root@k8s-master01 ~]# kubectl rollout status deploy pc-deployment -n dev
deployment "pc-deployment" successfully rolled out
```

查看升级历史记录

```shell
[root@k8s-master01 ~]# kubectl rollout history deploy pc-deployment -n dev
deployment.apps/pc-deployment
REVISION  CHANGE-CAUSE
1         kubectl create --filename=pc-deployment.yaml --record=true
2         kubectl create --filename=pc-deployment.yaml --record=true
3         kubectl create --filename=pc-deployment.yaml --record=true

# 可以发现有三次版本记录，说明完成过两次升级
# 仅当 Deployment 的 Pod 模板（.spec.template）发生更改时，才会创建新版本历史记录
```

`CHANGE-CAUSE` 的内容是从 Deployment 的 `kubernetes.io/change-cause` 注解复制过来的。 复制动作发生在修订版本创建时。你可以通过以下方式设置 `CHANGE-CAUSE` 消息：

- 使用 `kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1"` 为 Deployment 添加注解。
- 手动编辑资源的清单。

要查看修订历史的详细信息

```shell
[root@k8s-master01 ~]# kubectl rollout history deployment/pc-deployment --revision=2
```



###### 版本回滚

```shell
# 这里直接使用--to-revision=1回滚到了1版本， 如果省略这个选项，就是回退到上个版本，就是2版本
[root@k8s-master01 ~]# kubectl rollout undo deployment pc-deployment --to-revision=1 -n dev
deployment.apps/pc-deployment rolled back

# 查看发现，通过nginx镜像版本可以发现到了第一版
[root@k8s-master01 ~]# kubectl get deploy -n dev -o wide
NAME            READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         
pc-deployment   4/4     4            4           74m   nginx        nginx:1.17.1   

# 查看rs，发现第一个rs中有4个pod运行，后面两个版本的rs中pod未运行
# 其实deployment之所以可是实现版本的回滚，就是通过记录下历史rs来实现的，
# 一旦想回滚到哪个版本，只需要将当前版本pod数量降为0，然后将回滚版本的pod提升为目标数量就可以了
[root@k8s-master01 ~]# kubectl get rs -n dev
NAME                       DESIRED   CURRENT   READY   AGE
pc-deployment-6696798b78   4         4         4       78m
pc-deployment-966bf7f44    0         0         0       37m
pc-deployment-c848d767     0         0         0       71m
```



### 4 Pod-template-hash 标签

Deployment 控制器将 `pod-template-hash` 标签添加到 Deployment 所创建或收留的每个 ReplicaSet 。此标签用于确保 Deployment 的子 ReplicaSets 不重叠。

如下为两个不同的Deployment 控制器的标签情况：

```shell
[root@centos-master pod]# kubectl get pod --show-labels
NAME                              READY   STATUS    RESTARTS   AGE   LABELS
deploy-nginx-8599fc9455-5fvcp     1/1     Running   0          18m   app=deploy-nginx,pod-template-hash=8599fc9455
deploy-nginx-8599fc9455-pb9lt     1/1     Running   0          18m   app=deploy-nginx,pod-template-hash=8599fc9455
deploy-nginx-8599fc9455-wmhsx     1/1     Running   0          18m   app=deploy-nginx,pod-template-hash=8599fc9455
deploy-nginx01-6d86f69454-c8jgg   1/1     Running   0          48s   app=deploy-nginx01,pod-template-hash=6d86f69454
deploy-nginx01-6d86f69454-cjvl6   1/1     Running   0          48s   app=deploy-nginx01,pod-template-hash=6d86f69454
deploy-nginx01-6d86f69454-snmzd   1/1     Running   0          48s   app=deploy-nginx01,pod-template-hash=6d86f69454
[root@centos-master pod]# kubectl get rs --show-labels
NAME                        DESIRED   CURRENT   READY   AGE   LABELS
deploy-nginx-8599fc9455     3         3         3       18m   app=deploy-nginx,pod-template-hash=8599fc9455
deploy-nginx01-6d86f69454   3         3         3       58s   app=deploy-nginx01,pod-template-hash=6d86f69454
[root@centos-master pod]# kubectl get deployment --show-labels
NAME             READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
deploy-nginx     3/3     3            3           18m   app=deploy-nginx
deploy-nginx01   3/3     3            3           61s   app=deploy-nginx01

```



### 5 金丝雀发布

Deployment控制器支持控制更新过程中的控制，如“暂停(pause)”或“继续(resume)”更新操作。

比如有一批新的Pod资源创建完成后立即暂停更新过程，此时，仅存在一部分新版本的应用，主体部分还是旧的版本。然后，再筛选一小部分的用户请求路由到新版本的Pod应用，继续观察能否稳定地按期望的方式运行。确定没问题之后再继续完成余下的Pod资源滚动更新，否则立即回滚更新操作。这就是所谓的金丝雀发布。

```shell
# 更新deployment的版本，并配置暂停deployment
[root@k8s-master01 ~]#  kubectl set image deploy pc-deployment nginx=nginx:1.17.4 -n dev && kubectl rollout pause deployment pc-deployment  -n dev
deployment.apps/pc-deployment image updated
deployment.apps/pc-deployment paused

#观察更新状态
[root@k8s-master01 ~]# kubectl rollout status deploy pc-deployment -n dev　
Waiting for deployment "pc-deployment" rollout to finish: 2 out of 4 new replicas have been updated...

# 监控更新的过程，可以看到已经新增了一个资源，但是并未按照预期的状态去删除一个旧的资源，就是因为使用了pause暂停命令

[root@k8s-master01 ~]# kubectl get rs -n dev -o wide
NAME                       DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         
pc-deployment-5d89bdfbf9   3         3         3       19m     nginx        nginx:1.17.1   
pc-deployment-675d469f8b   0         0         0       14m     nginx        nginx:1.17.2   
pc-deployment-6c9f56fcfb   2         2         2       3m16s   nginx        nginx:1.17.4   
[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-5d89bdfbf9-rj8sq   1/1     Running   0          7m33s
pc-deployment-5d89bdfbf9-ttwgg   1/1     Running   0          7m35s
pc-deployment-5d89bdfbf9-v4wvc   1/1     Running   0          7m34s
pc-deployment-6c9f56fcfb-996rt   1/1     Running   0          3m31s
pc-deployment-6c9f56fcfb-j2gtj   1/1     Running   0          3m31s

# 确保更新的pod没问题了，继续更新
[root@k8s-master01 ~]# kubectl rollout resume deploy pc-deployment -n dev
deployment.apps/pc-deployment resumed

# 查看最后的更新情况
[root@k8s-master01 ~]# kubectl get rs -n dev -o wide
NAME                       DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         
pc-deployment-5d89bdfbf9   0         0         0       21m     nginx        nginx:1.17.1   
pc-deployment-675d469f8b   0         0         0       16m     nginx        nginx:1.17.2   
pc-deployment-6c9f56fcfb   4         4         4       5m11s   nginx        nginx:1.17.4   

[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6c9f56fcfb-7bfwh   1/1     Running   0          37s
pc-deployment-6c9f56fcfb-996rt   1/1     Running   0          5m27s
pc-deployment-6c9f56fcfb-j2gtj   1/1     Running   0          5m27s
pc-deployment-6c9f56fcfb-rf84v   1/1     Running   0          37s
```

**删除Deployment**

```shell
# 删除deployment，其下的rs和pod也将被删除
[root@k8s-master01 ~]# kubectl delete -f pc-deployment.yaml
deployment.apps "pc-deployment" deleted
```

## StatefulSet

StatefulSet 是用来管理有状态应用的工作负载 API 对象。StatefulSet 用来管理某 Pod 集合的部署和扩缩， 并为这些 Pod **提供持久存储和持久标识符**。

和 [Deployment](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/) 类似， StatefulSet 管理基于相同容器规约的一组 Pod。但和 Deployment 不同的是， **StatefulSet 为它们的每个 Pod 维护了一个有粘性的 ID。这些 Pod 是基于相同的规约来创建的， 但是不能相互替换：无论怎么调度，每个 Pod 都有一个永久不变的 ID。**



StatefulSet 控制器特点：

- 稳定的、唯一的网络标识符。

- 稳定的、持久的存储。

  ```shell
  当 Pod 或者 StatefulSet 被删除时，与 PersistentVolumeClaims 相关联的 PersistentVolume 并不会被删除。要删除它必须通过手动方式来完成。
  ```

  

- 有序的、优雅的部署和扩缩。

- 有序的、自动的滚动更新。



```shell
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: default
spec:
  minReadySeconds: 10        # 最小就绪时间， 默认值是 0，它指定新创建的 Pod 应该在没有任何容器崩溃的情况下运行并准备                                就绪，才能被认为是可用的。用于在使用滚动更新策略时检查滚动的进度。
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Retain
  podManagementPolicy: OrderedReady   # POD管理策略，默认为OrderedReady
  replicas: 7
  revisionHistoryLimit: 10   # 版本保存，用于版本滚动
  selector:
    matchLabels:  # 必须匹配 .spec.template.metadata.labels
      app: nginx
  serviceName: nginx    # 无头服务的名字
  template:
    metadata:
      labels:     # 必须匹配 .spec.selector.matchLabels
        app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          name: web
          protocol: TCP
      restartPolicy: Always       # 重启策略
      terminationGracePeriodSeconds: 10
  updateStrategy:         # 更新策略
    rollingUpdate:
      partition: 3
    type: RollingUpdate

```



### POD管理策略

```shell
[root@centos-master ~]# kubectl explain statefulSet.spec.podManagementPolicy
GROUP:      apps
KIND:       StatefulSet
VERSION:    v1

FIELD: podManagementPolicy <string>

DESCRIPTION:
    podManagementPolicy controls how pods are created during initial scale up,
    when replacing pods on nodes, or when scaling down. The default policy is
    `OrderedReady`, where pods are created in increasing order (pod-0, then
    pod-1, etc) and the controller will wait until each pod is ready before
    continuing. When scaling down, the pods are removed in the opposite order.
    The alternative policy is `Parallel` which will create pods in parallel to
    match the desired scale without waiting, and on scale down will delete all
    pods at once.
    
    Possible enum values:
     - `"OrderedReady"` will create pods in strictly increasing order on scale
    up and strictly decreasing order on scale down, progressing only when the
    previous pod is ready or terminated. At most one pod will be changed at any
    time.  # StatefulSet 的默认设置,有序管理
     - `"Parallel"` will create and delete pods as soon as the stateful set
    replica count is changed, and will not wait for pods to be ready or complete
    termination.  # 并行管理：

```

- Parallel：并行管理，让 StatefulSet 控制器并行的启动或终止所有的 Pod， 启动或者终止其他 Pod 前，无需等待 Pod 进入 Running 和 Ready 或者完全停止状态。 这个选项只会影响扩缩操作的行为，更新则不会被影响。
- OrderedReady：有序管理，是 StatefulSet 的默认设置。

yaml格式

```shell
spec:
  podManagementPolicy: OrderedReady
  
# 或者

spec:
  podManagementPolicy: Parallel

```

在 Kubernetes 中，`StatefulSet` 的 `podManagementPolicy` 是一个非常重要的配置项，它控制了 StatefulSet 中 Pod 的创建和删除行为。具体来说，`podManagementPolicy` 决定了 **Pod 的启动顺序** 和 **删除顺序**。

`podManagementPolicy` 主要有两个选项：

#### 1. **OrderedReady** (默认值)

- **描述**：Pod 会按照顺序启动和终止，保证每个 Pod 在启动和删除时都保持顺序。
- **启动顺序**：Pod 会按顺序启动，首先是 `myapp-0`，然后是 `myapp-1`，以此类推，直到所有 Pod 都启动完成。
- **删除顺序**：当删除 Pod 时，Pod 会按相反的顺序终止，首先删除 `myapp-N`（其中 N 为最后一个 Pod），然后是 `myapp-(N-1)`，依此类推，直到所有 Pod 删除完毕。
- **适用场景**：需要保持 Pod 顺序启动和停止的有状态应用，如数据库集群、缓存服务等。

**示例**：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp-statefulset
spec:
  podManagementPolicy: OrderedReady
  replicas: 3
  serviceName: "myapp"
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp-container
        image: myapp-image:latest
        ports:
        - containerPort: 80
```

在 `OrderedReady` 模式下，Kubernetes 会确保 Pod 按照顺序启动和终止。只有在前一个 Pod 完成启动（或终止）后，才会启动（或终止）下一个 Pod。

------

#### 2. **Parallel**

- **描述**：Pod 可以并行启动和删除，Kubernetes 不会按照顺序创建或删除 Pod，而是并行执行这些操作。
- **启动顺序**：Pod 会同时（并行）启动，不考虑顺序。这意味着 `myapp-0`、`myapp-1` 和 `myapp-2` 等 Pod 可能会同时启动。
- **删除顺序**：Pod 会同时（并行）删除，不保证 Pod 删除的顺序。
- **适用场景**：在某些情况下，可以使用 `Parallel` 模式提高 Pod 启动和删除的速度，尤其是当 Pod 启动/停止不会相互影响时，或者没有严格的顺序依赖。

**示例**：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp-statefulset
spec:
  podManagementPolicy: Parallel
  replicas: 3
  serviceName: "myapp"
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp-container
        image: myapp-image:latest
        ports:
        - containerPort: 80
```

在 `Parallel` 模式下，Kubernetes 会尝试同时创建或删除所有 Pod，这对于启动或停止不依赖顺序的服务来说是一个高效的方式。

------

####  3. **`podManagementPolicy` 对比**

| **特性**         | **OrderedReady**                                            | **Parallel**                                           |
| ---------------- | ----------------------------------------------------------- | ------------------------------------------------------ |
| **Pod 启动顺序** | 按顺序启动，确保每个 Pod 启动完成后再启动下一个             | Pod 并行启动，顺序不重要                               |
| **Pod 删除顺序** | 按顺序删除，确保每个 Pod 删除完成后再删除下一个             | Pod 并行删除，顺序不重要                               |
| **适用场景**     | 需要 Pod 顺序启动和删除的有状态应用（如数据库、分布式系统） | Pod 启动/停止不依赖顺序的应用，或希望加快启动/停止速度 |
| **默认值**       | `OrderedReady`（默认值）                                    | -                                                      |

------

#### **4. 选择合适的 `podManagementPolicy`**

- 选择 `OrderedReady`

  ：

  - 当你的应用依赖于 Pod 启动顺序时（例如：分布式数据库、消息队列等），`OrderedReady` 确保 Pod 按顺序启动和删除，以避免服务间的依赖性问题。

- 选择 `Parallel`

  ：

  - 当你的应用没有严格的顺序要求时，或者你希望加速 Pod 启动和停止的速度，可以使用 `Parallel` 模式。

#### **总结：**

- **`OrderedReady`** 保证 Pod 按顺序启动和终止，适用于需要稳定、顺序启动的有状态应用。
- **`Parallel`** 则允许 Pod 并行启动和删除，适用于启动/停止顺序不重要的场景，能提高启动速度。

根据你的应用特性，选择合适的 `podManagementPolicy` 可以帮助你更好地控制 Pod 的生命周期和管理。

### 更新策略（ updateStrategy）

statefulSet的更新策略有两种

- **OnDelete**：必须手动删除POD，然后依据`.spec.template`内容的变更触发控制器创建新POD。
- **RollingUpdate**：`RollingUpdate` 更新策略对 StatefulSet 中的 Pod 执行自动的滚动更新。这是默认的更新策略。

```shell
[root@centos-master ~]# kubectl explain statefulSet.spec.updateStrategy
GROUP:      apps
KIND:       StatefulSet
VERSION:    v1

FIELD: updateStrategy <StatefulSetUpdateStrategy>

DESCRIPTION:
    updateStrategy indicates the StatefulSetUpdateStrategy that will be employed
    to update Pods in the StatefulSet when a revision is made to Template.
    StatefulSetUpdateStrategy indicates the strategy that the StatefulSet
    controller will use to perform updates. It includes any additional
    parameters necessary to perform the update for the indicated strategy.
    
FIELDS:
  rollingUpdate	<RollingUpdateStatefulSetStrategy>
    RollingUpdate is used to communicate parameters when Type is
    RollingUpdateStatefulSetStrategyType.

  type	<string>
    Type indicates the type of the StatefulSetUpdateStrategy. Default is
    RollingUpdate.
    
    Possible enum values:
     - `"OnDelete"` triggers the legacy behavior. Version tracking and ordered
    rolling restarts are disabled. Pods are recreated from the StatefulSetSpec
    when they are manually deleted. When a scale operation is performed with
    this strategy,specification version indicated by the StatefulSet's
    currentRevision.
     - `"RollingUpdate"` indicates that update will be applied to all Pods in
    the StatefulSet with respect to the StatefulSet ordering constraints. When a
    scale operation is performed with this strategy, new Pods will be created
    from the specification version indicated by the StatefulSet's
    updateRevision.

```

yaml书写格式

```shell
updateStrategy:
      type: RollingUpdate
      rollingUpdate:
        partition: 0

# 或者

updateStrategy:
      type: OnDelete
```



### 滚动更新（RollingUpdate）

```shell
[root@centos-master ~]# kubectl explain statefulSet.spec.updateStrategy.rollingUpdate
GROUP:      apps
KIND:       StatefulSet
VERSION:    v1

FIELD: rollingUpdate <RollingUpdateStatefulSetStrategy>

DESCRIPTION:
    RollingUpdate is used to communicate parameters when Type is
    RollingUpdateStatefulSetStrategyType.
    RollingUpdateStatefulSetStrategy is used to communicate parameter for
    RollingUpdateStatefulSetStrategyType.
    
FIELDS:
  maxUnavailable	<IntOrString>
    The maximum number of pods that can be unavailable during the update. Value
    can be an absolute number (ex: 5) or a percentage of desired pods (ex: 10%).
    Absolute number is calculated from percentage by rounding up. This can not
    be 0. Defaults to 1. This field is alpha-level and is only honored by
    servers that enable the MaxUnavailableStatefulSet feature. The field applies
    to all pods in the range 0 to Replicas-1. That means if there is any
    unavailable pod in the range 0 to Replicas-1, it will be counted towards
    MaxUnavailable.

  partition	<integer>  # 分区滚动更新
    Partition indicates the ordinal at which the StatefulSet should be
    partitioned for updates. During a rolling update, all pods from ordinal
    Replicas-1 to Partition are updated. All pods from ordinal Partition-1 to 0
    remain untouched. This is helpful in being able to do a canary based
    deployment. The default value is 0.

```

yaml书写格式

```shell
updateStrategy:
      type: RollingUpdate
      rollingUpdate:
        partition: 3   # 只更新pod序号等于大于partition-1指定的pod,小于partition-1序号的POD不更新；若partition大                          于replica则POD不牵涉更新，默认值是0。

updateStrategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 3  # 指定一次滚动更新的POD数量，此数字必须小于replica，默认值是1。
                            #maxUnavailable 字段处于 Alpha 阶段，仅当 API 服务器启用了                                                    MaxUnavailableStatefulSet 特性门控时才起作用。
```



## Deployment 和 StatefulSet 的异同

`Deployment` 和 `StatefulSet` 都是 Kubernetes 中用于管理 Pod 的控制器，但它们在管理应用时有不同的设计目标和使用场景，特别是在处理 **有状态应用** 和 **无状态应用** 时的行为有所不同。

**主要区别总结：**

| 特性             | **Deployment**                                            | **StatefulSet**                                              |
| ---------------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| **用途**         | 主要用于无状态应用的部署和管理                            | 主要用于有状态应用的部署和管理                               |
| **Pod 标识符**   | Pod 没有稳定的标识符，Pod 会被动态调度                    | 每个 Pod 有一个稳定且唯一的标识符（如 `myapp-0`、`myapp-1`） |
| **存储（卷）**   | Pod 通常不会有持久化存储，存储会在 Pod 被销毁时丢失       | 每个 Pod 具有持久化存储，Pod 可以与特定的 `PersistentVolume` 绑定 |
| **Pod 启动顺序** | Pods 不保证启动顺序，所有 Pods 可以并行启动               | Pods 按顺序启动（`myapp-0` 会先启动，然后是 `myapp-1`，依此类推） |
| **网络标识符**   | Pod 没有稳定的 DNS 名称，Pod 的网络标识符不固定           | 每个 Pod 都有稳定的 DNS 名称，如 `myapp-0.myapp.svc.cluster.local` |
| **滚动更新**     | 支持滚动更新，Pod 会逐步被替换                            | 也支持滚动更新，但有更严格的顺序控制，确保 Pod 按顺序更新    |
| **副本数量**     | 副本数可以灵活调整，Pod 可以随时扩容或缩容                | 副本数固定且严格，Pod 的标识符是唯一且按顺序生成             |
| **适用场景**     | 适用于无状态服务，服务可以无序扩展并且不依赖于 Pod 的状态 | 适用于有状态服务，如数据库、缓存等，确保每个 Pod 都有自己的状态 |
| **删除行为**     | 删除 Pod 时不会保留数据，Pod 重新创建时会是全新的         | 删除 Pod 时，数据依然保留在持久化存储中，Pod 重新创建时会恢复之前的状态 |

------

**详细对比：**

1. **Pod 标识符和顺序性**

- Deployment
  - `Deployment` 中的 Pod 是无状态的，每个 Pod 都没有固定的标识符，Pod 会被动态调度和管理。
  - Pods 会按需启动，容器的副本数也会根据负载动态调整。
  - Pod 的启动和删除没有固定顺序，因此对于一些不依赖于顺序的应用（如 Web 服务）非常合适。
- StatefulSet
  - 每个 Pod 在 `StatefulSet` 中都会有一个稳定的标识符，像 `myapp-0`、`myapp-1` 等，确保每个 Pod 都能保持唯一性和顺序。
  - Pod 会按顺序启动和终止（`myapp-0` 会首先启动，然后是 `myapp-1`，依此类推），对于需要特定顺序启动的有状态应用非常有用。

2. **持久化存储**

- Deployment
  - `Deployment` 中的 Pods 通常没有持久化存储（例如，`Deployment` 中的容器会使用临时存储，当 Pod 被删除时，数据会丢失）。
  - 如果需要持久化存储，通常需要外部解决方案（如使用 `PersistentVolume` 和 `PersistentVolumeClaim` 来挂载存储）。
- StatefulSet
  - `StatefulSet` 中每个 Pod 都会自动创建一个持久化卷（`PersistentVolume`），确保即使 Pod 被删除和重新创建，它们依然能够访问到持久化的数据。
  - 每个 Pod 绑定到一个特定的持久化卷，这使得它们在删除后可以恢复之前的状态。

3. **网络标识符**

- Deployment
  - 在 `Deployment` 中，Pod 的 DNS 名称是临时的，每次 Pod 被重新调度或重新创建时，它的 DNS 名称都会发生变化。
- StatefulSet
  - 在 `StatefulSet` 中，每个 Pod 都有一个稳定的 DNS 名称，格式为 `pod-name.service-name`，如 `myapp-0.myapp.svc.cluster.local`。
  - 这对于有状态服务（如数据库、缓存服务）来说非常重要，因为它们通常需要通过稳定的网络标识符来访问和维护自己的状态。

4. **滚动更新与回滚**

- Deployment
  - 支持滚动更新功能，Pod 会逐个替换，保持服务的持续可用性。
  - 在滚动更新过程中，如果某个新版本的 Pod 出现问题，`Deployment` 支持回滚到旧版本，确保应用的可靠性。
- StatefulSet
  - 也支持滚动更新，但与 `Deployment` 不同，`StatefulSet` 会按顺序逐个更新 Pod，先停止一个 Pod，再启动一个新 Pod。这保证了在有状态应用中 Pod 顺序的稳定性。
  - `StatefulSet` 支持回滚功能，但回滚时也会严格按顺序恢复 Pod。

5. **删除 Pod**

- Deployment
  - 删除 Pod 时，Pod 会被立刻销毁，数据丢失（如果没有外部存储）。
  - 如果应用是无状态的，Pod 被销毁时不会有影响，新的 Pod 会被自动创建并替代它。
- StatefulSet
  - 删除 Pod 时，如果 Pod 关联了持久化存储，数据不会丢失（存储会保留）。
  - `StatefulSet` 会按顺序删除 Pod，确保不会影响应用的可用性。

**什么时候选择 `Deployment` 和 `StatefulSet`**

- **选择 `Deployment`**：
  - 如果应用是无状态的，Pod 之间没有依赖关系（例如：Web 应用、API 服务等）。
  - 如果你希望 Pod 数量能够灵活扩展，并且不需要持久化存储。
  - 如果你的应用可以容忍 Pod 的快速重启、删除和重新创建。
- **选择 `StatefulSet`**：
  - 如果应用是有状态的，Pod 之间有状态依赖，且需要持久化数据（例如：数据库、消息队列、缓存等）。
  - 如果应用需要保证每个 Pod 的稳定身份、网络标识符，并且 Pod 必须按顺序启动和终止。
  - 如果应用需要在 Pod 删除时保留数据，并且数据与特定 Pod 绑定。

**总结：**

- `Deployment` 适用于无状态应用，能够实现灵活扩展、滚动更新和回滚。
- `StatefulSet` 适用于有状态应用，能够保证 Pod 的稳定标识符、顺序启动、持久化存储等特性。

