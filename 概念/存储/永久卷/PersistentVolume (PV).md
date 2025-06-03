## 📌 PersistentVolume (PV) 

下面是一个标准的 Kubernetes **PersistentVolume (PV)** 的 YAML 模板，包含详细注释以说明各个字段的含义。

### 📌 **PersistentVolume (PV) YAML 示例**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example                # PV 的名称，必须唯一
  labels:
    type: local                   # 标签，用于与 PVC 进行匹配
spec:
  capacity:
    storage: 10Gi                 # PV 提供的存储容量，常用单位：Gi、Mi、Ti
  accessModes:
    - ReadWriteOnce               # 访问模式，详见下方说明
  persistentVolumeReclaimPolicy: Retain  # 回收策略，支持 Retain/Delete/Recycle
  storageClassName: manual        # 关联的 StorageClass 名称，PVC 需匹配该值
  mountOptions:                   # 挂载选项，取决于底层存储系统
    - hard
    - nfsvers=4.1
  volumeMode: Filesystem          # 存储卷模式 (Filesystem/Block)
  nfs:                            # 存储后端类型 (此处为 NFS 示例)
    path: /mnt/data               # NFS 服务器的共享路径
    server: 192.168.1.100         # NFS 服务器地址
  nodeAffinity:                   # 节点亲和性 (仅支持静态 PV)
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node-1          # 仅允许此 PV 在 node-1 节点上使用
```

------

📌 **主要字段说明**

| 字段                                 | 说明                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| `apiVersion`                         | API 版本，`v1` 为 PersistentVolume 对象的稳定版本。          |
| `kind`                               | 资源类型，这里是 `PersistentVolume`。                        |
| `metadata`                           | 元数据，包含名称、标签、注解等信息。                         |
| `spec.capacity.storage`              | 定义存储容量，单位常用 `Gi` (Gibibytes)。                    |
| `spec.accessModes`                   | 定义 PV 支持的访问模式，详见下面的访问模式说明。             |
| `spec.persistentVolumeReclaimPolicy` | 回收策略，决定 PVC 释放后 PV 的处理方式。                    |
| `spec.storageClassName`              | 与 PersistentVolumeClaim 绑定的 StorageClass 名称。          |
| `spec.mountOptions`                  | 存储卷挂载时的额外选项，通常用于特定的存储系统，如 NFS、Ceph。 |
| `spec.volumeMode`                    | 存储模式：`Filesystem` (文件系统) 或 `Block` (原始块设备)。  |
| `spec.nfs`                           | NFS 存储示例，其他存储类型如 `hostPath`、`cephfs` 亦支持。   |
| `spec.nodeAffinity`                  | 节点亲和性，限制 PV 只能被特定节点访问，适用于本地存储。     |

------

### 📌 **访问模式 (Access Modes)**

在 Kubernetes 中，**PersistentVolume (PV)** 必须显式指定 `accessModes`，否则该 PV **不会被接受或生效**。如果省略 `accessModes` 字段，Kubernetes 不会为该 PV 设置默认值，且该 PV 将无法与 **PersistentVolumeClaim (PVC)** 绑定。

✅ **为什么 PV 必须设置 `accessModes`？**

- Kubernetes 使用 **访问模式** 来确保 PV 只能按照指定的访问方式被 Pod 使用。
- **PVC** 申请存储时，必须与 PV 的访问模式匹配，才能完成绑定。

Kubernetes 支持以下几种卷访问模式：

| **访问模式**                  | **描述**                                                     | **支持的存储**                                |
| ----------------------------- | ------------------------------------------------------------ | --------------------------------------------- |
| **`ReadWriteOnce (RWO)`**     | 单节点单 Pod 读写，其他节点不可访问。                        | 本地存储、AWS EBS、Google Persistent Disk等。 |
| **`ReadOnlyMany (ROX)`**      | 多节点多 Pod 只读访问。                                      | NFS、CephFS、GlusterFS 等。                   |
| **`ReadWriteMany (RWX)`**     | 多节点多 Pod 读写访问。                                      | CephFS、GlusterFS、NFS 等。                   |
| **`ReadWriteOncePod (RWOP)`** | 单 Pod 独占读写权限。（此模式为 Kubernetes 1.22 引入的扩展访问模式，仅适用于支持的存储类型）。 | 特定 CSI 驱动、某些云存储插件。               |

📊 **注意事项**

1. **强制性字段**：`accessModes` 是必填字段，未设置会导致 PV 创建失败。

2. **PVC 匹配**：PVC 和 PV 的 `accessModes` 必须至少有一个重叠才能绑定。

3. 多值支持：**同一个 PV 可同时支持多个访问模式**，如：

   ```yaml
   accessModes:
     - ReadWriteOnce
     - ReadOnlyMany
   ```

📌 **示例：正确的 PV 访问模式**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:         # 配置PV访问模式
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/data"
```

