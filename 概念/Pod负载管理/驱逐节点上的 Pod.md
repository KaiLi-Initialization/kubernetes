在 Kubernetes 中移除节点（Node）通常分为以下几个步骤：

------

### 🚧 **1. 驱逐节点上的 Pod**

先把节点上运行的 Pod 驱逐到其他节点，确保服务不中断。

```bash
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data
```

参数解释：

- `--ignore-daemonsets`：忽略 DaemonSet 管理的 Pod（它们会自动在其他节点重建）
- `--delete-emptydir-data`：删除 `emptyDir` 卷挂载的数据（临时数据）
- `--force`（可选）：如果是 master 节点或者有 Pod 不能正常驱逐，可能需要加这个参数

------

### 🧯 **2. 确认节点已被设置为不可调度（cordon）**

`drain` 命令会自动 cordon 节点，但你也可以单独执行：

```bash
kubectl cordon <NODE_NAME>
```

这会防止新的 Pod 调度到该节点。

------

### ❌ **3. 从集群中删除节点**

```bash
kubectl delete node <NODE_NAME>
```

这将从 Kubernetes 集群的控制面（control plane）中移除该节点的记录。

> 注意：这不会关机或重启实际的服务器，只是从 Kubernetes 管理中移除。

------

