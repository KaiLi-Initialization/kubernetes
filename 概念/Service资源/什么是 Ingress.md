**官方文档：**https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/service-resources/ingress-v1/

在 **Kubernetes** 中，**Ingress** 是一种 API 资源，用于管理外部用户（通常是 HTTP 和 HTTPS 请求）如何访问集群内的服务。简单来说，**Ingress 是一种负载均衡器**，它通过 HTTP 和 HTTPS 路由规则控制外部流量如何进入 Kubernetes 集群并访问内部的服务。

Ingress 允许你在 Kubernetes 集群内定义 HTTP 和 HTTPS 路由规则，这些规则会根据请求的主机名、路径或其他参数来将请求转发到适当的服务上。通过使用 Ingress，用户可以方便地在集群外部访问应用程序，而无需为每个应用程序或服务创建单独的负载均衡器。

### 1. **Ingress 的基本概念**
Ingress 提供了一种集中管理访问集群内服务的方式，它通常依赖于 **Ingress Controller** 来实际处理流量的路由。**Ingress 本身只是定义了路由规则，而实际的流量转发和负载均衡是由 Ingress Controller 来完成的**。

#### 主要组件：
- **Ingress 资源**：定义流量如何被路由到集群内部的服务。
- **Ingress Controller**：是负责实现 Ingress 资源所定义的路由规则的控制器，通常是一个反向代理，如 Nginx、Traefik 或 HAProxy。

### 2. **Ingress 资源的结构**
Ingress 资源包括以下几个关键字段：

- **apiVersion**: 表示 API 版本。
- **kind**: 对象类型，通常是 `Ingress`。
- **metadata**: 资源的元数据，通常包含名称和命名空间。
- **spec**: 定义 Ingress 的详细配置，包括路由规则、TLS 配置等。

#### 示例：基本的 Ingress 配置
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /foo
            pathType: Prefix
            backend:
              service:
                name: foo-service
                port:
                  number: 80
          - path: /bar
            pathType: Prefix
            backend:
              service:
                name: bar-service
                port:
                  number: 80
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls-secret
```

在这个示例中：
- **host**：表示请求的主机名为 `myapp.example.com`。
- **paths**：定义了请求路径的路由规则，`/foo` 和 `/bar` 分别被路由到 `foo-service` 和 `bar-service` 服务的 80 端口。
- **tls**：启用了 TLS 加密，指定了一个名为 `myapp-tls-secret` 的 TLS 证书。

### 3. **Ingress 的工作原理**
Ingress 通过以下方式工作：

- **客户端请求**：外部客户端发送 HTTP/HTTPS 请求，Ingress Controller 根据请求的主机名（Host）和路径（Path）将流量路由到相应的 Kubernetes 服务。
- **Ingress Controller**：Ingress Controller 负责监听 Kubernetes API Server，查看集群中定义的 Ingress 资源，并根据这些资源配置流量路由规则。
- **反向代理**：Ingress Controller 实际上是一个反向代理，处理所有传入的流量，并将请求转发到正确的服务。

### 4. **常见的 Ingress Controller**
Ingress 资源本身并不会处理流量，实际的流量路由工作是由 **Ingress Controller** 来完成的。常见的 Ingress Controller 包括：

- **Nginx Ingress Controller**：一个流行的开源 Ingress Controller，基于 Nginx 反向代理。
- **Traefik**：另一个流行的 Ingress Controller，支持更多动态配置和自动化功能。
- **HAProxy**：高性能的 Ingress Controller，基于 HAProxy。
- **Istio**：一个更复杂的服务网格，提供丰富的流量管理功能，包括 Ingress 路由。

### 5. **Ingress 的路由规则**
Ingress 的路由规则允许你根据请求的不同特征（如主机名、路径等）来定义复杂的流量转发逻辑。

#### 主要路由规则：
- **Host**：基于请求的域名（如 `myapp.example.com`）来路由流量。
- **Path**：基于请求的路径（如 `/foo` 或 `/bar`）来路由流量。路径可以是精确匹配，也可以是前缀匹配。
- **PathType**：指定路径匹配的方式，有三种类型：
  - `Exact`：精确匹配路径。
  - `Prefix`：路径前缀匹配。
  - `ImplementationSpecific`：由 Ingress Controller 实现的路径匹配规则。
  
#### 示例：基于路径的路由规则
```yaml
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /foo
            pathType: Prefix
            backend:
              service:
                name: foo-service
                port:
                  number: 80
          - path: /bar
            pathType: Prefix
            backend:
              service:
                name: bar-service 
                port:
                  number: 80
```

在这个例子中，所有以 `/foo` 开头的请求会被转发到 `foo-service` 服务，而以 `/bar` 开头的请求会被转发到 `bar-service` 服务。

### 6. **TLS 和 SSL 配置**
Ingress 支持 HTTPS，允许你配置 TLS 证书，以确保通过安全连接访问服务。在 `spec.tls` 部分，你可以定义多个主机和对应的证书。

#### 示例：启用 TLS 加密
```yaml
spec:
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
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls-secret
```

- **secretName**：指向一个 Kubernetes Secret，Secret 中包含 TLS 证书和私钥。
- **hosts**：表示启用 TLS 的主机名。

### 7. **Ingress 的优点**
- **简化外部访问管理**：Ingress 将集群内服务的访问集中化，通过一个 Ingress 资源就可以定义多个服务的路由规则。
- **TLS 支持**：Ingress 可以直接管理服务的 HTTPS 访问，通过 TLS 配置提供加密连接。
- **灵活的路由规则**：支持基于主机名、路径、HTTP 请求头等条件的路由规则，允许复杂的流量管理。
- **减少外部负载均衡器**：通过 Ingress，多个服务可以共享一个外部访问点（例如，单一的 IP 地址或域名），无需为每个服务创建单独的负载均衡器。

### 8. **Ingress 和其他资源的关系**
- **Service**：Ingress 将流量路由到 Kubernetes 服务（Service）。每个 Ingress 路由规则都会指向一个 Service 的端口。
- **ConfigMap**：Ingress Controller 通常会使用 `ConfigMap` 来配置和定制其行为。例如，Nginx Ingress Controller 使用 `ConfigMap` 来管理 Nginx 的配置。

### 9. **常见问题和限制**
- **路径匹配问题**：不同的 Ingress Controller 对路径匹配的实现可能有所不同。例如，某些 Controller 对路径的尾部斜杠处理可能不一致。
- **负载均衡**：Ingress 本身并不直接提供负载均衡，但它依赖 Ingress Controller 来实现负载均衡功能。
- **性能考虑**：对于高负载或高流量的场景，确保选用高效的 Ingress Controller，如 Nginx 或 Traefik，并适当调整其配置。

### 10. **总结**
**Ingress** 是 Kubernetes 提供的一种管理外部 HTTP 和 HTTPS 流量访问集群内服务的机制。它通过定义灵活的路由规则，允许流量根据不同的请求特征（如主机名、路径）路由到不同的服务。Ingress 资源本身定义了路由规则，而实际的流量转发和负载均衡是由 Ingress Controller 来完成的。它为 Kubernetes 提供了集中化的流量管理解决方案，支持 TLS 加密、动态路由和负载均衡，并减少了为每个服务单独配置负载均衡器的需求。

