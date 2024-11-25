**官方文档：**https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/service-resources/ingress-class-v1/

在 Kubernetes 中，`IngressClass` 是用于指定 **Ingress Controller** 的一种资源类型。它允许用户指定哪些 Ingress Controller 处理某些 Ingress 资源。这是在 Kubernetes 1.18 版本中引入的，旨在更好地支持多种 Ingress Controller 的场景，从而提高了集群中多个控制器共存和管理的灵活性。

### 1. **什么是 IngressClass？**

`IngressClass` 是 Kubernetes 中定义和标识特定 Ingress Controller 的一种资源类型。它通过标签（例如，`controller` 字段）将某个 `Ingress` 资源与特定的 Ingress Controller 关联，从而决定该 `Ingress` 资源由哪个 Ingress Controller 来处理。

通过使用 `IngressClass`，集群管理员可以指定不同类型的 Ingress 由不同的 Ingress Controller 处理，这对于在一个集群中运行多个 Ingress Controller（例如，Nginx、Traefik、HAProxy 等）非常有用。

### 2. **IngressClass 的结构**

`IngressClass` 资源包括以下主要字段：

- **apiVersion**：表示 API 的版本，通常是 `networking.k8s.io/v1`。
- **kind**：该字段始终为 `IngressClass`，表示这是一个 `IngressClass` 资源对象。
- **metadata**：包含资源的元数据，如名称、命名空间等。
- **spec**：包含 `IngressClass` 的配置，包括控制器的名称和其他选项。

#### 示例：IngressClass 定义
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx-ingress-class
spec:
  controller: "k8s.io/ingress-nginx"
```

在这个示例中，`IngressClass` 资源的名称为 `nginx-ingress-class`，并且指定了一个控制器（`controller` 字段），值为 `k8s.io/ingress-nginx`。这个控制器是 Nginx Ingress Controller 的标识。

### 3. **IngressClass 的工作原理**

当你创建一个 `Ingress` 资源时，可以通过 `ingressClassName` 字段来指定该 `Ingress` 资源应该由哪个 Ingress Controller 处理。如果没有指定 `ingressClassName`，则 Kubernetes 将使用默认的 Ingress Controller。

- **IngressClass 的选择**：`IngressClass` 会被 `Ingress` 资源的 `ingressClassName` 字段引用。当你指定了 `ingressClassName` 时，Kubernetes 会通过该名称来找到对应的 `IngressClass` 资源，并由相应的 Ingress Controller 处理该 `Ingress` 资源。
- **默认 IngressClass**：如果一个 `Ingress` 资源没有显式指定 `ingressClassName`，那么 Kubernetes 会使用默认的 Ingress Controller。在很多情况下，Ingress Controller 会将自己注册为默认的控制器，并且在 `IngressClass` 中会有 `ingressclass.k8s.io/is-default-class` 标签标记为 `true`。

#### 示例：Ingress 资源使用 `IngressClass`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx-ingress-class
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

在这个例子中，`Ingress` 资源使用了 `ingressClassName: nginx-ingress-class`，因此这个 `Ingress` 将由名为 `nginx-ingress-class` 的 Ingress Controller 来处理。

### 4. **IngressClass 的默认控制器**

在一个 Kubernetes 集群中，可以有多个 `IngressClass`，每个 `IngressClass` 对应一个 Ingress Controller。当集群管理员没有显式指定 `ingressClassName` 时，Kubernetes 会选择一个默认的 Ingress Controller 来处理 `Ingress` 资源。

- **默认 IngressClass**：通常，一个 Ingress Controller 会注册自己为默认的 `IngressClass`，并且在 `IngressClass` 资源的元数据中添加 `ingressclass.k8s.io/is-default-class: "true"` 标签。
- **没有指定 `ingressClassName` 时的行为**：如果 `Ingress` 资源没有指定 `ingressClassName`，那么它会自动选择一个被标记为默认的 `IngressClass` 来处理。

#### 示例：标记默认 IngressClass
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx-ingress-class
  labels:
    ingressclass.k8s.io/is-default-class: "true"
spec:
  controller: "k8s.io/ingress-nginx"
```

在这个例子中，`nginx-ingress-class` 被标记为默认的 `IngressClass`，意味着如果 `Ingress` 没有指定 `ingressClassName`，Kubernetes 会自动选择由 `nginx-ingress-class` 所代表的 Nginx Ingress Controller 来处理流量。

### 5. **为什么需要 IngressClass？**
`IngressClass` 的引入为 Kubernetes 提供了以下几个主要的好处：

- **支持多个 Ingress Controller**：一个集群中可以同时运行多个不同的 Ingress Controller，例如 Nginx、Traefik 或 HAProxy。通过 `IngressClass`，管理员可以指定哪个 `Ingress` 资源由哪个 Ingress Controller 来处理。
- **提高灵活性**：可以在同一个集群中，根据需要为不同的应用程序、命名空间或服务选择不同的 Ingress Controller。
- **集群管理**：多个 Ingress Controller 可以帮助实现更细粒度的流量管理策略。例如，可以为某些特定的应用或服务选择特定的负载均衡器或流量路由策略。

### 6. **如何管理多个 Ingress Controller**

通过结合使用多个 `IngressClass` 和 `Ingress` 资源，管理员可以将流量路由到不同的 Ingress Controller，具体流程如下：

1. **创建多个 IngressClass**：为每个 Ingress Controller 创建一个对应的 `IngressClass`。
2. **创建 Ingress 资源**：在每个 `Ingress` 资源中指定 `ingressClassName` 字段，以告诉 Kubernetes 应使用哪个 Ingress Controller。
3. **灵活管理流量**：可以将不同的应用或服务流量路由到不同的 Ingress Controller，从而根据负载、性能或其他需求优化流量管理。

#### 示例：为不同应用选择不同的 Ingress Controller
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app1-ingress
spec:
  ingressClassName: nginx-ingress-class
  rules:
    - host: app1.example.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: app1-service
                port:
                  number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app2-ingress
spec:
  ingressClassName: traefik-ingress-class
  rules:
    - host: app2.example.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: app2-service
                port:
                  number: 80
```

在这个示例中：
- `app1-ingress` 使用 `nginx-ingress-class` 作为 Ingress Controller 处理流量。
- `app2-ingress` 使用 `traefik-ingress-class` 作为 Ingress Controller 处理流量。

### 7. **总结**
`IngressClass` 是 Kubernetes 1.18 中引入的一个资源类型，它为集群提供了更灵活的流量管理方式。通过 `IngressClass`，可以为不同的 `Ingress` 资源指定不同的 Ingress Controller，这对于运行多个 Ingress Controller 的集群非常有用。通过灵活地管理 `IngressClass`，Kubernetes 用户可以实现更加精细的流量控制、负载均衡以及服务路由策略，满足多种场景下的需求。