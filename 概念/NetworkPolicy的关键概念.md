Kubernetes中的NetworkPolicy是一种资源，用于控制集群中Pod之间的网络流量。它允许你。这有助于通过隔离应用程序的不同部分和控制对外部服务的访问来增强安全性。

### NetworkPolicy的实现

NetworkPolicy通过[网络插件](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)来实现，方式如下

1. 基于Pod和namespace的 NetworkPolicy时，以label来设定哪些流量可以进入或离开与该算符匹配的 Pod。
2. 基于 IP 的 NetworkPolicy 时，我们基于 IP 组块（CIDR 范围）来定义策略

### NetworkPolicy的作用

1. 定义规则来决定哪些Pod可以相互通信以及允许什么样的流量
2. 定义规则来决定哪些Pod可以与外界通信以及允许什么样的流量

### NetworkPolicy的关键概念

1. **Pod选择器（Pod Selector）**：基于标签指定策略应用于哪些Pod。

2. 策略类型（Policy Types）

   - **Ingress**：控制进入选定Pod的流量。
   - **Egress**：控制从选定Pod流出的流量。

   ```shell
   网络策略是相加的,如果策略适用于 Pod 某一特定方向的流量，Pod 在对应方向所允许的连接是适用的网络策略所允许的集合。
   
   要允许从源 Pod 到目的 Pod 的某个连接，源 Pod 的出口策略和目的 Pod 的入口策略都需要允许此连接。
   ```

   

3. **规则（Rules）**：定义允许或拒绝的特定流量，基于IP地址、端口和协议等标准。

### NetworkPolicy的基本结构

以下是一个简单的NetworkPolicy示例，该策略仅允许来自具有特定标签的Pod的Ingress流量：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific-ingress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: allowed-app
    ports:
    - protocol: TCP
      port: 80
```

### 解释

- **apiVersion**：NetworkPolicy的API版本（例如，`networking.k8s.io/v1`）。

- **kind**：资源类型（`NetworkPolicy`）。

- **metadata**：策略的元数据，包括其名称和命名空间。

- spec

  ：策略的详细规格。

  - **podSelector**：基于标签选择应用此策略的Pod。

  - **policyTypes**：指定策略类型（Ingress, Egress或两者）。

  - ingress

    ：定义Ingress规则。

    - from

      ：指定允许Ingress流量的来源。

      - **podSelector**：基于标签选择源Pod。

    - **ports**：指定允许的Ingress流量的端口和协议。

### 使用场景

1. **隔离Pod**：限制不同应用程序组件或环境（例如，开发、测试、生产）之间的通信。
2. **控制访问**：仅允许特定服务或外部来源与应用程序通信。
3. **增强安全性**：通过限制网络暴露减少攻击面，防止未经授权的访问。

### 使用NetworkPolicy的技巧

- **从默认拒绝开始**：创建一个默认拒绝策略以阻止所有流量，然后显式允许必要的流量。
- **测试策略**：在开发环境中测试你的NetworkPolicy，在应用到生产环境之前确保其工作正常。
- **监控和审计**：定期监控和审计你的NetworkPolicy，以确保其有效且最新。

NetworkPolicy是保护Kubernetes网络的强大工具，理解如何有效使用它们可以显著增强应用程序的安全性和可靠性。

### 进阶示例

#### 基于Pod

##### 默认拒绝所有入站流量

```shell
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

这确保即使没有被任何其他 NetworkPolicy 选择的 Pod 仍将被隔离以进行入口。 此策略不影响任何 Pod 的出口隔离。

##### 允许从特定IP范围的Egress流量

```shell
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific-egress
  namespace: default
spec:
  podSelector:
    matchLabels:   # 指定允许哪些Pod对外访问
      app: my-app
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:       # 指定哪些IP范围的pod对外访问
        cidr: 192.168.1.0/24
    ports:           # 指定pod通过什么协议哪个端口对外访问
    - protocol: TCP
      port: 80
```

##### 允许所有访问流量

```shell
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  ingress:
  - {}
  policyTypes:
  - Ingress
```

有了这个策略，任何额外的策略都不会导致到这些 Pod 的任何入站连接被拒绝。 此策略对任何 Pod 的出口隔离没有影响。

##### 拒绝所有出向流量

```shell
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

此策略可以确保即使没有被其他任何 NetworkPolicy 选择的 Pod 也不会被允许流出流量。 此策略不会更改任何 Pod 的入站流量隔离行为。

##### 允许所有出向流量

```shell
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress
```

有了这个策略，任何额外的策略都不会导致来自这些 Pod 的任何出站连接被拒绝。 此策略对进入任何 Pod 的隔离没有影响。

##### 默认拒绝所有Ingress和Egress流量

```shell
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

此策略可以确保即使没有被其他任何 NetworkPolicy 选择的 Pod 也不会被允许入站或出站流量。

#### 基于namespace

##### 仅允许特定命名空间中的Pod访问

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-namespace-ingress
  namespace: default
spec:
  podSelector:
    matchLabels:  #指定哪些pod被访问
      app: my-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:  # 接受哪些namespace的pod访问
        matchLabels:
          name: allowed-namespace
```

##### 按标签选择多个名字空间

在这种情况下，你的 `Egress` NetworkPolicy 使用名字空间的标签名称来将多个名字空间作为其目标。 为此，你需要为目标名字空间设置标签。例如：

```shell
kubectl label namespace frontend namespace=frontend
kubectl label namespace backend namespace=backend
```

在 NetworkPolicy 文档中的 `namespaceSelector` 下添加标签。例如：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-namespaces
spec:
  podSelector:
    matchLabels:  # 指定哪些pod对外访问
      app: myapp
  policyTypes:
    - Egress
  egress:
    - to:
      - namespaceSelector:  # 去访问哪些namespace的pod
          matchExpressions:
            - key: namespace
              operator: In
              values: ["frontend", "backend"]
```

**说明：**

```shell
你不可以在 NetworkPolicy 中直接指定名字空间的名称。 你必须使用带有 `matchLabels` 或 `matchExpressions` 的 `namespaceSelector` 来根据标签选择名字空间。
```



通过理解和应用这些策略，你可以更好地控制和保护Kubernetes集群中的网络流量。



#### 常见场景

文档：https://github.com/ahmetb/kubernetes-network-policy-recipes

#### 声明网络策略

文档：https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/declare-network-policy/