------

### 📌 卷模式（volumeModes）

Kubernetes 支持两种卷模式（`volumeModes`）：`Filesystem（文件系统）` 和 `Block（块）`。默认的卷模式是 `Filesystem`。

**Filesystem模式：**默认模式，表示将存储设备格式化为文件系统（如 `ext4`、`xfs`），然后以挂载目录的形式提供给 Pod。

- 数据表现形式：文件和目录（类似于普通磁盘使用方式）。

- **操作方式**：Pod 通过标准的文件系统操作（如 `ls`、`cat`、`read/write`）来访问数据。

- ✅ **示例配置**：

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: fs-pvc
  spec:
    accessModes:
      - ReadWriteOnce
    volumeMode: Filesystem  # 文件系统模式（默认值）
    resources:
      requests:
        storage: 10Gi
  ```

  在 Pod 中使用：

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: fs-pod
  spec:
    containers:
      - name: app
        image: busybox
        volumeMounts:
          - mountPath: /data
            name: fs-storage
    volumes:
      - name: fs-storage
        persistentVolumeClaim:
          claimName: fs-pvc
  ```



**Block模式：**将存储设备以原始块设备形式（类似于添加一块硬盘）提供给 Pod，直接进行 I/O 操作，不经过文件系统。

- **数据表现形式**：原始数据块（裸设备，未格式化）。

- **操作方式**：Pod 可以直接对块设备进行读写操作，通常需要应用程序自行管理数据格式。

- ✅ **示例配置**：

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: block-pvc
  spec:
    accessModes:
      - ReadWriteOnce
    volumeMode: Block  # 块设备模式
    resources:
      requests:
        storage: 20Gi
  ```

  在 Pod 中使用：

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: block-pod
  spec:
    containers:
      - name: app
        image: busybox
        command: ["sh", "-c", "dd if=/dev/zero of=/dev/xvda bs=4M count=100"]
        volumeDevices:
          - devicePath: /dev/xvda
            name: block-storage
    volumes:
      - name: block-storage
        persistentVolumeClaim:
          claimName: block-pvc
  ```

  

#### **核心区别**

| 特性             | Filesystem 模式                   | Block 模式                         |
| ---------------- | --------------------------------- | ---------------------------------- |
| **数据存储形式** | 文件系统（ext4、xfs 等）          | 原始块设备（未格式化的裸设备）     |
| **数据访问方式** | 通过目录路径访问 (`volumeMounts`) | 通过设备路径访问 (`volumeDevices`) |
| **性能开销**     | 需文件系统开销，稍慢              | 直接读写，延迟更低，性能更优       |
| **使用限制**     | 适用于大多数场景                  | 需应用程序支持块设备访问           |
| **典型场景**     | 普通数据存储、日志、文件共享      | 数据库、分布式存储、缓存           |
| **卷格式化**     | 自动格式化（首次使用时）          | 不自动格式化，需手动处理           |
| **数据迁移**     | 兼容性更好，易于跨系统迁移        | 设备依赖强，跨系统迁移难           |
| **数据保护**     | 文件系统自带一致性和元数据管理    | 需应用自行实现一致性管理           |

#### **何时选择 Filesystem vs. Block**

✅ **选择 Filesystem 模式：**

- 你的应用程序需要基于文件系统的访问（如 Nginx、MySQL、PostgreSQL）。
- 需要跨平台兼容性，数据可轻松迁移。
- 适用于大多数常规数据存储、共享和持久化场景。

✅ **选择 Block 模式：**

- 追求极致 I/O 性能和最低延迟（如 Redis、Cassandra）。
- 你的应用程序支持直接操作块设备。
- 需要更精细的存储控制（如镜像、分区管理）。

#### **性能对比与注意事项**

| 特性           | Filesystem 模式          | Block 模式           |
| -------------- | ------------------------ | -------------------- |
| **I/O 性能**   | 受文件系统缓存影响       | 直接访问，性能最佳   |
| **元数据管理** | 自动处理（文件名、权限） | 需应用程序自行管理   |
| **快照与备份** | 可通过文件系统工具完成   | 依赖底层存储快照能力 |
| **兼容性**     | 通用（支持几乎所有应用） | 仅适配特定应用程序   |
| **数据一致性** | 文件系统提供基本保证     | 需应用自带一致性机制 |
| **复杂性**     | 简单，易于管理           | 复杂，需深度了解设备 |

