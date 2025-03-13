

**官方文档：**https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/service-resources/endpoints-v1/

在 **Kubernetes** 中，`Endpoints` 是一个非常重要的资源对象，它与服务（Service）密切相关，负责定义与某个服务关联的实际 IP 地址和端口。具体来说，Kubernetes 的 `Endpoints` 资源表示某个服务（Service）背后所有 Pod 的网络地址，它充当了服务发现的角色，确保客户端请求能够正确地路由到后端的 Pod 上。

### 1. **什么是 Kubernetes Endpoints？**

在 Kubernetes 中，**Endpoints** 是与服务（Service）对象关联的资源，它们描述了一个服务的具体网络地址（IP 地址和端口号）。这些 Endpoints 通常指向 Kubernetes 集群内运行的 Pod。

每当 Kubernetes 中的 Pod 启动或终止，或者服务的后端 Pod 发生变化时，相关的 Endpoints 资源会自动更新，确保服务能够动态地发现并访问这些后端 Pod。

### 2. **Endpoints 的作用**

`Endpoints` 是 Kubernetes 服务发现的核心部分。当你创建一个 Kubernetes 服务（例如一个 ClusterIP、NodePort 或 LoadBalancer 类型的服务）时，Kubernetes 会自动创建一个与之关联的 `Endpoints` 资源。服务通过这些 Endpoints 获取背后实际 Pod 的 IP 地址和端口，进而实现流量转发。

- **服务发现**：Kubernetes 会自动维护服务与其后端 Pod 的对应关系，使得你可以通过服务名称来访问 Pod，而不需要关心 Pod 的实际 IP 地址。
- **动态更新**：当 Pod 启动、终止或发生其他变化时，`Endpoints` 会自动更新，以保证服务总是能够访问到可用的 Pod。

### 3. **Endpoints 的结构**

一个 Kubernetes `Endpoints` 对象包含以下主要字段：

- **metadata**：与 `Endpoints` 资源相关的元数据（如名称、命名空间等）。

- subsets

  ：表示与服务相关的所有网络地址和端口的集合。每个 subset 中通常包括以下内容：

  - **addresses**：后端 Pod 的 IP 地址。
  - **ports**：Pod 上暴露的端口信息。

  这些 addresses 和 ports 的组合构成了服务的后端实际地址。

#### 示例：

```
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
  namespace: default
subsets:
  - addresses:
      - ip: 10.244.0.1
      - ip: 10.244.0.2
    ports:
      - port: 8080
        protocol: TCP
  - addresses:
      - ip: 10.244.0.3
    ports:
      - port: 9090
        protocol: TCP
```

在上面的例子中，`my-service` 服务有三个后端 Pod。它们的 IP 地址分别是 `10.244.0.1`、`10.244.0.2` 和 `10.244.0.3`，它们分别监听 TCP 端口 `8080` 和 `9090`。

### 4. **Endpoints 和 Service 的关系**

Kubernetes 中的 `Service` 通过 `Endpoints` 资源来与后端的 Pod 进行关联。服务本身并不直接包含 Pod 的 IP 地址和端口信息，而是通过 `Endpoints` 来访问这些信息。

#### 创建一个 Service 和自动生成 Endpoints

假设你创建了一个简单的 Kubernetes 服务：

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  selector:
    app: my-app
  ports:
    - port: 8080
      targetPort: 80
      protocol: TCP
```

- 这个服务会自动查找所有具有标签 `app=my-app` 的 Pod，并将它们的 IP 地址和端口（在这里是 80）映射到服务的端口 8080。
- Kubernetes 会自动生成一个与 `my-service` 关联的 `Endpoints` 资源，并包含所有匹配 Pod 的 IP 地址和端口。

#### 手动管理 Endpoints

有时，你可能需要手动管理 `Endpoints`，例如，在某些特殊的配置或外部服务场景下。可以通过 `kubectl` 命令直接创建、更新或删除 `Endpoints`。

```
kubectl expose pod my-pod --name=my-service --port=8080 --target-port=80 --type=ClusterIP
```

上面的命令会为 `my-pod` 创建一个 `Service`，同时自动生成相应的 `Endpoints`。

### 5. **Endpoints 类型**

Kubernetes 支持几种类型的 `Service`，每种类型的服务背后可能会有不同的 Endpoints 配置方式：

- **ClusterIP**：默认类型，服务仅在集群内可访问，自动创建 Endpoints，表示服务的集群内部网络地址。
- **NodePort**：使服务可以通过集群节点的 IP 地址和某个端口访问，Endpoints 会同时显示 Pod 的地址和节点端口。
- **LoadBalancer**：当服务配置为 LoadBalancer 类型时，Kubernetes 会通过云平台的负载均衡器将流量引导到 Endpoints 上。
- **ExternalName**：通过外部 DNS 名称来访问外部服务，`Endpoints` 不包含 Pod 的 IP 地址，而是一个外部 DNS 名称。

### 6. **Endpoints 和 DNS**

Kubernetes 的 DNS 系统也会依赖于 `Endpoints` 资源。每当服务创建时，Kubernetes DNS 会通过查询 `Endpoints` 资源来解析服务的实际地址。DNS 会将服务名称（如 `my-service.default.svc.cluster.local`）映射到对应的 Pod IP 地址和端口。

### 7. **Endpoints 的使用场景**

- **服务发现**：Kubernetes 中的多个 Pod 可能会提供同一个服务，通过 `Endpoints` 确保流量能够正确分配到这些 Pod 上。
- **手动配置**：在一些复杂的场景中，你可能需要手动配置 `Endpoints`，例如，某些外部的服务，或者基于特定条件筛选的服务端点。
- **负载均衡**：`Endpoints` 作为服务的实际后端，结合 Kubernetes 内置的负载均衡机制，将流量均匀地分发到多个 Pod。

### 8. **查看和管理 Endpoints**

你可以通过以下命令来查看 Kubernetes 中的 Endpoints：

```
kubectl get endpoints
kubectl describe endpoints my-service
```

`kubectl describe` 命令会显示关于某个服务背后所有 Pod 的详细 Endpoints 信息。

### 总结

在 Kubernetes 中，`Endpoints` 资源是服务发现的核心组件，它定义了服务与后端 Pod 之间的网络关系。`Endpoints` 自动与服务同步，确保集群内外的客户端可以通过服务名称访问到实际的后端 Pod。通过 Kubernetes 内置的负载均衡和 DNS 系统，`Endpoints` 提供了动态、自动化的流量路由和服务发现机制。