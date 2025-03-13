在 **Kubernetes (K8s)** 中，**Persistent Volume (PV)** 和 **Persistent Volume Claim (PVC)** 都有各自的**状态（Status）**，用于表示它们的当前使用情况和生命周期阶段。

------

## 📊 **1. Persistent Volume (PV) 的状态**

`PersistentVolume` 是 K8s 中的持久化存储资源，以下是它的几种常见状态：

| **状态**      | **说明**                                        |
| ------------- | ----------------------------------------------- |
| **Available** | PV **未被绑定**，可供 PVC 使用。                |
| **Bound**     | PV **已绑定**到某个 PVC，正在被使用。           |
| **Released**  | 绑定的 PVC 已删除，**数据未清理**，需手动回收。 |
| **Failed**    | PV **自动回收失败**，可能需要手动处理。         |

### 📌 **查看 PV 状态**

```bash
kubectl get pv
```

示例输出：

```bash
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS   AGE
pv-test      10Gi       RWO            Retain           Bound       default/pvc-test    manual         5h
```

------

### 📌 **处理不同 PV 状态的操作建议**

- **Available**：PV 可用，等待 PVC 绑定。

- **Bound**：PVC 正在使用 PV，正常状态。

- Released：

  1. 如果需要重新使用 PV，需

     手动清理数据，并解除 PVC 绑定：

     ```bash
     kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'
     ```

  2. 如果不再需要该 PV，可直接删除：

     ```bash
     kubectl delete pv <pv-name>
     ```

- Failed：

  1. 查看 PV 详细信息，定位错误原因：

     ```bash
     kubectl describe pv <pv-name>
     ```

  2. 根据错误信息，修复问题或手动删除 PV。

------

## 📋 **2. Persistent Volume Claim (PVC) 的状态**

`PersistentVolumeClaim` 是对持久化卷的请求，以下是 PVC 的几种常见状态：

| **状态**    | **说明**                                  |
| ----------- | ----------------------------------------- |
| **Pending** | PVC **未绑定**，正在等待合适的 PV。       |
| **Bound**   | PVC **已成功绑定**到 PV，正在使用该卷。   |
| **Lost**    | PV 被删除或不可用，PVC 进入**丢失**状态。 |

### 📌 **查看 PVC 状态**

```bash
kubectl get pvc
```

示例输出：

```bash
NAME         STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-test     Bound     pv-test   10Gi       RWO            manual         5h
```

------

### 📌 **处理不同 PVC 状态的操作建议**

- **Pending**：

  1. 检查是否存在满足 PVC 要求的 PV：

     ```bash
     kubectl get pv
     ```

  2. 如果没有合适 PV，可手动创建或启用**动态供应**。

- **Bound**：正常状态，PVC 已绑定到 PV。

- **Lost**：

  1. 检查 PV 是否存在：

     ```bash
     kubectl get pv
     ```

  2. 如果 PV 丢失或已删除，需重新创建 PV 并重新绑定 PVC。

------

## 🔍 **3. 排查 PV/PVC 问题**

1. **检查 PVC 和 PV 绑定情况**：

   ```bash
   kubectl get pvc -o wide
   ```

2. **查看 PV 详细信息**：

   ```bash
   kubectl describe pv <pv-name>
   ```

3. **查看 PVC 详细信息**：

   ```bash
   kubectl describe pvc <pvc-name>
   ```

4. **检查事件和错误信息**：

   ```bash
   kubectl get events --sort-by='{.lastTimestamp}'
   ```

------

## ✅ **总结**

- **PV 状态**：Available、Bound、Released、Failed
- **PVC 状态**：Pending、Bound、Lost

选择正确的存储策略、回收模式（Retain、Delete、Recycle），可以有效管理持久化卷的生命周期。