#### **总结**：

- **文件系统模式 (Filesystem)** 是 Kubernetes 中的默认选择，适用于大部分应用程序，具有良好的兼容性和易用性。
- **块设备模式 (Block)** 适用于追求极致性能或需要直接访问底层存储的场景，适配特定的高性能应用。



### 📌 **回收策略 (Reclaim Policy)**

**永久卷（Persistent Volume, PV）** 的**回收策略（Reclaim Policy）**回收策略决定了当 PVC 被删除后，K8s 如何处理 PV。

| **策略**            | **描述**                                                     | **适用场景**                     |
| ------------------- | ------------------------------------------------------------ | -------------------------------- |
| `Retain（保留）`    | **保留数据**，需手动清理和回收 PV。删除 PVC 后，PV 状态会变为 `Released`（表示已解除绑定，但数据未清理） | 数据需长期保存，如数据库、日志。 |
| `Delete`            | 自动删除 PV 和存储后端资源（**适用于动态供应**）。           | 临时数据，动态创建和删除。       |
| `Recycle（再利用）` | **清理数据**（已弃用），PV 重新进入 Available。              | 早期测试使用，建议手动回收 PV。  |

**注意**：Recycle 策略在 Kubernetes **1.20 版本已废弃**，并在 **1.25 版本移除**，推荐使用 **Dynamic Provisioning** 进行动态存储管理。

**Dynamic Provisioning**：使用 StorageClass 动态创建和删除 PV，推荐用于大多数现代 Kubernetes 集群。

1️⃣ 创建一个 PV，使用不同的回收策略：

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

📌 **修改 PV 回收策略**

```bash
kubectl patch pv example-pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
```

------

### 📌 **适配的 Volume 类型**

| 类型                   | 说明                                          |
| ---------------------- | --------------------------------------------- |
| `hostPath`             | 使用节点本地路径，适合单节点开发环境。        |
| `nfs`                  | 使用 NFS 共享存储，适合多节点集群共享数据。   |
| `cephfs`               | 使用 Ceph 文件系统，支持多节点并发读写。      |
| `awsElasticBlockStore` | AWS EBS 卷，适用于 AWS 云环境。               |
| `gcePersistentDisk`    | Google Cloud Persistent Disk。                |
| `csi`                  | 使用 CSI (Container Storage Interface) 驱动。 |

------

### 📌 **PV 生命周期**

**persistentVolume**的生命周期是**独立于Pod的生命周期**，支持数据持久化，即使 Pod 被删除，数据仍然保留。

持久卷的生命周期分为以下几个阶段：

```
创建 → 绑定 → 使用 → 释放 → 回收/删除
```

### 📌 PV生命周期的状态

| **阶段**                          | **描述**                                            |
| --------------------------------- | --------------------------------------------------- |
| **Available（可用）**             | PV **可用状态**，等待与 PVC 绑定，尚未被使用。      |
| **Pending（待定）** **PVC的状态** | PVC 提交后，**等待合适的 PV** 进行绑定。            |
| **Bound（绑定）**                 | PV **已成功绑定** 到 PVC，Pod 可以使用该卷。        |
| **Released（释放）**              | PVC 被删除，但 PV **未被回收**，数据仍然保留。      |
| **Failed（失败）**                | PV 自动回收**失败**，需手动介入（如存储回收异常）。 |

**Released**状态的PV，如果需要重新使用 PV，需手动清理数据，并解除 PVC 绑定：

```yaml
kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'
```

注意：

- **无法直接删除已绑定的 PV**

  ```shell
  # 删除处于绑定状态的PV（会出现卡顿现象，无法完成PV的删除，此时PV状态为Terminating）
  root@ubuntu-master:/home/ubuntu/storage# kubectl delete -f pv.yaml
  
  # 查看PV和PVC状态
  root@ubuntu-master:/home/ubuntu# kubectl get pvc,pv
  NAME                        STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
  persistentvolumeclaim/pvc   Bound    pvc      10Gi       RWO                           <unset>                 3m25s
  
  NAME                   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM         STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
  persistentvolume/pvc   10Gi       RWO            Retain           Terminating   default/pvc                  <unset>                          7m3s
  
  # 删除PVC后PV被删除
  root@ubuntu-master:/home/ubuntu/storage# kubectl delete -f pvc.yaml
  persistentvolumeclaim "pvc" deleted
  root@ubuntu-master:/home/ubuntu/storage# kubectl get pvc,pv
  No resources found
  
  ```

  

  - 如果 PV 处于 `Bound` 状态，直接执行 `kubectl delete pv <pv-name>`，Kubernetes 不会立即删除 PV，而是等待 PVC 解除绑定。

  - 需要先删除或解除 PVC，才能删除PV

    

