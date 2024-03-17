service是为pod群组提供网络服务的一种达到负载均衡方式。

- 在pod群组中，pod的ip地址是随机分配的，当某一pod被重新删除与创建后ip会出现变动，无法确保固定访问。

- service是为pod群组提供一个固定的ip地址，来解决外部访问pod的一种方式。

- service也是通告labels选择器来选择访问pod的

  ```yaml
  spec:
    selector:
    	app: nginx
  ```

- service与pod一样是一个对象，可以创建，删除，修改。

定义一个service

service示例

```shell
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```



端口定义

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80
        name: http-web-svc    # 定义pod所要暴漏端口的名字

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP               # 此处定义service所要暴漏pod的哪个协议的端口，支持：TCP,UDP,SCTP
    port: 80                    # 此处定义service自身对外暴漏的端口
    targetPort: http-web-svc    # 此处的"http-web-svc" 是pod.spec.containers.ports.name定义的pod端口名字，代表service所要暴漏pod的端口；也可直接使用端口“80”，如：targetPort: 80 
```

多端口定义

Kubernetes 允许你为 Service 对象配置多个端口定义。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```



服务类型

服务类型是指暴漏访问service的ip的方式，共有四种类型

```yaml
spec:
  type: NodePort
```

**ClusterIP**类型

仅在集群内部公开service的ip，此时service只能在集群内部访问，默认为**ClusterIP**类型。

无头服务

有时你并不需要负载均衡，也不需要单独的 Service IP。你可以将 Service 的 `.spec.clusterIP` 设置为 `"None"`，来创建无头服务，此时 Kubernetes 不会为其分配 IP 地址。

无头 Service 不会获得集群 IP，kube-proxy 不会处理这类 Service， 而且平台也不会为它们提供负载均衡或路由支持。 取决于 Service 是否定义了选择算符，DNS 会以不同的方式被自动配置。

- 带选择算符的服务

  对定义了选择算符的无头 Service，Kubernetes 控制平面在 Kubernetes API 中创建 EndpointSlice 对象，并且修改 DNS 配置返回 A 或 AAAA 记录（IPv4 或 IPv6 地址）， 这些记录直接指向 Service 的后端 Pod 集合。

- 不带选择算符的服务

  对没有定义选择算符的无头 Service，控制平面不会创建 EndpointSlice 对象。 然而 DNS 系统会执行以下操作之一：

  - 对于 [`type: ExternalName`](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#externalname) Service，查找和配置其 CNAME 记录；
  - 对所有其他类型的 Service，针对 Service 的就绪端点的所有 IP 地址，查找和配置 DNS A / AAAA 记录：
    - 对于 IPv4 端点，DNS 系统创建 A 记录。
    - 对于 IPv6 端点，DNS 系统创建 AAAA 记录。

  当你定义无选择算符的无头 Service 时，`port` 必须与 `targetPort` 匹配。



[`NodePort`](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#type-nodeport)类型

可以通过访问Node上的ip和静态端口，来访问Kubernetes 会为 Service 配置集群 IP 地址（相当于你请求了 `type: ClusterIP` 的服务），从而访问pod终端。

Kubernetes 控制平面将在 --service-node-port-range 标志所指定的范围内分配端口（默认值：30000-32767）。每个节点将该端口（每个节点上的相同端口号）上的流量代理到你的 Service。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    # 默认情况下，为了方便起见，`targetPort` 被设置为与 `port` 字段相同的值。
    - port: 80
      targetPort: 80
      # 可选字段
      # 默认情况下，为了方便起见，Kubernetes 控制平面会从某个范围内分配一个端口号
      #（默认：30000-32767）
      nodePort: 30007  # 此为可选字段，Kubernetes 控制平面会分配一个，也可在（默认：30000-32767）范围内手动指定。
```

NodePort 端口规划

为 NodePort 服务分配端口的策略既适用于自动分配的情况，也适用于手动分配的场景。

通过启用特性门控 `ServiceNodePortStaticSubrange`，进而为 NodePort Service 使用不同的**端口分配策略**。用于 NodePort 服务的端口范围被分为两段。 动态端口分配默认使用较高的端口段，并且在较高的端口段耗尽时也可以使用较低的端口段。 用户可以从较低端口段中分配端口，降低端口冲突的风险。

NodePort 服务访问限制

你可以配置集群中的节点使用特定 IP 地址来支持 NodePort 服务。 如果每个节点都连接到多个网络（例如：一个网络用于应用流量，另一网络用于节点和控制平面之间的流量）， 你可能想要这样做。

通过设置kube-proxy 的 `--nodeport-addresses` 标志或 [kube-proxy 配置文件](https://kubernetes.io/zh-cn/docs/reference/config-api/kube-proxy-config.v1alpha1/)中的等效字段 `nodePortAddresses`来指定特定网段用于 NodePort 服务。

 `--nodeport-addresses` 接受逗号分隔的 IP 段列表（例如 `10.0.0.0/8`、`192.0.2.0/25`），用来设置 IP 地址范围。

`--nodeport-addresses` 的默认值是一个空的列表。 这意味着 kube-proxy 将认为所有可用网络接口都可用于 NodePort 服务.





[`LoadBalancer`](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#loadbalancer)类型

使用第三方云平台的负载均衡器向外部公开 Service。

[`ExternalName`](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#externalname)类型

类型为 ExternalName 的 Service 将 Service 映射到 DNS 名称，而不是典型的选择算符， 例如 `my-service` 或者 `cassandra`。你可以使用 `spec.externalName` 参数指定这些服务。

例如，以下 Service 定义将 `prod` 名字空间中的 `my-service` 服务映射到 `my.database.example.com`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

如果你想要将服务直接映射到某特定 IP 地址，请考虑使用[无头服务](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#headless-services)。



服务发现

对于在集群内运行的客户端，Kubernetes 支持两种主要的服务发现模式：环境变量和 DNS。



流量策略

参考文档：https://kubernetes.io/zh-cn/docs/reference/networking/virtual-ips/#traffic-policies

你可以设置 `.spec.internalTrafficPolicy` 和 `.spec.externalTrafficPolicy` 字段来控制 Kubernetes 如何将流量路由到健康（“就绪”）的后端。

- 内部流量策略

  可以设置 `.spec.internalTrafficPolicy` 字段来控制来自内部源的流量如何被路由。 有效值为 `Cluster` 和 `Local`。 将字段设置为 `Cluster` 会将内部流量路由到所有准备就绪的端点， 将字段设置为 `Local` 仅会将流量路由到本地节点准备就绪的端点。 如果流量策略为 `Local` 但没有本地节点端点，那么 kube-proxy 会丢弃该流量。



- 外部流量策略

  你可以设置 `.spec.externalTrafficPolicy` 字段来控制从外部源路由的流量。 有效值为 `Cluster` 和 `Local`。 将字段设置为 `Cluster` 会将外部流量路由到所有准备就绪的端点， 将字段设置为 `Local` 仅会将流量路由到本地节点上准备就绪的端点。 如果流量策略为 `Local` 并且没有本地节点端点， 那么 kube-proxy 不会转发与相关 Service 相关的任何流量。



会话亲和性



