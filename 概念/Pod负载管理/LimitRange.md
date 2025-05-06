## LimitRange简介

Kubernetes 中的 `LimitRange` 是一种限制策略，用于在命名空间级别设置默认资源请求和限制值，或者强制资源的最小/最大范围。

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: my-namespace
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: "1Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
```



**🧠 参数说明**

| 字段             | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| `type`           | 限制类型：通常是 `Container`（也有 `Pod` 和 `PersistentVolumeClaim`） |
| `max`            | 容器允许的最大资源使用量                                     |
| `min`            | 容器需要的最小资源请求                                       |
| `default`        | 如果用户没设置 limit，则自动使用这里的默认值（Pod的**request 不得大于 此limit**否则Pod无法部署） |
| `defaultRequest` | 如果用户没设置 request，则自动使用这里的默认值               |

**⚠️ 注意事项**

- `LimitRange` 只会在 Pod/容器**没有显式指定 request/limit 时**生效。
- 如果你已经设置了资源限制，`LimitRange` 不会覆盖这些设置。
- 通常与 `ResourceQuota` 搭配使用：`LimitRange` 控制单个容器或Pod的资源限制，`ResourceQuota` 控制整个 namespace 资源配额。
- 尽管你在 LimitRange 的配置文件中你没有声明默认值，默认值也会被自动创建，其值均与`limits.max`相同。
- 命名空间下容器的`request`不得小于`LimitRange`的`limits.min`，不得大于`LimitRange``limits.max`

------

## ✅ LimitRange 生效的条件：

`LimitRange` 在 Kubernetes 中只有在 **特定情况下**才会生效。简单来说，它在资源（如 Pod 或容器）**未完全指定资源限制时自动补全，或用于验证资源设置是否在允许范围内**。

1. **容器未显式设置 `resources.requests` 或 `resources.limits`**

- 这时，LimitRange 会补全默认值：
  - `defaultRequest`：如果用户没有设置 `resources.requests`，就用它补全。
  - `default`：如果用户没有设置 `resources.limits`，就用它补全。

👉 **这个补全发生在 API Server 的 Admission 阶段**，所以必须通过 API 创建的资源才会生效（比如 `kubectl apply` 或 CI/CD 系统）。

------

2. **容器设置了 requests/limits，但超出了 LimitRange 的 max/min 限制**

- `LimitRange` 也可以指定最大值 `max` 和最小值 `min`，用来做验证。
- 如果用户显式设置了资源，但超出这些范围，会被 API Server 拒绝。

------

3. **作用范围是 `Container` 类型的资源**

- `LimitRange` 支持三种 `type`：
  - `Container`：限制单个容器的资源（最常用）✅
  - `Pod`：限制整个 Pod 总资源（不常用）
  - `PersistentVolumeClaim`：限制 PVC 的最小/最大容量（用于 PVC）

------

## ❌ LimitRange 不生效的情况：

- 手动指定了所有 `requests` 和 `limits`，且在范围内 ✅ → LimitRange 不会做任何事。
- 创建资源时跳过 Admission 控制器（极少情况，比如某些测试或自定义集群）
- 使用的资源类型不支持（比如某些 CRD 或不受限资源）
- LimitRange 没有绑定到当前命名空间（它是 **namespace scoped** 的）

------

示例：什么时候 LimitRange 生效？

```yaml
# LimitRange 示例
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: dev
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    max:
      memory: 1Gi
    min:
      memory: 128Mi
    type: Container
```

用户提交这个 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  namespace: dev
spec:
  containers:
  - name: app
    image: nginx
    # 没有写 requests/limits
```

➡️ **LimitRange 会补全资源配置**，自动加上：

```yaml
resources:
  requests:
    memory: 256Mi
  limits:
    memory: 512Mi
```

------