### 📌 **PV 生命周期状态转换图**

```sql
                  +----------------+
      PVC创建     |   Available    |
  ---------------->  (可用/未绑定)  |
                  +--------+-------+
                           |
                      PVC绑定 (匹配)
                           ↓
                  +--------+-------+
                  |     Bound      |<------+
                  | (已绑定/使用中) |       |
                  +--------+-------+       |
                           |               |
          PVC删除 (释放)    |               |
                           ↓               |
                  +--------+-------+       |
                  |    Released    |       |
                  | (已释放/待回收) |       |
                  +--------+-------+       |
                           |               |
     Reclaim策略决定 (回收) |               |
        +------------------+---------------+
        ↓        ↓         ↓
    Delete    Retain    (Recycle已废弃)
   (删除)    (保留)        
```

------

### 📌 PV生命周期和状态转换关系

1. **📊 Provisioning(创建)**：创建 PV，手动 (静态) 或自动 (动态)。

   - ✅ **静态创建 (Static Provisioning)**：**管理员手动** 创建 `PersistentVolume (PV)` 对象。PV 需与 PVC (PersistentVolumeClaim) 匹配后才能被 Pod 使用。

     **示例**

     ```yaml
     apiVersion: v1
     kind: PersistentVolume
     metadata:
       name: pv-manual
     spec:
       capacity:
         storage: 10Gi                        # 存储容量
       accessModes:
         - ReadWriteOnce                      # 访问模式（RWO、ROX、RWX）
       persistentVolumeReclaimPolicy: Retain  # 回收策略：Retain、Delete、Recycle（已弃用）
       hostPath:
         path: "/mnt/data"                    # 本地路径（测试用，生产环境常用 NFS、云盘）
     ```

     

   - ✅ **动态创建 (Dynamic Provisioning)**：依赖于 **StorageClass**，当 PVC 提交时，Kubernetes **自动** 创建 PV 并绑定。

     **示例**：

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
           storage: 5Gi               # 请求 5Gi 存储
       storageClassName: standard
     ```

   

   ✅ **注意：**

   - PV **已创建但未与任何 PVC 绑定**，此时状态为 `Available`。

   - 只有 **静态创建的 PV** 才会经历此阶段，**动态创建的 PV** 会直接进入 `Bound` 状态。

     

2. **📊 Binding(绑定)**：与 PersistentVolumeClaim (PVC) 绑定。

   **当 PVC 提交请求**时，Kubernetes 调度程序会查找一个符合条件的 PV。匹配成功后，PV 和 PVC 进入 `Bound` 状态，Pod 可以使用该存储卷。

   

   ✅ **PVC 和 PV 绑定的两种方式**

   - **（1）静态绑定**

     - 如果 **PVC 设置了 `storageClassName`**，它**只会匹配**具有相同 `storageClassName` 的静态 PV。

     - 如果 **PVC 没有设置 `storageClassName`**（即 `storageClassName: ""`），它**只会匹配**那些**没有设置 `storageClassName`** 的静态 PV。

     - **示例 ：PVC 匹配静态 PV**

       #### PVC：

       ```yaml
       apiVersion: v1
       kind: PersistentVolumeClaim
       metadata:
         name: static-pvc
       spec:
         accessModes:
           - ReadWriteOnce
         resources:
           requests:
             storage: 5Gi
         storageClassName: ""
       ```

       #### PV（可以匹配成功）：

       ```yaml
       apiVersion: v1
       kind: PersistentVolume
       metadata:
         name: static-pv
       spec:
         capacity:
           storage: 10Gi
         accessModes:
           - ReadWriteOnce
         storageClassName: ""  # 必须为空，才能与 PVC 匹配
         hostPath:
           path: "/mnt/data"
       ```

   

   - **（2）动态绑定**

     - 只有当 **PVC 设置了 `storageClassName`**，且**没有符合条件的静态 PV** 时，Kubernetes 才会**根据 `StorageClass` 动态创建**一个 PV 进行绑定。

     - 如果 PVC **未设置 `storageClassName`**，则**不会触发**动态 PV 创建。

     - **示例 ：PVC 触发动态创建 PV**

       #### PVC：

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
             storage: 5Gi
         storageClassName: "my-storage-class"
       ```

       若没有手动创建的 PV，Kubernetes 会使用 `my-storage-class` 动态创建一个新的 PV。

       

   - ✅ **总结更精确的匹配逻辑**

     1. 如果 **PVC 设置了 `storageClassName`**：
        - 先尝试**匹配已有静态 PV**（同名 `storageClassName`）。
        - 如果找不到符合条件的静态 PV，**按 `StorageClass` 动态创建 PV**。
     2. 如果 **PVC 没有设置 `storageClassName`**：
        - 仅能匹配**没有设置 `storageClassName` 的静态 PV**。
        - **不会触发动态 PV**。

   💡 **关键区别：**

   - 设置了 `storageClassName` 的 PVC 有可能匹配静态 PV 或动态创建 PV。
   - 未设置 `storageClassName` 的 PVC 只能匹配没有 `storageClassName` 的静态 PV，且不会触发动态 PV 创建。

   

   **注意**：PVC 和 PV 的以下字段需匹配，才能完成绑定：

   - `accessModes`：访问模式
   - `storageClassName`：存储类
   - `resources.requests.storage`：存储容量
     - **PVC 请求 ≤ PV 提供** → ✅ 可以绑定
     - **PVC 请求 > PV 提供** → ❌ 无法绑定

   

