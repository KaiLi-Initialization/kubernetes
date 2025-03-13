在 **Kubernetes (K8s)** 中，**持久卷申领（Persistent Volume Claim，PVC）** 是用户对存储资源的请求，类似于向 Kubernetes 提交的“**我要使用多大、什么类型的存储**”的申请。K8s 通过 PVC 动态或静态绑定相应的 **持久卷（Persistent Volume，PV）**，以提供持久化存储。

------

## 📋 **1. 持久卷 (PV) 与持久卷申领 (PVC) 的关系**

- **PV**：由管理员或 Kubernetes 动态创建，表示存储资源，存储数据的实际物理介质（如云盘、NFS、Local Path 等）。
- **PVC**：用户提交的申请，描述所需存储的**大小**、**访问模式**、**存储类**等，K8s 会找到与之匹配的 PV 并进行绑定。

> 🎯 **PVC 和 PV 是一对一的绑定关系，PVC 被删除后，PV 根据回收策略决定如何处理。**

------

## 📐 **2. PVC 生命周期**

1. **创建 PVC**：用户提交 PVC 请求，定义存储需求（容量、访问模式等）。
2. **绑定 PV**：K8s 根据 PVC 匹配合适的 PV，完成绑定 (`Bound` 状态)。
3. **使用 PV**：Pod 挂载 PVC，读取和写入数据。
4. **释放 PV**：PVC 删除后，PV 状态变为 `Released`，等待回收。
5. **回收 PV**：根据 PV 的**回收策略（Reclaim Policy）**，执行删除、保留、回收操作。

------

## 🛠️ **3. 创建 PVC 示例**

### 1️⃣ **静态绑定：手动创建 PV 和 PVC**

#### ① **创建 PV**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manual
spec:
  capacity:
    storage: 5Gi                   # 存储容量
  accessModes:
    - ReadWriteOnce                 # 访问模式（RWO、ROX、RWX）
  persistentVolumeReclaimPolicy: Retain  # 回收策略：Retain、Delete、Recycle（已弃用）
  hostPath:
    path: "/mnt/data"               # 本地路径（测试用，生产环境常用 NFS、云盘）
```

**创建 PV：**

```bash
kubectl apply -f pv.yaml
```

#### ② **创建 PVC**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-manual
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi                  # 请求 5Gi 存储
```

**创建 PVC：**

```bash
kubectl apply -f pvc.yaml
```

#### ③ **检查 PVC 和 PV 状态**

```bash
kubectl get pv
kubectl get pvc
```

> **注意**：K8s 会自动将 PVC 和 PV 绑定，若没有符合要求的 PV，PVC 将保持 `Pending` 状态。

------

### 2️⃣ **动态绑定：使用 StorageClass 自动创建 PV**

如果启用了**动态供应**，Kubernetes 会根据 PVC 自动创建 PV，不需要手动定义 PV。

#### ① **创建 StorageClass**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: my-storageclass
provisioner: kubernetes.io/aws-ebs  # 云环境下的存储供应商（AWS、GCE、NFS、Local）
parameters:
  type: gp2                         # AWS EBS 类型
reclaimPolicy: Delete               # PVC 删除时自动清理 PV
```

**创建 StorageClass**：

```bash
kubectl apply -f storageclass.yaml
```

#### ② **创建 PVC，使用 StorageClass**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: my-storageclass  # 使用指定的 StorageClass
```

**创建 PVC：**

```bash
kubectl apply -f pvc-dynamic.yaml
```

> ✅ **动态供应**适用于云环境（如 AWS、Azure、GCP）和支持动态存储的插件。

------

### 3️⃣ **在 Pod 中使用 PVC**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-container
      image: nginx
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: my-volume
  volumes:
    - name: my-volume
      persistentVolumeClaim:
        claimName: pvc-manual   # 引用已存在的 PVC
```

**创建 Pod：**

```bash
kubectl apply -f pod.yaml
```

------

## 📊 **4. 查看 PVC 和 PV 状态**

### ✅ **检查 PVC 状态**

```bash
kubectl get pvc
```

示例输出：

```bash
NAME         STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS     AGE
pvc-manual   Bound    pv-manual    5Gi        RWO            manual           10m
```

### ✅ **检查 PV 状态**

```bash
kubectl get pv
```

示例输出：

```bash
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS     AGE
pv-manual    5Gi        RWO            Retain           Bound    default/pvc-manual  manual           10m
```

------

## 🔄 **5. PVC 和 PV 回收策略**

PV 在 PVC 释放后会根据回收策略执行相应操作：

| **回收策略** | **描述**                                        |
| ------------ | ----------------------------------------------- |
| **Retain**   | 保留数据，PV 进入 `Released` 状态，需手动清理。 |
| **Delete**   | 自动删除 PV 和其关联的物理存储资源。            |
| **Recycle**  | 简单清理（已废弃，推荐使用 `Delete` 替代）。    |

### ✅ **修改 PV 的回收策略**

```bash
kubectl patch pv pv-manual -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
```

------

## 📌 **6. PVC 绑定失败的排查方法**

1. **查看 PVC 和 PV 状态**

```bash
kubectl describe pvc <pvc-name>
kubectl describe pv <pv-name>
```

1. **常见原因和解决方案**

    | **问题**                       | **解决方法**                         | |--------------------------------|--------------------------------------| | PV 和 PVC 的容量不匹配         | 确保 `storage` 大小设置一致。         | | PV 和 PVC 的访问模式不匹配     | 确保 PV 和 PVC 使用相同的 `accessModes`。 | | 没有合适的 PV 可供绑定         | 手动创建符合要求的 PV，或使用动态供应。 | | StorageClass 不存在            | 确保 PVC 使用的 `storageClassName` 已创建。|

------

## 📣 **7. 总结**

- **静态绑定**：管理员创建 PV，用户申请 PVC。
- **动态绑定**：用户申请 PVC，K8s 自动创建 PV（需 StorageClass）。
- **PVC 生命周期**：创建 → 绑定 → 使用 → 释放 → 回收。
- **故障排查**：检查 PVC/PV 状态，确保配置匹配。

