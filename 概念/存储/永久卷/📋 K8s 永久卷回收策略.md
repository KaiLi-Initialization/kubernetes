在 **Kubernetes (K8s)** 中，**永久卷（Persistent Volume, PV）** 的**回收策略（Reclaim Policy）** 决定了当与之绑定的 **永久卷声明（Persistent Volume Claim, PVC）** 被删除后，K8s 对 PV 的处理方式。

------

### 📋 **K8s 永久卷回收策略概述**

Kubernetes 支持以下三种回收策略：

| 回收策略    | 说明                                                   |
| ----------- | ------------------------------------------------------ |
| **Retain**  | **保留**卷及其数据，需手动清理和再利用。               |
| **Delete**  | **删除**永久卷及其相关存储资源（适用于动态供应的卷）。 |
| **Recycle** | 对永久卷执行**基本清理**操作（已弃用，不建议使用）。   |

------

### 🛠️ **1. Retain（保留）**

- **行为**：当 PVC 被删除后，PV 和存储中的数据会被保留，需要管理员手动处理和回收。

- 适用场景：

  - 需要保护数据，防止数据意外丢失。
  - 适用于对数据安全要求高的场景（如数据库备份）。

- 操作指南：

  1. 删除 PVC 后，PV 状态会变为 `Released`（表示已解除绑定，但数据未清理）。

  2. 管理员可以手动清理数据，并通过以下方式将 PV 设置为可重新使用：

     ```bash
     kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'
     ```

------

### 🗑️ **2. Delete（删除）**

- **行为**：当 PVC 被删除后，K8s 会自动删除 PV 以及与之关联的后端存储资源（如 AWS EBS、GCE PD、Azure Disk 等）。
- 适用场景：
  - 动态供应的卷（通过 StorageClass 自动创建的 PV）。
  - 临时数据，使用完即可清理，节省资源。
- 注意事项：
  - 适用于**云存储**（如 AWS、GCP、Azure）等支持动态供应的环境。
  - **无法恢复**被删除的卷及其数据。

------

### 🔄 **3. Recycle（回收，已弃用）**

> ⚠️ **从 Kubernetes 1.20 版本起已被弃用**，不推荐使用，取而代之的是动态供应（Dynamic Provisioning）。

- **行为**：执行基本的 `rm -rf /` 清理数据，PV 可被重新使用。
- 适用场景：
  - 早期环境或遗留系统。
  - 仅限于 NFS 和 HostPath 类型的卷。

------

### 📊 **示例：设置 K8s PV 回收策略**

#### 1️⃣ 创建一个 PV，使用不同的回收策略：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  # 设置回收策略
  hostPath:
    path: "/mnt/data"
```

- 可选值：
  - `Retain`
  - `Delete`
  - `Recycle`（已弃用）

------

#### 2️⃣ 查看已有 PV 及其回收策略：

```bash
kubectl get pv
```

输出示例：

```bash
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                 STORAGECLASS   AGE
example-pv   10Gi       RWO            Retain           Bound    default/my-pvc        manual         5m
```

------

#### 3️⃣ 修改已存在 PV 的回收策略：

```bash
kubectl patch pv example-pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
```

------

### 📌 **选择合适的回收策略**

- **Retain**：数据敏感或需要保留重要数据时，适合**生产环境**。
- **Delete**：自动清理数据，适合**临时数据或测试环境**。
- **Recycle（不推荐）**：已弃用，尽量避免使用，改用**动态供应**。

------