3. **📊 Using(使用)**：Pod 使用 PVC，PVC 绑定 PV。

4. **📊 Releasing(释放)**：删除 PVC，PV 进入 Released 状态。

   - 当 PVC **被删除**时，Kubernetes 不会立即删除 PV，而是将其状态更新为 `Released`。
   - **数据仍然保留**，但该 PV 无法自动被新的 PVC 绑定。

   **注意**：如果回收策略是 `Retain`，需手动清理数据和回收 PV，才能再次使用。

5. **📊 Reclaiming(回收/删除)**：根据 `ReclaimPolicy` 执行PV的回收策略 (保留/删除)。

   PV 的回收方式由 `persistentVolumeReclaimPolicy` 决定，主要有以下几种：

   | 回收策略                    | 说明                                     |
   | --------------------------- | ---------------------------------------- |
   | **Retain**（保留）          | PV 数据保留，需手动清理和回收。          |
   | **Delete**（删除）          | 自动删除 PV 及其后端存储（仅动态创建）。 |
   | **Recycle**（回收，已弃用） | 清理 PV 数据并将其置为 `Available`。     |

   ✅ **如何手动回收 PV？**

   1. **清理数据**（如果使用 `Retain` 策略）。

   2. 删除 PV：

      ```bash
      kubectl delete pv pv-manual
      ```

   3. **重新创建 PV**。

6. **📊 Failed（失败）**

   如果 Kubernetes **无法正确使用 PV**，它会进入 `Failed` 状态，常见原因包括：

   - **后端存储异常**（如 NFS 挂载失败）。
   - **PV 定义不完整或参数错误**。

    **查看错误信息**：

   ```bash
   kubectl describe pv <pv-name>
   ```

   ------

   

7. **📊 Available (再次使用)**

- 如果 `persistentVolumeReclaimPolicy` 是 `Retain`，需手动清理数据后重置 PV

  **重置 PV**（清理绑定关系）：

  ```bash
  kubectl patch pv pv-manual -p '{"spec":{"claimRef": null}}'
  ```

- PV 状态恢复为 `Available`，可以被新的 PVC 绑定。



## ✅ PersistentVolumeClaim



下面是一个标准的 **PersistentVolumeClaim (PVC)** YAML 模板，包含详细注释以说明各个字段的含义。

### ✅ **PersistentVolumeClaim (PVC) YAML 示例**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example             # PVC 的名称，需唯一
  namespace: default            # PVC 所在的命名空间，默认是 "default"
  labels:                       # 标签，用于标识 PVC
    app: my-app
spec:
  accessModes:
    - ReadWriteOnce             # 访问模式，需与目标 PV 匹配 (详见下方说明)
  resources:
    requests:
      storage: 5Gi              # 请求的存储大小，单位常用 Gi、Mi、Ti
  storageClassName: manual      # 关联的 StorageClass (与 PV 保持一致)
  volumeMode: Filesystem        # 存储卷模式 (Filesystem/Block)
  selector:                     # PV 选择器，用于绑定符合条件的 PV (可选)
    matchLabels:
      type: local               # 选择带有指定标签的 PV
    matchExpressions:           # 复杂匹配规则 (可选)
      - key: environment
        operator: In
        values:
          - test
