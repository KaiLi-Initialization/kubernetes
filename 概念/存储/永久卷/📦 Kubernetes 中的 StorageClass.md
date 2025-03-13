### 📦 **Kubernetes 中的 StorageClass**

在 **Kubernetes (K8s)** 中，`StorageClass` 是一种定义**持久卷（PV）** 动态供应策略的资源对象，允许根据需要自动创建和管理持久化存储（Persistent Volume, PV）。它解决了手动创建 PV 的繁琐工作，适用于云存储、分布式存储等场景。

------

## 📋 **1. StorageClass 的作用**

1. **动态供应（Dynamic Provisioning）**：根据 PVC 的需求自动创建 PV。
2. **多种存储类型**：支持多种存储后端（如 AWS EBS、GCE Persistent Disk、NFS、Ceph、GlusterFS）。
3. **回收策略**：定义 PVC 删除后，PV 的处理方式（`Retain`、`Delete`、`Recycle`）。
4. **参数自定义**：通过参数传递存储相关的配置信息（如磁盘类型、IOPS、加密等）。
5. **多租户隔离**：使用不同的 StorageClass，满足不同用户、环境和性能需求。

------

## 📐 **2. StorageClass 的结构**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage  # StorageClass 名称
provisioner: kubernetes.io/aws-ebs  # 存储提供者
parameters:              # 存储配置参数
  type: gp2              # AWS EBS gp2 类型
reclaimPolicy: Delete    # PVC 删除时，自动清理 PV
volumeBindingMode: Immediate  # 立即绑定 PV
allowVolumeExpansion: true  # 支持存储扩容
```

### 📌 **字段解释**

| 字段                   | 含义                                               |
| ---------------------- | -------------------------------------------------- |
| `metadata.name`        | StorageClass 的名称                                |
| `provisioner`          | 存储供应商（决定使用何种存储）                     |
| `parameters`           | 传递给供应商的参数（如存储类型、IOPS）             |
| `reclaimPolicy`        | PV 的回收策略（`Retain`、`Delete`、`Recycle`）     |
| `volumeBindingMode`    | PV 绑定模式（`Immediate`、`WaitForFirstConsumer`） |
| `allowVolumeExpansion` | 是否允许 PVC 扩容（`true`、`false`）               |

------

## 📊 **3. 常用 StorageClass 供应商（Provisioner）**

| **云平台**     | **Provisioner 名称**           |
| -------------- | ------------------------------ |
| **AWS EBS**    | `kubernetes.io/aws-ebs`        |
| **GCE PD**     | `kubernetes.io/gce-pd`         |
| **Azure Disk** | `kubernetes.io/azure-disk`     |
| **Ceph RBD**   | `rbd.csi.ceph.com`             |
| **NFS**        | `nfs-client`                   |
| **OpenStack**  | `kubernetes.io/cinder`         |
| **Local Disk** | `kubernetes.io/no-provisioner` |

> **注意**：现代 Kubernetes 集群通常使用 **CSI（Container Storage Interface）** 驱动管理存储。

------

## 📦 **4. StorageClass 示例**

### ✅ **4.1 AWS EBS**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "50"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### ✅ **4.2 NFS（Network File System）**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: nfs-client
parameters:
  archiveOnDelete: "false" # 删除 PVC 时，是否保留数据
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

### ✅ **4.3 Local Persistent Volume**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

> 适用于本地存储，常用于数据库、高性能应用。

------

## 🔄 **5. StorageClass 与 PVC 结合使用**

### **1️⃣ 创建 PVC**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: aws-ssd
```

### **2️⃣ 查看 PVC 绑定状态**

```bash
kubectl get pvc
```

示例输出：

```bash
NAME          STATUS   VOLUME                                     CAPACITY  STORAGECLASS  AGE
dynamic-pvc   Bound    pvc-1234-abcd-5678                         10Gi      aws-ssd       5m
```

------

## 📊 **6. StorageClass 的管理命令**

### ✅ **1. 查看 StorageClass**

```bash
kubectl get storageclass
```

### ✅ **2. 查看 StorageClass 详情**

```bash
kubectl describe storageclass aws-ssd
```

### ✅ **3. 设置默认 StorageClass**