```

------

✅ **主要字段说明**

| 字段                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `apiVersion`              | API 版本，`v1` 是 PVC 资源的稳定版本。                       |
| `kind`                    | 资源类型，`PersistentVolumeClaim`。                          |
| `metadata`                | PVC 的元数据，包含名称、命名空间、标签等信息。               |
| `spec.accessModes`        | 定义 PVC 需要的访问模式，需与 PV 一致 (详见下面的访问模式)。 |
| `spec.resources.requests` | PVC 请求的存储资源大小，单位常用 `Gi`、`Mi`。                |
| `spec.storageClassName`   | 绑定的 StorageClass，PVC 仅能绑定同一 `storageClassName` 的 PV。 |
| `spec.volumeMode`         | 存储模式，`Filesystem` (文件系统) 或 `Block` (原始块设备)。  |
| `spec.selector`           | 指定 PV 选择器，限制 PVC 仅能绑定符合条件的 PV (标签匹配方式)。 |

------

### ✅ **访问模式 (Access Modes)**

**详细内容见PV访问模式**

**PVC** 申请存储时，必须与 PV 的访问模式匹配，才能完成绑定。

------

### ✅ **PVC 生命周期**

1. **Pending**：PVC 已创建，但尚未找到匹配的 PV。
2. **Bound**：PVC 绑定到符合条件的 PV，Pod 可使用该存储卷。
3. **Lost**：PVC 绑定的 PV 不可用 (如 PV 被删除或无法访问)。

------

### ✅ **PVC 与 PV 的绑定流程**

见《PersistentVolume（PV）》（PV生命周期和状态转换关系）章节中的**📊 Binding(绑定)**部分

------

### ✅ **常用存储类型与 StorageClass**

| 存储类型               | 说明                                          |
| ---------------------- | --------------------------------------------- |
| `hostPath`             | 使用节点的本地路径 (适用于单节点环境)。       |
| `nfs`                  | 使用 NFS 共享存储 (适用于多节点共享数据)。    |
| `cephfs`               | 使用 Ceph 文件系统，支持多节点并发读写。      |
| `awsElasticBlockStore` | AWS EBS 卷，适用于 AWS 云环境。               |
| `gcePersistentDisk`    | Google Cloud Persistent Disk。                |
| `csi`                  | 使用 CSI (Container Storage Interface) 驱动。 |

------

### ✅ **示例 1：使用动态存储**

以下示例会根据指定的 `StorageClass` 自动创建并绑定 PV：

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
  storageClassName: gp2  # Kubernetes 将使用 "gp2" 类型自动创建 PV
```

------

### ✅ **示例 2：绑定已有 PV (静态绑定)**

如果你想将 PVC 绑定到现有 PV，可以使用 `selector` 进行匹配：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual   
  selector:
    matchLabels:
      type: local
```

------

### ✅ **PVC 与 Pod 关联示例**

创建 PVC 后，可以在 Pod 中通过 `volumes` 字段挂载：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: "/data"
          name: storage
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: pvc-example  # 引用 PVC
```

------



## 🎯 申请PV示例

在 Kubernetes 中，**PersistentVolume (PV)** 是对物理存储的抽象，用户通过 **PersistentVolumeClaim (PVC)** 来申请和使用 PV。下面是一个用户申请 PV 的完整示例，包括 **PV 定义**、**PVC 申请** 以及 **验证和使用**。

------

### 🎯 **1. 创建 PersistentVolume (PV)**

管理员创建 PV，用户通过 PVC 申请时，Kubernetes 会根据 PVC 的需求匹配合适的 PV。

📄 **PV 定义示例**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi                      # PV 提供的存储容量
  accessModes:
    - ReadWriteOnce                     # 访问模式
  persistentVolumeReclaimPolicy: Retain # 回收策略
  storageClassName: manual              # 存储类
  hostPath:                             # 使用本地路径作为存储（测试环境常用）
    path: "/mnt/data"
```

📌 **字段解析**

| 字段                            | 说明                                               |
| ------------------------------- | -------------------------------------------------- |
| `capacity.storage`              | PV 提供的存储容量（如 5Gi、100Gi）。               |
| `accessModes`                   | 访问模式（如 `ReadWriteOnce`、`ReadOnlyMany`）。   |
| `persistentVolumeReclaimPolicy` | 回收策略（`Retain`、`Recycle`、`Delete`）。        |
| `storageClassName`              | 存储类，PVC 通过它匹配 PV。                        |
| `hostPath`                      | 本地路径（适合测试环境，生产环境常用 NFS、Ceph）。 |

------

### 🎯 **2. 创建 PersistentVolumeClaim (PVC)**

用户通过 PVC 申请存储，Kubernetes 会根据 PVC 要求匹配一个可用的 PV。

📄 **PVC 定义示例**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce             # 访问模式，需与 PV 一致
  resources:
    requests:
      storage: 5Gi              # 请求的存储大小
  storageClassName: manual      # 需与 PV 的 storageClassName 一致
```

📌 **字段解析**

| 字段                 | 说明                                 |
| -------------------- | ------------------------------------ |
| `accessModes`        | PVC 申请的访问模式，需与 PV 匹配。   |
| `resources.requests` | 申请的存储容量，不能超过 PV 的容量。 |
| `storageClassName`   | PVC 使用的存储类，需与 PV 对应。     |

------

### 🎯 **3. 验证 PV 和 PVC 状态**

1. **查看 PV 状态**

```bash
kubectl get pv
```

**输出示例**：

```
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    STORAGECLASS   AGE
pv-example   10Gi       RWO            Retain          Bound     manual         5m
```

1. **查看 PVC 状态**

```bash
kubectl get pvc
```

**输出示例**：

```
NAME         STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-example  Bound    pv-example   10Gi       RWO            manual         2m
```

> **STATUS** 为 `Bound`，说明 PVC 成功绑定了 PV。

------

### 🎯 **4. 在 Pod 中使用 PVC**

一旦 PVC 绑定了 PV，用户可以在 Pod 中使用此持久化存储。

📄 **使用 PVC 的 Pod 示例**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app-container
      image: nginx
      volumeMounts:
        - mountPath: "/usr/share/nginx/html" # 挂载路径
          name: storage
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: pvc-example               # 引用 PVC
```

📌 **说明**

- `volumeMounts`：将 PVC 提供的存储挂载到容器的 `/usr/share/nginx/html` 目录。
- `persistentVolumeClaim`：引用用户申请的 PVC。

------

### 🎯 **5. 验证 PVC 在 Pod 中的使用**

1. **创建 Pod**

```bash
kubectl apply -f app-pod.yaml
```

1. **查看 Pod 状态**

```bash
kubectl get pod
```

确保 Pod 状态为 `Running`。

1. **检查挂载数据**

```bash
kubectl exec -it app-pod -- /bin/sh
```

在容器中检查挂载数据：

```bash
ls /usr/share/nginx/html
```

------

### 🎯 **6. 删除 PVC 和 PV**

1. 删除 Pod（确保不再使用 PVC）：

```bash
kubectl delete pod app-pod
```

1. 删除 PVC：

```bash
kubectl delete pvc pvc-example
```

1. 删除 PV：

```bash
kubectl delete pv pv-example
```

------

### 🎯 **7. PV 回收策略**

`persistentVolumeReclaimPolicy` 控制 PV 在 PVC 删除后的处理方式：

| 策略      | 说明                                               |
| --------- | -------------------------------------------------- |
| `Retain`  | 保留数据，需手动清理并重新绑定。                   |
| `Delete`  | 自动删除 PV 和其后端存储（动态存储适用）。         |
| `Recycle` | 清理数据并将 PV 状态重置为 `Available`（已废弃）。 |

------

### 🎯 **8. 小结**

1. **管理员** 创建 PV，**用户** 通过 PVC 申请使用存储。
2. PV 和 PVC 需匹配以下条件才能绑定：
   - `accessModes`（访问模式）。
   - `storageClassName`（存储类名）。
   - `capacity`（存储容量）。
3. **回收策略** 决定 PV 在 PVC 删除后的处理方式。



## 🔍 动态申请PV

在 Kubernetes 中，**动态申请 PersistentVolume (PV)** 是指用户只需创建 **PersistentVolumeClaim (PVC)**，Kubernetes 会根据 PVC 的需求自动创建并绑定一个 PV，无需手动预先创建 PV。

这种方式通常依赖于 **StorageClass** 来定义存储类型、存储供应器（Provisioner）以及回收策略。

------

### ✅ **1. 动态申请 PV 的工作流程**

1. **管理员** 创建 `StorageClass`，定义存储的具体实现方式（如 NFS、Ceph、AWS EBS、PVC 供应器等）。
2. **用户** 创建 `PersistentVolumeClaim`，指定 `storageClassName`。
3. **Kubernetes** 自动根据 PVC 的要求，通过 `StorageClass` 动态创建 PV，并与 PVC 绑定。

------

### ✅ **2. 动态申请 PV 的完整示例**

#### 📌 **Step 1：创建 StorageClass**

以下示例使用 **hostPath**（适用于测试环境），在生产环境通常使用 **NFS、Ceph、AWS EBS、GCE PD** 等。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: my-storageclass         # StorageClass 名称
provisioner: kubernetes.io/no-provisioner # 不使用外部动态供应器 (适用于 hostPath)
reclaimPolicy: Retain           # 回收策略：Retain、Delete
volumeBindingMode: Immediate    # 立即绑定（或 WaitForFirstConsumer）
```