```bash
kubectl patch storageclass nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### ✅ **4. 删除 StorageClass**

```bash
kubectl delete storageclass aws-ssd
```

------

## 🔍 **7. StorageClass 参数解析**

### 📌 **回收策略（`reclaimPolicy`）**

| **值**    | **说明**                               |
| --------- | -------------------------------------- |
| `Retain`  | **保留** PV 及其数据，需手动删除。     |
| `Delete`  | **自动删除** PV 及存储资源。           |
| `Recycle` | **清空数据**，然后重新使用（已弃用）。 |

------

### 📌 **卷绑定模式（`volumeBindingMode`）**

| **模式**               | **说明**                                         |
| ---------------------- | ------------------------------------------------ |
| `Immediate`            | **立即**分配 PV（默认值）。                      |
| `WaitForFirstConsumer` | 等待**第一个 Pod** 调度到节点后，再动态分配 PV。 |

------

### 📌 **支持 PVC 动态扩容（`allowVolumeExpansion`）**

- **true**：允许 PVC 进行存储扩容。
- **false**：PVC 不允许扩容。

> **注意**：仅部分存储后端支持扩容，扩容后需删除 Pod 重新挂载。

------

## ✅ **8. 总结**

- **StorageClass** 实现**持久卷**的**动态供应**，简化存储管理。
- 通过设置 **`reclaimPolicy`** 决定 PV 回收策略。
- 选择适配的 **Provisioner** 支持多种存储后端（如 AWS、GCP、NFS）。
- 结合 **PVC**，按需自动分配和扩展存储资源。



## 默认存储类

在 Kubernetes 中，**默认 StorageClass** 是一个特殊的 `StorageClass`，用于在没有明确指定 `StorageClass` 的情况下，自动为动态存储卷（PV）提供存储资源。

当你创建一个 **PersistentVolumeClaim（PVC）** 并且没有指定 `storageClassName` 时，Kubernetes 会自动选择 **默认的 StorageClass** 来动态供应存储。只有在集群中定义了一个 `default` 标记的 `StorageClass` 时，Kubernetes 才会知道哪个 `StorageClass` 是默认的。

### 如何查看或设置默认的 `StorageClass`

#### 1. 查看当前的默认 `StorageClass`

可以通过以下命令来查看当前集群中的所有 `StorageClass` 以及哪个是默认的：

```bash
kubectl get storageclass
```

输出示例：

```bash
NAME                 PROVISIONER                    AGE
standard (default)   kubernetes.io/gce-pd           10d
fast                 kubernetes.io/aws-ebs          5d
```

在这个示例中，`standard` 是默认的 `StorageClass`，因为它带有 `(default)` 标签。

#### 2. 设置默认 `StorageClass`

如果你想设置一个特定的 `StorageClass` 为默认，可以使用 `kubectl patch` 命令来修改它。例如，假设我们想将 `fast` 设置为默认 `StorageClass`：

```bash
kubectl patch storageclass fast -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

如果想取消某个 `StorageClass` 的默认标记，可以使用以下命令：

```bash
kubectl patch storageclass fast -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

#### 3. 创建一个默认的 `StorageClass`

默认的 `StorageClass` 可以在 Kubernetes 配置文件中定义。以下是一个简单的 `StorageClass` 示例：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

在这个示例中，`standard` `StorageClass` 使用了 Google Cloud 的 **GCE Persistent Disk** 作为存储供应商，并且被标记为默认 `StorageClass`。

------

### 注意事项

- 如果集群中没有默认的 `StorageClass`，Kubernetes 在创建没有指定 `storageClassName` 的 PVC 时会失败。
- 通过设置 `annotations` 中的 `storageclass.kubernetes.io/is-default-class: "true"`，你可以指定一个特定的 `StorageClass` 为默认。
- 默认 `StorageClass` 在 PVC 没有显式指定时会自动应用。如果指定了 `storageClassName`，则会使用指定的 `StorageClass`。

### 总结

- **默认 `StorageClass`** 是 Kubernetes 自动选择的 `StorageClass`，用于动态供应存储。
- 通过 `kubectl get storageclass` 可以查看当前的默认 `StorageClass`。
- 可以使用 `kubectl patch` 命令设置或取消默认的 `StorageClass`。