> **常用 Provisioner**：
>
> - `kubernetes.io/no-provisioner`：本地存储（hostPath、Local PV）。
> - `kubernetes.io/aws-ebs`：AWS Elastic Block Store。
> - `kubernetes.io/gce-pd`：Google Persistent Disk。
> - `nfs-client`：NFS 存储。

------

#### 📌 **Step 2：创建 PersistentVolumeClaim**

用户提交 PVC，指定 `storageClassName`，Kubernetes 将自动创建并绑定 PV。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc                  # PVC 名称
spec:
  accessModes:
    - ReadWriteOnce             # 访问模式
  resources:
    requests:
      storage: 5Gi              # 申请 5Gi 存储
  storageClassName: my-storageclass  # 使用 StorageClass
```

------

#### 📌 **Step 3：使用 PVC 的 Pod**

创建一个 Pod，挂载通过 PVC 动态申请的存储。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app-container
      image: nginx
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: my-storage
  volumes:
    - name: my-storage
      persistentVolumeClaim:
        claimName: my-pvc      # 绑定 PVC
```

------

### ✅ **3. 验证动态创建 PV 是否成功**

#### 🔍 **1. 查看 StorageClass**

```bash
kubectl get storageclass
```

输出示例：

```
NAME              PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE      AGE
my-storageclass   kubernetes.io/no-provisioner Retain          Immediate              5m
```

------

#### 🔍 **2. 查看 PVC**

```bash
kubectl get pvc
```

输出示例：

```
NAME     STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS      AGE
my-pvc   Bound    pvc-xxxxx        5Gi       RWO            my-storageclass   2m
```

> ✅ **STATUS=Bound** 表示 PVC 已成功绑定一个 PV。

------

#### 🔍 **3. 查看 PV**

```bash
kubectl get pv
```

输出示例：

```
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM           STORAGECLASS      AGE
pvc-xxxxx        5Gi        RWO            Retain          Bound    default/my-pvc  my-storageclass   2m
```

------

### ✅ **4. 清理资源**

1. 删除 Pod：

```bash
kubectl delete pod app-pod
```

1. 删除 PVC：

```bash
kubectl delete pvc my-pvc
```

1. 删除 StorageClass：

```bash
kubectl delete storageclass my-storageclass
```

> **注意**：

- 如果 `reclaimPolicy` 为 `Retain`，即使 PVC 被删除，PV 仍会保留数据。
- 如果为 `Delete`，PVC 删除后，PV 及其数据会自动清理。

------

### ✅ **5. 动态 PV 的回收策略（Reclaim Policy）**

| 策略     | 说明                                    |
| -------- | --------------------------------------- |
| `Retain` | 保留数据，手动删除 PV，适用于重要数据。 |
| `Delete` | 自动删除 PV 和数据，适用于临时数据。    |

> **动态申请 PV 时，回收策略由 StorageClass 决定。**

------

### ✅ **6. StorageClass 的 VolumeBindingMode**

| 模式                   | 说明                                            |
| ---------------------- | ----------------------------------------------- |
| `Immediate` (默认)     | PVC 创建时立刻分配 PV，适用于大多数存储场景。   |
| `WaitForFirstConsumer` | 等待 Pod 使用 PVC 时才分配 PV，适用于延迟分配。 |

> **WaitForFirstConsumer** 适用于本地存储或多可用区环境，确保 PV 和 Pod 在同一区域。

------

### ✅ **7. StorageClass + Provisioner 示例**

### 📌 **① AWS EBS 动态申请**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### 📌 **② NFS 动态申请**

需要部署 NFS 动态供应器，如 [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: nfs-client
parameters:
  archiveOnDelete: "true"
reclaimPolicy: Retain
```

------

### 🎯 **8. 总结**

- **动态申请 PV** 通过 `StorageClass` 自动创建、绑定、回收存储资源。
- 用户只需创建 **PVC**，无需手动管理 PV。
- **StorageClass** 控制存储类型、回收策略、绑定模式。
- **不同环境** 需使用不同的 `Provisioner`（如 AWS、GCE、NFS）。



## Pod使用PVC

Pod 使用 PersistentVolumeClaim 作为存储示例：https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-persistent-volume-storage/
